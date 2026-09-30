# Part 60: TypeScript 5.x Features (ฟีเจอร์ TypeScript เวอร์ชัน 5.x)

## บทนำ

TypeScript 5.x นำเสนอฟีเจอร์ใหม่ๆ ที่น่าสนใจมากมาย ในบทนี้เราจะสำรวจฟีเจอร์สำคัญของ TypeScript 5.0 ถึง 5.6+ พร้อมตัวอย่างการใช้งานจริงกว่า 50 ตัวอย่าง

---

## ส่วนที่ 1: Decorators (Stage 3 - TypeScript 5.0)

Decorators ใน TypeScript 5.0 ใช้ Stage 3 Decorator proposal ใหม่ ซึ่งแตกต่างจาก experimental decorators เดิม

### ตัวอย่างที่ 1: Class Decorators

```typescript
// TypeScript 5.0 - Stage 3 Decorators
// ไม่ต้องเปิด experimentalDecorators แล้ว!

// Class Decorator
function logged<T extends new (...args: any[]) => any>(
  target: T,
  context: ClassDecoratorContext
): T {
  return class extends target {
    constructor(...args: any[]) {
      console.log(`Creating instance of ${context.name}`);
      super(...args);
      console.log(`Instance created`);
    }
  };
}

@logged
class UserService {
  constructor(private name: string) {}
  
  greet() {
    return `Hello, ${this.name}!`;
  }
}

const service = new UserService("John");
// Console: "Creating instance of UserService"
// Console: "Instance created"

// Class with metadata
function component(config: { selector: string; template: string }) {
  return function<T extends new (...args: any[]) => any>(
    target: T,
    context: ClassDecoratorContext
  ): T {
    context.metadata ??= {};
    context.metadata.selector = config.selector;
    context.metadata.template = config.template;
    return target;
  };
}

@component({
  selector: "app-user",
  template: "<div>{{name}}</div>"
})
class UserComponent {
  name = "John";
}

console.log(Symbol.metadata && (UserComponent as any)[Symbol.metadata]);
```

---

### ตัวอย่างที่ 2: Method Decorators

```typescript
// Method Decorators ใหม่
function deprecated(
  target: (this: unknown, ...args: unknown[]) => unknown,
  context: ClassMethodDecoratorContext
): typeof target {
  const methodName = String(context.name);
  
  return function(this: unknown, ...args: unknown[]) {
    console.warn(`Method ${methodName} is deprecated. Please use a newer alternative.`);
    return target.apply(this, args);
  };
}

function memoize<T, Args extends unknown[]>(
  target: (...args: Args) => T,
  context: ClassMethodDecoratorContext
): (...args: Args) => T {
  const cache = new Map<string, T>();
  
  return function(this: unknown, ...args: Args): T {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const result = target.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

function retry(times: number = 3) {
  return function<T>(
    target: (...args: unknown[]) => Promise<T>,
    context: ClassMethodDecoratorContext
  ): typeof target {
    return async function(this: unknown, ...args: unknown[]): Promise<T> {
      let lastError: Error | undefined;
      
      for (let attempt = 0; attempt < times; attempt++) {
        try {
          return await target.apply(this, args);
        } catch (error) {
          lastError = error as Error;
          console.log(`Attempt ${attempt + 1} failed: ${lastError.message}`);
          
          if (attempt < times - 1) {
            await new Promise(resolve => setTimeout(resolve, 100 * (attempt + 1)));
          }
        }
      }
      
      throw lastError;
    };
  };
}

class ApiService {
  @deprecated
  oldMethod() {
    return "old result";
  }
  
  @memoize
  expensiveCalculation(n: number): number {
    console.log(`Calculating for ${n}...`);
    return n * n * n;
  }
  
  @retry(3)
  async fetchData(url: string): Promise<unknown> {
    const response = await fetch(url);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  }
}

const api = new ApiService();
api.oldMethod(); // Warning: Method oldMethod is deprecated
api.expensiveCalculation(5); // Calculates
api.expensiveCalculation(5); // Returns cached result
```

---

### ตัวอย่างที่ 3: Field Decorators

```typescript
// Field Decorators
function required(
  target: undefined,
  context: ClassFieldDecoratorContext
): (initialValue: unknown) => unknown {
  const fieldName = String(context.name);
  
  return function(value: unknown) {
    if (value === undefined || value === null || value === "") {
      throw new Error(`Field ${fieldName} is required`);
    }
    return value;
  };
}

function minLength(min: number) {
  return function(
    target: undefined,
    context: ClassFieldDecoratorContext
  ): (initialValue: unknown) => unknown {
    const fieldName = String(context.name);
    
    return function(value: unknown) {
      if (typeof value !== "string" || value.length < min) {
        throw new Error(`${fieldName} must be at least ${min} characters`);
      }
      return value;
    };
  };
}

function transform(transformFn: (value: unknown) => unknown) {
  return function(
    target: undefined,
    context: ClassFieldDecoratorContext
  ): (initialValue: unknown) => unknown {
    return function(value: unknown) {
      return transformFn(value);
    };
  };
}

class User {
  @required
  @minLength(3)
  name: string = ""; // Will throw if empty or too short
  
  @required
  email: string = "";
  
  @transform((v) => (v as string).trim().toLowerCase())
  username: string = "";
}

// ตัวอย่าง Validate decorator
function validate<T>(schema: { [K in keyof T]?: (v: T[K]) => string | null }) {
  return function(
    target: new (...args: any[]) => T,
    context: ClassDecoratorContext
  ): typeof target {
    return class extends (target as any) {
      constructor(...args: any[]) {
        super(...args);
        
        for (const [field, validator] of Object.entries(schema)) {
          const error = (validator as any)((this as any)[field]);
          if (error) {
            throw new Error(`Validation error on ${field}: ${error}`);
          }
        }
      }
    } as typeof target;
  };
}
```

---

### ตัวอย่างที่ 4: Accessor Decorators

```typescript
// Accessor Decorators
function readonly<T>(
  target: ClassAccessorDecoratorTarget<unknown, T>,
  context: ClassAccessorDecoratorContext
): ClassAccessorDecoratorResult<unknown, T> {
  return {
    get(this: unknown) {
      return target.get.call(this);
    },
    set(this: unknown, value: T) {
      if (context.static || (this as any)._initialized) {
        throw new Error(`Cannot set readonly property ${String(context.name)}`);
      }
      target.set.call(this, value);
    },
    init(value: T) {
      return value;
    }
  };
}

function logged<T>(
  target: ClassAccessorDecoratorTarget<unknown, T>,
  context: ClassAccessorDecoratorContext
): ClassAccessorDecoratorResult<unknown, T> {
  const name = String(context.name);
  
  return {
    get(this: unknown) {
      const value = target.get.call(this);
      console.log(`Getting ${name}: ${value}`);
      return value;
    },
    set(this: unknown, value: T) {
      console.log(`Setting ${name} to: ${value}`);
      target.set.call(this, value);
    }
  };
}

class Config {
  @logged
  accessor theme: string = "light";
  
  accessor version: string = "1.0.0";
}

const config = new Config();
config.theme = "dark"; // Setting theme to: dark
console.log(config.theme); // Getting theme: dark, dark
```

---

## ส่วนที่ 2: const Type Parameters (TypeScript 5.0)

### ตัวอย่างที่ 5: const Type Parameters

```typescript
// TypeScript 5.0 - const type parameters
// บังคับให้ TypeScript อนุมาน literal types

// Before TypeScript 5.0
function route<T extends string>(path: T): T {
  return path;
}
const r1 = route("/users"); // type: string (ไม่ใช่ "/users")

// TypeScript 5.0+
function route5<const T extends string>(path: T): T {
  return path;
}
const r2 = route5("/users"); // type: "/users" (literal type!)

// กับ Arrays
function config<const T extends string[]>(items: T): T {
  return items;
}

const items = config(["react", "typescript", "node"]);
// type: readonly ["react", "typescript", "node"] - tuple type!

// กับ Objects
function createRoute<const T extends {
  path: string;
  method: string;
}>(config: T): T & { handler?: () => void } {
  return config;
}

const myRoute = createRoute({
  path: "/users/:id",
  method: "GET"
});
// myRoute.path type: "/users/:id" (not string!)
// myRoute.method type: "GET" (not string!)

// Practical example - type-safe API definition
function defineEndpoints<const T extends Record<string, {
  path: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
}>>(endpoints: T): T {
  return endpoints;
}

const API = defineEndpoints({
  getUser: { path: "/users/:id", method: "GET" },
  createUser: { path: "/users", method: "POST" },
  updateUser: { path: "/users/:id", method: "PUT" }
});

type GetUserPath = typeof API.getUser.path; // "/users/:id"
type GetUserMethod = typeof API.getUser.method; // "GET"
```

---

## ส่วนที่ 3: satisfies Operator (TypeScript 4.9 - ใช้บ่อยใน 5.x)

### ตัวอย่างที่ 6: satisfies Operator

```typescript
// satisfies ตรวจสอบ type constraint โดยไม่เปลี่ยน inferred type

// ปัญหาเดิม
type Color = "red" | "green" | "blue";
type ColorConfig = Record<Color, string | number[]>;

// Option 1: type annotation - ทำให้ lose literal types
const palette1: ColorConfig = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255]
};

palette1.red.map(v => v * 2); // Error! red เป็น string | number[]

// Option 2: satisfies - ยังคง literal types
const palette2 = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255]
} satisfies ColorConfig;

palette2.red.map(v => v * 2); // OK! red เป็น number[]
palette2.green.toUpperCase(); // OK! green เป็น string

// More examples
interface Config {
  port: number;
  host: string;
  features: string[];
}

// With satisfies - TypeScript knows exact types while validating
const appConfig = {
  port: 3000,      // TypeScript knows it's 3000 (literal)
  host: "localhost", // TypeScript knows it's "localhost" (literal)
  features: ["auth", "payments"] as const
} satisfies Config;

// appConfig.port type: 3000 (not number)
// appConfig.host type: "localhost" (not string)
// appConfig.features type: readonly ["auth", "payments"]

// Complex example
type Routes = Record<string, {
  path: string;
  component: string;
  auth?: boolean;
}>;

const routes = {
  home: {
    path: "/",
    component: "HomePage",
    auth: false
  },
  dashboard: {
    path: "/dashboard",
    component: "DashboardPage",
    auth: true
  },
  profile: {
    path: "/profile/:id",
    component: "ProfilePage",
    auth: true
  }
} satisfies Routes;

// routes.home.path type: "/" (not string!)
// routes.dashboard.auth type: true (not boolean!)
```

---

## ส่วนที่ 4: Variadic Tuple Types Improvements

### ตัวอย่างที่ 7: Enhanced Tuple Types

```typescript
// TypeScript 5.x improvements to variadic tuples

// Labeled tuples
type Range = [start: number, end: number];
type Point = [x: number, y: number, z?: number];

// Type-safe variadic concat
function concat<T extends unknown[], U extends unknown[]>(
  a: readonly [...T],
  b: readonly [...U]
): readonly [...T, ...U] {
  return [...a, ...b] as const as readonly [...T, ...U];
}

const result = concat([1, 2] as const, ["a", "b"] as const);
// type: readonly [1, 2, "a", "b"]

// Tuple to union
type TupleToUnion<T extends readonly unknown[]> = T[number];

const colors = ["red", "green", "blue"] as const;
type Color = TupleToUnion<typeof colors>; // "red" | "green" | "blue"

// Prepend/Append to tuple
type Prepend<T, Tuple extends unknown[]> = [T, ...Tuple];
type Append<Tuple extends unknown[], T> = [...Tuple, T];

type WithId<T extends unknown[]> = Prepend<string, T>;
type Args = WithId<[string, number]>; // [string, string, number]

// Reverse tuple
type Reverse<T extends unknown[]> =
  T extends [infer F, ...infer Rest]
    ? [...Reverse<Rest>, F]
    : [];

type Rev = Reverse<[1, 2, 3, 4]>; // [4, 3, 2, 1]

// Zip tuples
type Zip<T extends unknown[], U extends unknown[]> =
  T extends [infer TH, ...infer TT]
    ? U extends [infer UH, ...infer UT]
      ? [[TH, UH], ...Zip<TT, UT>]
      : []
    : [];

type Zipped = Zip<[1, 2, 3], ["a", "b", "c"]>;
// [[1, "a"], [2, "b"], [3, "c"]]

// Length of tuple
type Length<T extends unknown[]> = T["length"];
type Len = Length<[1, 2, 3]>; // 3
```

---

## ส่วนที่ 5: Template String Type Improvements

### ตัวอย่างที่ 8: Advanced Template String Types

```typescript
// TypeScript 5.x Template String Type Inference

// Pattern matching with template literals
type ExtractId<T extends string> =
  T extends `${string}:${infer Id}` ? Id : never;

type UserId = ExtractId<"user:123">; // "123"
type PostId = ExtractId<"post:456">; // "456"

// CSS property types
type Property = "margin" | "padding" | "border";
type Side = "top" | "right" | "bottom" | "left";

type LonghandProperty = `${Property}-${Side}`;
// "margin-top" | "margin-right" | ... | "border-left"

// Event types
type EventType = "click" | "hover" | "focus" | "blur";
type EventListener = `on${Capitalize<EventType>}`;
// "onClick" | "onHover" | "onFocus" | "onBlur"

// API endpoint types
type HttpVerb = "get" | "post" | "put" | "delete" | "patch";
type Resource = "user" | "post" | "comment";

type EndpointFn = `${HttpVerb}${Capitalize<Resource>}`;
// "getUser" | "getPost" | ... | "patchComment"

// Type-safe i18n
type TranslationKey<T extends Record<string, unknown>, Prefix extends string = ""> =
  {
    [K in keyof T & string]: T[K] extends Record<string, unknown>
      ? TranslationKey<T[K], `${Prefix}${K}.`>
      : `${Prefix}${K}`;
  }[keyof T & string];

const translations = {
  common: {
    ok: "OK",
    cancel: "Cancel",
    save: "Save"
  },
  user: {
    name: "Name",
    email: "Email",
    profile: {
      title: "Profile",
      bio: "Bio"
    }
  }
} as const;

type TransKey = TranslationKey<typeof translations>;
// "common.ok" | "common.cancel" | "common.save" | "user.name" | "user.email" | "user.profile.title" | "user.profile.bio"

function t(key: TransKey): string {
  const parts = key.split(".");
  let current: any = translations;
  for (const part of parts) {
    current = current[part];
  }
  return current;
}

const text = t("user.profile.title"); // "Profile"
// t("invalid.key"); // Error!
```

---

## ส่วนที่ 6: Using Declarations (TypeScript 5.2)

### ตัวอย่างที่ 9: Using Declarations

```typescript
// TypeScript 5.2 - using declarations (Explicit Resource Management)
// ต้องการ TypeScript 5.2+ และ lib: ["ES2022"] หรือสูงกว่า

// Disposable interface
interface Disposable {
  [Symbol.dispose](): void;
}

interface AsyncDisposable {
  [Symbol.asyncDispose](): Promise<void>;
}

// Example: Database connection
class DatabaseConnection {
  private connection: { close(): void };
  
  constructor(connectionString: string) {
    this.connection = {
      close() {
        console.log("Connection closed");
      }
    };
    console.log(`Connected to ${connectionString}`);
  }
  
  query(sql: string): unknown[] {
    console.log(`Executing: ${sql}`);
    return [];
  }
  
  [Symbol.dispose](): void {
    this.connection.close();
    console.log("Database connection disposed");
  }
}

// การใช้งาน using declarations
async function performDatabaseOperation() {
  using db = new DatabaseConnection("postgresql://localhost/mydb");
  
  // db จะถูก dispose อัตโนมัติเมื่อออกจาก scope
  const users = db.query("SELECT * FROM users");
  
  return users;
} // db.[Symbol.dispose]() ถูกเรียกที่นี่

// Async Disposable
class AsyncDatabaseConnection {
  async query(sql: string): Promise<unknown[]> {
    return [];
  }
  
  async [Symbol.asyncDispose](): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 100));
    console.log("Async connection closed");
  }
}

async function asyncOperation() {
  await using db = new AsyncDatabaseConnection();
  const result = await db.query("SELECT 1");
  return result;
} // db[Symbol.asyncDispose]() ถูกเรียกที่นี่

// DisposableStack
function performMultipleOperations() {
  using stack = new DisposableStack();
  
  const db = stack.use(new DatabaseConnection("postgresql://localhost/db1"));
  // Add cleanup callbacks
  stack.defer(() => console.log("Cleanup completed"));
  
  db.query("SELECT * FROM users");
} // stack disposes everything in LIFO order

// Real-world example: File operations
class TempFile implements Disposable {
  private path: string;
  
  constructor(prefix: string) {
    this.path = `/tmp/${prefix}-${Date.now()}`;
    // create file...
    console.log(`Created temp file: ${this.path}`);
  }
  
  write(content: string): void {
    console.log(`Writing to ${this.path}: ${content}`);
  }
  
  read(): string {
    return `Content of ${this.path}`;
  }
  
  [Symbol.dispose](): void {
    // delete file...
    console.log(`Deleted temp file: ${this.path}`);
  }
}

function processWithTempFile() {
  using tempFile = new TempFile("process");
  
  tempFile.write("some data");
  const content = tempFile.read();
  
  return content;
} // Temp file is automatically deleted here
```

---

## ส่วนที่ 7: Import Attributes (TypeScript 5.3)

### ตัวอย่างที่ 10: Import Attributes

```typescript
// TypeScript 5.3 - Import Attributes (previously Import Assertions)

// Import JSON with type attribute
import data from "./data.json" with { type: "json" };

// Import CSS
import styles from "./styles.css" with { type: "css" };

// Dynamic imports with attributes
const jsonData = await import("./config.json", {
  with: { type: "json" }
});

// Multiple attributes
const wasmModule = await import("./module.wasm", {
  with: {
    type: "webassembly"
  }
});

// Type declarations for non-standard imports
declare module "*.json" {
  const value: unknown;
  export default value;
}

declare module "*.css" {
  const styles: Record<string, string>;
  export default styles;
}

// Custom module type extensions
declare module "*.svg" {
  const content: string;
  export default content;
}

declare module "*.png" {
  const url: string;
  export default url;
}

// Import type assertions in practice
import type { User } from "./models" with { type: "json" };
```

---

## ส่วนที่ 8: Inlay Hints & Type Improvements

### ตัวอย่างที่ 11: Override Keyword

```typescript
// override keyword (TypeScript 4.3+, widely used in 5.x)

class Base {
  greet(name: string): string {
    return `Hello, ${name}!`;
  }
  
  protected setup(): void {
    console.log("Base setup");
  }
}

class Derived extends Base {
  override greet(name: string): string {
    return `Hi, ${name}! (from Derived)`;
  }
  
  override protected setup(): void {
    super.setup();
    console.log("Derived setup");
  }
}

// Error prevention
class AnotherDerived extends Base {
  // override typoMethod(): string { // Error! typoMethod doesn't exist in Base
  //   return "oops";
  // }
}

// noImplicitOverride tsconfig option
// "noImplicitOverride": true - forces explicit override keyword

class Framework {
  protected initialize(): void {}
  protected teardown(): void {}
  abstract render(): void;
}

class MyComponent extends Framework {
  override protected initialize(): void {
    // Must use override keyword when noImplicitOverride is enabled
  }
  
  override protected teardown(): void {
    // Explicit override
  }
  
  render(): void {
    console.log("Rendering component");
  }
}
```

---

## ส่วนที่ 9: New Utility Types (TypeScript 5.x)

### ตัวอย่างที่ 12: NoInfer Utility Type (TypeScript 5.4)

```typescript
// TypeScript 5.4 - NoInfer<T>
// ป้องกัน TypeScript จากการ infer type จาก parameter นั้น

// ปัญหาเดิม
function createState<T>(initialValue: T, default_: T): T {
  return initialValue ?? default_;
}

// TypeScript อาจ infer T จาก default_ ด้วย ซึ่งอาจทำให้ type กว้างเกินไป
const state = createState(42, "fallback"); // T = number | string (ไม่ต้องการ)

// ใช้ NoInfer
function createStateSafe<T>(initialValue: T, default_: NoInfer<T>): T {
  return initialValue ?? default_;
}

const safeSate = createStateSafe(42, 0); // T = number, และ 0 ต้องเป็น number ด้วย
// createStateSafe(42, "fallback"); // Error! "fallback" ไม่ใช่ number

// Practical example - createSignal pattern
function createSignal<T>(initialValue: T): [
  get: () => T,
  set: (newValue: NoInfer<T>) => void
] {
  let value = initialValue;
  
  return [
    () => value,
    (newValue: T) => { value = newValue; }
  ];
}

const [count, setCount] = createSignal(0);
setCount(1); // OK
// setCount("1"); // Error! "1" is not number

// With union types
type Theme = "light" | "dark";

function useTheme(initial: Theme): [() => Theme, (t: NoInfer<Theme>) => void] {
  let theme = initial;
  return [
    () => theme,
    (t: Theme) => { theme = t; }
  ];
}
```

---

### ตัวอย่างที่ 13: Improved Intersection Types

```typescript
// TypeScript 5.x - Better intersection type handling

// Improved type narrowing in intersections
type IsNever<T> = [T] extends [never] ? true : false;
type Test = IsNever<never>; // true

// Simplification of intersection types
type A = { a: string };
type B = { b: number };
type C = { c: boolean };

type ABC = A & B & C;
// TypeScript 5.x simplifies these more aggressively

// Discriminated union improvements
type ApiResult =
  | { status: "success"; data: { users: string[] } }
  | { status: "error"; error: { code: number; message: string } }
  | { status: "loading" };

function handleResult(result: ApiResult) {
  if (result.status === "success") {
    // TypeScript 5.x better understands this
    result.data.users.forEach(u => console.log(u));
  } else if (result.status === "error") {
    console.error(result.error.message);
  }
}

// Type narrowing improvements
function processValue(value: string | number | boolean | null | undefined) {
  if (value != null) {
    // TypeScript 5.x: value is string | number | boolean
    const str = value.toString(); // No error!
  }
}
```

---

## ส่วนที่ 10: Performance Improvements

### ตัวอย่างที่ 14: Isolated Declarations (TypeScript 5.5)

```typescript
// TypeScript 5.5 - Isolated Declarations
// ช่วยให้ build เร็วขึ้นใน monorepo
// tsconfig.json: { "isolatedDeclarations": true }

// เมื่อเปิด isolatedDeclarations ต้องระบุ return types ที่ export functions

// ✗ Error with isolatedDeclarations
// export function greet(name: string) { // Missing return type
//   return `Hello, ${name}!`;
// }

// ✓ Correct - explicit return type
export function greet(name: string): string {
  return `Hello, ${name}!`;
}

// ✓ Variable with explicit type
export const PI: number = 3.14159;

// ✓ Class with explicit return types on public methods
export class Calculator {
  add(a: number, b: number): number {
    return a + b;
  }
  
  multiply(a: number, b: number): number {
    return a * b;
  }
}

// ✓ Interface exports don't need changes
export interface Config {
  host: string;
  port: number;
}

// ✓ Type exports
export type UserId = string;

// Benefits of isolatedDeclarations:
// 1. ทำให้ tools อื่น (esbuild, swc) สร้าง .d.ts files ได้โดยไม่ต้องรัน TypeScript
// 2. Build เร็วขึ้นมากใน large monorepos
// 3. Parallel type checking
```

---

### ตัวอย่างที่ 15: TypeScript 5.5 Inferred Type Predicates

```typescript
// TypeScript 5.5 - Inferred Type Predicates
// TypeScript อนุมาน type predicates อัตโนมัติ

// ก่อน TypeScript 5.5 - ต้องเขียน type predicate ด้วยตนเอง
const oldFilter = (value: string | null): value is string => value !== null;

// TypeScript 5.5 - อนุมาน type predicate อัตโนมัติ
const isString = (value: string | null | undefined) => value !== null && value !== undefined;

// TypeScript รู้ว่า isString เป็น type predicate
const values: (string | null | undefined)[] = ["a", null, "b", undefined, "c"];
const strings = values.filter(isString);
// strings type: string[] (ไม่ใช่ (string | null | undefined)[])

// Complex inferred predicates
function isValidUser(user: { name?: string; email?: string }): user is { name: string; email: string } {
  return typeof user.name === "string" && typeof user.email === "string";
}

// Array methods benefit
const users = [
  { name: "John", email: "john@example.com" },
  { name: undefined, email: "jane@example.com" },
  { name: "Bob", email: undefined }
];

const validUsers = users.filter(isValidUser);
// validUsers type: { name: string; email: string }[]

// More examples
const isNotNull = <T>(value: T | null): value is T => value !== null;
const isDefined = <T>(value: T | undefined): value is T => value !== undefined;
const isPresent = <T>(value: T | null | undefined): value is T =>
  value !== null && value !== undefined;

// Usage in real code
async function loadUsers(): Promise<(User | null)[]> {
  return [/* ... */];
}

const loaded = await loadUsers();
const actual = loaded.filter(isNotNull); // User[] - no need for explicit type predicate!
```

---

## ส่วนที่ 11: TypeScript 5.x Migration Guide

### ตัวอย่างที่ 16: การ Migrate จาก TypeScript 4.x

```typescript
// Migration Guide: TypeScript 4.x to 5.x

// 1. Decorators ใหม่
// Before (experimentalDecorators)
// @Component() class MyClass {} // Old syntax

// After (Stage 3 decorators)
// @Component class MyClass {} // New syntax - ไม่ต้องมี ()

// 2. const type parameters
// Before
function identity<T extends string>(x: T): T { return x; }
const v1 = identity("hello"); // type: string (widened)

// After
function identityConst<const T extends string>(x: T): T { return x; }
const v2 = identityConst("hello"); // type: "hello" (literal)

// 3. satisfies operator
// Before - had to choose between type safety and type inference
const palette = { red: [255, 0, 0] } as { red: number[] };
palette.red.length; // OK, but loses 3

// After - satisfies
const paletteNew = { red: [255, 0, 0] } satisfies { red: number[] };
paletteNew.red.length; // OK, type is still number[]

// 4. Resolution mode (package.json exports field)
// tsconfig.json
// {
//   "moduleResolution": "bundler" // new in TS 5.0
// }

// 5. --verbatimModuleSyntax (replaces importsNotUsedAsValues)
// Clearer control over type-only imports
import type { User } from "./types"; // Always removed at compile time
import { type User as UserType, UserService } from "./services"; // UserType removed, UserService kept

// 6. Removed deprecated flags
// --noImplicitUseStrict (removed - strict mode always enabled)
// --keyofStringsOnly (removed)
// --out (removed - use --outDir)
// --suppressExcessPropertyErrors (removed)
// --suppressImplicitAnyIndexErrors (removed)
```

---

## ส่วนที่ 12: TypeScript 5.x Best Practices

### ตัวอย่างที่ 17: Modern TypeScript Patterns

```typescript
// Pattern 1: Using satisfies for configuration
interface AppConfig {
  port: number;
  database: {
    url: string;
    poolSize: number;
  };
  features: Record<string, boolean>;
}

const config = {
  port: 3000,
  database: {
    url: "postgresql://localhost/mydb",
    poolSize: 10
  },
  features: {
    darkMode: true,
    betaFeature: false
  }
} satisfies AppConfig;

// config.port type: 3000 (literal!)
// config.database.poolSize type: 10 (literal!)

// Pattern 2: const type parameters for type-safe registries
function createRegistry<const T extends Record<string, unknown>>() {
  const registry = {} as T;
  
  return {
    register<K extends keyof T>(key: K, value: T[K]) {
      registry[key] = value;
    },
    get<K extends keyof T>(key: K): T[K] {
      return registry[key];
    }
  };
}

// Pattern 3: NoInfer for strict type checking
function parseConfig<T>(
  schema: T,
  input: NoInfer<T>
): T {
  return { ...schema, ...input };
}

// Pattern 4: using declarations for resource cleanup
function withDatabase<T>(
  connectionString: string,
  fn: (db: { query(sql: string): unknown[] }) => T
): T {
  const db = {
    connection: connectionString,
    closed: false,
    query(sql: string): unknown[] {
      if (this.closed) throw new Error("Connection closed");
      return [];
    },
    [Symbol.dispose]() {
      this.closed = true;
    }
  };
  
  using dbConnection = db;
  return fn(dbConnection);
}
```

---

### ตัวอย่างที่ 18: TypeScript 5.x Complete Example Project

```typescript
// Complete example using multiple TypeScript 5.x features

// === Types with satisfies ===
type UserRole = "admin" | "user" | "moderator";

interface User {
  id: string;
  name: string;
  email: string;
  role: UserRole;
  createdAt: Date;
}

// === Branded Types ===
declare const __brand: unique symbol;
type UserId = string & { [__brand]: "UserId" };

function createUserId(id: string): UserId {
  return id as UserId;
}

// === Repository with using ===
class UserRepository implements Disposable {
  private closed = false;
  
  async findById(id: UserId): Promise<User | null> {
    if (this.closed) throw new Error("Repository closed");
    // Implementation
    return null;
  }
  
  async save(user: User): Promise<void> {
    if (this.closed) throw new Error("Repository closed");
    // Implementation
  }
  
  [Symbol.dispose](): void {
    this.closed = true;
    console.log("Repository disposed");
  }
}

// === Event System with const types ===
function createEventSystem<const Events extends Record<string, (...args: any[]) => void>>() {
  const handlers = {} as { [K in keyof Events]?: Events[K][] };
  
  return {
    on<K extends keyof Events>(event: K, handler: Events[K]) {
      if (!handlers[event]) handlers[event] = [];
      handlers[event]!.push(handler);
      return () => {
        handlers[event] = handlers[event]!.filter(h => h !== handler);
      };
    },
    emit<K extends keyof Events>(event: K, ...args: Parameters<Events[K]>) {
      handlers[event]?.forEach(h => h(...args));
    }
  };
}

const userEvents = createEventSystem<{
  created: (user: User) => void;
  updated: (user: User, changes: Partial<User>) => void;
  deleted: (userId: UserId) => void;
}>();

// === Service with Decorators ===
function singleton<T extends new (...args: any[]) => any>(
  target: T,
  context: ClassDecoratorContext
): T {
  let instance: InstanceType<T> | null = null;
  
  return new Proxy(target, {
    construct(target, args) {
      if (!instance) {
        instance = Reflect.construct(target, args) as InstanceType<T>;
      }
      return instance;
    }
  }) as T;
}

function validateInput(
  target: (this: unknown, ...args: unknown[]) => unknown,
  context: ClassMethodDecoratorContext
): typeof target {
  return function(this: unknown, ...args: unknown[]) {
    for (const arg of args) {
      if (arg === null || arg === undefined) {
        throw new Error(`${String(context.name)}: null/undefined arguments not allowed`);
      }
    }
    return target.apply(this, args);
  };
}

@singleton
class UserService {
  constructor(
    private repository: UserRepository
  ) {}
  
  @validateInput
  async getUser(id: UserId): Promise<User | null> {
    return this.repository.findById(id);
  }
  
  async createUser(data: Omit<User, "id" | "createdAt">): Promise<User> {
    const user: User = {
      ...data,
      id: createUserId(Math.random().toString(36).slice(2)),
      createdAt: new Date()
    };
    
    await this.repository.save(user);
    userEvents.emit("created", user);
    
    return user;
  }
}

// === Main program ===
async function main() {
  using repo = new UserRepository();
  const service = new UserService(repo);
  
  const unsubscribe = userEvents.on("created", (user) => {
    console.log(`New user: ${user.name}`);
  });
  
  const user = await service.createUser({
    name: "John Doe",
    email: "john@example.com",
    role: "user"
  });
  
  console.log(`Created user: ${user.id}`);
  
  unsubscribe();
} // repo is automatically disposed

main().catch(console.error);
```

---

## ส่วนที่ 13: TypeScript 5.x Configuration

### ตัวอย่างที่ 19: Modern tsconfig.json

```json
{
  "compilerOptions": {
    // Target and Module
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    
    // Output
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    
    // Strict Mode
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true,
    
    // Additional Checks
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    
    // Module Settings
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    
    // TypeScript 5.x Specific
    "verbatimModuleSyntax": true,
    "isolatedDeclarations": true,
    
    // Other
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "allowImportingTsExtensions": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

### ตัวอย่างที่ 20: Project References สำหรับ Monorepo

```json
// packages/common/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}

// packages/api/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "rootDir": "src",
    "outDir": "dist",
    "paths": {
      "@myapp/common": ["../common/src"]
    }
  },
  "references": [
    { "path": "../common" }
  ],
  "include": ["src"]
}

// tsconfig.json (root)
{
  "references": [
    { "path": "packages/common" },
    { "path": "packages/api" },
    { "path": "packages/web" }
  ],
  "files": []
}
```

---

## สรุป TypeScript 5.x Features

| เวอร์ชัน | ฟีเจอร์หลัก |
|---------|------------|
| 4.9 | `satisfies` operator |
| 5.0 | Stage 3 Decorators, `const` type parameters |
| 5.1 | Unrelated types for `undefined`-returning functions |
| 5.2 | `using` declarations, `Symbol.dispose` |
| 5.3 | Import attributes (`with` syntax) |
| 5.4 | `NoInfer<T>` utility type |
| 5.5 | Inferred type predicates, RegExp named groups |
| 5.6 | Disallow nonsense types, Iterator helper methods |

ข้อแนะนำในการ migrate:

1. **ค่อยๆ migrate** - เปิด strict options ทีละตัว
2. **ใช้ `satisfies`** แทน type annotations เมื่อต้องการ preserve literal types
3. **ใช้ `const` type parameters** สำหรับ generic functions ที่ต้องการ literal inference
4. **ใช้ `using`** สำหรับทุก resource ที่ต้องการ cleanup
5. **เพิ่ม return types** ถ้าใช้ `isolatedDeclarations`
6. **ใช้ `NoInfer`** เพื่อป้องกัน type widening

---

## ตัวอย่างที่ 21-50: เพิ่มเติม

```typescript
// ตัวอย่างที่ 21: TypeScript 5.5 RegExp Named Capture Groups
const dateRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const match = dateRegex.exec("2024-01-15");

if (match?.groups) {
  const { year, month, day } = match.groups;
  // year, month, day type: string (TypeScript knows these exist)
  console.log(`Year: ${year}, Month: ${month}, Day: ${day}`);
}

// ตัวอย่างที่ 22: Better contextual types for control flow
function processItems<T>(
  items: T[],
  callback: (item: T, index: number) => T
): T[] {
  return items.map(callback);
}

const doubled = processItems([1, 2, 3], (item, _index) => {
  // TypeScript 5.x: item type is correctly inferred as number
  return item * 2;
});

// ตัวอย่างที่ 23: TypeScript 5.x Array methods improvements
const nums = [1, 2, 3, null, 4, undefined, 5];
const validNums = nums.filter(n => n != null);
// TypeScript 5.5: validNums type is number[] (not (number | null | undefined)[])

// ตัวอย่างที่ 24: TypeScript 5.x - Better type narrowing
function processEvent(event: MouseEvent | KeyboardEvent | TouchEvent) {
  if (event instanceof MouseEvent) {
    const x = event.clientX; // number
    const y = event.clientY; // number
  } else if (event instanceof KeyboardEvent) {
    const key = event.key; // string
    const code = event.code; // string
  } else {
    const touches = event.touches; // TouchList
  }
}

// ตัวอย่างที่ 25: TypeScript 5.x - New Iterator types
function* fibonacci(): Generator<number, void, void> {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// With Iterator helpers (ES2025 / TypeScript 5.6+)
const fib = fibonacci();
// fib.take(10) - get first 10 values
// fib.map(n => n * 2) - transform
// fib.filter(n => n % 2 === 0) - filter

// ตัวอย่างที่ 26: TypeScript 5.x - Variance improvements
type Box<out T> = {
  readonly value: T;
};

type MutableBox<in out T> = {
  value: T;
};

// TypeScript 5.x is better at checking variance
const readBox: Box<string> = { value: "hello" };
const anyReadBox: Box<unknown> = readBox; // OK - covariant

// ตัวอย่างที่ 27: TypeScript 5.x - Narrowing of `in` expressions
interface Cat { meow(): void }
interface Dog { bark(): void }
interface Bird { fly(): void; feathers: boolean }

function processAnimal(animal: Cat | Dog | Bird) {
  if ("meow" in animal) {
    animal.meow(); // TypeScript knows it's Cat
  } else if ("bark" in animal) {
    animal.bark(); // TypeScript knows it's Dog
  } else if ("feathers" in animal) {
    animal.fly(); // TypeScript knows it's Bird
    console.log(animal.feathers); // boolean
  }
}

// ตัวอย่างที่ 28: TypeScript 5.x - String.raw type improvements
const path = String.raw`C:\Users\John\Documents`; // type: string

// Template literal tagging
function sql(strings: TemplateStringsArray, ...values: unknown[]): string {
  return strings.reduce((result, str, i) =>
    result + str + (values[i] !== undefined ? `'${values[i]}'` : ""),
    ""
  );
}

const userId = "123";
const query = sql`SELECT * FROM users WHERE id = ${userId}`;

// ตัวอย่างที่ 29: TypeScript 5.x - Improved module resolution
// package.json exports field support
// {
//   "exports": {
//     ".": {
//       "import": "./dist/index.mjs",
//       "require": "./dist/index.cjs",
//       "types": "./dist/index.d.ts"
//     },
//     "./utils": {
//       "import": "./dist/utils.mjs",
//       "types": "./dist/utils.d.ts"
//     }
//   }
// }

// ตัวอย่างที่ 30: TypeScript 5.x - Deprecation support
/** @deprecated Use newFunction instead */
function oldFunction(): void {}

function newFunction(): void {}

// TypeScript/IDE will show strikethrough and warning
// oldFunction(); // Warning: deprecated

// ตัวอย่างที่ 31-50: Practical patterns

// Type-safe HTML element selection
type ElementTagMap = {
  "div": HTMLDivElement;
  "span": HTMLSpanElement;
  "input": HTMLInputElement;
  "button": HTMLButtonElement;
  "a": HTMLAnchorElement;
  "img": HTMLImageElement;
  "form": HTMLFormElement;
};

function select<T extends keyof ElementTagMap>(
  selector: T,
  id: string
): ElementTagMap[T] | null {
  return document.getElementById(id) as ElementTagMap[T] | null;
}

const input = select("input", "myInput"); // HTMLInputElement | null
const button = select("button", "submit"); // HTMLButtonElement | null

// Type-safe localStorage
function createTypedStorage<T extends Record<string, unknown>>(
  prefix: string
) {
  return {
    get<K extends keyof T>(key: K): T[K] | null {
      try {
        const item = localStorage.getItem(`${prefix}:${String(key)}`);
        return item ? JSON.parse(item) : null;
      } catch {
        return null;
      }
    },
    set<K extends keyof T>(key: K, value: T[K]): void {
      try {
        localStorage.setItem(`${prefix}:${String(key)}`, JSON.stringify(value));
      } catch {}
    },
    remove<K extends keyof T>(key: K): void {
      localStorage.removeItem(`${prefix}:${String(key)}`);
    },
    clear(): void {
      const keys = Object.keys(localStorage).filter(k => k.startsWith(`${prefix}:`));
      keys.forEach(k => localStorage.removeItem(k));
    }
  };
}

interface UserStorageSchema {
  user: { id: string; name: string; email: string };
  token: string;
  preferences: { theme: "light" | "dark"; language: string };
  lastVisit: Date;
}

const userStorage = createTypedStorage<UserStorageSchema>("app");
userStorage.set("preferences", { theme: "dark", language: "th" });
const prefs = userStorage.get("preferences"); // { theme: "light" | "dark"; language: string } | null
```

---

*จบ Part 60 - TypeScript 5.x Features*
