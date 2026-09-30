# ส่วนที่ 9: Type Aliases ใน TypeScript

## บทนำ

Type Alias คือการสร้างชื่อใหม่ให้กับ type ที่มีอยู่แล้ว ทำให้โค้ดอ่านง่ายขึ้นและสามารถนำ type กลับมาใช้ใหม่ได้ Type Alias ใช้คีย์เวิร์ด `type` และสามารถใช้กับ primitive types, object types, union types, intersection types และอื่นๆ อีกมากมาย

ในบทนี้เราจะเรียนรู้:
- การประกาศ Type Alias
- Complex Type Compositions
- Type Aliases สำหรับ Functions
- Recursive Type Aliases
- Utility Types เบื้องต้น
- เมื่อไหร่ควรใช้ Type Alias แทน Interface
- Generic Type Aliases
- Template Literal Type Aliases

---

## 9.1 การประกาศ Type Alias พื้นฐาน

```typescript
// Type Alias สำหรับ primitive types
type UserID = number;
type ProductName = string;
type IsActive = boolean;

const userId: UserID = 123;
const productName: ProductName = "TypeScript Handbook";
const isActive: IsActive = true;
```

```typescript
// Type Alias สำหรับ Object Types
type Point = {
  x: number;
  y: number;
};

type Color = {
  red: number;
  green: number;
  blue: number;
  alpha?: number;
};

const origin: Point = { x: 0, y: 0 };
const red: Color = { red: 255, green: 0, blue: 0 };
const transparentBlue: Color = { red: 0, green: 0, blue: 255, alpha: 0.5 };

console.log(`Point: (${origin.x}, ${origin.y})`);
console.log(`Color: rgb(${red.red}, ${red.green}, ${red.blue})`);
```

```typescript
// Type Alias สำหรับ Union Types
type StringOrNumber = string | number;
type NullableString = string | null;
type OptionalBoolean = boolean | undefined;

function printValue(value: StringOrNumber): void {
  if (typeof value === "string") {
    console.log(`String: "${value}"`);
  } else {
    console.log(`Number: ${value}`);
  }
}

printValue("สวัสดี");  // String: "สวัสดี"
printValue(42);         // Number: 42
```

```typescript
// Type Alias สำหรับ Tuple Types
type Coordinate = [number, number];
type RGB = [number, number, number];
type NameAge = [string, number];
type KeyValuePair = [string, unknown];

const position: Coordinate = [10.5, 20.3];
const color: RGB = [255, 128, 0];
const person: NameAge = ["สมชาย", 30];

const [x, y] = position;
const [r, g, b] = color;
const [name, age] = person;

console.log(`ตำแหน่ง: (${x}, ${y})`);
console.log(`สี: rgb(${r}, ${g}, ${b})`);
console.log(`${name} อายุ ${age} ปี`);
```

---

## 9.2 Type Alias สำหรับ Literal Types

```typescript
// String Literal Types
type Direction = "north" | "south" | "east" | "west";
type CardinalDirection = "N" | "S" | "E" | "W" | "NE" | "NW" | "SE" | "SW";
type Status = "active" | "inactive" | "pending" | "suspended";
type Theme = "light" | "dark" | "system";

function movePlayer(direction: Direction, steps: number): void {
  console.log(`ย้ายผู้เล่น ${steps} ก้าวไปทาง ${direction}`);
}

movePlayer("north", 5); // OK
movePlayer("up", 3);    // Error! ไม่ใช่ Direction ที่ถูกต้อง
```

```typescript
// Numeric Literal Types
type DiceValue = 1 | 2 | 3 | 4 | 5 | 6;
type HttpStatusCode = 200 | 201 | 204 | 400 | 401 | 403 | 404 | 500;
type Month = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12;

function rollDice(): DiceValue {
  return (Math.floor(Math.random() * 6) + 1) as DiceValue;
}

const roll = rollDice();
console.log(`ทอยลูกเต๋าได้: ${roll}`);

function getMonthName(month: Month): string {
  const months = [
    "มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน",
    "พฤษภาคม", "มิถุนายน", "กรกฎาคม", "สิงหาคม",
    "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"
  ];
  return months[month - 1];
}

console.log(getMonthName(1));  // มกราคม
console.log(getMonthName(12)); // ธันวาคม
```

---

## 9.3 Complex Type Compositions

```typescript
// การรวม Type Aliases
type Name = {
  firstName: string;
  lastName: string;
  middleName?: string;
};

type ContactInfo = {
  email: string;
  phone?: string;
  website?: string;
};

type Address = {
  street: string;
  city: string;
  province: string;
  zipCode: string;
  country: string;
};

// รวม Types ด้วย Intersection (&)
type FullPerson = Name & ContactInfo & {
  id: number;
  dateOfBirth: Date;
  address: Address;
};

const employee: FullPerson = {
  id: 1,
  firstName: "สมชาย",
  lastName: "ใจดี",
  email: "somchai@company.com",
  phone: "081-234-5678",
  dateOfBirth: new Date("1990-05-15"),
  address: {
    street: "123 ถนนสุขุมวิท",
    city: "กรุงเทพมหานคร",
    province: "กรุงเทพมหานคร",
    zipCode: "10110",
    country: "Thailand"
  }
};
```

```typescript
// Conditional Types
type IsString<T> = T extends string ? true : false;
type IsArray<T> = T extends unknown[] ? true : false;

type CheckString = IsString<"hello">;   // true
type CheckNumber = IsString<42>;        // false
type CheckArray = IsArray<number[]>;    // true
type CheckObject = IsArray<object>;     // false

// Infer keyword ใน Conditional Types
type GetReturnType<T> = T extends (...args: unknown[]) => infer R ? R : never;
type GetFirstArg<T> = T extends (first: infer F, ...rest: unknown[]) => unknown ? F : never;

type StringReturn = GetReturnType<() => string>;        // string
type NumberReturn = GetReturnType<() => number>;        // number
type FirstArgType = GetFirstArg<(x: number, y: string) => void>; // number
```

```typescript
// Mapped Types
type Optional<T> = {
  [K in keyof T]?: T[K];
};

type Required<T> = {
  [K in keyof T]-?: T[K];
};

type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// การใช้งาน
type UserUpdate = Optional<FullPerson>;
// ทุก field ของ FullPerson จะกลายเป็น optional

type ImmutablePoint = Readonly<Point>;
// ทุก field ของ Point จะกลายเป็น readonly

const p: ImmutablePoint = { x: 1, y: 2 };
// p.x = 3; // Error!
```

---

## 9.4 Type Aliases สำหรับ Functions

```typescript
// Function Type Aliases
type Callback = () => void;
type StringCallback = (value: string) => void;
type ErrorCallback = (error: Error) => void;
type AsyncCallback<T> = (value: T) => Promise<void>;

// Higher-order function types
type Transformer<T, U> = (input: T) => U;
type Predicate<T> = (value: T) => boolean;
type Reducer<T, U> = (accumulator: U, current: T) => U;
type Comparator<T> = (a: T, b: T) => number;

// ตัวอย่างการใช้งาน
const doubleNumber: Transformer<number, number> = (n) => n * 2;
const toString: Transformer<number, string> = (n) => n.toString();
const isEven: Predicate<number> = (n) => n % 2 === 0;
const sum: Reducer<number, number> = (acc, curr) => acc + curr;

const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(doubleNumber);
const evens = numbers.filter(isEven);
const total = numbers.reduce(sum, 0);

console.log("ตัวเลขสองเท่า:", doubled);   // [2, 4, 6, 8, 10]
console.log("เลขคู่:", evens);              // [2, 4]
console.log("ผลรวม:", total);               // 15
```

```typescript
// Event Handler Types
type MouseEventHandler = (event: MouseEvent) => void;
type KeyboardEventHandler = (event: KeyboardEvent) => void;
type ChangeEventHandler<T extends HTMLElement> = (event: Event & { target: T }) => void;

// Middleware Types
type Middleware<T, U = T> = (data: T, next: (data: T) => U) => U;
type AsyncMiddleware<T, U = T> = (data: T, next: (data: T) => Promise<U>) => Promise<U>;

// API Handler Types
type RequestHandler<TBody = unknown, TResponse = unknown> = (
  body: TBody,
  params: Record<string, string>,
  query: Record<string, string>
) => Promise<TResponse>;

// ตัวอย่าง createUser handler
type CreateUserBody = {
  name: string;
  email: string;
  password: string;
};

type CreateUserResponse = {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
};

const createUser: RequestHandler<CreateUserBody, CreateUserResponse> = async (
  body,
  params,
  query
) => {
  // จำลองการสร้าง user
  return {
    id: Math.floor(Math.random() * 1000),
    name: body.name,
    email: body.email,
    createdAt: new Date()
  };
};
```

```typescript
// Curried Function Types
type Curried<T, U, V> = (a: T) => (b: U) => V;
type CurriedAdd = Curried<number, number, number>;
type CurriedTemplate = Curried<string, string, string>;

const add: CurriedAdd = (a) => (b) => a + b;
const add5 = add(5);
console.log(add5(3));  // 8
console.log(add5(10)); // 15

const greet: CurriedTemplate = (greeting) => (name) => `${greeting}, ${name}!`;
const sayHello = greet("สวัสดี");
console.log(sayHello("สมชาย")); // สวัสดี, สมชาย!
console.log(sayHello("สมหญิง")); // สวัสดี, สมหญิง!
```

---

## 9.5 Recursive Type Aliases

```typescript
// Recursive Type Alias สำหรับ JSON
type JSONValue = 
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const jsonData: JSONValue = {
  name: "John",
  age: 30,
  active: true,
  address: {
    street: "123 Main St",
    city: "Bangkok",
    coords: [13.7563, 100.5018]
  },
  tags: ["developer", "typescript"],
  metadata: null
};

function countJSONKeys(value: JSONValue): number {
  if (typeof value === "object" && value !== null && !Array.isArray(value)) {
    return Object.keys(value).length + 
      Object.values(value).reduce((sum, v) => sum + countJSONKeys(v), 0);
  }
  return 0;
}
```

```typescript
// Recursive Type สำหรับ Tree Structure
type TreeNode<T> = {
  value: T;
  children?: TreeNode<T>[];
};

type NumberTree = TreeNode<number>;
type StringTree = TreeNode<string>;

const fileSystem: TreeNode<string> = {
  value: "root",
  children: [
    {
      value: "src",
      children: [
        { value: "index.ts" },
        { value: "app.ts" },
        {
          value: "components",
          children: [
            { value: "Button.tsx" },
            { value: "Input.tsx" }
          ]
        }
      ]
    },
    {
      value: "public",
      children: [
        { value: "index.html" },
        { value: "styles.css" }
      ]
    }
  ]
};

function printTree(node: TreeNode<string>, indent = 0): void {
  console.log(" ".repeat(indent * 2) + node.value);
  node.children?.forEach(child => printTree(child, indent + 1));
}

printTree(fileSystem);
```

```typescript
// Recursive Type สำหรับ Linked List
type LinkedListNode<T> = {
  value: T;
  next: LinkedListNode<T> | null;
};

type LinkedList<T> = {
  head: LinkedListNode<T> | null;
  size: number;
};

function createLinkedList<T>(): LinkedList<T> {
  return { head: null, size: 0 };
}

function prepend<T>(list: LinkedList<T>, value: T): LinkedList<T> {
  return {
    head: { value, next: list.head },
    size: list.size + 1
  };
}

function toArray<T>(list: LinkedList<T>): T[] {
  const result: T[] = [];
  let current = list.head;
  while (current !== null) {
    result.push(current.value);
    current = current.next;
  }
  return result;
}

let list = createLinkedList<number>();
list = prepend(list, 3);
list = prepend(list, 2);
list = prepend(list, 1);

console.log(toArray(list)); // [1, 2, 3]
```

```typescript
// Recursive Type สำหรับ Deep Partial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object 
    ? DeepPartial<T[K]> 
    : T[K];
};

type UserConfig = {
  profile: {
    name: string;
    avatar: string;
    bio: string;
  };
  settings: {
    theme: string;
    language: string;
    notifications: {
      email: boolean;
      push: boolean;
      sms: boolean;
    };
  };
};

// DeepPartial ทำให้ทุก nested field เป็น optional
type UserConfigUpdate = DeepPartial<UserConfig>;

const update: UserConfigUpdate = {
  settings: {
    theme: "dark",
    notifications: {
      email: true
      // push และ sms ไม่จำเป็นต้องระบุ
    }
  }
};
```

---

## 9.6 Utility Types Preview

TypeScript มี built-in utility types ที่ช่วยให้เราสร้าง types ใหม่จาก types ที่มีอยู่

```typescript
// Partial<T> - ทำให้ทุก property เป็น optional
interface UpdateUserDTO {
  name?: string;
  email?: string;
  phone?: string;
}
// เทียบเท่ากับ
type UpdateUserDTO2 = Partial<FullPerson>;

// Required<T> - ทำให้ทุก property เป็น required
type RequiredUserDTO = Required<UpdateUserDTO>;

// Readonly<T> - ทำให้ทุก property เป็น readonly
type ImmutableUser = Readonly<FullPerson>;

// Pick<T, K> - เลือกเฉพาะ properties ที่ต้องการ
type UserSummary = Pick<FullPerson, "id" | "firstName" | "lastName" | "email">;

// Omit<T, K> - ยกเว้น properties ที่ไม่ต้องการ
type UserWithoutPassword = Omit<FullPerson, "password">;

// Record<K, V> - สร้าง object type จาก keys และ values
type UserRoles = Record<string, string[]>;
type ErrorMessages = Record<string, string>;
type ConfigMap = Record<string, string | number | boolean>;

const roles: UserRoles = {
  admin: ["read", "write", "delete"],
  user: ["read"],
  moderator: ["read", "write"]
};
```

```typescript
// Exclude<T, U> - ยกเว้น type ออกจาก union
type AllColors = "red" | "green" | "blue" | "yellow" | "purple";
type PrimaryColors = Exclude<AllColors, "yellow" | "purple">;
// PrimaryColors = "red" | "green" | "blue"

// Extract<T, U> - เอาเฉพาะ type ที่ต้องการจาก union
type OnlyStringOrNumber = Extract<string | number | boolean | null, string | number>;
// OnlyStringOrNumber = string | number

// NonNullable<T> - ยกเว้น null และ undefined
type NotNullString = NonNullable<string | null | undefined>;
// NotNullString = string

// ReturnType<T> - ได้ return type ของ function
function getUserById(id: number) {
  return { id, name: "test", email: "test@test.com" };
}
type UserFromDB = ReturnType<typeof getUserById>;
// UserFromDB = { id: number; name: string; email: string; }

// Parameters<T> - ได้ parameter types ของ function
type GetUserParams = Parameters<typeof getUserById>;
// GetUserParams = [id: number]
```

```typescript
// InstanceType<T> - ได้ type ของ instance จาก constructor
class ApiClient {
  constructor(
    private baseUrl: string,
    private apiKey: string
  ) {}
  
  get(path: string): Promise<unknown> {
    return fetch(`${this.baseUrl}${path}`);
  }
}

type ApiClientInstance = InstanceType<typeof ApiClient>;
// เทียบเท่ากับ ApiClient

// Awaited<T> - ได้ type ที่ resolve แล้วจาก Promise
type UserFromApi = Awaited<Promise<FullPerson>>;
// UserFromApi = FullPerson

type NestedPromise = Awaited<Promise<Promise<string>>>;
// NestedPromise = string
```

---

## 9.7 เมื่อไหร่ควรใช้ Type Alias แทน Interface

```typescript
// ============= ใช้ Type Alias เมื่อ =============

// 1. Union Types
type Result<T> = { success: true; data: T } | { success: false; error: string };
// Interface ทำไม่ได้: cannot use | in interface

// 2. Primitive Type Aliases
type ID = string | number;
type Milliseconds = number;
type Percentage = number;

// 3. Tuple Types
type Vector2D = [number, number];
type Vector3D = [number, number, number];
type RGB = [number, number, number];
type RGBA = [number, number, number, number];

// 4. Computed/Mapped Types
type ReadonlyUser = Readonly<FullPerson>;
type PartialUser = Partial<FullPerson>;

// 5. Conditional Types
type NonNullable<T> = T extends null | undefined ? never : T;
type Flatten<T> = T extends Array<infer U> ? U : T;

// 6. Complex Function Types
type EventHandler<T extends Event = Event> = (event: T) => void;
type AsyncFunction<T = void> = (...args: unknown[]) => Promise<T>;
```

```typescript
// ============= ใช้ Interface เมื่อ =============

// 1. กำหนด shape ของ object สำหรับ class
interface Serializable {
  serialize(): string;
  deserialize(data: string): void;
}

// 2. Class contracts
interface Printable {
  print(): void;
  preview(): string;
}

// 3. เมื่อต้องการ Declaration Merging
interface GlobalConfig {
  apiUrl: string;
}
// เพิ่มภายหลังได้
interface GlobalConfig {
  debug: boolean;
}

// 4. เมื่อต้องการ extends
interface AdminUser extends FullPerson {
  role: "admin";
  permissions: string[];
  lastLogin: Date;
}
```

```typescript
// ============= ใช้ได้ทั้งคู่ =============

// Object Type ทั่วไป (ทำได้ทั้งคู่)
interface UserInterface {
  id: number;
  name: string;
}

type UserType = {
  id: number;
  name: string;
};

// Generic Container (ทำได้ทั้งคู่)
interface Container<T> {
  value: T;
  transform<U>(fn: (v: T) => U): Container<U>;
}

type ContainerType<T> = {
  value: T;
  transform<U>(fn: (v: T) => U): ContainerType<U>;
};
```

---

## 9.8 Generic Type Aliases

```typescript
// Generic Type Alias พื้นฐาน
type Maybe<T> = T | null | undefined;
type Either<L, R> = { left: L; right?: never } | { left?: never; right: R };
type Nullable<T> = T | null;
type Optional<T> = T | undefined;

// การใช้งาน Maybe
function findUser(id: number): Maybe<FullPerson> {
  // ถ้าหาไม่เจอ return null
  if (id <= 0) return null;
  return {
    id,
    firstName: "สมชาย",
    lastName: "ใจดี",
    email: "somchai@example.com",
    dateOfBirth: new Date("1990-01-01"),
    address: {
      street: "123 Test",
      city: "Bangkok",
      province: "Bangkok",
      zipCode: "10110",
      country: "Thailand"
    }
  };
}

const user = findUser(1);
if (user !== null && user !== undefined) {
  console.log(user.firstName); // TypeScript รู้ว่าไม่ใช่ null/undefined
}
```

```typescript
// Generic Type สำหรับ Result Pattern
type Success<T> = {
  readonly kind: "success";
  readonly value: T;
};

type Failure<E = Error> = {
  readonly kind: "failure";
  readonly error: E;
};

type Result<T, E = Error> = Success<T> | Failure<E>;

// Helper functions
function success<T>(value: T): Success<T> {
  return { kind: "success", value };
}

function failure<E = Error>(error: E): Failure<E> {
  return { kind: "failure", error };
}

function isSuccess<T, E>(result: Result<T, E>): result is Success<T> {
  return result.kind === "success";
}

// การใช้งาน
function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return failure("ไม่สามารถหารด้วยศูนย์ได้");
  }
  return success(a / b);
}

const result = divide(10, 2);
if (isSuccess(result)) {
  console.log(`ผลลัพธ์: ${result.value}`); // 5
} else {
  console.error(`ข้อผิดพลาด: ${result.error}`);
}

const errorResult = divide(10, 0);
if (!isSuccess(errorResult)) {
  console.error(`ข้อผิดพลาด: ${errorResult.error}`); // ไม่สามารถหารด้วยศูนย์ได้
}
```

```typescript
// Generic Type สำหรับ Pagination
type Page<T> = {
  items: T[];
  page: number;
  pageSize: number;
  total: number;
  totalPages: number;
};

type SortedPage<T> = Page<T> & {
  sortBy: keyof T;
  sortOrder: "asc" | "desc";
};

// Generic Pair และ Triple
type Pair<A, B> = { first: A; second: B };
type Triple<A, B, C> = { first: A; second: B; third: C };
type Swap<P extends Pair<unknown, unknown>> = Pair<P["second"], P["first"]>;

const pair: Pair<string, number> = { first: "hello", second: 42 };
const swapped: Swap<typeof pair> = { first: 42, second: "hello" };
```

---

## 9.9 Template Literal Type Aliases

```typescript
// Template Literal Types
type EventName = `on${string}`;
type GetterName = `get${Capitalize<string>}`;
type SetterName = `set${Capitalize<string>}`;

// ตัวอย่างการใช้งาน
type CSSProperty = `--${string}`;  // CSS Custom Properties
type DataAttribute = `data-${string}`;  // HTML Data Attributes

const theme: CSSProperty = "--primary-color";
const testId: DataAttribute = "data-testid";
```

```typescript
// Template Literal Types กับ Union
type Color = "red" | "green" | "blue";
type Size = "sm" | "md" | "lg";

type ButtonVariant = `${Color}-${Size}`;
// "red-sm" | "red-md" | "red-lg" | "green-sm" | ...

type EventType = "click" | "hover" | "focus" | "blur";
type ElementType = "button" | "input" | "select";
type HandlerName = `on${Capitalize<EventType>}${Capitalize<ElementType>}`;
// "onClickButton" | "onClickInput" | ... | "onBlurSelect"
```

```typescript
// Template Literal Types สำหรับ API Routes
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
type ApiVersion = "v1" | "v2" | "v3";
type ResourceName = "users" | "products" | "orders" | "categories";

type ApiRoute = `/${ApiVersion}/${ResourceName}`;
type ApiRouteWithId = `/${ApiVersion}/${ResourceName}/${number}`;

// สร้าง typed API client
type ApiEndpoint = {
  [K in `${Lowercase<HttpMethod>}${Capitalize<ResourceName>}`]: (
    path: string,
    data?: unknown
  ) => Promise<unknown>;
};
```

```typescript
// Template Literal Types สำหรับ CSS
type CSSUnit = "px" | "em" | "rem" | "%" | "vh" | "vw";
type CSSValue = `${number}${CSSUnit}`;
type CSSColorHex = `#${string}`;
type CSSRgb = `rgb(${number}, ${number}, ${number})`;
type CSSRgba = `rgba(${number}, ${number}, ${number}, ${number})`;

// ตัวอย่างการใช้งาน
const fontSize: CSSValue = "16px";
const margin: CSSValue = "1.5rem";
const primaryColor: CSSColorHex = "#3B82F6";
```

```typescript
// Template Literal Types สำหรับ Database Queries
type TableName = "users" | "products" | "orders";
type ColumnName = "id" | "name" | "email" | "createdAt";
type OrderDirection = "ASC" | "DESC";

type OrderByClause = `ORDER BY ${ColumnName} ${OrderDirection}`;
type SelectClause = `SELECT ${string} FROM ${TableName}`;

// Type-safe SQL builder (simplified)
type WhereOperator = "=" | "!=" | ">" | "<" | ">=" | "<=" | "LIKE" | "IN";
type WhereClause = `${ColumnName} ${WhereOperator} ?`;
```

---

## 9.10 ตัวอย่างจริง: Type Aliases ในระบบจริง

```typescript
// Type aliases สำหรับ E-commerce System

// ID Types
type ProductID = number;
type CategoryID = number;
type CustomerID = number;
type OrderID = string;  // UUID format
type SKU = string;      // Stock Keeping Unit

// Status Types
type ProductStatus = "active" | "inactive" | "draft" | "archived";
type OrderStatus = "pending" | "confirmed" | "processing" | "shipped" | "delivered" | "cancelled" | "refunded";
type PaymentStatus = "pending" | "processing" | "completed" | "failed" | "refunded";
type ShipmentStatus = "not_shipped" | "preparing" | "shipped" | "in_transit" | "delivered" | "returned";

// Money Type (ป้องกัน floating point issues)
type Money = {
  amount: number;       // จำนวน (เป็น integer เช่น satang)
  currency: string;     // สกุลเงิน เช่น "THB"
};

// Percentage Type
type Percentage = number; // 0-100

// Discount Types
type DiscountType = "percentage" | "fixed_amount" | "free_shipping";
type Discount = {
  type: DiscountType;
  value: number;
  maxAmount?: Money;
  minOrderAmount?: Money;
  applicableProducts?: ProductID[];
  applicableCategories?: CategoryID[];
};

// Price Configuration
type PriceConfig = {
  basePrice: Money;
  salePrice?: Money;
  costPrice: Money;
  taxRate: Percentage;
  discounts?: Discount[];
};
```

```typescript
// Type aliases สำหรับ Authentication System

type UserRole = "super_admin" | "admin" | "manager" | "staff" | "customer";
type Permission = 
  | "users:read" | "users:write" | "users:delete"
  | "products:read" | "products:write" | "products:delete"
  | "orders:read" | "orders:write" | "orders:delete"
  | "reports:view" | "settings:manage";

type TokenType = "access" | "refresh" | "reset_password" | "email_verification";

type JWTPayload = {
  sub: string;        // subject (user ID)
  iat: number;        // issued at
  exp: number;        // expiration
  jti: string;        // JWT ID
  role: UserRole;
  permissions: Permission[];
};

type AuthToken = {
  type: TokenType;
  token: string;
  expiresAt: Date;
};

type AuthResponse = {
  user: {
    id: CustomerID;
    email: string;
    role: UserRole;
  };
  tokens: {
    access: AuthToken;
    refresh: AuthToken;
  };
};
```

```typescript
// Type aliases สำหรับ Notification System

type NotificationType = 
  | "order_placed"
  | "order_confirmed"
  | "order_shipped"
  | "payment_received"
  | "payment_failed"
  | "account_created"
  | "password_reset"
  | "promotion";

type NotificationChannel = "email" | "sms" | "push" | "in_app";
type NotificationPriority = "low" | "normal" | "high" | "urgent";

type NotificationTemplate = {
  id: string;
  type: NotificationType;
  channel: NotificationChannel;
  subject?: string;       // สำหรับ email
  body: string;
  variables: string[];    // placeholders เช่น {{customerName}}
};

type NotificationData = {
  templateId: string;
  recipient: {
    userId?: CustomerID;
    email?: string;
    phone?: string;
    deviceToken?: string;
  };
  variables: Record<string, string | number>;
  priority: NotificationPriority;
  scheduledAt?: Date;
};
```

---

## 9.11 Type Aliases กับ Discriminated Unions

```typescript
// Discriminated Union Pattern
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
  }
}

const circle: Shape = { kind: "circle", radius: 5 };
const rect: Shape = { kind: "rectangle", width: 4, height: 6 };
const tri: Shape = { kind: "triangle", base: 3, height: 8 };

console.log(`วงกลม: ${calculateArea(circle).toFixed(2)}`);    // 78.54
console.log(`สี่เหลี่ยม: ${calculateArea(rect)}`);            // 24
console.log(`สามเหลี่ยม: ${calculateArea(tri)}`);             // 12
```

```typescript
// Action Types สำหรับ State Management
type Action =
  | { type: "INCREMENT"; payload?: number }
  | { type: "DECREMENT"; payload?: number }
  | { type: "RESET" }
  | { type: "SET_VALUE"; payload: number };

type CounterState = {
  value: number;
  history: number[];
};

function counterReducer(state: CounterState, action: Action): CounterState {
  switch (action.type) {
    case "INCREMENT":
      return {
        value: state.value + (action.payload ?? 1),
        history: [...state.history, state.value]
      };
    case "DECREMENT":
      return {
        value: state.value - (action.payload ?? 1),
        history: [...state.history, state.value]
      };
    case "RESET":
      return { value: 0, history: [...state.history, state.value] };
    case "SET_VALUE":
      return {
        value: action.payload,
        history: [...state.history, state.value]
      };
  }
}

const initialState: CounterState = { value: 0, history: [] };
let state = counterReducer(initialState, { type: "INCREMENT" });
state = counterReducer(state, { type: "INCREMENT", payload: 5 });
state = counterReducer(state, { type: "DECREMENT", payload: 2 });

console.log(`ค่าปัจจุบัน: ${state.value}`);     // 4
console.log(`ประวัติ: ${state.history}`);        // [0, 1, 6]
```

---

## 9.12 Type Aliases สำหรับ Advanced Patterns

```typescript
// Builder Pattern Types
type Builder<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => Builder<T>;
} & {
  build(): T;
};

// Fluent Interface Type
type Fluent<T> = {
  [K in keyof T]: T[K] extends (...args: infer A) => T
    ? (...args: A) => Fluent<T>
    : T[K];
};

// Proxy Type
type Proxied<T> = {
  [K in keyof T]: T[K] extends (...args: infer A) => infer R
    ? (...args: A) => Promise<R>
    : T[K];
};
```

```typescript
// Type สำหรับ Validation
type ValidationRule<T> = {
  validate: (value: T) => boolean;
  message: string;
};

type ValidationSchema<T> = {
  [K in keyof T]?: ValidationRule<T[K]>[];
};

type ValidationResult<T> = {
  isValid: boolean;
  errors: Partial<Record<keyof T, string[]>>;
};

// ตัวอย่างการใช้งาน
type RegistrationForm = {
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
  age: number;
};

const registrationSchema: ValidationSchema<RegistrationForm> = {
  username: [
    {
      validate: (v) => v.length >= 3,
      message: "ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร"
    },
    {
      validate: (v) => /^[a-zA-Z0-9_]+$/.test(v),
      message: "ชื่อผู้ใช้ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _"
    }
  ],
  email: [
    {
      validate: (v) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
      message: "รูปแบบอีเมลไม่ถูกต้อง"
    }
  ],
  age: [
    {
      validate: (v) => v >= 18,
      message: "ต้องมีอายุ 18 ปีขึ้นไป"
    }
  ]
};
```

---

## 9.13 สรุปบทที่ 9

ในบทนี้เราได้เรียนรู้:

1. **Type Alias Declaration** - การสร้าง type aliases ด้วย `type`
2. **Literal Types** - string, number, boolean literal types
3. **Complex Type Compositions** - การรวม types ด้วย &, Mapped types
4. **Function Type Aliases** - types สำหรับ functions, callbacks, handlers
5. **Recursive Type Aliases** - JSON, Tree, Linked List
6. **Utility Types** - Partial, Required, Readonly, Pick, Omit และอื่นๆ
7. **เมื่อไหร่ควรใช้ Type vs Interface**
8. **Generic Type Aliases** - Maybe, Result, Page
9. **Template Literal Types** - สร้าง string pattern types
10. **Real-world Examples** - E-commerce, Auth, Notification systems

### สรุปการเลือก Type Alias vs Interface

| สถานการณ์ | Type Alias | Interface |
|-----------|-----------|-----------|
| Union Types | ✅ | ❌ |
| Intersection Types | ✅ | ✅ (extends) |
| Primitive Type Names | ✅ | ❌ |
| Tuple Types | ✅ | ❌ |
| Mapped/Conditional Types | ✅ | ❌ |
| Class Contracts | ✅ | ✅ (นิยมกว่า) |
| Declaration Merging | ❌ | ✅ |
| Generic Container | ✅ | ✅ |
| Object Shape | ✅ | ✅ |

---

## แบบฝึกหัดบทที่ 9

**แบบฝึกหัดที่ 1:** สร้าง Type Alias สำหรับระบบ Todo ที่มี TodoItem, TodoList, FilterType, SortType

**แบบฝึกหัดที่ 2:** สร้าง Generic Result Type ที่รองรับ success, failure, loading states

**แบบฝึกหัดที่ 3:** ใช้ Template Literal Types สร้าง type-safe CSS class names

**แบบฝึกหัดที่ 4:** สร้าง Recursive Type สำหรับ Menu system ที่มี sub-menus

---

*จบบทที่ 9: Type Aliases*
*บทต่อไป: Part 10 - Union & Intersection Types*
