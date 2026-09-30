# ส่วนที่ 48: การเพิ่มประสิทธิภาพ (Performance Optimization) ใน TypeScript

## บทนำ

การเพิ่มประสิทธิภาพของแอปพลิเคชัน TypeScript เป็นสิ่งสำคัญอย่างยิ่งในการพัฒนาซอฟต์แวร์ระดับ Production บทนี้จะครอบคลุมเทคนิคและกลยุทธ์ต่างๆ ที่จะช่วยให้โค้ดของคุณทำงานได้เร็วขึ้น ใช้หน่วยความจำน้อยลง และตอบสนองได้ดีขึ้น

---

## 1. TypeScript Performance Best Practices (แนวทางปฏิบัติที่ดีสำหรับประสิทธิภาพ TypeScript)

### 1.1 การเลือกใช้ Type ที่เหมาะสม

การเลือกใช้ Type ที่เหมาะสมสามารถส่งผลต่อประสิทธิภาพของ Type Checker และเวลาในการ Compile

```typescript
// ❌ หลีกเลี่ยงการใช้ any ซึ่งทำให้เสียประโยชน์จาก TypeScript
function processData(data: any): any {
  return data.map((item: any) => item.value);
}

// ✅ ใช้ Type ที่ชัดเจน
interface DataItem {
  id: number;
  value: string;
  timestamp: Date;
}

function processDataTyped(data: DataItem[]): string[] {
  return data.map(item => item.value);
}

// ✅ ใช้ Generic สำหรับ Reusability
function processGeneric<T extends { value: string }>(data: T[]): string[] {
  return data.map(item => item.value);
}
```

### 1.2 การใช้ Const Assertions

```typescript
// ❌ Array ที่ mutate ได้ทำให้ TypeScript ต้องทำงานมากขึ้น
const colors = ['red', 'green', 'blue']; // string[]

// ✅ Const assertion ทำให้ TypeScript รู้ว่าค่าไม่เปลี่ยนแปลง
const colorsConst = ['red', 'green', 'blue'] as const; // readonly ['red', 'green', 'blue']

type Color = typeof colorsConst[number]; // 'red' | 'green' | 'blue'

// ✅ Object const assertion
const config = {
  api: {
    baseUrl: 'https://api.example.com',
    timeout: 5000,
    retries: 3,
  },
  cache: {
    ttl: 3600,
    maxSize: 1000,
  },
} as const;

type ApiConfig = typeof config.api;
// { readonly baseUrl: 'https://api.example.com'; readonly timeout: 5000; readonly retries: 3 }
```

### 1.3 การใช้ Readonly Types

```typescript
// ✅ ใช้ Readonly เพื่อป้องกันการแก้ไขโดยไม่ตั้งใจ
interface ImmutableUser {
  readonly id: number;
  readonly name: string;
  readonly email: string;
}

// ✅ ReadonlyArray สำหรับ arrays
function processUsers(users: ReadonlyArray<ImmutableUser>): string[] {
  // ไม่สามารถ push หรือแก้ไข users ได้
  return users.map(u => u.name);
}

// ✅ Deep Readonly utility type
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

interface Config {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
}

const appConfig: DeepReadonly<Config> = {
  database: {
    host: 'localhost',
    port: 5432,
    credentials: {
      username: 'admin',
      password: 'secret',
    },
  },
};

// appConfig.database.host = 'newhost'; // Error! Cannot assign
```

---

## 2. Avoiding Performance Pitfalls (การหลีกเลี่ยงปัญหาด้านประสิทธิภาพ)

### 2.1 หลีกเลี่ยง Union Types ที่ซับซ้อนเกินไป

```typescript
// ❌ Union Type ที่ซับซ้อนเกินไปทำให้ Type Checker ช้า
type ComplexUnion = 
  | { type: 'a'; data: string }
  | { type: 'b'; data: number }
  | { type: 'c'; data: boolean }
  | { type: 'd'; data: Date }
  | { type: 'e'; data: string[] }
  | { type: 'f'; data: Record<string, unknown> };

// ✅ ใช้ Discriminated Union ที่มีโครงสร้างชัดเจน
interface BaseEvent {
  type: string;
  timestamp: Date;
}

interface StringEvent extends BaseEvent {
  type: 'string';
  data: string;
}

interface NumberEvent extends BaseEvent {
  type: 'number';
  data: number;
}

type AppEvent = StringEvent | NumberEvent;

function handleEvent(event: AppEvent): void {
  switch (event.type) {
    case 'string':
      console.log(event.data.toUpperCase());
      break;
    case 'number':
      console.log(event.data.toFixed(2));
      break;
  }
}
```

### 2.2 หลีกเลี่ยงการสร้าง Object ซ้ำๆ

```typescript
// ❌ การสร้าง object ใหม่ทุกครั้งที่เรียกฟังก์ชัน
function getDefaultConfig() {
  return {  // สร้าง object ใหม่ทุกครั้ง!
    timeout: 5000,
    retries: 3,
    headers: { 'Content-Type': 'application/json' }
  };
}

// ✅ ใช้ constant object ที่ share กันได้
const DEFAULT_CONFIG = Object.freeze({
  timeout: 5000,
  retries: 3,
  headers: Object.freeze({ 'Content-Type': 'application/json' })
});

function getConfig(override?: Partial<typeof DEFAULT_CONFIG>) {
  return { ...DEFAULT_CONFIG, ...override };
}

// ✅ ใช้ Class สำหรับ object ที่สร้างบ่อยๆ
class Point {
  constructor(
    public readonly x: number,
    public readonly y: number
  ) {}

  distanceTo(other: Point): number {
    return Math.sqrt(
      Math.pow(this.x - other.x, 2) + 
      Math.pow(this.y - other.y, 2)
    );
  }
  
  // Pool pattern สำหรับ reuse objects
  static readonly ORIGIN = new Point(0, 0);
}
```

### 2.3 การใช้ Short-circuit Evaluation

```typescript
// ✅ Short-circuit evaluation ลดการคำนวณที่ไม่จำเป็น
function expensiveOperation(): string {
  // จำลองการทำงานที่ใช้เวลานาน
  return 'result';
}

function processConditional(condition: boolean, value?: string): string {
  // ✅ ถ้า condition เป็น false จะไม่เรียก expensiveOperation
  return condition ? (value ?? expensiveOperation()) : 'default';
}

// ✅ Nullish coalescing สำหรับ default values
const username = null;
const displayName = username ?? 'Anonymous'; // ไม่คำนวณเพิ่มเติม

// ✅ Optional chaining ลด null checks
interface UserProfile {
  user?: {
    address?: {
      city?: string;
    };
  };
}

function getCity(profile: UserProfile): string {
  return profile.user?.address?.city ?? 'Unknown';
}
```

---

## 3. Type-Level Performance (ประสิทธิภาพระดับ Type)

### 3.1 การเพิ่มประสิทธิภาพ Conditional Types

```typescript
// ❌ Conditional types ที่ซับซ้อนทำให้ TypeScript ช้า
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

// ✅ ใช้ Interface แทนเมื่อเป็นไปได้
interface PartialConfig {
  timeout?: number;
  retries?: number;
  baseUrl?: string;
}

// ✅ ใช้ Mapped Types อย่างมีประสิทธิภาพ
type Nullable<T> = { [P in keyof T]: T[P] | null };
type Optional<T> = { [P in keyof T]?: T[P] };

// ✅ ใช้ infer อย่างระมัดระวัง
type ReturnType<T extends (...args: any) => any> = 
  T extends (...args: any) => infer R ? R : never;

type PromiseValue<T> = T extends Promise<infer V> ? V : T;

// ตัวอย่างการใช้งาน
async function fetchUser(): Promise<{ id: number; name: string }> {
  return { id: 1, name: 'John' };
}

type UserData = PromiseValue<ReturnType<typeof fetchUser>>;
// { id: number; name: string }
```

### 3.2 Interface vs Type Alias Performance

```typescript
// Interface ถูก merge ได้และมักจะเร็วกว่าสำหรับ object types
interface UserBase {
  id: number;
  name: string;
}

interface UserWithEmail extends UserBase {
  email: string;
}

// Type alias เหมาะสำหรับ union types และ complex types
type UserId = number | string;
type UserOrNull = UserBase | null;

// ✅ ใช้ Interface สำหรับ Object shapes
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<void>;
}

// ✅ ใช้ Type สำหรับ Function signatures ที่ซับซ้อน
type AsyncHandler<T, R> = (input: T) => Promise<R>;
type Middleware<T> = (ctx: T, next: () => Promise<void>) => Promise<void>;
```

### 3.3 การใช้ Template Literal Types อย่างมีประสิทธิภาพ

```typescript
// ✅ Template Literal Types สำหรับ string manipulation
type EventName = 'click' | 'hover' | 'focus';
type EventHandler = `on${Capitalize<EventName>}`;
// 'onClick' | 'onHover' | 'onFocus'

type CSSProperty = 'color' | 'background' | 'margin' | 'padding';
type CSSValue = `${number}px` | `${number}%` | `${number}rem` | 'auto';

interface StyleMap {
  [K in CSSProperty]?: CSSValue;
}

// ✅ ใช้สำหรับ API route typing
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
type ApiRoute = `/${string}`;
type ApiEndpoint = `${HttpMethod} ${ApiRoute}`;

const endpoint: ApiEndpoint = 'GET /users';

// ✅ สร้าง typed event system
type AppEvents = {
  'user:login': { userId: string; timestamp: Date };
  'user:logout': { userId: string };
  'data:loaded': { count: number; source: string };
  'error:occurred': { message: string; code: number };
};

type EventKey = keyof AppEvents;

function emit<K extends EventKey>(event: K, data: AppEvents[K]): void {
  console.log(`Event: ${event}`, data);
}

emit('user:login', { userId: '123', timestamp: new Date() });
// emit('user:login', { userId: 123 }); // Error! wrong type
```

---

## 4. Lazy Loading (การโหลดแบบ Lazy)

### 4.1 Dynamic Imports

```typescript
// ✅ ใช้ Dynamic Import สำหรับ code ที่ไม่ต้องการทันที
async function loadHeavyModule() {
  const { HeavyComponent } = await import('./HeavyComponent');
  return new HeavyComponent();
}

// ✅ Lazy loading ด้วย Intersection Observer
class LazyLoader {
  private observer: IntersectionObserver;
  private loadedModules = new Map<string, unknown>();

  constructor() {
    this.observer = new IntersectionObserver(
      this.handleIntersection.bind(this),
      { rootMargin: '100px' }
    );
  }

  private async handleIntersection(entries: IntersectionObserverEntry[]) {
    for (const entry of entries) {
      if (entry.isIntersecting) {
        const element = entry.target as HTMLElement;
        const moduleName = element.dataset.lazyModule;
        
        if (moduleName && !this.loadedModules.has(moduleName)) {
          try {
            const module = await import(`./modules/${moduleName}`);
            this.loadedModules.set(moduleName, module);
            this.observer.unobserve(element);
          } catch (error) {
            console.error(`Failed to load module: ${moduleName}`, error);
          }
        }
      }
    }
  }

  observe(element: HTMLElement): void {
    this.observer.observe(element);
  }

  disconnect(): void {
    this.observer.disconnect();
  }
}

// ✅ Route-based lazy loading สำหรับ SPA
interface Route {
  path: string;
  loadComponent: () => Promise<{ default: new () => HTMLElement }>;
}

const routes: Route[] = [
  {
    path: '/home',
    loadComponent: () => import('./pages/HomePage'),
  },
  {
    path: '/profile',
    loadComponent: () => import('./pages/ProfilePage'),
  },
  {
    path: '/settings',
    loadComponent: () => import('./pages/SettingsPage'),
  },
];

async function renderRoute(path: string): Promise<HTMLElement | null> {
  const route = routes.find(r => r.path === path);
  if (!route) return null;
  
  const { default: Component } = await route.loadComponent();
  return new Component();
}
```

### 4.2 Lazy Initialization Pattern

```typescript
// ✅ Lazy initialization ด้วย getter
class Database {
  private _connection: DatabaseConnection | null = null;

  private get connection(): DatabaseConnection {
    if (!this._connection) {
      this._connection = this.createConnection();
    }
    return this._connection;
  }

  private createConnection(): DatabaseConnection {
    console.log('Creating database connection...');
    return {
      query: async (sql: string) => [],
      close: async () => {},
    };
  }

  async query(sql: string): Promise<unknown[]> {
    return this.connection.query(sql);
  }
}

interface DatabaseConnection {
  query(sql: string): Promise<unknown[]>;
  close(): Promise<void>;
}

// ✅ Lazy Singleton
class ConfigManager {
  private static instance: ConfigManager | null = null;
  private config: Record<string, string> = {};

  private constructor() {
    this.loadConfig();
  }

  private loadConfig(): void {
    // โหลด config จาก environment variables
    this.config = {
      apiUrl: process.env.API_URL ?? 'http://localhost:3000',
      dbUrl: process.env.DATABASE_URL ?? 'localhost:5432',
    };
  }

  static getInstance(): ConfigManager {
    if (!ConfigManager.instance) {
      ConfigManager.instance = new ConfigManager();
    }
    return ConfigManager.instance;
  }

  get(key: string): string | undefined {
    return this.config[key];
  }
}

// ✅ Lazy computed values
class DataProcessor {
  private _sortedData: number[] | null = null;
  private _statistics: Statistics | null = null;

  constructor(private readonly rawData: number[]) {}

  get sortedData(): number[] {
    if (!this._sortedData) {
      this._sortedData = [...this.rawData].sort((a, b) => a - b);
    }
    return this._sortedData;
  }

  get statistics(): Statistics {
    if (!this._statistics) {
      const sorted = this.sortedData;
      const sum = sorted.reduce((acc, val) => acc + val, 0);
      this._statistics = {
        min: sorted[0],
        max: sorted[sorted.length - 1],
        mean: sum / sorted.length,
        median: sorted[Math.floor(sorted.length / 2)],
      };
    }
    return this._statistics;
  }
}

interface Statistics {
  min: number;
  max: number;
  mean: number;
  median: number;
}
```

---

## 5. Code Splitting (การแบ่งโค้ด)

### 5.1 Webpack Code Splitting

```typescript
// webpack.config.ts
import { Configuration } from 'webpack';
import path from 'path';

const config: Configuration = {
  entry: {
    main: './src/index.ts',
    vendor: ['react', 'react-dom', 'lodash'],
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js',
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 20,
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'async',
          priority: 10,
          reuseExistingChunk: true,
        },
      },
    },
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
};

export default config;
```

### 5.2 Dynamic Import สำหรับ Feature Modules

```typescript
// ✅ Feature-based code splitting
type FeatureModule = {
  initialize: () => Promise<void>;
  cleanup: () => Promise<void>;
};

class FeatureManager {
  private loadedFeatures = new Map<string, FeatureModule>();

  async loadFeature(featureName: string): Promise<FeatureModule> {
    if (this.loadedFeatures.has(featureName)) {
      return this.loadedFeatures.get(featureName)!;
    }

    let module: FeatureModule;

    switch (featureName) {
      case 'analytics':
        const analyticsModule = await import(
          /* webpackChunkName: "feature-analytics" */
          './features/analytics'
        );
        module = analyticsModule.default;
        break;
      
      case 'chat':
        const chatModule = await import(
          /* webpackChunkName: "feature-chat" */
          './features/chat'
        );
        module = chatModule.default;
        break;
      
      default:
        throw new Error(`Unknown feature: ${featureName}`);
    }

    await module.initialize();
    this.loadedFeatures.set(featureName, module);
    return module;
  }

  async unloadFeature(featureName: string): Promise<void> {
    const feature = this.loadedFeatures.get(featureName);
    if (feature) {
      await feature.cleanup();
      this.loadedFeatures.delete(featureName);
    }
  }
}
```

---

## 6. Tree Shaking (การตัดโค้ดที่ไม่ใช้)

### 6.1 การเขียนโค้ดที่ Tree-shakeable

```typescript
// ❌ การ export แบบ default ทำให้ tree shaking ยาก
class Utils {
  static formatDate(date: Date): string {
    return date.toISOString();
  }

  static calculateAge(birthDate: Date): number {
    const today = new Date();
    const age = today.getFullYear() - birthDate.getFullYear();
    return age;
  }

  static generateId(): string {
    return Math.random().toString(36).substring(2);
  }
}

export default Utils;

// ✅ Named exports ช่วย tree shaking ได้ดีกว่า
export function formatDate(date: Date): string {
  return date.toISOString();
}

export function calculateAge(birthDate: Date): number {
  const today = new Date();
  return today.getFullYear() - birthDate.getFullYear();
}

export function generateId(): string {
  return Math.random().toString(36).substring(2);
}

// ✅ Pure functions ที่ไม่มี side effects
export const formatCurrency = (amount: number, currency = 'THB'): string => {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency,
  }).format(amount);
};

// ✅ tsconfig.json สำหรับ tree shaking
// {
//   "compilerOptions": {
//     "module": "ES2020",
//     "moduleResolution": "bundler",
//     "target": "ES2020"
//   }
// }
```

### 6.2 Side-effect-free Code

```typescript
// ✅ package.json ที่ระบุ sideEffects
// {
//   "sideEffects": false  // หรือระบุไฟล์ที่มี side effects
//   "sideEffects": ["./src/styles.css", "./src/polyfills.ts"]
// }

// ✅ Pure utility functions
export const arrayUtils = {
  unique: <T>(arr: T[]): T[] => [...new Set(arr)],
  flatten: <T>(arr: T[][]): T[] => arr.flat(),
  chunk: <T>(arr: T[], size: number): T[][] => {
    const chunks: T[][] = [];
    for (let i = 0; i < arr.length; i += size) {
      chunks.push(arr.slice(i, i + size));
    }
    return chunks;
  },
  groupBy: <T, K extends string | number>(
    arr: T[],
    key: (item: T) => K
  ): Record<K, T[]> => {
    return arr.reduce((groups, item) => {
      const groupKey = key(item);
      return {
        ...groups,
        [groupKey]: [...(groups[groupKey] ?? []), item],
      };
    }, {} as Record<K, T[]>);
  },
};

// Import เฉพาะที่ใช้
import { arrayUtils } from './utils';
const { unique, chunk } = arrayUtils;
```

---

## 7. Bundle Optimization (การเพิ่มประสิทธิภาพ Bundle)

### 7.1 การตั้งค่า tsconfig.json สำหรับ Production

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2020", "DOM"],
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "removeComments": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "importHelpers": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts", "**/*.test.ts"]
}
```

### 7.2 Bundle Analysis

```typescript
// scripts/analyze-bundle.ts
import { execSync } from 'child_process';
import path from 'path';
import fs from 'fs';

interface BundleStats {
  totalSize: number;
  gzippedSize: number;
  files: FileStats[];
}

interface FileStats {
  name: string;
  size: number;
  gzippedSize: number;
  percentage: number;
}

async function analyzeBundleSize(): Promise<BundleStats> {
  const distPath = path.resolve(process.cwd(), 'dist');
  
  if (!fs.existsSync(distPath)) {
    throw new Error('dist directory not found. Run build first.');
  }

  const files = fs.readdirSync(distPath)
    .filter(f => f.endsWith('.js'))
    .map(f => path.join(distPath, f));

  let totalSize = 0;
  let totalGzippedSize = 0;
  const fileStats: FileStats[] = [];

  for (const file of files) {
    const stats = fs.statSync(file);
    const size = stats.size;
    
    // จำลองขนาด gzip (ปกติจะใช้ zlib)
    const gzippedSize = Math.floor(size * 0.3);
    
    totalSize += size;
    totalGzippedSize += gzippedSize;
    
    fileStats.push({
      name: path.basename(file),
      size,
      gzippedSize,
      percentage: 0, // คำนวณหลัง
    });
  }

  const statsWithPercentage = fileStats.map(f => ({
    ...f,
    percentage: (f.size / totalSize) * 100,
  }));

  return {
    totalSize,
    gzippedSize: totalGzippedSize,
    files: statsWithPercentage,
  };
}

function formatBytes(bytes: number): string {
  if (bytes === 0) return '0 B';
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return `${(bytes / Math.pow(k, i)).toFixed(2)} ${sizes[i]}`;
}

// ใช้งาน
analyzeBundleSize().then(stats => {
  console.log('\n=== Bundle Analysis ===');
  console.log(`Total size: ${formatBytes(stats.totalSize)}`);
  console.log(`Gzipped: ${formatBytes(stats.gzippedSize)}\n`);
  
  stats.files.forEach(file => {
    console.log(
      `${file.name.padEnd(40)} ${formatBytes(file.size).padStart(10)} ` +
      `(${file.percentage.toFixed(1)}%)`
    );
  });
});
```

---

## 8. Memoization Patterns (รูปแบบ Memoization)

### 8.1 Simple Memoization

```typescript
// ✅ Basic memoization function
function memoize<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => TReturn
): (...args: TArgs) => TReturn {
  const cache = new Map<string, TReturn>();
  
  return function(...args: TArgs): TReturn {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// ตัวอย่างการใช้งาน
function expensiveCalculation(n: number): number {
  console.log(`Computing fibonacci(${n})...`);
  if (n <= 1) return n;
  return expensiveCalculation(n - 1) + expensiveCalculation(n - 2);
}

const memoizedFib = memoize(expensiveCalculation);
console.log(memoizedFib(10)); // คำนวณครั้งแรก
console.log(memoizedFib(10)); // ใช้ cache

// ✅ Memoization ด้วย LRU Cache
class LRUCache<K, V> {
  private cache: Map<K, V>;
  private maxSize: number;

  constructor(maxSize: number) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    
    // ย้าย item ไปท้าย (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // ลบ item แรก (least recently used)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }

  has(key: K): boolean {
    return this.cache.has(key);
  }

  clear(): void {
    this.cache.clear();
  }

  get size(): number {
    return this.cache.size;
  }
}

// ✅ Memoize ด้วย TTL (Time To Live)
interface CacheEntry<T> {
  value: T;
  expiresAt: number;
}

function memoizeWithTTL<TArgs extends unknown[], TReturn>(
  fn: (...args: TArgs) => TReturn,
  ttlMs: number
): (...args: TArgs) => TReturn {
  const cache = new Map<string, CacheEntry<TReturn>>();
  
  return function(...args: TArgs): TReturn {
    const key = JSON.stringify(args);
    const now = Date.now();
    const entry = cache.get(key);
    
    if (entry && entry.expiresAt > now) {
      return entry.value;
    }
    
    const result = fn(...args);
    cache.set(key, { value: result, expiresAt: now + ttlMs });
    return result;
  };
}
```

### 8.2 React useMemo Pattern (TypeScript)

```typescript
import { useMemo, useCallback, useState } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  rating: number;
}

interface FilterOptions {
  category: string;
  minPrice: number;
  maxPrice: number;
  minRating: number;
}

// ✅ useMemo สำหรับ expensive calculations
function useFilteredProducts(
  products: Product[],
  filters: FilterOptions
) {
  const filteredProducts = useMemo(() => {
    return products.filter(product => {
      const matchesCategory = 
        !filters.category || product.category === filters.category;
      const matchesPrice = 
        product.price >= filters.minPrice && 
        product.price <= filters.maxPrice;
      const matchesRating = product.rating >= filters.minRating;
      
      return matchesCategory && matchesPrice && matchesRating;
    });
  }, [products, filters]); // เฉพาะ recalculate เมื่อ dependencies เปลี่ยน

  const sortedProducts = useMemo(() => {
    return [...filteredProducts].sort((a, b) => b.rating - a.rating);
  }, [filteredProducts]);

  const categories = useMemo(() => {
    return [...new Set(products.map(p => p.category))].sort();
  }, [products]);

  return { filteredProducts, sortedProducts, categories };
}

// ✅ useCallback สำหรับ stable function references
function useProductActions(onUpdate: (product: Product) => void) {
  const handlePriceUpdate = useCallback(
    (productId: number, newPrice: number) => {
      console.log(`Updating price for product ${productId} to ${newPrice}`);
      // เรียก API update
    },
    [] // ไม่มี dependencies
  );

  const handleCategoryChange = useCallback(
    (productId: number, category: string) => {
      console.log(`Changing category for product ${productId}`);
    },
    []
  );

  return { handlePriceUpdate, handleCategoryChange };
}
```

---

## 9. Web Workers กับ TypeScript

### 9.1 การตั้งค่า Web Worker

```typescript
// worker.ts - ไฟล์ Worker
/// <reference lib="webworker" />

interface WorkerMessage<T = unknown> {
  type: string;
  id: string;
  data: T;
}

interface ComputeRequest {
  numbers: number[];
  operation: 'sum' | 'average' | 'max' | 'min' | 'sort';
}

interface ComputeResult {
  result: number | number[];
  duration: number;
}

self.onmessage = function(event: MessageEvent<WorkerMessage<ComputeRequest>>) {
  const { type, id, data } = event.data;
  const startTime = performance.now();
  
  let result: number | number[];
  
  switch (data.operation) {
    case 'sum':
      result = data.numbers.reduce((a, b) => a + b, 0);
      break;
    case 'average':
      result = data.numbers.reduce((a, b) => a + b, 0) / data.numbers.length;
      break;
    case 'max':
      result = Math.max(...data.numbers);
      break;
    case 'min':
      result = Math.min(...data.numbers);
      break;
    case 'sort':
      result = [...data.numbers].sort((a, b) => a - b);
      break;
    default:
      throw new Error(`Unknown operation: ${data.operation}`);
  }
  
  const duration = performance.now() - startTime;
  
  const response: WorkerMessage<ComputeResult> = {
    type: 'result',
    id,
    data: { result, duration },
  };
  
  self.postMessage(response);
};

// worker-manager.ts - จัดการ Workers
class WorkerPool {
  private workers: Worker[] = [];
  private queue: Array<{
    message: WorkerMessage<ComputeRequest>;
    resolve: (value: ComputeResult) => void;
    reject: (error: Error) => void;
  }> = [];
  private busyWorkers = new Set<Worker>();

  constructor(private readonly workerUrl: string, private poolSize: number) {
    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerUrl, { type: 'module' });
      worker.onmessage = this.handleMessage.bind(this, worker);
      worker.onerror = this.handleError.bind(this, worker);
      this.workers.push(worker);
    }
  }

  private pendingRequests = new Map<
    string,
    { resolve: (value: ComputeResult) => void; reject: (error: Error) => void }
  >();

  private handleMessage(worker: Worker, event: MessageEvent<WorkerMessage<ComputeResult>>) {
    const { id, data } = event.data;
    const pending = this.pendingRequests.get(id);
    
    if (pending) {
      pending.resolve(data);
      this.pendingRequests.delete(id);
    }
    
    this.busyWorkers.delete(worker);
    this.processQueue();
  }

  private handleError(worker: Worker, error: ErrorEvent) {
    console.error('Worker error:', error);
    this.busyWorkers.delete(worker);
    this.processQueue();
  }

  private processQueue() {
    if (this.queue.length === 0) return;
    
    const availableWorker = this.workers.find(w => !this.busyWorkers.has(w));
    if (!availableWorker) return;
    
    const { message, resolve, reject } = this.queue.shift()!;
    this.busyWorkers.add(availableWorker);
    this.pendingRequests.set(message.id, { resolve, reject });
    availableWorker.postMessage(message);
  }

  compute(request: ComputeRequest): Promise<ComputeResult> {
    return new Promise((resolve, reject) => {
      const id = Math.random().toString(36).substring(2);
      const message: WorkerMessage<ComputeRequest> = {
        type: 'compute',
        id,
        data: request,
      };
      
      const availableWorker = this.workers.find(w => !this.busyWorkers.has(w));
      
      if (availableWorker) {
        this.busyWorkers.add(availableWorker);
        this.pendingRequests.set(id, { resolve, reject });
        availableWorker.postMessage(message);
      } else {
        this.queue.push({ message, resolve, reject });
      }
    });
  }

  terminate() {
    this.workers.forEach(w => w.terminate());
    this.workers = [];
  }
}

// การใช้งาน
const pool = new WorkerPool('/worker.js', 4);

async function processLargeDataset(data: number[]): Promise<void> {
  const chunkSize = Math.ceil(data.length / 4);
  const chunks: number[][] = [];
  
  for (let i = 0; i < data.length; i += chunkSize) {
    chunks.push(data.slice(i, i + chunkSize));
  }
  
  const results = await Promise.all(
    chunks.map(chunk => pool.compute({ numbers: chunk, operation: 'sum' }))
  );
  
  const totalSum = results.reduce((sum, r) => sum + (r.result as number), 0);
  console.log('Total sum:', totalSum);
}
```

---

## 10. Node.js Performance

### 10.1 การใช้ Streams

```typescript
import { Transform, TransformCallback, pipeline } from 'stream';
import { createReadStream, createWriteStream } from 'fs';
import { promisify } from 'util';
import zlib from 'zlib';

const pipelineAsync = promisify(pipeline);

// ✅ Transform Stream สำหรับประมวลผล data ขนาดใหญ่
class JSONTransform extends Transform {
  private buffer = '';

  constructor() {
    super({ objectMode: true });
  }

  _transform(chunk: Buffer, encoding: string, callback: TransformCallback): void {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    
    // เก็บ line สุดท้ายที่อาจจะยังไม่สมบูรณ์
    this.buffer = lines.pop() ?? '';
    
    for (const line of lines) {
      if (line.trim()) {
        try {
          const parsed = JSON.parse(line);
          this.push(parsed);
        } catch {
          this.emit('error', new Error(`Invalid JSON: ${line}`));
        }
      }
    }
    
    callback();
  }

  _flush(callback: TransformCallback): void {
    if (this.buffer.trim()) {
      try {
        this.push(JSON.parse(this.buffer));
      } catch {
        this.emit('error', new Error(`Invalid JSON: ${this.buffer}`));
      }
    }
    callback();
  }
}

// ✅ ประมวลผลไฟล์ขนาดใหญ่ด้วย Stream
async function processLargeFile(
  inputPath: string,
  outputPath: string
): Promise<void> {
  const input = createReadStream(inputPath);
  const jsonTransform = new JSONTransform();
  const processTransform = new Transform({
    objectMode: true,
    transform(record: Record<string, unknown>, encoding, callback) {
      // ประมวลผล record
      const processed = {
        ...record,
        processed: true,
        timestamp: new Date().toISOString(),
      };
      callback(null, JSON.stringify(processed) + '\n');
    },
  });
  const gzip = zlib.createGzip();
  const output = createWriteStream(outputPath);

  await pipelineAsync(input, jsonTransform, processTransform, gzip, output);
  console.log('File processed successfully');
}
```

### 10.2 Clustering

```typescript
import cluster from 'cluster';
import { cpus } from 'os';
import http from 'http';

// ✅ Cluster module สำหรับ multi-core utilization
if (cluster.isPrimary) {
  const numCPUs = cpus().length;
  console.log(`Primary process ${process.pid} running`);
  console.log(`Starting ${numCPUs} workers...`);
  
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code})`);
    console.log('Starting new worker...');
    cluster.fork();
  });
  
  cluster.on('online', (worker) => {
    console.log(`Worker ${worker.process.pid} is online`);
  });
} else {
  // Worker process
  const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      message: 'Hello from TypeScript!',
      worker: process.pid,
      timestamp: new Date().toISOString(),
    }));
  });
  
  server.listen(3000, () => {
    console.log(`Worker ${process.pid} listening on port 3000`);
  });
}
```

---

## 11. Profiling TypeScript Applications

### 11.1 Performance Timing

```typescript
// ✅ Performance measurement utilities
class PerformanceTracker {
  private measurements = new Map<string, number[]>();

  async measure<T>(
    name: string,
    fn: () => Promise<T>
  ): Promise<{ result: T; duration: number }> {
    const start = performance.now();
    const result = await fn();
    const duration = performance.now() - start;
    
    const existing = this.measurements.get(name) ?? [];
    existing.push(duration);
    this.measurements.set(name, existing);
    
    return { result, duration };
  }

  measureSync<T>(
    name: string,
    fn: () => T
  ): { result: T; duration: number } {
    const start = performance.now();
    const result = fn();
    const duration = performance.now() - start;
    
    const existing = this.measurements.get(name) ?? [];
    existing.push(duration);
    this.measurements.set(name, existing);
    
    return { result, duration };
  }

  getStats(name: string): {
    count: number;
    total: number;
    average: number;
    min: number;
    max: number;
    p95: number;
  } | null {
    const times = this.measurements.get(name);
    if (!times || times.length === 0) return null;
    
    const sorted = [...times].sort((a, b) => a - b);
    const total = sorted.reduce((a, b) => a + b, 0);
    const p95Index = Math.floor(sorted.length * 0.95);
    
    return {
      count: sorted.length,
      total,
      average: total / sorted.length,
      min: sorted[0],
      max: sorted[sorted.length - 1],
      p95: sorted[p95Index],
    };
  }

  printReport(): void {
    console.log('\n=== Performance Report ===');
    for (const [name] of this.measurements) {
      const stats = this.getStats(name);
      if (stats) {
        console.log(`\n${name}:`);
        console.log(`  Count: ${stats.count}`);
        console.log(`  Average: ${stats.average.toFixed(2)}ms`);
        console.log(`  Min: ${stats.min.toFixed(2)}ms`);
        console.log(`  Max: ${stats.max.toFixed(2)}ms`);
        console.log(`  P95: ${stats.p95.toFixed(2)}ms`);
      }
    }
  }

  reset(): void {
    this.measurements.clear();
  }
}

// การใช้งาน
const tracker = new PerformanceTracker();

async function apiCall(url: string): Promise<unknown> {
  const { result, duration } = await tracker.measure('api-call', async () => {
    const response = await fetch(url);
    return response.json();
  });
  
  console.log(`API call took ${duration.toFixed(2)}ms`);
  return result;
}
```

---

## 12. Memory Management (การจัดการหน่วยความจำ)

### 12.1 WeakMap และ WeakRef

```typescript
// ✅ WeakMap สำหรับ caching ที่ไม่ prevent garbage collection
const cache = new WeakMap<object, unknown>();

function processObject<T extends object>(obj: T): unknown {
  if (cache.has(obj)) {
    return cache.get(obj);
  }
  
  const result = expensiveProcess(obj);
  cache.set(obj, result);
  return result;
}

function expensiveProcess(obj: object): unknown {
  // จำลองการประมวลผลที่ใช้เวลา
  return JSON.stringify(obj);
}

// ✅ WeakRef สำหรับ optional references
class ResourceManager {
  private resources = new Map<string, WeakRef<Resource>>();
  private registry = new FinalizationRegistry<string>((id) => {
    console.log(`Resource ${id} was garbage collected`);
    this.resources.delete(id);
  });

  add(id: string, resource: Resource): void {
    const ref = new WeakRef(resource);
    this.resources.set(id, ref);
    this.registry.register(resource, id);
  }

  get(id: string): Resource | undefined {
    return this.resources.get(id)?.deref();
  }

  has(id: string): boolean {
    const ref = this.resources.get(id);
    if (!ref) return false;
    return ref.deref() !== undefined;
  }
}

interface Resource {
  id: string;
  data: unknown;
  cleanup(): void;
}

// ✅ Memory-efficient data structures
class CircularBuffer<T> {
  private buffer: (T | undefined)[];
  private head = 0;
  private tail = 0;
  private count = 0;

  constructor(private readonly capacity: number) {
    this.buffer = new Array(capacity).fill(undefined);
  }

  push(item: T): void {
    if (this.count === this.capacity) {
      // เต็มแล้ว ลบ item เก่าสุด
      this.head = (this.head + 1) % this.capacity;
      this.count--;
    }
    
    this.buffer[this.tail] = item;
    this.tail = (this.tail + 1) % this.capacity;
    this.count++;
  }

  pop(): T | undefined {
    if (this.count === 0) return undefined;
    
    const item = this.buffer[this.head];
    this.buffer[this.head] = undefined;
    this.head = (this.head + 1) % this.capacity;
    this.count--;
    return item;
  }

  get size(): number {
    return this.count;
  }

  toArray(): T[] {
    const result: T[] = [];
    let index = this.head;
    
    for (let i = 0; i < this.count; i++) {
      result.push(this.buffer[index] as T);
      index = (index + 1) % this.capacity;
    }
    
    return result;
  }
}
```

---

## 13. Caching Strategies (กลยุทธ์การ Cache)

### 13.1 Multi-Level Cache

```typescript
// ✅ Multi-level caching system
interface CacheLayer<K, V> {
  get(key: K): Promise<V | null>;
  set(key: K, value: V, ttl?: number): Promise<void>;
  delete(key: K): Promise<void>;
  clear(): Promise<void>;
}

class MemoryCache<K, V> implements CacheLayer<K, V> {
  private store = new Map<K, { value: V; expiresAt: number }>();

  async get(key: K): Promise<V | null> {
    const entry = this.store.get(key);
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return null;
    }
    return entry.value;
  }

  async set(key: K, value: V, ttl = 60000): Promise<void> {
    this.store.set(key, {
      value,
      expiresAt: Date.now() + ttl,
    });
  }

  async delete(key: K): Promise<void> {
    this.store.delete(key);
  }

  async clear(): Promise<void> {
    this.store.clear();
  }
}

class MultiLevelCache<K extends string, V> {
  private layers: CacheLayer<K, V>[];

  constructor(layers: CacheLayer<K, V>[]) {
    this.layers = layers;
  }

  async get(key: K): Promise<V | null> {
    for (let i = 0; i < this.layers.length; i++) {
      const value = await this.layers[i].get(key);
      
      if (value !== null) {
        // Backfill ชั้น cache ที่ miss
        for (let j = 0; j < i; j++) {
          await this.layers[j].set(key, value);
        }
        return value;
      }
    }
    return null;
  }

  async set(key: K, value: V, ttl?: number): Promise<void> {
    await Promise.all(
      this.layers.map(layer => layer.set(key, value, ttl))
    );
  }

  async delete(key: K): Promise<void> {
    await Promise.all(
      this.layers.map(layer => layer.delete(key))
    );
  }

  async clear(): Promise<void> {
    await Promise.all(
      this.layers.map(layer => layer.clear())
    );
  }
}

// การใช้งาน
const l1Cache = new MemoryCache<string, unknown>(); // in-memory
// const l2Cache = new RedisCache(...); // distributed cache

const cache = new MultiLevelCache<string, unknown>([
  l1Cache,
  // l2Cache,
]);

// ✅ Cache-aside pattern
async function getUser(userId: string): Promise<{ id: string; name: string } | null> {
  const cacheKey = `user:${userId}`;
  const cached = await cache.get(cacheKey);
  
  if (cached) {
    return cached as { id: string; name: string };
  }
  
  // โหลดจาก database
  const user = await fetchUserFromDB(userId);
  
  if (user) {
    await cache.set(cacheKey, user, 5 * 60 * 1000); // 5 minutes TTL
  }
  
  return user;
}

async function fetchUserFromDB(userId: string): Promise<{ id: string; name: string } | null> {
  // จำลอง DB query
  return { id: userId, name: 'Test User' };
}
```

---

## 14. Database Query Optimization (การเพิ่มประสิทธิภาพ Database Query)

### 14.1 Query Builder Pattern

```typescript
// ✅ Type-safe query builder
type OrderDirection = 'ASC' | 'DESC';

interface QueryOptions<T> {
  select?: (keyof T)[];
  where?: Partial<Record<keyof T, unknown>>;
  orderBy?: { field: keyof T; direction: OrderDirection }[];
  limit?: number;
  offset?: number;
}

class QueryBuilder<T extends Record<string, unknown>> {
  private options: QueryOptions<T> = {};
  private tableName: string;

  constructor(tableName: string) {
    this.tableName = tableName;
  }

  select(...fields: (keyof T)[]): this {
    this.options.select = fields;
    return this;
  }

  where(conditions: Partial<Record<keyof T, unknown>>): this {
    this.options.where = { ...this.options.where, ...conditions };
    return this;
  }

  orderBy(field: keyof T, direction: OrderDirection = 'ASC'): this {
    if (!this.options.orderBy) this.options.orderBy = [];
    this.options.orderBy.push({ field, direction });
    return this;
  }

  limit(n: number): this {
    this.options.limit = n;
    return this;
  }

  offset(n: number): this {
    this.options.offset = n;
    return this;
  }

  build(): { sql: string; params: unknown[] } {
    const params: unknown[] = [];
    let paramIndex = 1;

    // SELECT clause
    const selectFields = this.options.select?.length
      ? this.options.select.map(f => String(f)).join(', ')
      : '*';

    let sql = `SELECT ${selectFields} FROM ${this.tableName}`;

    // WHERE clause
    if (this.options.where && Object.keys(this.options.where).length > 0) {
      const conditions = Object.entries(this.options.where).map(([key, value]) => {
        params.push(value);
        return `${key} = $${paramIndex++}`;
      });
      sql += ` WHERE ${conditions.join(' AND ')}`;
    }

    // ORDER BY clause
    if (this.options.orderBy?.length) {
      const orderClauses = this.options.orderBy.map(
        o => `${String(o.field)} ${o.direction}`
      );
      sql += ` ORDER BY ${orderClauses.join(', ')}`;
    }

    // LIMIT
    if (this.options.limit !== undefined) {
      sql += ` LIMIT ${this.options.limit}`;
    }

    // OFFSET
    if (this.options.offset !== undefined) {
      sql += ` OFFSET ${this.options.offset}`;
    }

    return { sql, params };
  }
}

// การใช้งาน
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  createdAt: Date;
}

const query = new QueryBuilder<User>('users')
  .select('id', 'name', 'email')
  .where({ age: 25 })
  .orderBy('createdAt', 'DESC')
  .limit(10)
  .offset(0)
  .build();

console.log(query.sql);
// SELECT id, name, email FROM users WHERE age = $1 ORDER BY createdAt DESC LIMIT 10 OFFSET 0
console.log(query.params); // [25]

// ✅ N+1 Query Prevention ด้วย DataLoader pattern
class DataLoader<K, V> {
  private batch: K[] = [];
  private callbacks = new Map<K, Array<(value: V | null) => void>>();
  private scheduled = false;

  constructor(private batchFn: (keys: K[]) => Promise<Map<K, V>>) {}

  load(key: K): Promise<V | null> {
    return new Promise((resolve) => {
      this.batch.push(key);
      
      const existing = this.callbacks.get(key) ?? [];
      existing.push(resolve);
      this.callbacks.set(key, existing);
      
      if (!this.scheduled) {
        this.scheduled = true;
        process.nextTick(() => this.dispatch());
      }
    });
  }

  private async dispatch(): Promise<void> {
    const keys = [...new Set(this.batch)];
    this.batch = [];
    this.scheduled = false;
    
    const results = await this.batchFn(keys);
    
    for (const key of keys) {
      const value = results.get(key) ?? null;
      const callbacks = this.callbacks.get(key) ?? [];
      
      for (const callback of callbacks) {
        callback(value);
      }
      
      this.callbacks.delete(key);
    }
  }
}

// การใช้งาน DataLoader
const userLoader = new DataLoader<number, User>(async (ids) => {
  console.log(`Batch loading ${ids.length} users...`);
  // query เดียวสำหรับทุก ids
  const users = await Promise.resolve(ids.map(id => ({
    id,
    name: `User ${id}`,
    email: `user${id}@example.com`,
    age: 25,
    createdAt: new Date(),
  })));
  
  return new Map(users.map(u => [u.id, u]));
});

// แทนที่จะทำ N queries
async function loadPosts(): Promise<void> {
  const posts = [
    { id: 1, userId: 1, title: 'Post 1' },
    { id: 2, userId: 2, title: 'Post 2' },
    { id: 3, userId: 1, title: 'Post 3' }, // same userId as post 1!
  ];
  
  // DataLoader จะ batch userId 1, 2 เป็น query เดียว
  const postsWithUsers = await Promise.all(
    posts.map(async post => ({
      ...post,
      user: await userLoader.load(post.userId), // batched!
    }))
  );
  
  console.log('Posts with users:', postsWithUsers);
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **TypeScript Performance Best Practices** - การเลือกใช้ Type ที่เหมาะสม, Const Assertions, Readonly Types
2. **Avoiding Performance Pitfalls** - หลีกเลี่ยง Union Types ที่ซับซ้อน, การสร้าง Object ซ้ำ
3. **Type-Level Performance** - Conditional Types ที่มีประสิทธิภาพ, Interface vs Type
4. **Lazy Loading** - Dynamic Imports, Lazy Initialization Pattern
5. **Code Splitting** - Webpack Configuration, Feature-based splitting
6. **Tree Shaking** - Named exports, Side-effect-free code
7. **Bundle Optimization** - tsconfig สำหรับ Production, Bundle Analysis
8. **Memoization Patterns** - Simple memoize, LRU Cache, TTL Cache
9. **Web Workers** - Worker Pool, Parallel Processing
10. **Node.js Performance** - Streams, Clustering
11. **Profiling** - Performance measurement utilities
12. **Memory Management** - WeakMap, WeakRef, Circular Buffer
13. **Caching Strategies** - Multi-level cache, Cache-aside pattern
14. **Database Query Optimization** - Query Builder, DataLoader pattern

การเพิ่มประสิทธิภาพเป็นกระบวนการต่อเนื่อง ควรใช้ profiling tools เพื่อระบุ bottlenecks จริงๆ ก่อนที่จะ optimize และควร measure ผลลัพธ์หลังจาก optimize แต่ละส่วน

---

*หัวข้อถัดไป: Part 49 - Security Best Practices*
