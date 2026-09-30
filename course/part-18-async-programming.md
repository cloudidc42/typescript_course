# ตอนที่ 18: Async Programming ใน TypeScript

## บทนำ

การเขียนโปรแกรมแบบ asynchronous เป็นหัวใจสำคัญของ JavaScript/TypeScript โดยเฉพาะสำหรับการจัดการ I/O operations เช่น API calls, การอ่านไฟล์, และ database queries TypeScript เพิ่ม type safety ให้กับ async code ทำให้พัฒนาได้ง่ายและปลอดภัยขึ้น

---

## 1. Promises กับ TypeScript Types

### 1.1 Promise พื้นฐาน

```typescript
// Promise<T> - T คือ type ที่ resolve
const fetchUser = (id: number): Promise<{ id: number; name: string; email: string }> => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (id > 0) {
        resolve({ id, name: 'สมชาย', email: 'somchai@example.com' });
      } else {
        reject(new Error('Invalid user ID'));
      }
    }, 1000);
  });
};

// การใช้งาน
fetchUser(1)
  .then(user => console.log(`ผู้ใช้: ${user.name}`))
  .catch(error => console.error('เกิดข้อผิดพลาด:', error.message));
```

### 1.2 Promise Types ต่างๆ

```typescript
// Promise<void> - ไม่มีค่า return
function sendEmail(to: string, subject: string, body: string): Promise<void> {
  return new Promise((resolve, reject) => {
    console.log(`ส่งอีเมลไปยัง: ${to}`);
    resolve(); // resolve โดยไม่มีค่า
  });
}

// Promise<boolean>
function checkUserExists(email: string): Promise<boolean> {
  return new Promise(resolve => {
    resolve(email.includes('@'));
  });
}

// Promise<string[]>
function getPermissions(userId: number): Promise<string[]> {
  return new Promise(resolve => {
    resolve(['read', 'write', 'execute']);
  });
}

// Promise<null | User>
interface User {
  id: number;
  name: string;
  email: string;
}

function findUserByEmail(email: string): Promise<User | null> {
  return new Promise(resolve => {
    if (email === 'admin@example.com') {
      resolve({ id: 1, name: 'Admin', email });
    } else {
      resolve(null);
    }
  });
}
```

### 1.3 Generic Promise Functions

```typescript
function delay<T>(ms: number, value: T): Promise<T> {
  return new Promise(resolve => {
    setTimeout(() => resolve(value), ms);
  });
}

function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error(`Timeout หลังจาก ${ms}ms`)), ms);
  });
  
  return Promise.race([promise, timeout]);
}

function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  delayMs: number = 1000
): Promise<T> {
  return new Promise(async (resolve, reject) => {
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        const result = await fn();
        resolve(result);
        return;
      } catch (error) {
        console.warn(`ครั้งที่ ${attempt}/${maxAttempts} ล้มเหลว`);
        if (attempt === maxAttempts) {
          reject(error);
        } else {
          await delay(delayMs * attempt, undefined);
        }
      }
    }
  });
}
```

### 1.4 Promise Chaining กับ Types

```typescript
interface Order {
  id: number;
  userId: number;
  products: number[];
  total: number;
}

interface OrderWithUser extends Order {
  user: User;
}

interface FullOrder extends OrderWithUser {
  productDetails: { id: number; name: string; price: number }[];
}

async function getFullOrder(orderId: number): Promise<FullOrder> {
  const order = await fetchOrder(orderId);
  const user = await fetchUser(order.userId);
  const productDetails = await Promise.all(
    order.products.map(id => fetchProduct(id))
  );
  
  return { ...order, user, productDetails };
}

// Helper functions (stubs)
async function fetchOrder(id: number): Promise<Order> {
  return { id, userId: 1, products: [1, 2, 3], total: 500 };
}

async function fetchProduct(id: number): Promise<{ id: number; name: string; price: number }> {
  return { id, name: `Product ${id}`, price: 100 };
}
```

---

## 2. async/await Typing

### 2.1 async Function Types

```typescript
// async function return type เป็น Promise<T> เสมอ
async function getNumber(): Promise<number> {
  return 42; // TypeScript รู้ว่านี่คือ Promise<number>
}

// async arrow function
const getText = async (): Promise<string> => {
  return "Hello TypeScript";
};

// TypeScript infer return type อัตโนมัติ
async function getUser(id: number) {
  // TypeScript จะ infer เป็น Promise<{ id: number; name: string }>
  return { id, name: 'ผู้ใช้' };
}
```

### 2.2 await กับ Types

```typescript
async function processUser(): Promise<void> {
  // TypeScript รู้ type ของ user
  const user: User = await fetchUser(1);
  console.log(user.name); // TypeScript รู้ว่า name เป็น string
  
  // Destructuring กับ await
  const { id, name, email } = await fetchUser(2);
  
  // ใช้ type assertion เมื่อจำเป็น
  const response = await fetch('https://api.example.com/data');
  const data = (await response.json()) as { items: string[]; total: number };
}
```

### 2.3 async Class Methods

```typescript
class UserRepository {
  private baseUrl = 'https://api.example.com';
  
  async findById(id: number): Promise<User | null> {
    try {
      const response = await fetch(`${this.baseUrl}/users/${id}`);
      if (response.status === 404) return null;
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json() as Promise<User>;
    } catch (error) {
      console.error('findById error:', error);
      throw error;
    }
  }
  
  async findAll(page: number = 1, limit: number = 10): Promise<{
    users: User[];
    total: number;
    page: number;
  }> {
    const response = await fetch(
      `${this.baseUrl}/users?page=${page}&limit=${limit}`
    );
    return response.json();
  }
  
  async create(data: Omit<User, 'id'>): Promise<User> {
    const response = await fetch(`${this.baseUrl}/users`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    
    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(errorData.message || 'Failed to create user');
    }
    
    return response.json();
  }
  
  async update(id: number, data: Partial<User>): Promise<User> {
    const response = await fetch(`${this.baseUrl}/users/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    
    if (!response.ok) throw new Error(`Failed to update user ${id}`);
    return response.json();
  }
  
  async delete(id: number): Promise<void> {
    const response = await fetch(`${this.baseUrl}/users/${id}`, {
      method: 'DELETE'
    });
    
    if (!response.ok) throw new Error(`Failed to delete user ${id}`);
  }
}
```

---

## 3. Promise.all, Promise.race, Promise.allSettled

### 3.1 Promise.all

```typescript
// Promise.all รอทุก Promise สำเร็จ ถ้า reject หนึ่งตัว ทั้งหมด reject
async function fetchMultipleUsers(ids: number[]): Promise<User[]> {
  const promises = ids.map(id => fetchUser(id));
  const users = await Promise.all(promises);
  return users;
}

// Type inference กับ Promise.all
async function fetchDashboardData(): Promise<{
  users: User[];
  orders: Order[];
  stats: { totalRevenue: number; activeUsers: number };
}> {
  const [users, orders, stats] = await Promise.all([
    fetchAllUsers(),
    fetchAllOrders(),
    fetchStats()
  ]);
  
  return { users, orders, stats };
}

// Helper stubs
async function fetchAllUsers(): Promise<User[]> { return []; }
async function fetchAllOrders(): Promise<Order[]> { return []; }
async function fetchStats(): Promise<{ totalRevenue: number; activeUsers: number }> {
  return { totalRevenue: 0, activeUsers: 0 };
}
```

### 3.2 Promise.race

```typescript
// Promise.race - ตัวแรกที่ settle ชนะ
async function fetchWithFallback<T>(
  primary: Promise<T>,
  fallback: Promise<T>,
  timeoutMs: number = 3000
): Promise<T> {
  const timeoutPromise = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), timeoutMs)
  );
  
  try {
    return await Promise.race([primary, timeoutPromise]);
  } catch {
    return fallback;
  }
}

// Implement timeout ด้วย Promise.race
function withRaceTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number,
  errorMessage?: string
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(
      () => reject(new Error(errorMessage || `Timeout หลังจาก ${timeoutMs}ms`)),
      timeoutMs
    )
  );
  
  return Promise.race([promise, timeout]);
}

// ใช้งาน
async function fetchWithTimeout() {
  const data = await withRaceTimeout(
    fetch('https://api.example.com/slow-endpoint').then(r => r.json()),
    5000,
    'API response timeout'
  );
  return data;
}
```

### 3.3 Promise.allSettled

```typescript
interface SuccessResult<T> {
  status: 'fulfilled';
  value: T;
}

interface FailureResult {
  status: 'rejected';
  reason: any;
}

type SettledResult<T> = SuccessResult<T> | FailureResult;

// Promise.allSettled - รอทุก Promise ไม่ว่าจะ resolve หรือ reject
async function fetchUsersWithStatus(ids: number[]): Promise<{
  successful: User[];
  failed: { id: number; error: string }[];
}> {
  const promises = ids.map(id =>
    fetchUser(id).then(user => ({ id, user }))
  );
  
  const results = await Promise.allSettled(promises);
  
  const successful: User[] = [];
  const failed: { id: number; error: string }[] = [];
  
  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      successful.push(result.value.user);
    } else {
      failed.push({
        id: ids[index],
        error: result.reason?.message || 'Unknown error'
      });
    }
  });
  
  return { successful, failed };
}

// การใช้งาน
async function main() {
  const { successful, failed } = await fetchUsersWithStatus([1, 2, 3, 999]);
  
  console.log(`โหลดสำเร็จ: ${successful.length} คน`);
  console.log(`โหลดล้มเหลว: ${failed.length} คน`);
  failed.forEach(f => console.log(`  - User ${f.id}: ${f.error}`));
}
```

### 3.4 Promise.any

```typescript
// Promise.any - รอตัวแรกที่ resolve สำเร็จ
async function fetchFromMultipleSources<T>(
  sources: (() => Promise<T>)[]
): Promise<T> {
  const promises = sources.map(fn => fn());
  return Promise.any(promises);
}

// CDN fallback pattern
async function loadScript(urls: string[]): Promise<string> {
  const fetchPromises = urls.map(url =>
    fetch(url).then(r => {
      if (!r.ok) throw new Error(`Failed: ${url}`);
      return url;
    })
  );
  
  try {
    return await Promise.any(fetchPromises);
  } catch {
    throw new Error('ไม่สามารถโหลด script ได้จากทุก URL');
  }
}
```

---

## 4. Error Handling ใน Async Code

### 4.1 try/catch กับ async/await

```typescript
class ApiError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public originalError?: unknown
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

async function safeFetch<T>(url: string): Promise<T> {
  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      const errorBody = await response.json().catch(() => ({}));
      throw new ApiError(
        response.status,
        errorBody.message || `HTTP Error ${response.status}`
      );
    }
    
    return response.json();
  } catch (error) {
    if (error instanceof ApiError) throw error;
    
    throw new ApiError(
      500,
      error instanceof Error ? error.message : 'Unknown error occurred',
      error
    );
  }
}
```

### 4.2 async Error Propagation

```typescript
// Error propagation ผ่าน call stack
async function level3(id: number): Promise<User> {
  const user = await fetchUser(id);
  if (!user) throw new Error(`User ${id} not found`);
  return user;
}

async function level2(id: number): Promise<{ user: User; orders: Order[] }> {
  const user = await level3(id); // error propagates up
  const orders = await fetchOrdersByUserId(id);
  return { user, orders };
}

async function level1(id: number): Promise<void> {
  try {
    const { user, orders } = await level2(id);
    console.log(`User: ${user.name}, Orders: ${orders.length}`);
  } catch (error) {
    if (error instanceof Error) {
      console.error(`Error in level1: ${error.message}`);
    }
    throw error; // re-throw ถ้าต้องการ
  }
}

async function fetchOrdersByUserId(userId: number): Promise<Order[]> {
  return [];
}
```

### 4.3 Async Error Boundary Pattern

```typescript
type AsyncResult<T> = { success: true; data: T } | { success: false; error: Error };

async function safeAsync<T>(
  fn: () => Promise<T>
): Promise<AsyncResult<T>> {
  try {
    const data = await fn();
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error : new Error(String(error))
    };
  }
}

// การใช้งาน
async function processOrder(orderId: number) {
  const result = await safeAsync(() => fetchOrder(orderId));
  
  if (!result.success) {
    console.error('Failed to fetch order:', result.error.message);
    return null;
  }
  
  const order = result.data;
  console.log('Order:', order);
  return order;
}
```

---

## 5. Async Generators และ Iterators

### 5.1 Async Generator พื้นฐาน

```typescript
// Async Generator function
async function* generateNumbers(start: number, end: number): AsyncGenerator<number> {
  for (let i = start; i <= end; i++) {
    await new Promise(resolve => setTimeout(resolve, 100)); // จำลอง async
    yield i;
  }
}

// การใช้งาน
async function main() {
  const numbers = generateNumbers(1, 5);
  
  for await (const num of numbers) {
    console.log(`ตัวเลข: ${num}`);
  }
}
```

### 5.2 Async Pagination Generator

```typescript
interface Page<T> {
  items: T[];
  nextCursor?: string;
  hasMore: boolean;
}

async function* paginate<T>(
  fetchPage: (cursor?: string) => Promise<Page<T>>
): AsyncGenerator<T[], void, unknown> {
  let cursor: string | undefined;
  
  do {
    const page = await fetchPage(cursor);
    yield page.items;
    cursor = page.nextCursor;
    
    if (!page.hasMore) break;
  } while (cursor);
}

// การใช้งาน - ดึงผู้ใช้ทั้งหมดทีละหน้า
async function getAllUsers(): Promise<User[]> {
  const allUsers: User[] = [];
  
  const userPages = paginate(async (cursor) => {
    const response = await fetch(
      `/api/users?cursor=${cursor || ''}&limit=100`
    );
    return response.json();
  });
  
  for await (const users of userPages) {
    allUsers.push(...users);
    console.log(`โหลดผู้ใช้แล้ว ${allUsers.length} คน`);
  }
  
  return allUsers;
}
```

### 5.3 Event Stream Generator

```typescript
async function* streamEvents(
  url: string
): AsyncGenerator<{ type: string; data: any }> {
  const response = await fetch(url);
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  
  let buffer = '';
  
  while (true) {
    const { done, value } = await reader.read();
    
    if (done) break;
    
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop() || '';
    
    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        if (data === '[DONE]') return;
        
        try {
          yield JSON.parse(data);
        } catch {
          console.warn('Failed to parse event:', data);
        }
      }
    }
  }
}

// การใช้งาน
async function processStream() {
  const stream = streamEvents('https://api.example.com/events');
  
  for await (const event of stream) {
    console.log('Event:', event.type, event.data);
  }
}
```

### 5.4 Async Iterator Protocol

```typescript
class DatabaseCursor<T> implements AsyncIterator<T> {
  private offset = 0;
  private done = false;
  
  constructor(
    private query: string,
    private batchSize: number = 100
  ) {}
  
  async next(): Promise<IteratorResult<T>> {
    if (this.done) {
      return { value: undefined as any, done: true };
    }
    
    const batch = await this.fetchBatch();
    
    if (batch.length === 0) {
      this.done = true;
      return { value: undefined as any, done: true };
    }
    
    this.offset += batch.length;
    // ส่ง batch ทีละตัว (simplified)
    return { value: batch[0] as T, done: false };
  }
  
  private async fetchBatch(): Promise<T[]> {
    const response = await fetch(
      `/api/query?sql=${encodeURIComponent(this.query)}&offset=${this.offset}&limit=${this.batchSize}`
    );
    return response.json();
  }
  
  [Symbol.asyncIterator]() {
    return this;
  }
}

class AsyncCollection<T> {
  constructor(private source: AsyncIterable<T>) {}
  
  async toArray(): Promise<T[]> {
    const result: T[] = [];
    for await (const item of this.source) {
      result.push(item);
    }
    return result;
  }
  
  async *filter(predicate: (item: T) => boolean | Promise<boolean>): AsyncGenerator<T> {
    for await (const item of this.source) {
      if (await predicate(item)) {
        yield item;
      }
    }
  }
  
  async *map<U>(transform: (item: T) => U | Promise<U>): AsyncGenerator<U> {
    for await (const item of this.source) {
      yield await transform(item);
    }
  }
  
  async *take(count: number): AsyncGenerator<T> {
    let taken = 0;
    for await (const item of this.source) {
      if (taken >= count) break;
      yield item;
      taken++;
    }
  }
}
```

---

## 6. Async Patterns

### 6.1 Retry Pattern

```typescript
interface RetryOptions {
  maxAttempts?: number;
  baseDelay?: number;
  maxDelay?: number;
  exponential?: boolean;
  onRetry?: (error: Error, attempt: number) => void;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const {
    maxAttempts = 3,
    baseDelay = 1000,
    maxDelay = 30000,
    exponential = true,
    onRetry
  } = options;
  
  let lastError: Error;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));
      
      if (attempt === maxAttempts) break;
      
      const delay = exponential
        ? Math.min(baseDelay * Math.pow(2, attempt - 1), maxDelay)
        : baseDelay;
      
      onRetry?.(lastError, attempt);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  
  throw lastError!;
}

// การใช้งาน
const data = await withRetry(
  () => fetch('https://api.example.com/data').then(r => r.json()),
  {
    maxAttempts: 5,
    baseDelay: 500,
    exponential: true,
    onRetry: (error, attempt) => {
      console.warn(`ลองใหม่ครั้งที่ ${attempt}: ${error.message}`);
    }
  }
);
```

### 6.2 Timeout Pattern

```typescript
class TimeoutError extends Error {
  constructor(ms: number) {
    super(`Operation timed out after ${ms}ms`);
    this.name = 'TimeoutError';
  }
}

function createTimeout(ms: number): { promise: Promise<never>; cancel: () => void } {
  let timeoutId: ReturnType<typeof setTimeout>;
  
  const promise = new Promise<never>((_, reject) => {
    timeoutId = setTimeout(() => reject(new TimeoutError(ms)), ms);
  });
  
  const cancel = () => clearTimeout(timeoutId);
  
  return { promise, cancel };
}

async function withTimeout2<T>(
  operation: () => Promise<T>,
  ms: number
): Promise<T> {
  const { promise: timeoutPromise, cancel } = createTimeout(ms);
  
  try {
    const result = await Promise.race([operation(), timeoutPromise]);
    cancel(); // ยกเลิก timeout เมื่อสำเร็จ
    return result;
  } catch (error) {
    cancel();
    throw error;
  }
}

// การใช้งาน
async function fetchWithTimeoutExample() {
  try {
    const user = await withTimeout2(
      () => fetchUser(1),
      3000
    );
    console.log('User:', user);
  } catch (error) {
    if (error instanceof TimeoutError) {
      console.error('Request timed out!');
    }
  }
}
```

### 6.3 Debounce Pattern

```typescript
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => Promise<Awaited<ReturnType<T>>> {
  let timeoutId: ReturnType<typeof setTimeout> | null = null;
  let resolveRef: ((value: any) => void) | null = null;
  let rejectRef: ((reason?: any) => void) | null = null;
  
  return function(...args: Parameters<T>): Promise<Awaited<ReturnType<T>>> {
    return new Promise((resolve, reject) => {
      if (timeoutId) {
        clearTimeout(timeoutId);
        rejectRef?.(new Error('Debounced'));
      }
      
      resolveRef = resolve;
      rejectRef = reject;
      
      timeoutId = setTimeout(async () => {
        try {
          const result = await fn(...args);
          resolveRef?.(result);
        } catch (error) {
          rejectRef?.(error);
        } finally {
          timeoutId = null;
          resolveRef = null;
          rejectRef = null;
        }
      }, delay);
    });
  };
}

// การใช้งาน
const debouncedSearch = debounce(async (query: string) => {
  const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
  return response.json();
}, 300);

// เรียกหลายครั้ง แต่จะ execute แค่ครั้งเดียว
async function handleSearchInput(event: any) {
  const results = await debouncedSearch(event.target.value);
  console.log('ผลการค้นหา:', results);
}
```

### 6.4 Throttle Pattern

```typescript
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => Promise<Awaited<ReturnType<T>>> | undefined {
  let inThrottle = false;
  let lastResult: Awaited<ReturnType<T>>;
  
  return function(...args: Parameters<T>) {
    if (!inThrottle) {
      inThrottle = true;
      
      const result = fn(...args);
      
      if (result instanceof Promise) {
        return result.then(value => {
          lastResult = value;
          setTimeout(() => { inThrottle = false; }, limit);
          return value;
        });
      }
      
      lastResult = result;
      setTimeout(() => { inThrottle = false; }, limit);
      return Promise.resolve(lastResult);
    }
    return undefined;
  };
}

// Queue-based throttle ที่ไม่ทิ้ง calls
class ThrottledQueue<T> {
  private queue: Array<{
    fn: () => Promise<T>;
    resolve: (value: T) => void;
    reject: (reason?: any) => void;
  }> = [];
  
  private processing = false;
  
  constructor(private delayMs: number) {}
  
  enqueue(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject });
      if (!this.processing) {
        this.process();
      }
    });
  }
  
  private async process(): Promise<void> {
    this.processing = true;
    
    while (this.queue.length > 0) {
      const { fn, resolve, reject } = this.queue.shift()!;
      
      try {
        const result = await fn();
        resolve(result);
      } catch (error) {
        reject(error);
      }
      
      if (this.queue.length > 0) {
        await new Promise(resolve => setTimeout(resolve, this.delayMs));
      }
    }
    
    this.processing = false;
  }
}
```

### 6.5 Circuit Breaker Pattern

```typescript
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerOptions {
  failureThreshold: number;
  successThreshold: number;
  timeout: number;
}

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private nextAttemptTime = 0;
  
  constructor(private options: CircuitBreakerOptions) {}
  
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttemptTime) {
        throw new Error('Circuit breaker is OPEN - service unavailable');
      }
      this.state = 'HALF_OPEN';
      this.successCount = 0;
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  private onSuccess(): void {
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
        console.log('Circuit breaker: CLOSED (service recovered)');
      }
    } else {
      this.failureCount = 0;
    }
  }
  
  private onFailure(): void {
    this.failureCount++;
    
    if (this.failureCount >= this.options.failureThreshold || this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
      this.nextAttemptTime = Date.now() + this.options.timeout;
      console.log(`Circuit breaker: OPEN (will retry after ${this.options.timeout}ms)`);
    }
  }
  
  getState(): CircuitState {
    return this.state;
  }
}

// การใช้งาน
const apiBreaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 30000
});

async function callApi() {
  return apiBreaker.execute(() =>
    fetch('https://api.example.com/data').then(r => r.json())
  );
}
```

---

## 7. Event Emitters กับ Types

### 7.1 Typed EventEmitter

```typescript
type EventMap = {
  [event: string]: any[];
};

class TypedEventEmitter<Events extends EventMap> {
  private listeners = new Map<keyof Events, Function[]>();
  
  on<K extends keyof Events>(
    event: K,
    listener: (...args: Events[K]) => void
  ): this {
    const existing = this.listeners.get(event) || [];
    this.listeners.set(event, [...existing, listener]);
    return this;
  }
  
  off<K extends keyof Events>(
    event: K,
    listener: (...args: Events[K]) => void
  ): this {
    const existing = this.listeners.get(event) || [];
    this.listeners.set(event, existing.filter(l => l !== listener));
    return this;
  }
  
  once<K extends keyof Events>(
    event: K,
    listener: (...args: Events[K]) => void
  ): this {
    const wrapper = (...args: Events[K]) => {
      listener(...args);
      this.off(event, wrapper as any);
    };
    return this.on(event, wrapper as any);
  }
  
  emit<K extends keyof Events>(event: K, ...args: Events[K]): boolean {
    const listeners = this.listeners.get(event) || [];
    listeners.forEach(listener => listener(...args));
    return listeners.length > 0;
  }
  
  removeAllListeners(event?: keyof Events): this {
    if (event) {
      this.listeners.delete(event);
    } else {
      this.listeners.clear();
    }
    return this;
  }
}

// การใช้งาน
interface AppEvents {
  userLogin: [userId: number, timestamp: Date];
  userLogout: [userId: number];
  error: [error: Error, context: string];
  dataUpdated: [table: string, id: number, data: any];
}

const emitter = new TypedEventEmitter<AppEvents>();

emitter.on('userLogin', (userId, timestamp) => {
  console.log(`User ${userId} logged in at ${timestamp}`);
});

emitter.on('error', (error, context) => {
  console.error(`Error in ${context}:`, error.message);
});

emitter.emit('userLogin', 1, new Date());
emitter.emit('error', new Error('Database connection failed'), 'UserService');
```

### 7.2 Async Event Emitter

```typescript
class AsyncEventEmitter<Events extends EventMap> {
  private listeners = new Map<keyof Events, Array<(...args: any[]) => Promise<void> | void>>();
  
  on<K extends keyof Events>(
    event: K,
    listener: (...args: Events[K]) => Promise<void> | void
  ): this {
    const existing = this.listeners.get(event) || [];
    this.listeners.set(event, [...existing, listener]);
    return this;
  }
  
  async emit<K extends keyof Events>(event: K, ...args: Events[K]): Promise<void> {
    const listeners = this.listeners.get(event) || [];
    await Promise.all(listeners.map(listener => listener(...args)));
  }
  
  async emitSequential<K extends keyof Events>(event: K, ...args: Events[K]): Promise<void> {
    const listeners = this.listeners.get(event) || [];
    for (const listener of listeners) {
      await listener(...args);
    }
  }
}
```

---

## 8. Observable Pattern

### 8.1 Simple Observable Implementation

```typescript
type Observer<T> = {
  next: (value: T) => void;
  error?: (error: Error) => void;
  complete?: () => void;
};

type Subscriber<T> = (observer: Observer<T>) => (() => void) | void;

class Observable<T> {
  constructor(private subscriber: Subscriber<T>) {}
  
  subscribe(observer: Observer<T> | ((value: T) => void)): { unsubscribe: () => void } {
    const obs: Observer<T> = typeof observer === 'function'
      ? { next: observer }
      : observer;
    
    const cleanup = this.subscriber(obs) || (() => {});
    
    return { unsubscribe: cleanup };
  }
  
  pipe<R>(operator: (source: Observable<T>) => Observable<R>): Observable<R> {
    return operator(this);
  }
  
  static from<T>(values: T[]): Observable<T> {
    return new Observable<T>(observer => {
      values.forEach(value => observer.next(value));
      observer.complete?.();
    });
  }
  
  static interval(ms: number): Observable<number> {
    return new Observable<number>(observer => {
      let count = 0;
      const id = setInterval(() => observer.next(count++), ms);
      return () => clearInterval(id);
    });
  }
  
  static fromPromise<T>(promise: Promise<T>): Observable<T> {
    return new Observable<T>(observer => {
      promise
        .then(value => {
          observer.next(value);
          observer.complete?.();
        })
        .catch(error => observer.error?.(error));
    });
  }
  
  map<R>(transform: (value: T) => R): Observable<R> {
    return new Observable<R>(observer => {
      return this.subscribe({
        next: value => observer.next(transform(value)),
        error: observer.error,
        complete: observer.complete
      }).unsubscribe;
    });
  }
  
  filter(predicate: (value: T) => boolean): Observable<T> {
    return new Observable<T>(observer => {
      return this.subscribe({
        next: value => {
          if (predicate(value)) observer.next(value);
        },
        error: observer.error,
        complete: observer.complete
      }).unsubscribe;
    });
  }
  
  take(count: number): Observable<T> {
    return new Observable<T>(observer => {
      let taken = 0;
      const { unsubscribe } = this.subscribe({
        next: value => {
          if (taken < count) {
            observer.next(value);
            taken++;
            if (taken >= count) {
              observer.complete?.();
              unsubscribe();
            }
          }
        },
        error: observer.error,
        complete: observer.complete
      });
      return unsubscribe;
    });
  }
}

// การใช้งาน
const timer$ = Observable.interval(1000).take(5);

const subscription = timer$.map(n => n * 2).subscribe({
  next: value => console.log('Value:', value),
  complete: () => console.log('Completed!')
});

// ยกเลิก subscription เมื่อต้องการ
// subscription.unsubscribe();
```

---

## 9. ตัวอย่าง Async จริง

### 9.1 API Client ที่สมบูรณ์

```typescript
interface RequestConfig {
  headers?: Record<string, string>;
  timeout?: number;
  retries?: number;
}

interface ApiClientConfig {
  baseUrl: string;
  defaultHeaders?: Record<string, string>;
  timeout?: number;
  maxRetries?: number;
  interceptors?: {
    request?: (config: RequestInit) => RequestInit;
    response?: (response: Response) => Response | Promise<Response>;
    error?: (error: Error) => Error;
  };
}

class HttpClient {
  private config: Required<ApiClientConfig>;
  
  constructor(config: ApiClientConfig) {
    this.config = {
      baseUrl: config.baseUrl,
      defaultHeaders: config.defaultHeaders || {},
      timeout: config.timeout || 10000,
      maxRetries: config.maxRetries || 3,
      interceptors: config.interceptors || {}
    };
  }
  
  private async request<T>(
    path: string,
    options: RequestInit = {},
    reqConfig: RequestConfig = {}
  ): Promise<T> {
    const url = `${this.config.baseUrl}${path}`;
    
    let init: RequestInit = {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...this.config.defaultHeaders,
        ...options.headers,
        ...reqConfig.headers
      }
    };
    
    // Apply request interceptor
    if (this.config.interceptors.request) {
      init = this.config.interceptors.request(init);
    }
    
    const timeout = reqConfig.timeout ?? this.config.timeout;
    const maxRetries = reqConfig.retries ?? this.config.maxRetries;
    
    return withRetry(async () => {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), timeout);
      
      try {
        let response = await fetch(url, {
          ...init,
          signal: controller.signal
        });
        
        // Apply response interceptor
        if (this.config.interceptors.response) {
          response = await this.config.interceptors.response(response);
        }
        
        if (!response.ok) {
          const errorData = await response.json().catch(() => ({}));
          throw new ApiError(response.status, errorData.message || response.statusText);
        }
        
        const contentType = response.headers.get('content-type');
        if (contentType?.includes('application/json')) {
          return response.json();
        }
        return response.text() as any;
      } finally {
        clearTimeout(timeoutId);
      }
    }, { maxAttempts: maxRetries });
  }
  
  get<T>(path: string, config?: RequestConfig): Promise<T> {
    return this.request<T>(path, { method: 'GET' }, config);
  }
  
  post<T>(path: string, data?: any, config?: RequestConfig): Promise<T> {
    return this.request<T>(path, {
      method: 'POST',
      body: data ? JSON.stringify(data) : undefined
    }, config);
  }
  
  put<T>(path: string, data?: any, config?: RequestConfig): Promise<T> {
    return this.request<T>(path, {
      method: 'PUT',
      body: data ? JSON.stringify(data) : undefined
    }, config);
  }
  
  patch<T>(path: string, data?: any, config?: RequestConfig): Promise<T> {
    return this.request<T>(path, {
      method: 'PATCH',
      body: data ? JSON.stringify(data) : undefined
    }, config);
  }
  
  delete<T>(path: string, config?: RequestConfig): Promise<T> {
    return this.request<T>(path, { method: 'DELETE' }, config);
  }
}

// การใช้งาน
const api = new HttpClient({
  baseUrl: 'https://api.example.com',
  defaultHeaders: {
    'Authorization': 'Bearer token123'
  },
  timeout: 5000,
  maxRetries: 3
});

interface UserListResponse {
  users: User[];
  total: number;
  page: number;
}

const users = await api.get<UserListResponse>('/users');
const newUser = await api.post<User>('/users', {
  name: 'สมชาย',
  email: 'somchai@example.com'
});
```

### 9.2 Database Query Builder

```typescript
class QueryBuilder<T> {
  private tableName: string;
  private conditions: string[] = [];
  private orderByClause?: string;
  private limitValue?: number;
  private offsetValue?: number;
  private selectColumns: string[] = ['*'];
  
  constructor(table: string) {
    this.tableName = table;
  }
  
  select(...columns: (keyof T | string)[]): this {
    this.selectColumns = columns as string[];
    return this;
  }
  
  where(condition: string): this {
    this.conditions.push(condition);
    return this;
  }
  
  orderBy(column: keyof T, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.orderByClause = `${String(column)} ${direction}`;
    return this;
  }
  
  limit(value: number): this {
    this.limitValue = value;
    return this;
  }
  
  offset(value: number): this {
    this.offsetValue = value;
    return this;
  }
  
  build(): string {
    let query = `SELECT ${this.selectColumns.join(', ')} FROM ${this.tableName}`;
    
    if (this.conditions.length > 0) {
      query += ` WHERE ${this.conditions.join(' AND ')}`;
    }
    
    if (this.orderByClause) {
      query += ` ORDER BY ${this.orderByClause}`;
    }
    
    if (this.limitValue !== undefined) {
      query += ` LIMIT ${this.limitValue}`;
    }
    
    if (this.offsetValue !== undefined) {
      query += ` OFFSET ${this.offsetValue}`;
    }
    
    return query;
  }
  
  async execute(db: any): Promise<T[]> {
    const sql = this.build();
    console.log('Executing:', sql);
    return db.query(sql);
  }
}

// การใช้งาน
const query = new QueryBuilder<User>('users')
  .select('id', 'name', 'email')
  .where('age > 18')
  .where('isActive = true')
  .orderBy('name')
  .limit(10)
  .offset(20);

console.log(query.build());
// SELECT id, name, email FROM users WHERE age > 18 AND isActive = true ORDER BY name ASC LIMIT 10 OFFSET 20
```

### 9.3 Concurrent Request Manager

```typescript
class ConcurrentRequestManager {
  private activeRequests = 0;
  private queue: Array<{
    fn: () => Promise<any>;
    resolve: (value: any) => void;
    reject: (reason?: any) => void;
    priority: number;
  }> = [];
  
  constructor(private maxConcurrent: number = 5) {}
  
  async execute<T>(
    fn: () => Promise<T>,
    priority: number = 0
  ): Promise<T> {
    if (this.activeRequests < this.maxConcurrent) {
      return this.run(fn);
    }
    
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject, priority });
      this.queue.sort((a, b) => b.priority - a.priority); // สูงกว่า = ก่อน
    });
  }
  
  private async run<T>(fn: () => Promise<T>): Promise<T> {
    this.activeRequests++;
    
    try {
      const result = await fn();
      return result;
    } finally {
      this.activeRequests--;
      this.processQueue();
    }
  }
  
  private processQueue(): void {
    if (this.queue.length > 0 && this.activeRequests < this.maxConcurrent) {
      const { fn, resolve, reject } = this.queue.shift()!;
      this.run(fn).then(resolve).catch(reject);
    }
  }
  
  getStats() {
    return {
      activeRequests: this.activeRequests,
      queuedRequests: this.queue.length,
      maxConcurrent: this.maxConcurrent
    };
  }
}

// การใช้งาน
const manager = new ConcurrentRequestManager(3);

// ส่ง 10 requests พร้อมกัน แต่จะ execute ได้แค่ 3 ตัวพร้อมกัน
const results = await Promise.all(
  Array.from({ length: 10 }, (_, i) =>
    manager.execute(() => fetchUser(i + 1), i > 5 ? 10 : 0) // priority สูงสำหรับ high-value requests
  )
);
```

---

## 10. ตัวอย่าง File Operations

```typescript
import fs from 'fs/promises';
import path from 'path';

interface FileInfo {
  name: string;
  path: string;
  size: number;
  extension: string;
  createdAt: Date;
  modifiedAt: Date;
  isDirectory: boolean;
}

async function getFileInfo(filePath: string): Promise<FileInfo> {
  const stats = await fs.stat(filePath);
  const name = path.basename(filePath);
  
  return {
    name,
    path: filePath,
    size: stats.size,
    extension: path.extname(name),
    createdAt: stats.birthtime,
    modifiedAt: stats.mtime,
    isDirectory: stats.isDirectory()
  };
}

async function readJsonFile<T>(filePath: string): Promise<T> {
  const content = await fs.readFile(filePath, 'utf-8');
  return JSON.parse(content) as T;
}

async function writeJsonFile<T>(filePath: string, data: T): Promise<void> {
  const content = JSON.stringify(data, null, 2);
  await fs.mkdir(path.dirname(filePath), { recursive: true });
  await fs.writeFile(filePath, content, 'utf-8');
}

async function* walkDirectory(dir: string): AsyncGenerator<FileInfo> {
  const entries = await fs.readdir(dir, { withFileTypes: true });
  
  for (const entry of entries) {
    const fullPath = path.join(dir, entry.name);
    
    if (entry.isDirectory()) {
      yield* walkDirectory(fullPath);
    } else {
      yield await getFileInfo(fullPath);
    }
  }
}

async function copyDirectory(src: string, dest: string): Promise<void> {
  await fs.mkdir(dest, { recursive: true });
  const entries = await fs.readdir(src, { withFileTypes: true });
  
  await Promise.all(entries.map(async entry => {
    const srcPath = path.join(src, entry.name);
    const destPath = path.join(dest, entry.name);
    
    if (entry.isDirectory()) {
      await copyDirectory(srcPath, destPath);
    } else {
      await fs.copyFile(srcPath, destPath);
    }
  }));
}
```

---

## สรุป

Async Programming ใน TypeScript มีเครื่องมือมากมาย:

1. **Promises** - พื้นฐานของ async programming พร้อม full type safety
2. **async/await** - syntax ที่อ่านง่ายสำหรับ Promise chains
3. **Promise combinators** - all, race, allSettled, any สำหรับจัดการหลาย Promises
4. **Async Generators** - สร้าง streams ข้อมูลแบบ async
5. **Patterns** - retry, timeout, debounce, throttle, circuit breaker
6. **Event Emitters** - type-safe event handling
7. **Observables** - reactive programming patterns
8. **Real-world applications** - HTTP clients, database, file operations

การใช้ TypeScript กับ async code ช่วยให้ตรวจจับ bugs ได้เร็วขึ้น และทำให้โค้ดมีความน่าเชื่อถือสูงขึ้น
