# Part 55: Advanced React Patterns กับ TypeScript

## บทนำ

Advanced React Patterns คือเทคนิคการออกแบบ components ที่ยืดหยุ่น นำกลับมาใช้ใหม่ได้ และบำรุงรักษาง่าย บทนี้จะครอบคลุม patterns ต่างๆ พร้อม TypeScript types ที่ถูกต้อง

---

## 1. Compound Component Pattern

Compound Components ช่วยสร้าง component ที่มีหลายส่วนทำงานร่วมกัน

```tsx
// ตัวอย่าง: Tabs Component แบบ Compound

import React, { createContext, useContext, useState, ReactNode } from 'react';

// Context สำหรับ share state ระหว่าง components
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(): TabsContextValue {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('useTabsContext must be used within Tabs component');
  }
  return context;
}

// Main Tabs Component
interface TabsProps {
  defaultTab: string;
  children: ReactNode;
  onChange?: (tab: string) => void;
}

function Tabs({ defaultTab, children, onChange }: TabsProps): JSX.Element {
  const [activeTab, setActiveTab] = useState(defaultTab);
  
  const handleTabChange = (tab: string) => {
    setActiveTab(tab);
    onChange?.(tab);
  };
  
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab: handleTabChange }}>
      <div className="tabs-container">{children}</div>
    </TabsContext.Provider>
  );
}

// Tab List Component
interface TabListProps {
  children: ReactNode;
  className?: string;
}

function TabList({ children, className = '' }: TabListProps): JSX.Element {
  return (
    <div role="tablist" className={`tab-list ${className}`}>
      {children}
    </div>
  );
}

// Tab Button Component
interface TabProps {
  value: string;
  children: ReactNode;
  disabled?: boolean;
}

function Tab({ value, children, disabled = false }: TabProps): JSX.Element {
  const { activeTab, setActiveTab } = useTabsContext();
  const isActive = activeTab === value;
  
  return (
    <button
      role="tab"
      aria-selected={isActive}
      aria-disabled={disabled}
      disabled={disabled}
      onClick={() => !disabled && setActiveTab(value)}
      className={`tab ${isActive ? 'tab--active' : ''} ${disabled ? 'tab--disabled' : ''}`}
    >
      {children}
    </button>
  );
}

// Tab Panel Component
interface TabPanelProps {
  value: string;
  children: ReactNode;
  className?: string;
}

function TabPanel({ value, children, className = '' }: TabPanelProps): JSX.Element | null {
  const { activeTab } = useTabsContext();
  
  if (activeTab !== value) return null;
  
  return (
    <div
      role="tabpanel"
      className={`tab-panel ${className}`}
    >
      {children}
    </div>
  );
}

// ผูก sub-components เข้ากับ main component
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// การใช้งาน
function App(): JSX.Element {
  return (
    <Tabs defaultTab="overview" onChange={(tab) => console.log('Tab changed:', tab)}>
      <Tabs.List>
        <Tabs.Tab value="overview">Overview</Tabs.Tab>
        <Tabs.Tab value="details">Details</Tabs.Tab>
        <Tabs.Tab value="settings" disabled>Settings</Tabs.Tab>
      </Tabs.List>
      
      <Tabs.Panel value="overview">
        <h2>Overview Content</h2>
        <p>This is the overview section.</p>
      </Tabs.Panel>
      
      <Tabs.Panel value="details">
        <h2>Details Content</h2>
        <p>This is the details section.</p>
      </Tabs.Panel>
    </Tabs>
  );
}
```

```tsx
// Compound Component: Accordion

interface AccordionContextValue {
  openItems: Set<string>;
  toggleItem: (id: string) => void;
  allowMultiple: boolean;
}

const AccordionContext = createContext<AccordionContextValue | null>(null);

interface AccordionProps {
  children: ReactNode;
  allowMultiple?: boolean;
  defaultOpen?: string[];
}

function Accordion({ children, allowMultiple = false, defaultOpen = [] }: AccordionProps) {
  const [openItems, setOpenItems] = useState<Set<string>>(new Set(defaultOpen));
  
  const toggleItem = (id: string) => {
    setOpenItems(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) {
          next.clear();
        }
        next.add(id);
      }
      return next;
    });
  };
  
  return (
    <AccordionContext.Provider value={{ openItems, toggleItem, allowMultiple }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

interface AccordionItemProps {
  id: string;
  children: ReactNode;
}

function AccordionItem({ id, children }: AccordionItemProps) {
  const context = useContext(AccordionContext);
  if (!context) throw new Error('AccordionItem must be used within Accordion');
  
  const isOpen = context.openItems.has(id);
  
  return (
    <div className={`accordion-item ${isOpen ? 'accordion-item--open' : ''}`}>
      {React.Children.map(children, child => {
        if (React.isValidElement(child)) {
          return React.cloneElement(child as React.ReactElement<any>, { itemId: id, isOpen });
        }
        return child;
      })}
    </div>
  );
}

interface AccordionHeaderProps {
  children: ReactNode;
  itemId?: string;
  isOpen?: boolean;
}

function AccordionHeader({ children, itemId, isOpen }: AccordionHeaderProps) {
  const context = useContext(AccordionContext);
  if (!context) throw new Error('AccordionHeader must be used within Accordion');
  
  return (
    <button
      className="accordion-header"
      onClick={() => itemId && context.toggleItem(itemId)}
      aria-expanded={isOpen}
    >
      {children}
      <span>{isOpen ? '▲' : '▼'}</span>
    </button>
  );
}

Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
```

---

## 2. Render Props Pattern

```tsx
// Render Props Pattern: Mouse Tracker

interface MousePosition {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  render: (position: MousePosition) => ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps): JSX.Element {
  const [position, setPosition] = useState<MousePosition>({ x: 0, y: 0 });
  
  const handleMouseMove = (event: React.MouseEvent) => {
    setPosition({ x: event.clientX, y: event.clientY });
  };
  
  return (
    <div onMouseMove={handleMouseMove} style={{ height: '100vh' }}>
      {render(position)}
    </div>
  );
}

// การใช้งาน
function App(): JSX.Element {
  return (
    <MouseTracker
      render={({ x, y }) => (
        <div>
          <p>Mouse is at: ({x}, {y})</p>
          <div
            style={{
              position: 'absolute',
              left: x,
              top: y,
              width: 10,
              height: 10,
              background: 'red',
              borderRadius: '50%',
              transform: 'translate(-50%, -50%)'
            }}
          />
        </div>
      )}
    />
  );
}

// Render Props: Data Fetcher
interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

interface DataFetcherProps<T> {
  url: string;
  children: (state: FetchState<T>) => ReactNode;
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>): JSX.Element {
  const [state, setState] = useState<Omit<FetchState<T>, 'refetch'>>({
    data: null,
    loading: true,
    error: null
  });
  
  const fetchData = React.useCallback(async () => {
    setState(prev => ({ ...prev, loading: true, error: null }));
    
    try {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      const data: T = await response.json();
      setState({ data, loading: false, error: null });
    } catch (error) {
      setState({
        data: null,
        loading: false,
        error: error instanceof Error ? error : new Error('Unknown error')
      });
    }
  }, [url]);
  
  React.useEffect(() => {
    fetchData();
  }, [fetchData]);
  
  return <>{children({ ...state, refetch: fetchData })}</>;
}

// การใช้งาน DataFetcher
interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: number }): JSX.Element {
  return (
    <DataFetcher<User> url={`/api/users/${userId}`}>
      {({ data, loading, error, refetch }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.message} <button onClick={refetch}>Retry</button></div>;
        if (!data) return null;
        
        return (
          <div>
            <h2>{data.name}</h2>
            <p>{data.email}</p>
            <button onClick={refetch}>Refresh</button>
          </div>
        );
      }}
    </DataFetcher>
  );
}
```

---

## 3. HOC with TypeScript

```tsx
// Higher Order Components (HOC) กับ TypeScript

// HOC: withLoading
interface WithLoadingProps {
  loading: boolean;
}

function withLoading<TProps extends object>(
  WrappedComponent: React.ComponentType<TProps>
): React.FC<TProps & WithLoadingProps> {
  const displayName = WrappedComponent.displayName || WrappedComponent.name || 'Component';
  
  const ComponentWithLoading: React.FC<TProps & WithLoadingProps> = ({
    loading,
    ...props
  }) => {
    if (loading) {
      return (
        <div className="loading-container">
          <div className="spinner" />
          <p>Loading...</p>
        </div>
      );
    }
    
    return <WrappedComponent {...(props as TProps)} />;
  };
  
  ComponentWithLoading.displayName = `withLoading(${displayName})`;
  
  return ComponentWithLoading;
}

// HOC: withAuth
interface AuthUser {
  id: string;
  name: string;
  roles: string[];
}

interface WithAuthProps {
  currentUser: AuthUser;
}

function withAuth<TProps extends object>(
  WrappedComponent: React.ComponentType<TProps & WithAuthProps>
): React.FC<Omit<TProps, keyof WithAuthProps>> {
  const ComponentWithAuth: React.FC<Omit<TProps, keyof WithAuthProps>> = (props) => {
    const user = useCurrentUser(); // custom hook
    
    if (!user) {
      return <Navigate to="/login" />;
    }
    
    return <WrappedComponent {...(props as TProps)} currentUser={user} />;
  };
  
  ComponentWithAuth.displayName = `withAuth(${WrappedComponent.displayName || WrappedComponent.name})`;
  
  return ComponentWithAuth;
}

// HOC: withErrorBoundary
interface ErrorState {
  hasError: boolean;
  error: Error | null;
}

interface WithErrorBoundaryOptions {
  fallback?: ReactNode;
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

function withErrorBoundary<TProps extends object>(
  WrappedComponent: React.ComponentType<TProps>,
  options: WithErrorBoundaryOptions = {}
): React.ComponentClass<TProps, ErrorState> {
  const { fallback = <div>Something went wrong</div>, onError } = options;
  
  class WithErrorBoundaryComponent extends React.Component<TProps, ErrorState> {
    state: ErrorState = { hasError: false, error: null };
    
    static getDerivedStateFromError(error: Error): ErrorState {
      return { hasError: true, error };
    }
    
    componentDidCatch(error: Error, info: React.ErrorInfo): void {
      console.error('Error caught by boundary:', error, info);
      onError?.(error, info);
    }
    
    render(): ReactNode {
      if (this.state.hasError) {
        return fallback;
      }
      return <WrappedComponent {...this.props} />;
    }
  }
  
  return WithErrorBoundaryComponent;
}

// HOC: withPermission
type Permission = 'read' | 'write' | 'delete' | 'admin';

function withPermission(requiredPermission: Permission) {
  return function<TProps extends object>(
    WrappedComponent: React.ComponentType<TProps>
  ): React.FC<TProps> {
    const ComponentWithPermission: React.FC<TProps> = (props) => {
      const { hasPermission } = usePermissions();
      
      if (!hasPermission(requiredPermission)) {
        return (
          <div className="no-permission">
            <p>You don't have permission to view this content.</p>
          </div>
        );
      }
      
      return <WrappedComponent {...props} />;
    };
    
    return ComponentWithPermission;
  };
}

// การใช้งาน HOCs รวมกัน
interface AdminDashboardProps {
  currentUser: AuthUser;
}

const AdminDashboard = withAuth(
  withPermission('admin')(
    withErrorBoundary(
      ({ currentUser }: AdminDashboardProps) => (
        <div>
          <h1>Admin Dashboard</h1>
          <p>Welcome, {currentUser.name}</p>
        </div>
      ),
      { fallback: <div>Failed to load dashboard</div> }
    )
  )
);
```

---

## 4. Hooks Composition

```tsx
// Custom Hooks ที่ทำงานร่วมกัน

// useDebounce Hook
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  React.useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(handler);
  }, [value, delay]);
  
  return debouncedValue;
}

// usePrevious Hook
function usePrevious<T>(value: T): T | undefined {
  const ref = React.useRef<T | undefined>(undefined);
  
  React.useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

// useLocalStorage Hook
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });
  
  const setValue = (value: T) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error('Error saving to localStorage:', error);
    }
  };
  
  return [storedValue, setValue];
}

// useAsync Hook
interface AsyncState<T> {
  status: 'idle' | 'pending' | 'success' | 'error';
  data: T | null;
  error: Error | null;
}

function useAsync<T>(
  asyncFunction: () => Promise<T>,
  immediate = true
) {
  const [state, setState] = useState<AsyncState<T>>({
    status: 'idle',
    data: null,
    error: null
  });
  
  const execute = React.useCallback(async () => {
    setState({ status: 'pending', data: null, error: null });
    
    try {
      const data = await asyncFunction();
      setState({ status: 'success', data, error: null });
    } catch (error) {
      setState({
        status: 'error',
        data: null,
        error: error instanceof Error ? error : new Error(String(error))
      });
    }
  }, [asyncFunction]);
  
  React.useEffect(() => {
    if (immediate) {
      execute();
    }
  }, [execute, immediate]);
  
  return { ...state, execute };
}

// useSearch Hook (combining multiple hooks)
interface SearchOptions<T> {
  data: T[];
  searchKey: keyof T;
  debounceMs?: number;
}

function useSearch<T extends Record<string, any>>({
  data,
  searchKey,
  debounceMs = 300
}: SearchOptions<T>) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, debounceMs);
  
  const results = React.useMemo(() => {
    if (!debouncedQuery.trim()) return data;
    
    return data.filter(item => {
      const value = item[searchKey];
      if (typeof value === 'string') {
        return value.toLowerCase().includes(debouncedQuery.toLowerCase());
      }
      return false;
    });
  }, [data, searchKey, debouncedQuery]);
  
  return {
    query,
    setQuery,
    results,
    isSearching: query !== debouncedQuery
  };
}

// การใช้งาน
interface Product {
  id: number;
  name: string;
  price: number;
}

function ProductSearch({ products }: { products: Product[] }) {
  const { query, setQuery, results, isSearching } = useSearch({
    data: products,
    searchKey: 'name',
    debounceMs: 300
  });
  
  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search products..."
      />
      {isSearching && <span>Searching...</span>}
      <ul>
        {results.map(product => (
          <li key={product.id}>
            {product.name} - ฿{product.price}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 5. Context + Reducer Pattern

```tsx
// Context + Reducer สำหรับ State Management

// Shopping Cart State Management
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
  image?: string;
}

interface CartState {
  items: CartItem[];
  isOpen: boolean;
  couponCode: string | null;
  discount: number;
}

// Actions
type CartAction =
  | { type: 'ADD_ITEM'; payload: Omit<CartItem, 'quantity'> }
  | { type: 'REMOVE_ITEM'; payload: { id: string } }
  | { type: 'UPDATE_QUANTITY'; payload: { id: string; quantity: number } }
  | { type: 'CLEAR_CART' }
  | { type: 'TOGGLE_CART' }
  | { type: 'OPEN_CART' }
  | { type: 'CLOSE_CART' }
  | { type: 'APPLY_COUPON'; payload: { code: string; discount: number } }
  | { type: 'REMOVE_COUPON' };

// Reducer
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existingItem = state.items.find(item => item.id === action.payload.id);
      
      if (existingItem) {
        return {
          ...state,
          items: state.items.map(item =>
            item.id === action.payload.id
              ? { ...item, quantity: item.quantity + 1 }
              : item
          )
        };
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
      return { ...state, items: [], couponCode: null, discount: 0 };
    
    case 'TOGGLE_CART':
      return { ...state, isOpen: !state.isOpen };
    
    case 'OPEN_CART':
      return { ...state, isOpen: true };
    
    case 'CLOSE_CART':
      return { ...state, isOpen: false };
    
    case 'APPLY_COUPON':
      return {
        ...state,
        couponCode: action.payload.code,
        discount: action.payload.discount
      };
    
    case 'REMOVE_COUPON':
      return { ...state, couponCode: null, discount: 0 };
    
    default:
      return state;
  }
}

// Context
interface CartContextValue {
  state: CartState;
  dispatch: React.Dispatch<CartAction>;
  // Computed values
  totalItems: number;
  subtotal: number;
  total: number;
  // Actions (convenience methods)
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  toggleCart: () => void;
}

const CartContext = createContext<CartContextValue | null>(null);

const initialCartState: CartState = {
  items: [],
  isOpen: false,
  couponCode: null,
  discount: 0
};

interface CartProviderProps {
  children: ReactNode;
  initialState?: Partial<CartState>;
}

function CartProvider({ children, initialState }: CartProviderProps): JSX.Element {
  const [state, dispatch] = React.useReducer(
    cartReducer,
    { ...initialCartState, ...initialState }
  );
  
  const totalItems = React.useMemo(
    () => state.items.reduce((sum, item) => sum + item.quantity, 0),
    [state.items]
  );
  
  const subtotal = React.useMemo(
    () => state.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [state.items]
  );
  
  const total = React.useMemo(
    () => subtotal - (subtotal * state.discount / 100),
    [subtotal, state.discount]
  );
  
  const addItem = React.useCallback(
    (item: Omit<CartItem, 'quantity'>) => dispatch({ type: 'ADD_ITEM', payload: item }),
    []
  );
  
  const removeItem = React.useCallback(
    (id: string) => dispatch({ type: 'REMOVE_ITEM', payload: { id } }),
    []
  );
  
  const updateQuantity = React.useCallback(
    (id: string, quantity: number) => dispatch({ type: 'UPDATE_QUANTITY', payload: { id, quantity } }),
    []
  );
  
  const clearCart = React.useCallback(
    () => dispatch({ type: 'CLEAR_CART' }),
    []
  );
  
  const toggleCart = React.useCallback(
    () => dispatch({ type: 'TOGGLE_CART' }),
    []
  );
  
  const value: CartContextValue = {
    state,
    dispatch,
    totalItems,
    subtotal,
    total,
    addItem,
    removeItem,
    updateQuantity,
    clearCart,
    toggleCart
  };
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

function useCart(): CartContextValue {
  const context = useContext(CartContext);
  if (!context) {
    throw new Error('useCart must be used within CartProvider');
  }
  return context;
}

// การใช้งาน
function ProductCard({ product }: { product: any }): JSX.Element {
  const { addItem } = useCart();
  
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>฿{product.price}</p>
      <button onClick={() => addItem(product)}>
        Add to Cart
      </button>
    </div>
  );
}

function CartIcon(): JSX.Element {
  const { totalItems, toggleCart } = useCart();
  
  return (
    <button onClick={toggleCart} className="cart-icon">
      🛒 {totalItems > 0 && <span className="badge">{totalItems}</span>}
    </button>
  );
}
```

---

## 6. Observer Pattern in React

```tsx
// Observer Pattern สำหรับ event-driven components

// Event Emitter
type Listener<T> = (data: T) => void;

class TypedEventEmitter<TEvents extends Record<string, any>> {
  private listeners: {
    [K in keyof TEvents]?: Set<Listener<TEvents[K]>>
  } = {};
  
  on<K extends keyof TEvents>(event: K, listener: Listener<TEvents[K]>): () => void {
    if (!this.listeners[event]) {
      this.listeners[event] = new Set();
    }
    this.listeners[event]!.add(listener);
    
    // Return unsubscribe function
    return () => this.off(event, listener);
  }
  
  off<K extends keyof TEvents>(event: K, listener: Listener<TEvents[K]>): void {
    this.listeners[event]?.delete(listener);
  }
  
  emit<K extends keyof TEvents>(event: K, data: TEvents[K]): void {
    this.listeners[event]?.forEach(listener => listener(data));
  }
}

// Application Events
interface AppEvents {
  'user:login': { userId: string; name: string };
  'user:logout': void;
  'notification:new': { message: string; type: 'info' | 'success' | 'error' };
  'cart:updated': { totalItems: number };
}

const appEvents = new TypedEventEmitter<AppEvents>();

// Hook สำหรับ subscribe to events
function useEventListener<K extends keyof AppEvents>(
  event: K,
  handler: Listener<AppEvents[K]>
): void {
  const handlerRef = React.useRef(handler);
  handlerRef.current = handler;
  
  React.useEffect(() => {
    const unsubscribe = appEvents.on(event, (data) => {
      handlerRef.current(data);
    });
    
    return unsubscribe;
  }, [event]);
}

// Component ที่ใช้ Observer Pattern
function NotificationCenter(): JSX.Element {
  const [notifications, setNotifications] = useState<Array<{
    id: string;
    message: string;
    type: string;
  }>>([]);
  
  useEventListener('notification:new', ({ message, type }) => {
    const id = Date.now().toString();
    setNotifications(prev => [...prev, { id, message, type }]);
    
    // Auto remove after 5 seconds
    setTimeout(() => {
      setNotifications(prev => prev.filter(n => n.id !== id));
    }, 5000);
  });
  
  return (
    <div className="notification-center">
      {notifications.map(notification => (
        <div
          key={notification.id}
          className={`notification notification--${notification.type}`}
        >
          {notification.message}
        </div>
      ))}
    </div>
  );
}

function UserStatus(): JSX.Element {
  const [user, setUser] = useState<{ userId: string; name: string } | null>(null);
  
  useEventListener('user:login', (userData) => {
    setUser(userData);
  });
  
  useEventListener('user:logout', () => {
    setUser(null);
  });
  
  if (!user) return <button onClick={() => {}}>Login</button>;
  
  return (
    <div>
      <span>Welcome, {user.name}</span>
      <button onClick={() => appEvents.emit('user:logout', undefined as any)}>
        Logout
      </button>
    </div>
  );
}
```

---

## 7. Virtualization (react-window)

```tsx
// Virtualization สำหรับ list ขนาดใหญ่
import { FixedSizeList, VariableSizeList, FixedSizeGrid } from 'react-window';
import AutoSizer from 'react-virtualized-auto-sizer';

// ข้อมูลตัวอย่าง
interface DataItem {
  id: number;
  name: string;
  email: string;
  role: string;
  status: 'active' | 'inactive';
}

// Fixed Size List
function VirtualizedUserList({ users }: { users: DataItem[] }): JSX.Element {
  const ITEM_HEIGHT = 60;
  
  const UserRow = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const user = users[index];
    
    return (
      <div style={style} className="user-row">
        <div className="user-info">
          <strong>{user.name}</strong>
          <span>{user.email}</span>
        </div>
        <span className={`badge badge--${user.status}`}>
          {user.status}
        </span>
      </div>
    );
  };
  
  return (
    <AutoSizer>
      {({ height, width }) => (
        <FixedSizeList
          height={height}
          width={width}
          itemCount={users.length}
          itemSize={ITEM_HEIGHT}
          overscanCount={5}
        >
          {UserRow}
        </FixedSizeList>
      )}
    </AutoSizer>
  );
}

// Variable Size List
interface Message {
  id: number;
  text: string;
  sender: string;
  timestamp: Date;
}

function VirtualizedMessageList({ messages }: { messages: Message[] }): JSX.Element {
  const listRef = React.useRef<VariableSizeList>(null);
  const rowHeights = React.useRef<{ [key: number]: number }>({});
  
  const getItemSize = (index: number) => {
    return rowHeights.current[index] || 80;
  };
  
  const setRowHeight = (index: number, height: number) => {
    if (rowHeights.current[index] !== height) {
      rowHeights.current[index] = height;
      listRef.current?.resetAfterIndex(index);
    }
  };
  
  const MessageRow = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const rowRef = React.useRef<HTMLDivElement>(null);
    const message = messages[index];
    
    React.useEffect(() => {
      if (rowRef.current) {
        setRowHeight(index, rowRef.current.getBoundingClientRect().height);
      }
    }, [index, message]);
    
    return (
      <div style={style}>
        <div ref={rowRef} className="message">
          <strong>{message.sender}</strong>
          <p>{message.text}</p>
          <small>{message.timestamp.toLocaleTimeString()}</small>
        </div>
      </div>
    );
  };
  
  return (
    <AutoSizer>
      {({ height, width }) => (
        <VariableSizeList
          ref={listRef}
          height={height}
          width={width}
          itemCount={messages.length}
          itemSize={getItemSize}
          overscanCount={10}
        >
          {MessageRow}
        </VariableSizeList>
      )}
    </AutoSizer>
  );
}

// Virtualized Grid
function VirtualizedGrid({ items }: { items: any[] }): JSX.Element {
  const COLUMN_COUNT = 4;
  const ROW_HEIGHT = 200;
  const COLUMN_WIDTH = 250;
  
  const Cell = ({
    columnIndex,
    rowIndex,
    style
  }: {
    columnIndex: number;
    rowIndex: number;
    style: React.CSSProperties;
  }) => {
    const itemIndex = rowIndex * COLUMN_COUNT + columnIndex;
    
    if (itemIndex >= items.length) return null;
    
    const item = items[itemIndex];
    
    return (
      <div style={{ ...style, padding: 8 }}>
        <div className="grid-item">
          <img src={item.image} alt={item.name} />
          <h3>{item.name}</h3>
          <p>฿{item.price}</p>
        </div>
      </div>
    );
  };
  
  const rowCount = Math.ceil(items.length / COLUMN_COUNT);
  
  return (
    <AutoSizer>
      {({ height, width }) => (
        <FixedSizeGrid
          height={height}
          width={width}
          columnCount={COLUMN_COUNT}
          columnWidth={COLUMN_WIDTH}
          rowCount={rowCount}
          rowHeight={ROW_HEIGHT}
          overscanRowCount={3}
        >
          {Cell}
        </FixedSizeGrid>
      )}
    </AutoSizer>
  );
}
```

---

## 8. Suspense และ Lazy Loading

```tsx
// React Suspense และ Lazy Loading

// Lazy loaded components
const HeavyChart = React.lazy(() => import('./HeavyChart'));
const AdminPanel = React.lazy(() => import('./AdminPanel'));
const UserSettings = React.lazy(() => import('./UserSettings'));

// Loading components
function LoadingSpinner(): JSX.Element {
  return (
    <div className="loading-container">
      <div className="spinner" role="status">
        <span className="sr-only">Loading...</span>
      </div>
    </div>
  );
}

function LoadingSkeleton(): JSX.Element {
  return (
    <div className="skeleton-container">
      {Array.from({ length: 3 }).map((_, i) => (
        <div key={i} className="skeleton-item">
          <div className="skeleton skeleton--image" />
          <div className="skeleton skeleton--title" />
          <div className="skeleton skeleton--text" />
        </div>
      ))}
    </div>
  );
}

// Route-based code splitting
function App(): JSX.Element {
  return (
    <React.Suspense fallback={<LoadingSpinner />}>
      <Router>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route
            path="/admin"
            element={
              <React.Suspense fallback={<LoadingSkeleton />}>
                <AdminPanel />
              </React.Suspense>
            }
          />
          <Route
            path="/settings"
            element={
              <React.Suspense fallback={<LoadingSpinner />}>
                <UserSettings />
              </React.Suspense>
            }
          />
        </Routes>
      </Router>
    </React.Suspense>
  );
}

// Suspense สำหรับ Data Fetching
// resource.ts
function createResource<T>(promise: Promise<T>) {
  let status: 'pending' | 'success' | 'error' = 'pending';
  let result: T;
  let error: Error;
  
  const suspender = promise.then(
    (data) => { status = 'success'; result = data; },
    (err) => { status = 'error'; error = err; }
  );
  
  return {
    read(): T {
      if (status === 'pending') throw suspender;
      if (status === 'error') throw error;
      return result;
    }
  };
}

// ใช้กับ Suspense
const userResource = createResource(fetchUser('user-001'));

function UserProfile(): JSX.Element {
  // จะ throw Promise ถ้ายังโหลดไม่เสร็จ
  const user = userResource.read();
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

function UserPage(): JSX.Element {
  return (
    <React.Suspense fallback={<LoadingSpinner />}>
      <UserProfile />
    </React.Suspense>
  );
}
```

---

## 9. Error Boundaries

```tsx
// Error Boundary ที่ครอบคลุม

interface ErrorInfo {
  componentStack: string;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
  errorInfo: ErrorInfo | null;
}

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  resetKeys?: any[];
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null,
    errorInfo: null
  };
  
  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    this.setState({ errorInfo });
    
    // Log to error tracking service
    console.error('Uncaught error:', error, errorInfo);
    
    // Report to Sentry, etc.
    this.props.onError?.(error, errorInfo as ErrorInfo);
  }
  
  componentDidUpdate(prevProps: ErrorBoundaryProps): void {
    // Reset error state when resetKeys change
    if (
      this.state.hasError &&
      this.props.resetKeys?.some(
        (key, index) => key !== prevProps.resetKeys?.[index]
      )
    ) {
      this.reset();
    }
  }
  
  reset = (): void => {
    this.setState({ hasError: false, error: null, errorInfo: null });
  };
  
  render(): ReactNode {
    if (this.state.hasError && this.state.error) {
      const { fallback } = this.props;
      
      if (typeof fallback === 'function') {
        return fallback(this.state.error, this.reset);
      }
      
      if (fallback) {
        return fallback;
      }
      
      return (
        <div className="error-boundary-fallback">
          <h2>Something went wrong</h2>
          <details>
            <summary>Error Details</summary>
            <pre>{this.state.error.message}</pre>
            {process.env.NODE_ENV === 'development' && (
              <pre>{this.state.errorInfo?.componentStack}</pre>
            )}
          </details>
          <button onClick={this.reset}>Try Again</button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// Functional wrapper สำหรับ Error Boundary
function withErrorBoundaryFn<TProps extends object>(
  Component: React.ComponentType<TProps>,
  fallback?: ReactNode
) {
  return function WrappedComponent(props: TProps): JSX.Element {
    return (
      <ErrorBoundary fallback={fallback}>
        <Component {...props} />
      </ErrorBoundary>
    );
  };
}

// การใช้งาน
function RiskyComponent(): JSX.Element {
  if (Math.random() > 0.5) {
    throw new Error('Random error occurred!');
  }
  return <div>Success!</div>;
}

function App(): JSX.Element {
  return (
    <ErrorBoundary
      fallback={(error, reset) => (
        <div className="error-page">
          <h1>Oops! Something went wrong</h1>
          <p>{error.message}</p>
          <button onClick={reset}>Try Again</button>
        </div>
      )}
      onError={(error) => {
        // ส่ง error ไป logging service
        console.log('Error reported:', error.message);
      }}
    >
      <RiskyComponent />
    </ErrorBoundary>
  );
}
```

---

## 10. Portal Pattern

```tsx
// React Portal สำหรับ Modal, Tooltip, Dropdown

import { createPortal } from 'react-dom';

// Portal Hook
function usePortal(elementId: string): HTMLElement {
  const [element] = useState<HTMLElement>(() => {
    let el = document.getElementById(elementId);
    if (!el) {
      el = document.createElement('div');
      el.setAttribute('id', elementId);
      document.body.appendChild(el);
    }
    return el;
  });
  
  return element;
}

// Modal Component ที่ใช้ Portal
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: ReactNode;
  size?: 'sm' | 'md' | 'lg' | 'xl';
}

function Modal({ isOpen, onClose, title, children, size = 'md' }: ModalProps): JSX.Element | null {
  const portalElement = usePortal('modal-root');
  
  // Handle keyboard events
  React.useEffect(() => {
    if (!isOpen) return;
    
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    
    document.addEventListener('keydown', handleEscape);
    document.body.style.overflow = 'hidden';
    
    return () => {
      document.removeEventListener('keydown', handleEscape);
      document.body.style.overflow = '';
    };
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  return createPortal(
    <div
      className="modal-overlay"
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose();
      }}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div className={`modal modal--${size}`}>
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            className="modal-close"
            onClick={onClose}
            aria-label="Close modal"
          >
            ×
          </button>
        </div>
        <div className="modal-content">{children}</div>
      </div>
    </div>,
    portalElement
  );
}

// Tooltip Component
interface TooltipProps {
  content: string;
  children: React.ReactElement;
  placement?: 'top' | 'bottom' | 'left' | 'right';
}

function Tooltip({ content, children, placement = 'top' }: TooltipProps): JSX.Element {
  const [visible, setVisible] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const triggerRef = React.useRef<HTMLElement>(null);
  const portalElement = usePortal('tooltip-root');
  
  const updatePosition = () => {
    if (!triggerRef.current) return;
    
    const rect = triggerRef.current.getBoundingClientRect();
    
    const positions = {
      top: { top: rect.top - 40, left: rect.left + rect.width / 2 },
      bottom: { top: rect.bottom + 8, left: rect.left + rect.width / 2 },
      left: { top: rect.top + rect.height / 2, left: rect.left - 120 },
      right: { top: rect.top + rect.height / 2, left: rect.right + 8 }
    };
    
    setPosition(positions[placement]);
  };
  
  const handleMouseEnter = () => {
    updatePosition();
    setVisible(true);
  };
  
  const clone = React.cloneElement(children, {
    ref: triggerRef,
    onMouseEnter: handleMouseEnter,
    onMouseLeave: () => setVisible(false)
  });
  
  return (
    <>
      {clone}
      {visible && createPortal(
        <div
          className={`tooltip tooltip--${placement}`}
          style={{
            position: 'fixed',
            top: position.top,
            left: position.left,
            zIndex: 9999
          }}
        >
          {content}
        </div>,
        portalElement
      )}
    </>
  );
}

// การใช้งาน
function App(): JSX.Element {
  const [showModal, setShowModal] = useState(false);
  
  return (
    <div>
      <Tooltip content="คลิกเพื่อเปิด Modal" placement="top">
        <button onClick={() => setShowModal(true)}>
          Open Modal
        </button>
      </Tooltip>
      
      <Modal
        isOpen={showModal}
        onClose={() => setShowModal(false)}
        title="ยืนยันการทำรายการ"
        size="md"
      >
        <p>คุณแน่ใจหรือไม่ที่จะดำเนินการต่อ?</p>
        <div className="modal-footer">
          <button onClick={() => setShowModal(false)}>ยกเลิก</button>
          <button className="btn-primary">ยืนยัน</button>
        </div>
      </Modal>
    </div>
  );
}
```

---

## 11. Performance Patterns

```tsx
// React.memo - ป้องกัน re-render ที่ไม่จำเป็น

interface UserCardProps {
  user: {
    id: string;
    name: string;
    avatar: string;
    role: string;
  };
  onSelect: (id: string) => void;
}

// ❌ ไม่ใช้ memo - re-render ทุกครั้งที่ parent re-render
function UserCardWithoutMemo({ user, onSelect }: UserCardProps): JSX.Element {
  console.log(`Rendering UserCard for ${user.name}`);
  return (
    <div onClick={() => onSelect(user.id)}>
      <img src={user.avatar} alt={user.name} />
      <h3>{user.name}</h3>
      <span>{user.role}</span>
    </div>
  );
}

// ✅ ใช้ memo - re-render เฉพาะเมื่อ props เปลี่ยน
const UserCardWithMemo = React.memo(function UserCard(
  { user, onSelect }: UserCardProps
): JSX.Element {
  console.log(`Rendering UserCard for ${user.name}`);
  return (
    <div onClick={() => onSelect(user.id)}>
      <img src={user.avatar} alt={user.name} />
      <h3>{user.name}</h3>
      <span>{user.role}</span>
    </div>
  );
}, (prevProps, nextProps) => {
  // Custom comparison function
  return (
    prevProps.user.id === nextProps.user.id &&
    prevProps.user.name === nextProps.user.name &&
    prevProps.user.role === nextProps.user.role &&
    prevProps.onSelect === nextProps.onSelect
  );
});

// useCallback - Memoize functions
function UserList({ users }: { users: any[] }): JSX.Element {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [filter, setFilter] = useState('');
  
  // ❌ สร้าง function ใหม่ทุกครั้ง
  const handleSelectBad = (id: string) => {
    setSelectedId(id);
  };
  
  // ✅ Memoize function
  const handleSelect = React.useCallback((id: string) => {
    setSelectedId(id);
  }, []); // ไม่ต้อง dependency เพราะ setSelectedId เป็น stable function
  
  // ✅ useMemo สำหรับ computed values
  const filteredUsers = React.useMemo(
    () => users.filter(user =>
      user.name.toLowerCase().includes(filter.toLowerCase())
    ),
    [users, filter]
  );
  
  return (
    <div>
      <input
        value={filter}
        onChange={e => setFilter(e.target.value)}
        placeholder="Filter users..."
      />
      {filteredUsers.map(user => (
        <UserCardWithMemo
          key={user.id}
          user={user}
          onSelect={handleSelect}
        />
      ))}
    </div>
  );
}

// useMemo Deep Dive
function ExpensiveCalculation({ data }: { data: number[] }): JSX.Element {
  const [multiplier, setMultiplier] = useState(1);
  const [unrelated, setUnrelated] = useState(0);
  
  // ❌ คำนวณใหม่ทุกครั้งที่ unrelated เปลี่ยน
  const resultBad = data.reduce((sum, n) => sum + n * multiplier, 0);
  
  // ✅ คำนวณใหม่เฉพาะเมื่อ data หรือ multiplier เปลี่ยน
  const result = React.useMemo(
    () => {
      console.log('Performing expensive calculation...');
      return data.reduce((sum, n) => sum + n * multiplier, 0);
    },
    [data, multiplier]
  );
  
  return (
    <div>
      <p>Result: {result}</p>
      <button onClick={() => setMultiplier(m => m + 1)}>
        Multiplier: {multiplier}
      </button>
      <button onClick={() => setUnrelated(n => n + 1)}>
        Unrelated: {unrelated}
      </button>
    </div>
  );
}

// useTransition สำหรับ non-urgent updates
function SearchWithTransition(): JSX.Element {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = React.useTransition();
  
  const handleSearch = (value: string) => {
    setQuery(value); // Urgent: update input immediately
    
    startTransition(() => {
      // Non-urgent: update results
      const filtered = massiveDataset.filter(item =>
        item.toLowerCase().includes(value.toLowerCase())
      );
      setResults(filtered);
    });
  };
  
  return (
    <div>
      <input
        value={query}
        onChange={e => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      {isPending && <span>Updating results...</span>}
      <ul>
        {results.slice(0, 100).map((item, i) => (
          <li key={i}>{item}</li>
        ))}
      </ul>
    </div>
  );
}

// useDeferredValue
function DeferredExample({ query }: { query: string }): JSX.Element {
  const deferredQuery = React.useDeferredValue(query);
  
  const isStale = query !== deferredQuery;
  
  const results = React.useMemo(
    () => expensiveSearch(deferredQuery),
    [deferredQuery]
  );
  
  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      {results.map((result, i) => (
        <div key={i}>{result}</div>
      ))}
    </div>
  );
}
```

---

## 12. Ref Patterns

```tsx
// Advanced Ref Patterns

// forwardRef กับ TypeScript
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const Input = React.forwardRef<HTMLInputElement, InputProps>(
  function Input({ label, error, ...props }, ref): JSX.Element {
    const id = React.useId();
    
    return (
      <div className="form-field">
        <label htmlFor={id}>{label}</label>
        <input
          ref={ref}
          id={id}
          className={`input ${error ? 'input--error' : ''}`}
          aria-describedby={error ? `${id}-error` : undefined}
          {...props}
        />
        {error && (
          <span id={`${id}-error`} className="error-message">
            {error}
          </span>
        )}
      </div>
    );
  }
);

Input.displayName = 'Input';

// useImperativeHandle
interface VideoPlayerRef {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
  getCurrentTime: () => number;
}

interface VideoPlayerProps {
  src: string;
  onTimeUpdate?: (time: number) => void;
}

const VideoPlayer = React.forwardRef<VideoPlayerRef, VideoPlayerProps>(
  function VideoPlayer({ src, onTimeUpdate }, ref): JSX.Element {
    const videoRef = React.useRef<HTMLVideoElement>(null);
    
    React.useImperativeHandle(ref, () => ({
      play: () => videoRef.current?.play(),
      pause: () => videoRef.current?.pause(),
      seek: (time: number) => {
        if (videoRef.current) {
          videoRef.current.currentTime = time;
        }
      },
      getCurrentTime: () => videoRef.current?.currentTime || 0
    }), []);
    
    return (
      <video
        ref={videoRef}
        src={src}
        onTimeUpdate={() => {
          onTimeUpdate?.(videoRef.current?.currentTime || 0);
        }}
        controls
      />
    );
  }
);

// การใช้งาน VideoPlayer
function VideoPage(): JSX.Element {
  const playerRef = React.useRef<VideoPlayerRef>(null);
  const [currentTime, setCurrentTime] = useState(0);
  
  return (
    <div>
      <VideoPlayer
        ref={playerRef}
        src="/video/sample.mp4"
        onTimeUpdate={setCurrentTime}
      />
      <div>Time: {currentTime.toFixed(1)}s</div>
      <div>
        <button onClick={() => playerRef.current?.play()}>Play</button>
        <button onClick={() => playerRef.current?.pause()}>Pause</button>
        <button onClick={() => playerRef.current?.seek(0)}>Rewind</button>
      </div>
    </div>
  );
}

// useCallback ref (ref callback)
function useMeasure(): [
  (element: HTMLElement | null) => void,
  { width: number; height: number }
] {
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  const ref = React.useCallback((element: HTMLElement | null) => {
    if (!element) return;
    
    const observer = new ResizeObserver(entries => {
      const { width, height } = entries[0].contentRect;
      setSize({ width, height });
    });
    
    observer.observe(element);
    
    // Initial measurement
    const { width, height } = element.getBoundingClientRect();
    setSize({ width, height });
    
    return () => observer.disconnect();
  }, []);
  
  return [ref, size];
}

// การใช้งาน
function ResponsiveComponent(): JSX.Element {
  const [ref, size] = useMeasure();
  
  return (
    <div ref={ref}>
      <p>Width: {size.width}px, Height: {size.height}px</p>
      {size.width > 600 ? (
        <WideLayout />
      ) : (
        <NarrowLayout />
      )}
    </div>
  );
}
```

---

## สรุป

Advanced React Patterns ช่วยให้เราสร้าง components ที่:

1. **Compound Components** - ยืดหยุ่นและ composable
2. **Render Props** - แชร์ logic โดยไม่ผูกกับ UI
3. **HOC** - เพิ่ม functionality โดยไม่แก้ original component
4. **Hooks Composition** - แยก business logic ออกจาก UI
5. **Context + Reducer** - จัดการ state ที่ซับซ้อน
6. **Observer Pattern** - event-driven communication
7. **Virtualization** - แสดงข้อมูลจำนวนมากอย่างมีประสิทธิภาพ
8. **Suspense** - จัดการ async loading อย่างสวยงาม
9. **Error Boundaries** - จัดการ errors อย่างสง่างาม
10. **Portals** - render ใน DOM nodes ที่ต้องการ
11. **Ref Patterns** - เข้าถึง DOM และ component APIs
12. **Performance Patterns** - optimize re-renders ด้วย memo, useCallback, useMemo
