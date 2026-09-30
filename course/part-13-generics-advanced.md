# ส่วนที่ 13: Advanced Generics

## บทนำ

ในบทนี้เราจะเจาะลึก TypeScript features ที่ทรงพลังยิ่งขึ้น ได้แก่ keyof, typeof, Conditional Types, Mapped Types และ Utility Types

---

## 13.1 keyof Operator

### keyof พื้นฐาน

```typescript
// keyof ดึง union ของ keys จาก object type
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type UserKeys = keyof User; // "id" | "name" | "email" | "age"

// ใช้งาน
function getUserProperty(user: User, key: UserKeys): User[typeof key] {
  return user[key];
}

const user: User = { id: 1, name: "สมชาย", email: "a@b.com", age: 25 };
const name = getUserProperty(user, "name"); // string
const age = getUserProperty(user, "age");   // number
```

### keyof กับ Generic

```typescript
// type-safe getter
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// type-safe setter
function set<T, K extends keyof T>(obj: T, key: K, value: T[K]): T {
  return { ...obj, [key]: value };
}

interface Config {
  host: string;
  port: number;
  debug: boolean;
  timeout: number;
}

const config: Config = {
  host: "localhost",
  port: 3000,
  debug: false,
  timeout: 5000
};

const host = get(config, "host");   // type: string
const port = get(config, "port");   // type: number

const newConfig = set(config, "port", 8080);  // OK
// Error: set(config, "port", "8080"); // ไม่ตรง type
```

### keyof กับ Index Signatures

```typescript
// keyof กับ index signature
interface StringMap {
  [key: string]: string;
}

type StringMapKeys = keyof StringMap; // string | number
// (number เพราะ JavaScript แปลง number key เป็น string)

interface NumberMap {
  [key: number]: string;
}

type NumberMapKeys = keyof NumberMap; // number

// keyof กับ Array
type ArrayKeys = keyof string[]; // number | "length" | "push" | "pop" | ...
```

### ตัวอย่างขั้นสูง: Object Mapper

```typescript
// Map ค่าใน object โดยรักษา key structure
function mapObjectValues<T extends object, U>(
  obj: T,
  fn: <K extends keyof T>(key: K, value: T[K]) => U
): Record<keyof T, U> {
  const result = {} as Record<keyof T, U>;
  
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      result[key as keyof T] = fn(key as keyof T, obj[key as keyof T]);
    }
  }
  
  return result;
}

interface Scores {
  math: number;
  science: number;
  english: number;
}

const scores: Scores = { math: 85, science: 90, english: 78 };

const grades = mapObjectValues(scores, (key, value) => {
  if (value >= 90) return "A";
  if (value >= 80) return "B";
  if (value >= 70) return "C";
  return "F";
});

// type: Record<"math" | "science" | "english", string>
console.log(grades); // { math: "B", science: "A", english: "C" }
```

---

## 13.2 typeof Operator

### typeof พื้นฐาน

```typescript
// typeof ดึง type จาก value
const user = {
  id: 1,
  name: "สมชาย",
  address: {
    city: "กรุงเทพ",
    zipCode: "10110"
  }
};

type UserType = typeof user;
// {
//   id: number;
//   name: string;
//   address: { city: string; zipCode: string; }
// }

// ใช้ type ที่ได้
function cloneUser(u: typeof user): typeof user {
  return { ...u, address: { ...u.address } };
}
```

### typeof กับ Functions

```typescript
// ดึง type ของฟังก์ชัน
function createUser(name: string, age: number) {
  return { id: Math.random(), name, age, createdAt: new Date() };
}

type CreateUserFn = typeof createUser;
// (name: string, age: number) => { id: number; name: string; age: number; createdAt: Date; }

// ใช้ ReturnType เพื่อดึง return type
type CreatedUser = ReturnType<typeof createUser>;
// { id: number; name: string; age: number; createdAt: Date; }
```

### typeof กับ Enums และ Constants

```typescript
// typeof กับ const object (เป็น pattern ที่ดีแทน enum)
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT"
} as const;

type DirectionType = typeof Direction;
// { readonly Up: "UP"; readonly Down: "DOWN"; ... }

type DirectionValue = (typeof Direction)[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT"

function move(direction: DirectionValue): void {
  console.log(`เคลื่อนที่ไปทาง: ${direction}`);
}

move(Direction.Up);    // OK
move("UP");            // OK
// Error: move("DIAGONAL"); // ไม่ใช่ค่าที่ถูกต้อง
```

### typeof ใน Conditional

```typescript
// ใช้ typeof ตรวจสอบ type
function processValue(value: string | number | boolean): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // TypeScript รู้ว่าเป็น string
  }
  if (typeof value === "number") {
    return value.toFixed(2); // TypeScript รู้ว่าเป็น number
  }
  return value.toString(); // TypeScript รู้ว่าเป็น boolean
}
```

---

## 13.3 Indexed Access Types (T[K])

### พื้นฐาน Indexed Access

```typescript
interface Person {
  name: string;
  age: number;
  address: {
    street: string;
    city: string;
    country: string;
  };
}

// ดึง type ของ property เฉพาะ
type PersonName = Person["name"];     // string
type PersonAge = Person["age"];       // number
type PersonAddress = Person["address"]; // { street: string; city: string; country: string; }
type PersonCity = Person["address"]["city"]; // string - nested access

// ใช้กับ Union of Keys
type NameOrAge = Person["name" | "age"]; // string | number
```

### Indexed Access กับ Array

```typescript
const products = [
  { id: 1, name: "สินค้า A", price: 100, inStock: true },
  { id: 2, name: "สินค้า B", price: 200, inStock: false }
];

// ดึง type ของ element ใน array
type Product = (typeof products)[number];
// { id: number; name: string; price: number; inStock: boolean; }

// ดึง property ของ element
type ProductId = (typeof products)[number]["id"]; // number
type ProductName = (typeof products)[number]["name"]; // string
```

### Indexed Access กับ Generic

```typescript
// ดึง type ของค่าใน array ด้วย Generics
type ElementType<T extends readonly unknown[]> = T[number];

const colors = ["red", "green", "blue"] as const;
type Color = ElementType<typeof colors>; // "red" | "green" | "blue"

// ดึง type ของ value ใน object
type ValueOf<T> = T[keyof T];

interface Theme {
  primary: string;
  secondary: string;
  background: string;
  text: string;
}

type ThemeColor = ValueOf<Theme>; // string (ในกรณีนี้ทุก value เป็น string)

// ตัวอย่างที่มีหลาย type
interface Config {
  host: string;
  port: number;
  debug: boolean;
}

type ConfigValue = ValueOf<Config>; // string | number | boolean
```

---

## 13.4 Conditional Types

### พื้นฐาน Conditional Types

```typescript
// syntax: T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false
type C = IsString<"hello">; // true (string literal เป็น subtype ของ string)
```

### Conditional Types ที่ใช้บ่อย

```typescript
// ตรวจสอบว่าเป็น Array หรือไม่
type IsArray<T> = T extends unknown[] ? true : false;

type D = IsArray<number[]>;    // true
type E = IsArray<string[]>;    // true
type F = IsArray<string>;      // false

// ดึง element type ของ Array
type UnpackArray<T> = T extends (infer U)[] ? U : T;

type G = UnpackArray<number[]>; // number
type H = UnpackArray<string>;   // string (ไม่ใช่ array คืนตัวเอง)

// Flatten array type (1 ระดับ)
type Flatten<T> = T extends Array<infer Item> ? Item : T;
```

### Non-Nullable Conditional

```typescript
// ตัดค่า null และ undefined ออก
type NonNullable<T> = T extends null | undefined ? never : T;

type I = NonNullable<string | null>;           // string
type J = NonNullable<number | undefined>;       // number
type K = NonNullable<string | null | undefined>; // string
```

### Conditional Types กับ Union (Distributive)

```typescript
// Conditional types กระจายผ่าน Union โดยอัตโนมัติ
type ToArray<T> = T extends unknown ? T[] : never;

type L = ToArray<string | number>; // string[] | number[]
// ไม่ใช่ (string | number)[] - มันกระจาย!

// ป้องกันการกระจายด้วย tuple syntax
type ToArrayNonDistributive<T> = [T] extends [unknown] ? T[] : never;
type M = ToArrayNonDistributive<string | number>; // (string | number)[]
```

---

## 13.5 infer Keyword

### infer พื้นฐาน

```typescript
// infer สร้าง type variable ใน conditional type
type ReturnType<T> = T extends (...args: unknown[]) => infer R ? R : never;

function greet(name: string): string {
  return `สวัสดี ${name}`;
}

type GreetReturn = ReturnType<typeof greet>; // string

// Parameters type
type Parameters<T> = T extends (...args: infer P) => unknown ? P : never;

function createUser(name: string, age: number, email: string): void {}

type CreateUserParams = Parameters<typeof createUser>; // [string, number, string]
```

### infer กับ Nested Types

```typescript
// ดึง Promise value type
type Awaited<T> = T extends Promise<infer V> ? Awaited<V> : T;

type N = Awaited<Promise<string>>;           // string
type O = Awaited<Promise<Promise<number>>>;  // number
type P = Awaited<string>;                    // string

// ดึง Array element type
type ArrayElement<T> = T extends (infer E)[] ? E : never;

type Q = ArrayElement<number[]>; // number
type R = ArrayElement<string[]>; // string

// ดึง first argument type
type FirstArgument<T> = T extends (first: infer A, ...rest: unknown[]) => unknown ? A : never;

function test(x: number, y: string, z: boolean): void {}
type TestFirstArg = FirstArgument<typeof test>; // number
```

### infer กับ Object Types

```typescript
// ดึง constructor parameter types
type ConstructorParameters<T extends new (...args: unknown[]) => unknown> =
  T extends new (...args: infer P) => unknown ? P : never;

class UserService {
  constructor(private db: string, private cache: boolean) {}
}

type UserServiceParams = ConstructorParameters<typeof UserService>; // [string, boolean]

// ดึง instance type
type InstanceType<T extends new (...args: unknown[]) => unknown> =
  T extends new (...args: unknown[]) => infer R ? R : never;

type UserServiceInstance = InstanceType<typeof UserService>; // UserService
```

---

## 13.6 Distributive Conditional Types

### การกระจายแบบ Distributive

```typescript
// Union ถูกกระจาย (distributed) ผ่าน conditional types
type StringOrNumber<T> = T extends string ? "string" : "number";

type Result1 = StringOrNumber<string>;         // "string"
type Result2 = StringOrNumber<number>;         // "number"
type Result3 = StringOrNumber<string | number>; // "string" | "number" - กระจาย!

// ตัวอย่างการใช้งาน
type Nullable<T> = T extends unknown ? T | null : never;

type NullableString = Nullable<string>;          // string | null
type NullableUnion = Nullable<string | number>;  // string | null | number | null = string | number | null
```

### หยุดการกระจาย

```typescript
// ใช้ [] เพื่อป้องกันการกระจาย
type NoDistribute<T> = [T] extends [unknown] ? T[] : never;

type R1 = NoDistribute<string | number>; // (string | number)[]

// ตัวอย่าง: IsUnion
type IsUnion<T, U extends T = T> =
  T extends unknown
    ? ([U] extends [T] ? false : true)
    : never;

type CheckUnion1 = IsUnion<string | number>; // boolean (true)
type CheckUnion2 = IsUnion<string>;          // false
```

### Filter Types

```typescript
// กรองออก types ที่ไม่ต้องการ
type Filter<T, U> = T extends U ? T : never;
type Exclude<T, U> = T extends U ? never : T;

type OnlyStrings = Filter<string | number | boolean, string>; // string
type NoStrings = Exclude<string | number | boolean, string>;  // number | boolean

// Extract strings from union
type StringsOnly<T> = T extends string ? T : never;
type Test = StringsOnly<string | number | string[]>; // string
```

---

## 13.7 Template Literal Types

### พื้นฐาน Template Literal

```typescript
// สร้าง string type ด้วย template literal
type Greeting = `สวัสดี ${string}`;

const g1: Greeting = "สวัสดี สมชาย";   // OK
const g2: Greeting = "สวัสดี TypeScript"; // OK
// Error: const g3: Greeting = "ไม่ใช่ greeting";

// รวม literal types
type Color = "red" | "blue" | "green";
type Size = "sm" | "md" | "lg";
type ButtonClass = `btn-${Color}-${Size}`;
// "btn-red-sm" | "btn-red-md" | ... (9 combinations)
```

### Template Literal Utilities

```typescript
// TypeScript built-in string manipulation types
type Upper = Uppercase<"hello">;       // "HELLO"
type Lower = Lowercase<"HELLO">;       // "hello"
type Capitalized = Capitalize<"hello">; // "Hello"
type Uncapitalized = Uncapitalize<"Hello">; // "hello"

// สร้าง getter/setter names
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface User {
  name: string;
  age: number;
  email: string;
}

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; getEmail: () => string; }

type UserSetters = Setters<User>;
// { setName: (value: string) => void; ... }
```

### Template Literal ขั้นสูง

```typescript
// Event handler naming
type EventName = "click" | "focus" | "blur" | "change";
type EventHandler = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur" | "onChange"

// CSS class generation
type Modifier = "hover" | "focus" | "active" | "disabled";
type BaseClass = "btn" | "input" | "card";
type ModifiedClass = `${BaseClass}:${Modifier}`;
// "btn:hover" | "btn:focus" | ... (12 combinations)

// API Route types
type HttpMethod = "get" | "post" | "put" | "delete" | "patch";
type ApiPath = `/api/${string}`;
type ApiEndpoint = `${Uppercase<HttpMethod>} ${ApiPath}`;
// "GET /api/..." | "POST /api/..." | ...

// SQL Column naming
type TableName = "user" | "product" | "order";
type ColumnPrefix<T extends string> = `${T}_id` | `${T}_created_at` | `${T}_updated_at`;
type UserColumns = ColumnPrefix<"user">;
// "user_id" | "user_created_at" | "user_updated_at"
```

---

## 13.8 Mapped Types

### Mapped Types พื้นฐาน

```typescript
// แปลง properties ทั้งหมดเป็น optional
type Optional<T> = {
  [K in keyof T]?: T[K];
};

// แปลง properties ทั้งหมดเป็น readonly
type ReadonlyAll<T> = {
  readonly [K in keyof T]: T[K];
};

// แปลงค่าทั้งหมดเป็น string
type Stringify<T> = {
  [K in keyof T]: string;
};

interface Config {
  host: string;
  port: number;
  debug: boolean;
}

type OptionalConfig = Optional<Config>;
// { host?: string; port?: number; debug?: boolean; }

type ReadonlyConfig = ReadonlyAll<Config>;
// { readonly host: string; readonly port: number; readonly debug: boolean; }
```

### Mapped Types กับ Key Remapping

```typescript
// เปลี่ยนชื่อ key ด้วย as
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

// กรอง keys ด้วย never
type OmitNever<T> = {
  [K in keyof T as T[K] extends never ? never : K]: T[K];
};

// แปลงเป็น function types
type FunctionValues<T> = {
  [K in keyof T]: T[K] extends (...args: unknown[]) => unknown ? T[K] : never;
};

// กรองเฉพาะ method
type Methods<T> = {
  [K in keyof T as T[K] extends (...args: unknown[]) => unknown ? K : never]: T[K];
};

class Calculator {
  value: number = 0;
  add(n: number): this { this.value += n; return this; }
  subtract(n: number): this { this.value -= n; return this; }
  multiply(n: number): this { this.value *= n; return this; }
  getResult(): number { return this.value; }
}

type CalculatorMethods = Methods<Calculator>;
// { add: ..., subtract: ..., multiply: ..., getResult: ... }
// value ถูกตัดออก เพราะไม่ใช่ function
```

### Mapped Types ซับซ้อน

```typescript
// Deep Partial
type DeepPartial<T> = T extends object ? {
  [K in keyof T]?: DeepPartial<T[K]>;
} : T;

interface AppState {
  user: {
    id: number;
    profile: {
      name: string;
      avatar: string;
    };
  };
  settings: {
    theme: "light" | "dark";
    language: string;
  };
}

type PartialAppState = DeepPartial<AppState>;
// ทุก property และ nested property กลายเป็น optional

const partialState: PartialAppState = {
  user: {
    profile: {
      name: "สมชาย"
      // avatar ไม่จำเป็น
    }
    // id ไม่จำเป็น
  }
  // settings ไม่จำเป็น
};
```

---

## 13.9 Utility Types เจาะลึก

### Partial<T> และ Required<T>

```typescript
interface UserProfile {
  id: number;
  name: string;
  email: string;
  bio?: string;
  avatar?: string;
  phoneNumber?: string;
}

// Partial ทำให้ทุก property เป็น optional
type UpdateUserInput = Partial<UserProfile>;
// ใช้สำหรับ PATCH requests

// Required ทำให้ทุก property เป็น required
type CompleteProfile = Required<UserProfile>;
// { id: number; name: string; email: string; bio: string; ... }

// ใช้งานจริง
async function updateUser(id: number, updates: Partial<UserProfile>): Promise<UserProfile> {
  // อัปเดตเฉพาะ fields ที่ส่งมา
  return { id, name: "", email: "", ...updates };
}
```

### Readonly<T>

```typescript
interface MutableConfig {
  host: string;
  port: number;
  options: {
    timeout: number;
    retries: number;
  };
}

type ImmutableConfig = Readonly<MutableConfig>;
// ไม่สามารถแก้ไข host, port, options ได้
// แต่ options.timeout ยังแก้ไขได้ (shallow readonly!)

const config: ImmutableConfig = {
  host: "localhost",
  port: 3000,
  options: { timeout: 5000, retries: 3 }
};

// config.host = "remote"; // Error!
config.options.timeout = 10000; // OK! (shallow readonly)

// Deep Readonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

type TrulyImmutableConfig = DeepReadonly<MutableConfig>;
const frozenConfig: TrulyImmutableConfig = {
  host: "localhost",
  port: 3000,
  options: { timeout: 5000, retries: 3 }
};

// frozenConfig.options.timeout = 10000; // Error! now
```

### Record<K, V>

```typescript
// Record สร้าง object type ที่มี keys เป็น K และ values เป็น V
type StringRecord = Record<string, string>;
type NumberRecord = Record<string, number>;

// กำหนด keys เฉพาะ
type UserRoles = Record<"admin" | "user" | "guest", string[]>;
const rolePermissions: UserRoles = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  guest: ["read"]
};

// ใช้กับ mapped types
type StatusMap = Record<"active" | "inactive" | "pending", {
  label: string;
  color: string;
}>;

const statuses: StatusMap = {
  active: { label: "ใช้งาน", color: "green" },
  inactive: { label: "ไม่ใช้งาน", color: "red" },
  pending: { label: "รอดำเนินการ", color: "yellow" }
};

// สร้าง index จาก array
function indexBy<T, K extends keyof T>(
  items: T[],
  key: K
): Record<string, T> {
  return items.reduce((acc, item) => {
    acc[String(item[key])] = item;
    return acc;
  }, {} as Record<string, T>);
}

const users = [
  { id: 1, name: "สมชาย" },
  { id: 2, name: "สมหญิง" }
];

const usersById = indexBy(users, "id");
// { "1": { id: 1, name: "สมชาย" }, "2": { id: 2, name: "สมหญิง" } }
```

### Pick<T, K> และ Omit<T, K>

```typescript
interface FullUser {
  id: number;
  name: string;
  email: string;
  password: string;
  role: string;
  createdAt: Date;
  lastLogin: Date;
}

// Pick - เลือกเฉพาะ properties ที่ต้องการ
type PublicUser = Pick<FullUser, "id" | "name" | "email" | "role">;
type UserCredentials = Pick<FullUser, "email" | "password">;

// Omit - ตัด properties ที่ไม่ต้องการออก
type SafeUser = Omit<FullUser, "password">;
type UserWithoutDates = Omit<FullUser, "createdAt" | "lastLogin">;

// ใช้งาน
function createSafeResponse(user: FullUser): SafeUser {
  const { password, ...safeUser } = user;
  return safeUser;
}

// Pick ใช้สำหรับ DTO (Data Transfer Objects)
type CreateUserDTO = Pick<FullUser, "name" | "email" | "password">;
type UpdateUserDTO = Partial<Pick<FullUser, "name" | "email">>;

async function createUser(dto: CreateUserDTO): Promise<SafeUser> {
  // hash password, save to DB, etc.
  return {
    id: 1,
    name: dto.name,
    email: dto.email,
    role: "user",
    createdAt: new Date(),
    lastLogin: new Date()
  };
}
```

### Exclude<T, U> และ Extract<T, U>

```typescript
// Exclude - เอาออกจาก union
type Numbers = Exclude<string | number | boolean | null, string | boolean>;
// number | null

type NonNullable<T> = Exclude<T, null | undefined>;

type Status = "active" | "inactive" | "pending" | "deleted";
type ActiveStatus = Exclude<Status, "deleted" | "inactive">;
// "active" | "pending"

// Extract - เอาเฉพาะที่ตรงกัน
type OnlyStrings = Extract<string | number | boolean, string | boolean>;
// string | boolean (เอาเฉพาะที่อยู่ใน second type ด้วย)

type ActiveOrPending = Extract<Status, "active" | "pending" | "archived">;
// "active" | "pending"
```

### ReturnType<T> และ Parameters<T>

```typescript
// ReturnType
function fetchUser(): Promise<{ id: number; name: string }> {
  return Promise.resolve({ id: 1, name: "สมชาย" });
}

type FetchUserReturn = ReturnType<typeof fetchUser>;
// Promise<{ id: number; name: string }>

type UnwrappedReturn = Awaited<ReturnType<typeof fetchUser>>;
// { id: number; name: string }

// Parameters
function sendEmail(to: string, subject: string, body: string, cc?: string[]): void {}

type SendEmailParams = Parameters<typeof sendEmail>;
// [to: string, subject: string, body: string, cc?: string[] | undefined]

type FirstParam = Parameters<typeof sendEmail>[0]; // string
type ThirdParam = Parameters<typeof sendEmail>[2]; // string

// ใช้ Parameters สำหรับ wrapper functions
function logAndCall<T extends (...args: unknown[]) => unknown>(
  fn: T,
  ...args: Parameters<T>
): ReturnType<T> {
  console.log(`เรียกฟังก์ชัน: ${fn.name}`);
  return fn(...args) as ReturnType<T>;
}
```

### InstanceType<T>

```typescript
class DatabaseConnection {
  private connected = false;
  
  connect(host: string): void {
    this.connected = true;
    console.log(`เชื่อมต่อกับ ${host}`);
  }
  
  disconnect(): void {
    this.connected = false;
  }
  
  isConnected(): boolean {
    return this.connected;
  }
}

type DbConnection = InstanceType<typeof DatabaseConnection>;
// DatabaseConnection

// ใช้งาน
function useConnection<T extends new (...args: unknown[]) => unknown>(
  Constructor: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new Constructor(...args) as InstanceType<T>;
}
```

---

## 13.10 Custom Utility Types

### Flatten

```typescript
// Flatten nested object type
type Flatten<T, Prefix extends string = ""> = {
  [K in keyof T as T[K] extends object
    ? never
    : Prefix extends ""
    ? string & K
    : `${Prefix}.${string & K}`
  ]: T[K];
} & {
  [K in keyof T as T[K] extends object
    ? keyof Flatten<T[K], Prefix extends "" ? string & K : `${Prefix}.${string & K}`>
    : never
  ]: T[K] extends object ? Flatten<T[K]>[keyof Flatten<T[K]>] : never;
};
```

### Mutable<T>

```typescript
// ตรงข้ามของ Readonly
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

interface ImmutablePoint {
  readonly x: number;
  readonly y: number;
}

type MutablePoint = Mutable<ImmutablePoint>;
// { x: number; y: number; } - ไม่มี readonly แล้ว

const point: MutablePoint = { x: 0, y: 0 };
point.x = 5; // OK
```

### OptionalKeys และ RequiredKeys

```typescript
// ดึง keys ที่เป็น optional
type OptionalKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? K : never;
}[keyof T];

// ดึง keys ที่เป็น required
type RequiredKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? never : K;
}[keyof T];

interface UserForm {
  name: string;
  email: string;
  phone?: string;
  bio?: string;
}

type RequiredFormKeys = RequiredKeys<UserForm>; // "name" | "email"
type OptionalFormKeys = OptionalKeys<UserForm>; // "phone" | "bio"
```

### WritableKeys

```typescript
// ดึง keys ที่เขียนได้ (ไม่ใช่ readonly)
type WritableKeys<T> = {
  [K in keyof T]-?: Equal<
    { [P in K]: T[P] },
    { -readonly [P in K]: T[P] }
  > extends true ? K : never;
}[keyof T];

// Helper type
type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends
  (<T>() => T extends Y ? 1 : 2) ? true : false;
```

### Overwrite<T, U>

```typescript
// แทนที่ properties บางส่วนของ T ด้วย U
type Overwrite<T, U extends Partial<T>> = Omit<T, keyof U> & U;

interface BaseUser {
  id: number;
  name: string;
  role: string;
  createdAt: Date;
}

type AdminUser = Overwrite<BaseUser, { role: "admin" }>;
// { id: number; name: string; createdAt: Date; role: "admin"; }

const admin: AdminUser = {
  id: 1,
  name: "ผู้ดูแล",
  role: "admin", // ต้องเป็น "admin" เท่านั้น
  createdAt: new Date()
};
```

### XOR (Exclusive Or)

```typescript
// ต้องมีอย่างใดอย่างหนึ่ง แต่ไม่ใช่ทั้งคู่
type Without<T, U> = { [P in Exclude<keyof T, keyof U>]?: never };
type XOR<T, U> = (T | U) extends object
  ? (Without<T, U> & U) | (Without<U, T> & T)
  : T | U;

// ตัวอย่าง: Login ด้วย email หรือ username แต่ไม่ใช่ทั้งคู่
type EmailLogin = { email: string; password: string };
type UsernameLogin = { username: string; password: string };
type LoginInput = XOR<EmailLogin, UsernameLogin>;

const emailLogin: LoginInput = { email: "a@b.com", password: "123" }; // OK
const usernameLogin: LoginInput = { username: "somchai", password: "123" }; // OK
// Error: const bothLogin: LoginInput = { email: "a@b.com", username: "somchai", password: "123" };
```

---

## 13.11 ตัวอย่างขั้นสูง

### Type-Safe Event System

```typescript
type EventMap = Record<string, unknown>;

type EventListener<T> = (event: T) => void | Promise<void>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners = new Map<
    keyof Events,
    Set<EventListener<Events[keyof Events]>>
  >();

  on<K extends keyof Events>(
    event: K,
    listener: EventListener<Events[K]>
  ): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener as EventListener<Events[keyof Events]>);
    
    // คืน unsubscribe function
    return () => this.off(event, listener);
  }

  off<K extends keyof Events>(
    event: K,
    listener: EventListener<Events[K]>
  ): void {
    this.listeners.get(event)?.delete(listener as EventListener<Events[keyof Events]>);
  }

  async emit<K extends keyof Events>(event: K, data: Events[K]): Promise<void> {
    const handlers = this.listeners.get(event);
    if (handlers) {
      await Promise.all([...handlers].map(handler => handler(data)));
    }
  }

  once<K extends keyof Events>(
    event: K,
    listener: EventListener<Events[K]>
  ): void {
    const wrapper: EventListener<Events[K]> = async (data) => {
      await listener(data);
      this.off(event, wrapper);
    };
    this.on(event, wrapper);
  }
}

// ใช้งาน
interface ShopEvents {
  orderCreated: { orderId: string; total: number; userId: string };
  orderShipped: { orderId: string; trackingNumber: string };
  orderDelivered: { orderId: string; deliveredAt: Date };
  paymentFailed: { orderId: string; reason: string };
}

const shopEmitter = new TypedEventEmitter<ShopEvents>();

const unsubscribe = shopEmitter.on("orderCreated", async (event) => {
  console.log(`คำสั่งซื้อ ${event.orderId} มูลค่า ${event.total} บาท`);
  // ส่ง email แจ้ง user
});

await shopEmitter.emit("orderCreated", {
  orderId: "ORD-001",
  total: 1500,
  userId: "USR-001"
});

// ยกเลิก listener เมื่อไม่ต้องการ
unsubscribe();
```

### Builder Pattern กับ Generics

```typescript
type BuilderState<T> = {
  [K in keyof T]: T[K] | undefined;
};

class Builder<T extends object, Built extends Partial<T> = {}> {
  private state: BuilderState<T>;

  constructor(state?: BuilderState<T>) {
    this.state = state ?? ({} as BuilderState<T>);
  }

  set<K extends keyof T>(
    key: K,
    value: T[K]
  ): Builder<T, Built & Pick<T, K>> {
    return new Builder({ ...this.state, [key]: value } as BuilderState<T>);
  }

  build(this: Builder<T, T>): T {
    return this.state as T;
  }
}

interface PersonData {
  name: string;
  age: number;
  email: string;
}

const person = new Builder<PersonData>()
  .set("name", "สมชาย")
  .set("age", 25)
  .set("email", "somchai@example.com")
  .build();
// person มี type PersonData
```

---

## 13.12 Pattern Matching กับ Conditional Types

```typescript
// Pattern matching ด้วย conditional types
type UnwrapPromise<T> = T extends Promise<infer U> ? UnwrapPromise<U> : T;

type Match<T, Pattern extends [unknown, unknown][]> = {
  [K in keyof Pattern]: Pattern[K] extends [infer When, infer Then]
    ? T extends When ? Then : never
    : never;
}[number];

// เทียบกับ switch statement สำหรับ types
type HttpStatusMessage = Match<
  number,
  [
    [200, "OK"],
    [201, "Created"],
    [400, "Bad Request"],
    [401, "Unauthorized"],
    [403, "Forbidden"],
    [404, "Not Found"],
    [500, "Internal Server Error"]
  ]
>;
// "OK" | "Created" | "Bad Request" | ...

// ตัวอย่างที่ใช้งานได้จริง
type IsNever<T> = [T] extends [never] ? true : false;
type IsAny<T> = 0 extends (1 & T) ? true : false;
type IsUnknown<T> = IsNever<T> extends false
  ? T extends unknown
    ? unknown extends T
      ? IsAny<T> extends false
        ? true
        : false
      : false
    : false
  : false;
```

---

## สรุปบทนี้

ในบทนี้เราได้เรียนรู้:

- **keyof** ดึง union ของ keys จาก type
- **typeof** ดึง type จาก value หรือ variable
- **Indexed Access (T[K])** ดึง type ของ property
- **Conditional Types** สร้าง type แบบมีเงื่อนไข
- **infer** สร้าง type variable ภายใน conditional type
- **Distributive Conditional** การกระจายผ่าน union
- **Template Literal Types** สร้าง string types
- **Mapped Types** แปลง object types
- **Utility Types** เครื่องมือสร้าง types สำเร็จรูป
- **Custom Utility Types** สร้าง utility types เอง

บทถัดไปจะเรียนรู้ Type Guards & Narrowing
