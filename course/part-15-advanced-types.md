# ส่วนที่ 15: Advanced Types

## บทนำ

ในบทนี้เราจะเรียนรู้ Advanced Types ที่ช่วยให้เราเขียน TypeScript ได้อย่างทรงพลังและยืดหยุ่น รวมถึง Recursive Types, Variadic Tuple Types และ Template Literal Types ในงานจริง

---

## 15.1 Readonly<T> และ Deep Readonly

### Readonly พื้นฐาน

```typescript
interface UserSettings {
  theme: "light" | "dark";
  language: string;
  notifications: boolean;
  fontSize: number;
}

// Readonly ทำให้ไม่สามารถแก้ไขค่าได้
const settings: Readonly<UserSettings> = {
  theme: "light",
  language: "th",
  notifications: true,
  fontSize: 16
};

// settings.theme = "dark"; // Error! Cannot assign to 'theme'

// แต่ shallow readonly - nested object ยังแก้ไขได้
interface AppConfig {
  server: {
    host: string;
    port: number;
  };
  database: {
    url: string;
    name: string;
  };
}

const config: Readonly<AppConfig> = {
  server: { host: "localhost", port: 3000 },
  database: { url: "mongodb://localhost", name: "mydb" }
};

// config.server = { host: "remote", port: 80 }; // Error!
config.server.host = "remote"; // OK! (shallow readonly)
```

### Deep Readonly

```typescript
// Deep Readonly - ทุก level ไม่สามารถแก้ไขได้
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

interface ComplexConfig {
  server: {
    host: string;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
  features: string[];
  limits: {
    maxUsers: number;
    maxRequests: number;
  };
}

type FrozenConfig = DeepReadonly<ComplexConfig>;

const frozenConfig: FrozenConfig = {
  server: {
    host: "localhost",
    ssl: { enabled: true, cert: "/path/to/cert" }
  },
  features: ["auth", "logging"],
  limits: { maxUsers: 1000, maxRequests: 10000 }
};

// frozenConfig.server.host = "remote"; // Error!
// frozenConfig.server.ssl.enabled = false; // Error!
// frozenConfig.features.push("new"); // Error!
// frozenConfig.features[0] = "other"; // Error!

// Immutable helper
function freeze<T>(obj: T): DeepReadonly<T> {
  // Deep freeze ใน runtime
  Object.freeze(obj);
  Object.keys(obj as object).forEach(key => {
    const value = (obj as Record<string, unknown>)[key];
    if (typeof value === "object" && value !== null) {
      freeze(value);
    }
  });
  return obj as DeepReadonly<T>;
}
```

### ReadonlyArray

```typescript
// ReadonlyArray ป้องกันการแก้ไข array
function processItems(items: ReadonlyArray<string>): string {
  // items.push("new"); // Error!
  // items.pop(); // Error!
  // items[0] = "changed"; // Error!
  
  // แต่ read operations ยังได้
  return items.join(", ");
}

// Readonly Tuple
type ReadonlyPoint = Readonly<[number, number]>;
// [number, number] ที่ไม่สามารถแก้ไขได้

const point: ReadonlyPoint = [0, 0];
// point[0] = 5; // Error!

// ใช้ as const สำหรับ literal types
const colors = ["red", "green", "blue"] as const;
// type: readonly ["red", "green", "blue"]
type Color = (typeof colors)[number]; // "red" | "green" | "blue"
```

---

## 15.2 Partial<T> และ Deep Partial

### Partial พื้นฐาน

```typescript
interface BlogPost {
  id: number;
  title: string;
  content: string;
  author: string;
  publishedAt: Date;
  tags: string[];
  featured: boolean;
}

// Partial สำหรับ update operations
type UpdateBlogPost = Partial<BlogPost>;

// ฟังก์ชัน update
async function updatePost(
  id: number,
  updates: Partial<Omit<BlogPost, "id">>
): Promise<BlogPost> {
  // ในงานจริงจะ fetch post แล้ว merge
  const existing: BlogPost = {
    id,
    title: "เดิม",
    content: "เนื้อหาเดิม",
    author: "สมชาย",
    publishedAt: new Date(),
    tags: [],
    featured: false
  };
  
  return { ...existing, ...updates };
}

// ใช้งาน
await updatePost(1, { title: "ชื่อใหม่", featured: true });
// ไม่ต้องส่ง field ทั้งหมด
```

### Deep Partial

```typescript
// Deep Partial สำหรับ nested objects
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface AppState {
  user: {
    profile: {
      name: string;
      avatar: string;
      bio: string;
    };
    preferences: {
      theme: "light" | "dark";
      language: string;
      notifications: {
        email: boolean;
        push: boolean;
        sms: boolean;
      };
    };
  };
  cart: {
    items: Array<{
      productId: string;
      quantity: number;
      price: number;
    }>;
    couponCode: string | null;
  };
}

type PartialAppState = DeepPartial<AppState>;

// ตอนนี้ทุก nested property เป็น optional
const stateUpdate: PartialAppState = {
  user: {
    preferences: {
      theme: "dark"
      // ไม่ต้องระบุ language, notifications
    }
    // ไม่ต้องระบุ profile
  }
  // ไม่ต้องระบุ cart
};

// merge function
function mergeState<T>(current: T, updates: DeepPartial<T>): T {
  if (typeof updates !== "object" || updates === null) {
    return updates as T;
  }
  
  const result = { ...current } as Record<string, unknown>;
  
  for (const key in updates) {
    const updateValue = (updates as Record<string, unknown>)[key];
    const currentValue = (current as Record<string, unknown>)[key];
    
    if (typeof updateValue === "object" && updateValue !== null && 
        typeof currentValue === "object" && currentValue !== null) {
      result[key] = mergeState(currentValue, updateValue as DeepPartial<typeof currentValue>);
    } else if (updateValue !== undefined) {
      result[key] = updateValue;
    }
  }
  
  return result as T;
}
```

---

## 15.3 Required<T>

### Required พื้นฐาน

```typescript
interface UserRegistration {
  name: string;
  email: string;
  password: string;
  phone?: string;
  bio?: string;
  avatar?: string;
  dateOfBirth?: Date;
}

// Required ทำให้ทุก property เป็น required
type CompleteUserProfile = Required<UserRegistration>;
// { name: string; email: string; password: string; phone: string; bio: string; ... }

// ใช้ Required เพื่อกำหนดว่าต้องกรอกข้อมูลครบ
function validateCompleteProfile(profile: Required<UserRegistration>): boolean {
  return Object.values(profile).every(v => v !== null && v !== undefined && v !== "");
}
```

### Required กับ Specific Fields

```typescript
// Required บาง field เท่านั้น
type RequireFields<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

interface ProductDraft {
  name?: string;
  description?: string;
  price?: number;
  category?: string;
  images?: string[];
  stock?: number;
}

// ต้องมี name และ price แต่อื่นๆ optional
type ProductForPublish = RequireFields<ProductDraft, "name" | "price">;

function publishProduct(product: ProductForPublish): void {
  // product.name และ product.price มีค่าแน่นอน
  console.log(`เผยแพร่สินค้า: ${product.name} ราคา ${product.price}`);
}

// สร้าง partial ที่มีบาง field required
type PartialWithRequired<T, K extends keyof T> = Partial<T> & Required<Pick<T, K>>;

interface FormData {
  firstName: string;
  lastName: string;
  email: string;
  phone?: string;
  address?: string;
}

// ต้องมี firstName, lastName, email
type RequiredFormData = PartialWithRequired<FormData, "firstName" | "lastName" | "email">;
```

---

## 15.4 Record<K, V>

### Record พื้นฐาน

```typescript
// Record สำหรับ dictionary-like objects
type CurrencyCode = "THB" | "USD" | "EUR" | "JPY" | "GBP";

interface CurrencyInfo {
  name: string;
  symbol: string;
  decimalPlaces: number;
}

const currencies: Record<CurrencyCode, CurrencyInfo> = {
  THB: { name: "Thai Baht", symbol: "฿", decimalPlaces: 2 },
  USD: { name: "US Dollar", symbol: "$", decimalPlaces: 2 },
  EUR: { name: "Euro", symbol: "€", decimalPlaces: 2 },
  JPY: { name: "Japanese Yen", symbol: "¥", decimalPlaces: 0 },
  GBP: { name: "British Pound", symbol: "£", decimalPlaces: 2 }
};

function formatCurrency(amount: number, currency: CurrencyCode): string {
  const info = currencies[currency];
  return `${info.symbol}${amount.toFixed(info.decimalPlaces)}`;
}

console.log(formatCurrency(1500, "THB")); // ฿1500.00
console.log(formatCurrency(100, "JPY"));  // ¥100
```

### Record ซับซ้อน

```typescript
// Permission system
type Permission = "read" | "write" | "delete" | "admin";
type Resource = "users" | "products" | "orders" | "reports";

type PermissionMatrix = Record<Permission, Record<Resource, boolean>>;

const adminPermissions: PermissionMatrix = {
  read: { users: true, products: true, orders: true, reports: true },
  write: { users: true, products: true, orders: true, reports: false },
  delete: { users: false, products: true, orders: false, reports: false },
  admin: { users: true, products: false, orders: false, reports: false }
};

function hasPermission(
  matrix: PermissionMatrix,
  permission: Permission,
  resource: Resource
): boolean {
  return matrix[permission][resource];
}

console.log(hasPermission(adminPermissions, "write", "products")); // true
console.log(hasPermission(adminPermissions, "delete", "users"));   // false

// Dynamic Record
type StoreMap<T> = Record<string, T>;

function createStore<T>(): {
  get: (key: string) => T | undefined;
  set: (key: string, value: T) => void;
  delete: (key: string) => boolean;
  keys: () => string[];
} {
  const store: StoreMap<T> = {};
  
  return {
    get: (key) => store[key],
    set: (key, value) => { store[key] = value; },
    delete: (key) => {
      if (key in store) {
        delete store[key];
        return true;
      }
      return false;
    },
    keys: () => Object.keys(store)
  };
}
```

---

## 15.5 Pick<T, K> และ Omit<T, K>

### Pick และ Omit ขั้นสูง

```typescript
interface FullProduct {
  id: number;
  sku: string;
  name: string;
  description: string;
  price: number;
  costPrice: number;    // ข้อมูลภายใน
  margin: number;       // ข้อมูลภายใน
  stock: number;
  category: string;
  tags: string[];
  images: string[];
  weight: number;
  dimensions: { w: number; h: number; d: number };
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

// สำหรับ public API - ตัดข้อมูลภายในออก
type PublicProduct = Omit<FullProduct, "costPrice" | "margin" | "deletedAt">;

// สำหรับ list view - เฉพาะ field ที่จำเป็น
type ProductListItem = Pick<FullProduct, "id" | "name" | "price" | "images" | "category">;

// สำหรับ cart
type CartProduct = Pick<FullProduct, "id" | "name" | "price" | "weight" | "stock">;

// สำหรับ admin
type AdminProduct = Omit<FullProduct, "deletedAt">;

// Recursive Pick
type DeepPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// ใช้งาน
function formatForList(product: FullProduct): ProductListItem {
  const { id, name, price, images, category } = product;
  return { id, name, price, images, category };
}
```

### Nested Pick/Omit

```typescript
// Helper type สำหรับ nested operations
type PickNested<T, K1 extends keyof T, K2 extends keyof T[K1]> = {
  [P in K1]: Pick<T[P], K2>;
};

interface CompanyEmployee {
  id: number;
  name: string;
  department: {
    id: number;
    name: string;
    budget: number;     // sensitive
    headCount: number;
  };
  salary: number;       // sensitive
  bankAccount: string;  // sensitive
}

// ตัด sensitive info ออก
type SafeEmployee = Omit<CompanyEmployee, "salary" | "bankAccount"> & {
  department: Omit<CompanyEmployee["department"], "budget">;
};

function getSafeEmployee(employee: CompanyEmployee): SafeEmployee {
  const { salary, bankAccount, department, ...rest } = employee;
  const { budget, ...safeDept } = department;
  return { ...rest, department: safeDept };
}
```

---

## 15.6 Exclude<T, U> และ Extract<T, U>

### ตัวอย่างที่ซับซ้อน

```typescript
// ฟิลเตอร์ union types
type AllEvents =
  | "click"
  | "focus"
  | "blur"
  | "change"
  | "input"
  | "submit"
  | "keydown"
  | "keyup"
  | "mouseenter"
  | "mouseleave";

type FormEvents = Extract<AllEvents, "change" | "input" | "submit" | "focus" | "blur">;
// "change" | "input" | "submit" | "focus" | "blur"

type MouseEvents = Extract<AllEvents, `mouse${string}`>;
// "mouseenter" | "mouseleave"

type KeyboardEvents = Extract<AllEvents, `key${string}`>;
// "keydown" | "keyup"

type NonFormEvents = Exclude<AllEvents, FormEvents>;
// "click" | "keydown" | "keyup" | "mouseenter" | "mouseleave"

// ใช้ Exclude สำหรับ permission control
type AdminPermissions = "read" | "write" | "delete" | "admin" | "superadmin";
type UserPermissions = Exclude<AdminPermissions, "admin" | "superadmin">;
// "read" | "write" | "delete"

// ใช้ Extract สำหรับ type matching
type StringKeys<T> = Extract<keyof T, string>;
type NumberKeys<T> = Extract<keyof T, number>;

interface Mixed {
  name: string;
  0: number;
  1: boolean;
  age: number;
}

type MixedStringKeys = StringKeys<Mixed>; // "name" | "age"
type MixedNumberKeys = NumberKeys<Mixed>; // 0 | 1
```

---

## 15.7 NonNullable<T>

### NonNullable ในงานจริง

```typescript
// ลบ null และ undefined ออกจาก type
type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>; // User

// ฟังก์ชัน filter null values
function filterNullable<T>(arr: (T | null | undefined)[]): T[] {
  return arr.filter((item): item is T => item != null);
}

const maybeUsers: (User | null | undefined)[] = [
  { id: 1, name: "สมชาย", email: "a@b.com" },
  null,
  { id: 2, name: "สมหญิง", email: "b@c.com" },
  undefined
];

const definiteUsers = filterNullable(maybeUsers); // User[]
console.log(definiteUsers.length); // 2

// NonNullable กับ Record
type NullableRecord = Record<string, string | null | undefined>;
type NonNullableRecord = { [K in keyof NullableRecord]: NonNullable<NullableRecord[K]> };

// ล้าง null values ออกจาก object
function removeNullValues<T extends object>(
  obj: T
): { [K in keyof T]: NonNullable<T[K]> } {
  const result = {} as { [K in keyof T]: NonNullable<T[K]> };
  
  for (const key in obj) {
    if (obj[key] != null) {
      result[key] = obj[key] as NonNullable<T[typeof key]>;
    }
  }
  
  return result;
}
```

---

## 15.8 ReturnType<T> และ Parameters<T>

### การใช้งานขั้นสูง

```typescript
// สร้าง type จาก existing functions
const apiCalls = {
  getUser: async (id: number) => ({ id, name: "สมชาย", email: "a@b.com" }),
  getProduct: async (id: number) => ({ id, name: "สินค้า", price: 100 }),
  createOrder: async (userId: number, items: string[]) => ({ orderId: "ORD-001" })
};

// ดึง return types
type GetUserResult = Awaited<ReturnType<typeof apiCalls.getUser>>;
// { id: number; name: string; email: string; }

type GetProductResult = Awaited<ReturnType<typeof apiCalls.getProduct>>;
// { id: number; name: string; price: number; }

// ดึง parameter types
type GetUserParams = Parameters<typeof apiCalls.getUser>;
// [id: number]

type CreateOrderParams = Parameters<typeof apiCalls.createOrder>;
// [userId: number, items: string[]]

// ใช้สำหรับ mock functions
type MockFunction<T extends (...args: unknown[]) => unknown> = (
  ...args: Parameters<T>
) => ReturnType<T>;

// Memoization ที่ type-safe
function memoize<T extends (...args: unknown[]) => unknown>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();
  
  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const result = fn(...args) as ReturnType<T>;
    cache.set(key, result);
    return result;
  }) as T;
}

function expensiveCalc(n: number): number {
  console.log(`คำนวณ ${n}...`);
  return n * n;
}

const memoizedCalc = memoize(expensiveCalc);
console.log(memoizedCalc(5)); // คำนวณ 5... 25
console.log(memoizedCalc(5)); // 25 (จาก cache)
```

---

## 15.9 Template Literal Types - Real World Patterns

### API Route Generation

```typescript
// สร้าง type-safe API routes
type ApiVersion = "v1" | "v2" | "v3";
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type ResourceName = "users" | "products" | "orders" | "payments";

type ApiRoute = `/${ApiVersion}/${ResourceName}`;
type ApiRouteWithId = `${ApiRoute}/${number}`;

// ฟังก์ชัน type-safe fetch
async function apiFetch<T>(
  method: HttpMethod,
  route: ApiRoute | ApiRouteWithId,
  body?: unknown
): Promise<T> {
  const response = await fetch(route, {
    method,
    headers: { "Content-Type": "application/json" },
    body: body ? JSON.stringify(body) : undefined
  });
  return response.json();
}

// CSS Class Generation
type Breakpoint = "sm" | "md" | "lg" | "xl" | "2xl";
type SpacingSize = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 8 | 10 | 12 | 16 | 20 | 24;

type PaddingClass = `p-${SpacingSize}` | `px-${SpacingSize}` | `py-${SpacingSize}`;
type MarginClass = `m-${SpacingSize}` | `mx-${SpacingSize}` | `my-${SpacingSize}`;
type ResponsiveClass<T extends string> = T | `${Breakpoint}:${T}`;

type UtilityClass = ResponsiveClass<PaddingClass | MarginClass>;

function buildClassName(...classes: UtilityClass[]): string {
  return classes.join(" ");
}

const className = buildClassName("p-4", "md:p-8", "mx-2", "lg:mx-0");
```

### Event System ด้วย Template Literals

```typescript
// Type-safe event names
type EntityName = "user" | "product" | "order" | "payment";
type EventVerb = "created" | "updated" | "deleted" | "published";

type EntityEvent = `${EntityName}:${EventVerb}`;
// "user:created" | "user:updated" | ... (16 combinations)

type EventPayloads = {
  "user:created": { id: number; name: string; email: string };
  "user:updated": { id: number; changes: Partial<{ name: string; email: string }> };
  "user:deleted": { id: number; deletedBy: string };
  "product:created": { id: number; name: string; price: number };
  "product:updated": { id: number; changes: Partial<{ name: string; price: number }> };
  "product:deleted": { id: number };
  "order:created": { orderId: string; userId: number; total: number };
  "order:updated": { orderId: string; status: string };
};

class TypedEventBus {
  private handlers = new Map<string, Set<(payload: unknown) => void>>();

  on<E extends keyof EventPayloads>(
    event: E,
    handler: (payload: EventPayloads[E]) => void
  ): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler as (payload: unknown) => void);
    
    return () => {
      this.handlers.get(event)?.delete(handler as (payload: unknown) => void);
    };
  }

  emit<E extends keyof EventPayloads>(event: E, payload: EventPayloads[E]): void {
    this.handlers.get(event)?.forEach(handler => handler(payload));
  }
}

const bus = new TypedEventBus();

bus.on("user:created", (payload) => {
  // payload มี type: { id: number; name: string; email: string }
  console.log(`ผู้ใช้ใหม่: ${payload.name} (${payload.email})`);
});

bus.emit("user:created", { id: 1, name: "สมชาย", email: "somchai@example.com" });
```

---

## 15.10 Recursive Types

### Recursive Type พื้นฐาน

```typescript
// Tree structure
interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[];
}

// สร้าง tree
function createNode<T>(value: T, children: TreeNode<T>[] = []): TreeNode<T> {
  return { value, children };
}

// traverse tree
function traverseTree<T>(
  node: TreeNode<T>,
  callback: (value: T, depth: number) => void,
  depth = 0
): void {
  callback(node.value, depth);
  node.children.forEach(child => traverseTree(child, callback, depth + 1));
}

// ตัวอย่าง: Category tree
interface Category {
  id: number;
  name: string;
}

const categoryTree = createNode<Category>(
  { id: 1, name: "ทั้งหมด" },
  [
    createNode<Category>({ id: 2, name: "อิเล็กทรอนิกส์" }, [
      createNode<Category>({ id: 4, name: "โทรศัพท์" }),
      createNode<Category>({ id: 5, name: "แล็ปท็อป" })
    ]),
    createNode<Category>({ id: 3, name: "เสื้อผ้า" }, [
      createNode<Category>({ id: 6, name: "เสื้อ" }),
      createNode<Category>({ id: 7, name: "กางเกง" })
    ])
  ]
);

traverseTree(categoryTree, (cat, depth) => {
  console.log("  ".repeat(depth) + cat.name);
});
```

### JSON Type (Recursive)

```typescript
// Recursive type สำหรับ JSON
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

// ฟังก์ชัน type-safe JSON operations
function getJSONPath(data: JSONValue, path: string[]): JSONValue | undefined {
  if (path.length === 0) return data;
  
  if (typeof data !== "object" || data === null) return undefined;
  
  const [first, ...rest] = path;
  
  if (Array.isArray(data)) {
    const index = parseInt(first);
    if (isNaN(index)) return undefined;
    return getJSONPath(data[index], rest);
  }
  
  return getJSONPath(data[first], rest);
}

const jsonData: JSONValue = {
  users: [
    { name: "สมชาย", age: 25, hobbies: ["อ่านหนังสือ", "เล่นกีฬา"] },
    { name: "สมหญิง", age: 30 }
  ],
  total: 2
};

const firstUserName = getJSONPath(jsonData, ["users", "0", "name"]);
// "สมชาย"

const hobby = getJSONPath(jsonData, ["users", "0", "hobbies", "1"]);
// "เล่นกีฬา"
```

### Deeply Nested Type Operations

```typescript
// Path type - สร้าง all possible paths ใน object
type Paths<T, Prefix extends string = ""> = {
  [K in keyof T & string]: T[K] extends object
    ? K | `${K}.${Paths<T[K]>}`
    : K;
}[keyof T & string];

interface UserData {
  id: number;
  name: string;
  address: {
    street: string;
    city: string;
    country: string;
  };
}

type UserPaths = Paths<UserData>;
// "id" | "name" | "address" | "address.street" | "address.city" | "address.country"

// PathValue - ดึง type ณ path นั้น
type PathValue<T, P extends string> =
  P extends `${infer K}.${infer Rest}`
    ? K extends keyof T
      ? PathValue<T[K], Rest>
      : never
    : P extends keyof T
    ? T[P]
    : never;

type CityType = PathValue<UserData, "address.city">; // string
type NameType = PathValue<UserData, "name">; // string
```

---

## 15.11 Variadic Tuple Types

### Tuple Types พื้นฐาน

```typescript
// Tuple - array ที่กำหนด length และ types ชัดเจน
type Pair<T, U> = [T, U];
type Triple<T, U, V> = [T, U, V];

const pair: Pair<string, number> = ["hello", 42];
const triple: Triple<string, number, boolean> = ["yes", 1, true];

// Variadic Tuple - tuple ที่ยืดหยุ่น
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];

type AB = Concat<[string, number], [boolean, Date]>;
// [string, number, boolean, Date]

// Prepend/Append
type Prepend<T, Arr extends unknown[]> = [T, ...Arr];
type Append<Arr extends unknown[], T> = [...Arr, T];

type PrependString = Prepend<string, [number, boolean]>; // [string, number, boolean]
type AppendDate = Append<[string, number], Date>;         // [string, number, Date]
```

### Variadic Tuple ใน Functions

```typescript
// ฟังก์ชัน curry ที่ type-safe
function curry<T extends unknown[], R>(
  fn: (...args: T) => R,
  ...partialArgs: Partial<T>
): (...remainingArgs: T extends [...(typeof partialArgs), ...infer Rest] ? Rest : never) => R {
  return (...remainingArgs) => fn(...([...partialArgs, ...remainingArgs] as T));
}

// ฟังก์ชัน compose
function compose<T>(...fns: Array<(x: T) => T>): (x: T) => T {
  return (x: T) => fns.reduceRight((acc, fn) => fn(acc), x);
}

const processString = compose<string>(
  s => s.toUpperCase(),
  s => s.trim(),
  s => s.replace(/\s+/g, " ")
);

console.log(processString("  hello   world  ")); // "HELLO WORLD"

// Zip หลาย arrays
function zipAll<T extends unknown[][]>(
  ...arrays: { [K in keyof T]: T[K] extends unknown[] ? T[K] : never }
): { [K in keyof T]: T[K] extends (infer U)[] ? U : never }[] {
  const minLength = Math.min(...arrays.map(a => a.length));
  return Array.from({ length: minLength }, (_, i) =>
    arrays.map(arr => arr[i]) as { [K in keyof T]: T[K] extends (infer U)[] ? U : never }
  );
}

const names = ["สมชาย", "สมหญิง", "สมปอง"];
const ages = [25, 30, 28];
const scores = [90, 85, 95];

const zipped = zipAll(names, ages, scores);
// [string, number, number][]
```

---

## 15.12 Advanced Patterns

### Builder Pattern สำหรับ Complex Types

```typescript
// Type-safe query builder
type WhereCondition<T> = {
  [K in keyof T]?: T[K] | {
    eq?: T[K];
    ne?: T[K];
    gt?: T[K] extends number ? number : never;
    lt?: T[K] extends number ? number : never;
    gte?: T[K] extends number ? number : never;
    lte?: T[K] extends number ? number : never;
    in?: T[K][];
    like?: T[K] extends string ? string : never;
  };
};

class QueryBuilder<T extends object> {
  private conditions: WhereCondition<T> = {};
  private orderByField: keyof T | null = null;
  private orderDirection: "asc" | "desc" = "asc";
  private limitValue: number | null = null;
  private offsetValue: number = 0;

  where(conditions: WhereCondition<T>): this {
    this.conditions = { ...this.conditions, ...conditions };
    return this;
  }

  orderBy(field: keyof T, direction: "asc" | "desc" = "asc"): this {
    this.orderByField = field;
    this.orderDirection = direction;
    return this;
  }

  limit(n: number): this {
    this.limitValue = n;
    return this;
  }

  offset(n: number): this {
    this.offsetValue = n;
    return this;
  }

  build(): {
    where: WhereCondition<T>;
    orderBy: { field: keyof T; direction: "asc" | "desc" } | null;
    limit: number | null;
    offset: number;
  } {
    return {
      where: this.conditions,
      orderBy: this.orderByField
        ? { field: this.orderByField, direction: this.orderDirection }
        : null,
      limit: this.limitValue,
      offset: this.offsetValue
    };
  }
}

interface UserQuery {
  name: string;
  age: number;
  email: string;
  active: boolean;
}

const query = new QueryBuilder<UserQuery>()
  .where({ age: { gte: 18, lte: 65 } })
  .where({ active: true })
  .orderBy("name", "asc")
  .limit(10)
  .offset(20)
  .build();
```

### Opaque Types (Branded Types)

```typescript
// Branded types เพื่อป้องกันการสับสน
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type ProductId = Brand<number, "ProductId">;
type OrderId = Brand<string, "OrderId">;
type Email = Brand<string, "Email">;

// Helper functions ในการสร้าง branded types
function createUserId(id: number): UserId {
  return id as UserId;
}

function createProductId(id: number): ProductId {
  return id as ProductId;
}

function createOrderId(id: string): OrderId {
  return id as OrderId;
}

function createEmail(email: string): Email {
  if (!email.includes("@")) {
    throw new Error("Invalid email");
  }
  return email as Email;
}

// ฟังก์ชันที่ type-safe
async function getUser(id: UserId): Promise<{ id: UserId; name: string }> {
  return { id, name: "สมชาย" };
}

async function getProduct(id: ProductId): Promise<{ id: ProductId; name: string }> {
  return { id, name: "สินค้า" };
}

const userId = createUserId(1);
const productId = createProductId(1);

// ไม่สามารถสับสน UserId กับ ProductId ได้แม้ทั้งคู่เป็น number
getUser(userId);    // OK
getProduct(productId); // OK
// getUser(productId); // Error! TypeScript จับได้
```

### Phantom Types

```typescript
// Phantom types สำหรับ state management
type State = "idle" | "loading" | "success" | "error";

interface AsyncValue<T, S extends State = "idle"> {
  _state: S;
  data: S extends "success" ? T : never;
  error: S extends "error" ? Error : never;
}

type IdleValue<T> = AsyncValue<T, "idle">;
type LoadingValue<T> = AsyncValue<T, "loading">;
type SuccessValue<T> = AsyncValue<T, "success">;
type ErrorValue<T> = AsyncValue<T, "error">;

// ฟังก์ชันที่ return type ตาม state
function createIdle<T>(): IdleValue<T> {
  return { _state: "idle" } as IdleValue<T>;
}

function createLoading<T>(): LoadingValue<T> {
  return { _state: "loading" } as LoadingValue<T>;
}

function createSuccess<T>(data: T): SuccessValue<T> {
  return { _state: "success", data } as SuccessValue<T>;
}

function createError<T>(error: Error): ErrorValue<T> {
  return { _state: "error", error } as ErrorValue<T>;
}
```

---

## 15.13 Type-Level Programming

### Fibonacci ใน Type System

```typescript
// นับด้วย TypeScript type system (สำหรับ illustrative purposes)
type BuildTuple<N extends number, T extends unknown[] = []> =
  T["length"] extends N ? T : BuildTuple<N, [...T, unknown]>;

type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"] & number;

type Subtract<A extends number, B extends number> =
  BuildTuple<A> extends [...BuildTuple<B>, ...infer Rest]
    ? Rest["length"] & number
    : never;

// ใช้ในงานจริง: ตรวจสอบ array length
type HasAtLeastOneElement<T extends unknown[]> = T extends [unknown, ...unknown[]] ? true : false;

type EmptyArray = HasAtLeastOneElement<[]>;        // false
type OneElement = HasAtLeastOneElement<[string]>;  // true
type MultipleElements = HasAtLeastOneElement<[string, number]>; // true
```

### Infer Pattern ขั้นสูง

```typescript
// Extract type จาก complex generic
type UnpackPromise<T> = T extends Promise<infer U> ? UnpackPromise<U> : T;
type UnpackArray<T> = T extends (infer U)[] ? UnpackArray<U> : T;
type UnpackBoth<T> = T extends Promise<infer U>
  ? UnpackBoth<U>
  : T extends (infer U)[]
  ? UnpackBoth<U>
  : T;

type A = UnpackBoth<Promise<string[]>>; // string
type B = UnpackBoth<Promise<Promise<number[][]>>>; // number

// Function overload types
interface Overloaded {
  (x: string): string;
  (x: number): number;
  (x: boolean): boolean;
}

// ดึง overload signatures (advanced)
type OverloadToUnion<T> = T extends {
  (...args: infer A1): infer R1;
  (...args: infer A2): infer R2;
  (...args: infer A3): infer R3;
} ? [(...args: A1) => R1, (...args: A2) => R2, (...args: A3) => R3] : never;
```

---

## 15.14 Integration Example: Type-Safe ORM

```typescript
// Type-safe ORM-like interface
type ColumnType = "string" | "number" | "boolean" | "date" | "json";

interface ColumnDefinition {
  type: ColumnType;
  nullable?: boolean;
  default?: unknown;
}

type SchemaDefinition = Record<string, ColumnDefinition>;

type InferSchemaType<T extends SchemaDefinition> = {
  [K in keyof T]: T[K]["nullable"] extends true
    ? InferColumnType<T[K]["type"]> | null
    : InferColumnType<T[K]["type"]>;
};

type InferColumnType<T extends ColumnType> =
  T extends "string" ? string :
  T extends "number" ? number :
  T extends "boolean" ? boolean :
  T extends "date" ? Date :
  T extends "json" ? Record<string, unknown> :
  never;

// Schema ของตาราง
const userSchema = {
  id: { type: "number" as const },
  name: { type: "string" as const },
  email: { type: "string" as const },
  age: { type: "number" as const, nullable: true },
  isActive: { type: "boolean" as const, default: true },
  metadata: { type: "json" as const, nullable: true },
  createdAt: { type: "date" as const }
} satisfies SchemaDefinition;

type UserRow = InferSchemaType<typeof userSchema>;
// {
//   id: number;
//   name: string;
//   email: string;
//   age: number | null;
//   isActive: boolean;
//   metadata: Record<string, unknown> | null;
//   createdAt: Date;
// }

// Type-safe model class
class Model<Schema extends SchemaDefinition> {
  constructor(private schema: Schema) {}

  create(data: InferSchemaType<Schema>): InferSchemaType<Schema> {
    return data;
  }

  find(predicate: (row: InferSchemaType<Schema>) => boolean): InferSchemaType<Schema>[] {
    return []; // stub
  }

  update(
    id: number,
    updates: Partial<Omit<InferSchemaType<Schema>, "id">>
  ): InferSchemaType<Schema> | null {
    return null; // stub
  }
}

const UserModel = new Model(userSchema);

const newUser = UserModel.create({
  id: 1,
  name: "สมชาย",
  email: "somchai@example.com",
  age: null,
  isActive: true,
  metadata: null,
  createdAt: new Date()
});
// TypeScript ตรวจสอบ type ของ newUser อัตโนมัติ
```

---

## สรุปบทนี้

ในบทนี้เราได้เรียนรู้:

- **Readonly<T>** และ Deep Readonly ป้องกันการแก้ไขข้อมูล
- **Partial<T>** และ Deep Partial ทำ properties เป็น optional
- **Required<T>** บังคับทุก property ให้มีค่า
- **Record<K, V>** สร้าง dictionary-like types
- **Pick<T, K>** และ **Omit<T, K>** เลือก/ตัด properties
- **Exclude<T, U>** และ **Extract<T, U>** กรอง union types
- **NonNullable<T>** ลบ null/undefined
- **ReturnType<T>** และ **Parameters<T>** ดึง types จาก functions
- **Template Literal Types** สร้าง string types ที่ซับซ้อน
- **Recursive Types** สำหรับ nested/recursive data structures
- **Variadic Tuple Types** สำหรับ flexible array types
- **Branded Types** ป้องกันการสับสนระหว่าง types เดียวกัน

---

## แบบฝึกหัดท้ายบท

1. สร้าง `DeepFreeze<T>` utility type ที่ทำ readonly ทุก level และ implement runtime function
2. สร้าง `Flatten<T>` type ที่ flatten nested object เป็น flat object พร้อม dot-notation keys
3. สร้าง type-safe router ที่ใช้ Template Literal Types สำหรับ route params
4. Implement `Result<T, E>` type พร้อม `map`, `flatMap`, `fold` methods ที่ fully type-safe
5. สร้าง Schema validation library ขนาดเล็กที่ใช้ Recursive Types และ Conditional Types

---

## สรุปรายวิชา (Part 12-15)

เราได้เรียนรู้ TypeScript ขั้นสูงครอบคลุม:

**Part 12 - Generics Basics**: การสร้างโค้ดที่ยืดหยุ่น type-safe
**Part 13 - Advanced Generics**: keyof, typeof, Conditional Types, Mapped Types
**Part 14 - Type Guards**: Narrowing, Type Predicates, Exhaustive Checking
**Part 15 - Advanced Types**: Utility Types, Recursive Types, Template Literals

ทักษะเหล่านี้จะช่วยให้เขียน TypeScript ได้อย่างมีประสิทธิภาพสูงสุด ทั้ง maintainable และ bug-free
