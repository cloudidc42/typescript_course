# ส่วนที่ 10: Union & Intersection Types ใน TypeScript

## บทนำ

Union Types และ Intersection Types เป็นเครื่องมือที่ทรงพลังใน TypeScript สำหรับสร้าง types ที่ยืดหยุ่น Union Type (`|`) ทำให้ค่าสามารถเป็นได้หนึ่งในหลาย types ในขณะที่ Intersection Type (`&`) สร้าง type ใหม่ที่รวมทุก properties จากหลาย types เข้าด้วยกัน

ในบทนี้เราจะเรียนรู้:
- Union Types และการใช้งาน
- Intersection Types และการใช้งาน
- Discriminated Unions
- Type Narrowing กับ Union Types
- Practical Patterns
- Union vs Optional Properties
- การใช้งานกับ Functions
- การรวมกับ Generics
- Real-world Use Cases

---

## 10.1 Union Types พื้นฐาน

Union Type ใช้ `|` เพื่อบอกว่าค่าสามารถเป็น type ใดก็ได้ในที่ระบุ

```typescript
// Union Type พื้นฐาน
type StringOrNumber = string | number;
type BooleanOrNull = boolean | null;
type ID = string | number;

let value: StringOrNumber;
value = "สวัสดี";    // OK
value = 42;          // OK
// value = true;     // Error!

function printId(id: ID): void {
  if (typeof id === "string") {
    console.log(`ID (string): ${id.toUpperCase()}`);
  } else {
    console.log(`ID (number): ${id}`);
  }
}

printId("abc123");  // ID (string): ABC123
printId(456);       // ID (number): 456
```

```typescript
// Union Types กับ Object Types
type Admin = {
  role: "admin";
  adminPrivileges: string[];
  name: string;
};

type Customer = {
  role: "customer";
  purchaseHistory: string[];
  name: string;
};

type Employee = {
  role: "employee";
  department: string;
  name: string;
};

type User = Admin | Customer | Employee;

function greetUser(user: User): string {
  return `สวัสดี ${user.name} (${user.role})`;
}

const admin: User = {
  role: "admin",
  adminPrivileges: ["manage-users", "view-reports"],
  name: "สมชาย"
};

const customer: User = {
  role: "customer",
  purchaseHistory: ["order-001", "order-002"],
  name: "สมหญิง"
};

console.log(greetUser(admin));    // สวัสดี สมชาย (admin)
console.log(greetUser(customer)); // สวัสดี สมหญิง (customer)
```

```typescript
// Union ของ Literal Types
type Direction = "up" | "down" | "left" | "right";
type Status = "loading" | "success" | "error" | "idle";
type LogLevel = "debug" | "info" | "warn" | "error" | "fatal";

function handleDirection(dir: Direction): void {
  const moves: Record<Direction, string> = {
    up: "เลื่อนขึ้น",
    down: "เลื่อนลง",
    left: "เลื่อนซ้าย",
    right: "เลื่อนขวา"
  };
  console.log(moves[dir]);
}

handleDirection("up");    // เลื่อนขึ้น
handleDirection("left");  // เลื่อนซ้าย
```

---

## 10.2 Intersection Types พื้นฐาน

Intersection Type ใช้ `&` เพื่อรวม types หลายตัวเข้าด้วยกัน

```typescript
// Intersection Type พื้นฐาน
type HasName = { name: string };
type HasAge = { age: number };
type HasEmail = { email: string };

type Person = HasName & HasAge & HasEmail;

const person: Person = {
  name: "สมชาย ใจดี",
  age: 30,
  email: "somchai@example.com"
};

console.log(`${person.name}, อายุ ${person.age} ปี`);
```

```typescript
// Intersection Types กับ Class-like objects
type Serializable = {
  serialize(): string;
  deserialize(data: string): void;
};

type Loggable = {
  log(message: string): void;
  getLog(): string[];
};

type Persistable = {
  save(): Promise<void>;
  load(id: string): Promise<void>;
};

type DataService = Serializable & Loggable & Persistable;

// การ implement DataService
const userService: DataService = {
  serialize() {
    return JSON.stringify({ type: "userService" });
  },
  deserialize(data: string) {
    console.log("Deserialized:", JSON.parse(data));
  },
  log(message: string) {
    console.log(`[UserService] ${message}`);
  },
  getLog() {
    return ["log entry 1", "log entry 2"];
  },
  async save() {
    console.log("บันทึกข้อมูล...");
  },
  async load(id: string) {
    console.log(`โหลดข้อมูล id: ${id}`);
  }
};
```

```typescript
// Intersection Types สำหรับ Mixins
type Timestamped<T> = T & {
  createdAt: Date;
  updatedAt: Date;
};

type WithId<T, ID = number> = T & {
  id: ID;
};

type BaseEntity<T> = WithId<Timestamped<T>>;

type CreateProductInput = {
  name: string;
  price: number;
  stock: number;
};

type Product = BaseEntity<CreateProductInput>;

const product: Product = {
  id: 1,
  name: "TypeScript Handbook",
  price: 299,
  stock: 50,
  createdAt: new Date("2024-01-01"),
  updatedAt: new Date("2024-06-01")
};

console.log(`${product.name} - ${product.price} บาท (ID: ${product.id})`);
```

---

## 10.3 Discriminated Unions

Discriminated Union คือ pattern ที่ใช้ property พิเศษ (discriminant) เพื่อแยกแยะ type ต่างๆ ในกลุ่ม union

```typescript
// Discriminated Union ด้วย "kind" property
type Circle = {
  kind: "circle";
  radius: number;
};

type Square = {
  kind: "square";
  side: number;
};

type Rectangle = {
  kind: "rectangle";
  width: number;
  height: number;
};

type Triangle = {
  kind: "triangle";
  base: number;
  height: number;
};

type Shape = Circle | Square | Rectangle | Triangle;

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
    // TypeScript รู้ว่า switch ครอบคลุมทุก case แล้ว
  }
}

function describeShape(shape: Shape): string {
  switch (shape.kind) {
    case "circle":
      return `วงกลม รัศมี ${shape.radius}`;
    case "square":
      return `สี่เหลี่ยมจัตุรัส ด้าน ${shape.side}`;
    case "rectangle":
      return `สี่เหลี่ยมผืนผ้า ${shape.width}x${shape.height}`;
    case "triangle":
      return `สามเหลี่ยม ฐาน ${shape.base} สูง ${shape.height}`;
  }
}

const shapes: Shape[] = [
  { kind: "circle", radius: 5 },
  { kind: "square", side: 4 },
  { kind: "rectangle", width: 6, height: 3 },
  { kind: "triangle", base: 8, height: 5 }
];

shapes.forEach(shape => {
  console.log(`${describeShape(shape)}: พื้นที่ = ${calculateArea(shape).toFixed(2)}`);
});
```

```typescript
// Discriminated Union สำหรับ API State
type LoadingState = {
  status: "loading";
};

type SuccessState<T> = {
  status: "success";
  data: T;
  timestamp: Date;
};

type ErrorState = {
  status: "error";
  error: Error;
  code: number;
};

type IdleState = {
  status: "idle";
};

type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState | IdleState;

interface User {
  id: number;
  name: string;
  email: string;
}

function renderUserState(state: AsyncState<User>): string {
  switch (state.status) {
    case "idle":
      return "กรุณากดปุ่มเพื่อโหลดข้อมูล";
    case "loading":
      return "กำลังโหลดข้อมูล...";
    case "success":
      return `ยินดีต้อนรับ ${state.data.name}!`;
    case "error":
      return `เกิดข้อผิดพลาด: ${state.error.message} (${state.code})`;
  }
}

const states: AsyncState<User>[] = [
  { status: "idle" },
  { status: "loading" },
  {
    status: "success",
    data: { id: 1, name: "สมชาย", email: "s@e.com" },
    timestamp: new Date()
  },
  { status: "error", error: new Error("ไม่พบข้อมูล"), code: 404 }
];

states.forEach(state => console.log(renderUserState(state)));
```

```typescript
// Discriminated Union สำหรับ Redux-style Actions
type FetchUsersRequest = {
  type: "FETCH_USERS_REQUEST";
};

type FetchUsersSuccess = {
  type: "FETCH_USERS_SUCCESS";
  payload: User[];
};

type FetchUsersFailure = {
  type: "FETCH_USERS_FAILURE";
  error: string;
};

type CreateUserRequest = {
  type: "CREATE_USER_REQUEST";
  payload: { name: string; email: string };
};

type CreateUserSuccess = {
  type: "CREATE_USER_SUCCESS";
  payload: User;
};

type UserAction =
  | FetchUsersRequest
  | FetchUsersSuccess
  | FetchUsersFailure
  | CreateUserRequest
  | CreateUserSuccess;

type UsersState = {
  users: User[];
  loading: boolean;
  error: string | null;
};

function usersReducer(
  state: UsersState = { users: [], loading: false, error: null },
  action: UserAction
): UsersState {
  switch (action.type) {
    case "FETCH_USERS_REQUEST":
      return { ...state, loading: true, error: null };
    case "FETCH_USERS_SUCCESS":
      return { ...state, loading: false, users: action.payload };
    case "FETCH_USERS_FAILURE":
      return { ...state, loading: false, error: action.error };
    case "CREATE_USER_REQUEST":
      return { ...state, loading: true };
    case "CREATE_USER_SUCCESS":
      return {
        ...state,
        loading: false,
        users: [...state.users, action.payload]
      };
  }
}
```

---

## 10.4 Type Narrowing กับ Union Types

Type Narrowing คือการบอก TypeScript ว่า type ใน union คืออะไร ณ จุดนั้น

```typescript
// typeof Narrowing
function processValue(value: string | number | boolean): string {
  if (typeof value === "string") {
    return `String: ${value.toUpperCase()}`;  // TypeScript รู้ว่าเป็น string
  } else if (typeof value === "number") {
    return `Number: ${value.toFixed(2)}`;     // TypeScript รู้ว่าเป็น number
  } else {
    return `Boolean: ${value ? "true" : "false"}`; // TypeScript รู้ว่าเป็น boolean
  }
}

console.log(processValue("hello"));  // String: HELLO
console.log(processValue(3.14));     // Number: 3.14
console.log(processValue(true));     // Boolean: true
```

```typescript
// instanceof Narrowing
class Dog {
  bark(): string { return "โฮ่ง!"; }
  fetch(): string { return "เอาลูกบอลมาให้แล้ว!"; }
}

class Cat {
  meow(): string { return "เมี้ยว!"; }
  purr(): string { return "กรรๆๆ"; }
}

type Pet = Dog | Cat;

function makeSound(pet: Pet): string {
  if (pet instanceof Dog) {
    return pet.bark();  // TypeScript รู้ว่าเป็น Dog
  } else {
    return pet.meow();  // TypeScript รู้ว่าเป็น Cat
  }
}

const dog = new Dog();
const cat = new Cat();

console.log(makeSound(dog)); // โฮ่ง!
console.log(makeSound(cat)); // เมี้ยว!
```

```typescript
// in Operator Narrowing
type Fish = {
  swim(): void;
  breatheUnderwater: boolean;
};

type Bird = {
  fly(): void;
  wingspan: number;
};

type Animal = Fish | Bird;

function move(animal: Animal): void {
  if ("swim" in animal) {
    animal.swim();  // TypeScript รู้ว่าเป็น Fish
    console.log(`ปลาสามารถหายใจใต้น้ำ: ${animal.breatheUnderwater}`);
  } else {
    animal.fly();   // TypeScript รู้ว่าเป็น Bird
    console.log(`นกมีปีกกว้าง: ${animal.wingspan} ซม.`);
  }
}
```

```typescript
// Type Guards (Type Predicate)
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNumber(value: unknown): value is number {
  return typeof value === "number";
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value &&
    "email" in value
  );
}

function isError(value: unknown): value is Error {
  return value instanceof Error;
}

// การใช้งาน
function processData(data: unknown): string {
  if (isString(data)) {
    return `String: ${data.length} ตัวอักษร`;
  } else if (isNumber(data)) {
    return `Number: ${data}`;
  } else if (isUser(data)) {
    return `User: ${data.name} (${data.email})`;
  } else if (isError(data)) {
    return `Error: ${data.message}`;
  }
  return "ไม่รู้จัก type";
}

console.log(processData("สวัสดี"));                           // String: 7 ตัวอักษร
console.log(processData(42));                               // Number: 42
console.log(processData({ id: 1, name: "John", email: "j@e.com" })); // User: John
console.log(processData(new Error("เกิดข้อผิดพลาด")));       // Error: เกิดข้อผิดพลาด
```

```typescript
// Exhaustiveness Checking
type NetworkState =
  | { state: "waiting" }
  | { state: "connecting"; connectionAttempts: number }
  | { state: "connected"; ping: number; bandwidth: number }
  | { state: "disconnected"; reason: string; canReconnect: boolean };

function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${x}`);
}

function handleNetworkState(state: NetworkState): string {
  switch (state.state) {
    case "waiting":
      return "รอการเชื่อมต่อ...";
    case "connecting":
      return `กำลังเชื่อมต่อ (ครั้งที่ ${state.connectionAttempts})...`;
    case "connected":
      return `เชื่อมต่อแล้ว - Ping: ${state.ping}ms, Bandwidth: ${state.bandwidth}Mbps`;
    case "disconnected":
      return `การเชื่อมต่อถูกตัด: ${state.reason}. สามารถเชื่อมต่อใหม่: ${state.canReconnect}`;
    default:
      return assertNever(state); // TypeScript จะ Error ถ้ามี case ที่ไม่ครอบคลุม
  }
}
```

---

## 10.5 Practical Union Patterns

```typescript
// Optional Chain กับ Union Types
type Config = {
  database?: {
    host: string;
    port?: number;
  };
  cache?: {
    type: "redis" | "memory";
    ttl: number;
  };
};

function getDbPort(config: Config): number | undefined {
  return config.database?.port;
}

// Nullish Coalescing กับ Union Types
function getPort(config: Config): number {
  return config.database?.port ?? 5432;
}

// การรวม Union Types
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ReadMethod = Extract<HttpMethod, "GET">;
type WriteMethod = Exclude<HttpMethod, "GET" | "DELETE">;

// ReadMethod = "GET"
// WriteMethod = "POST" | "PUT"
```

```typescript
// Union Types สำหรับ Error Handling
class ValidationError extends Error {
  constructor(
    message: string,
    public readonly field: string,
    public readonly code: string
  ) {
    super(message);
    this.name = "ValidationError";
  }
}

class DatabaseError extends Error {
  constructor(
    message: string,
    public readonly query?: string,
    public readonly code?: string
  ) {
    super(message);
    this.name = "DatabaseError";
  }
}

class NetworkError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly url: string
  ) {
    super(message);
    this.name = "NetworkError";
  }
}

type AppError = ValidationError | DatabaseError | NetworkError;

function handleError(error: AppError): string {
  if (error instanceof ValidationError) {
    return `ข้อมูลไม่ถูกต้อง: ${error.field} - ${error.message}`;
  } else if (error instanceof DatabaseError) {
    return `ข้อผิดพลาดฐานข้อมูล: ${error.message}`;
  } else {
    return `ข้อผิดพลาดเครือข่าย (${error.statusCode}): ${error.message}`;
  }
}

const validationErr = new ValidationError("อีเมลไม่ถูกต้อง", "email", "INVALID_EMAIL");
const dbErr = new DatabaseError("ไม่สามารถเชื่อมต่อฐานข้อมูลได้");
const networkErr = new NetworkError("ไม่พบ endpoint", 404, "/api/users");

console.log(handleError(validationErr));
console.log(handleError(dbErr));
console.log(handleError(networkErr));
```

```typescript
// Union Types สำหรับ Form State
type FormField<T = string> = {
  value: T;
  error: string | null;
  touched: boolean;
  dirty: boolean;
};

type FormState<T extends Record<string, unknown>> = {
  [K in keyof T]: FormField<T[K]>;
} & {
  isSubmitting: boolean;
  isValid: boolean;
  submitCount: number;
};

type LoginForm = {
  email: string;
  password: string;
  rememberMe: boolean;
};

type LoginFormState = FormState<LoginForm>;

const initialLoginState: LoginFormState = {
  email: { value: "", error: null, touched: false, dirty: false },
  password: { value: "", error: null, touched: false, dirty: false },
  rememberMe: { value: false, error: null, touched: false, dirty: false },
  isSubmitting: false,
  isValid: false,
  submitCount: 0
};
```

---

## 10.6 Intersection Types ขั้นสูง

```typescript
// Intersection สำหรับ Mixin Pattern
type Constructor<T = object> = new (...args: unknown[]) => T;

type Timestamped<TBase extends Constructor> = TBase & Constructor<{
  createdAt: Date;
  updatedAt: Date;
}>;

// Intersection Types สำหรับ Object Merging
type DeepMerge<T, U> = {
  [K in keyof T | keyof U]: K extends keyof T & keyof U
    ? T[K] extends object
      ? U[K] extends object
        ? DeepMerge<T[K], U[K]>
        : T[K] | U[K]
      : T[K] | U[K]
    : K extends keyof T
    ? T[K]
    : K extends keyof U
    ? U[K]
    : never;
};
```

```typescript
// Intersection Types สำหรับ Permission System
type ReadPermission = { canRead: true };
type WritePermission = { canWrite: true };
type DeletePermission = { canDelete: true };
type AdminPermission = { isAdmin: true; canManageUsers: true };

type ViewerPermissions = ReadPermission;
type EditorPermissions = ReadPermission & WritePermission;
type ModeratorPermissions = ReadPermission & WritePermission & DeletePermission;
type AdminPermissions = ModeratorPermissions & AdminPermission;

function checkAccess(
  permissions: EditorPermissions,
  action: "read" | "write"
): boolean {
  if (action === "read") return permissions.canRead;
  return permissions.canWrite;
}

const editorPerms: EditorPermissions = { canRead: true, canWrite: true };
console.log(checkAccess(editorPerms, "write")); // true
```

```typescript
// Intersection สำหรับ API Response Enhancement
type BaseApiResponse = {
  requestId: string;
  timestamp: Date;
  version: string;
};

type PaginatedMeta = {
  page: number;
  pageSize: number;
  total: number;
  totalPages: number;
};

type SortedMeta = {
  sortBy: string;
  sortOrder: "asc" | "desc";
};

type FilteredMeta = {
  filters: Record<string, unknown>;
};

type ListResponse<T> = BaseApiResponse & {
  data: T[];
  meta: PaginatedMeta & SortedMeta & FilteredMeta;
};

type SingleResponse<T> = BaseApiResponse & {
  data: T;
};

// ตัวอย่างการใช้งาน
const usersResponse: ListResponse<User> = {
  requestId: "req-123",
  timestamp: new Date(),
  version: "1.0",
  data: [
    { id: 1, name: "สมชาย", email: "s@e.com" }
  ],
  meta: {
    page: 1,
    pageSize: 10,
    total: 100,
    totalPages: 10,
    sortBy: "name",
    sortOrder: "asc",
    filters: { active: true }
  }
};
```

---

## 10.7 Union vs Optional Properties

```typescript
// Optional Properties vs Union with undefined
// ทั้งสองวิธีต่างกัน

// วิธีที่ 1: Optional Property
interface UserOptional {
  name: string;
  phone?: string;  // string | undefined
}

// วิธีที่ 2: Union with undefined
interface UserUnion {
  name: string;
  phone: string | undefined; // ต้องระบุ แต่ค่าอาจเป็น undefined
}

// ความแตกต่าง:
const u1: UserOptional = { name: "John" };        // OK - ไม่ต้องระบุ phone
const u2: UserUnion = { name: "John", phone: undefined }; // ต้องระบุ phone!
// const u3: UserUnion = { name: "John" };         // Error! phone is missing

// ตัวอย่างที่ชัดเจนกว่า:
type ApiConfig = {
  timeout: number;
  retries?: number;              // ไม่ต้องระบุ
  onError: ((err: Error) => void) | undefined; // ต้องระบุ แต่อาจเป็น undefined
};

const config: ApiConfig = {
  timeout: 5000,
  onError: undefined  // ต้องระบุ
};
```

```typescript
// เมื่อไหร่ควรใช้อะไร

// ใช้ Optional Properties เมื่อ property ไม่จำเป็นต้องระบุ
interface SearchParams {
  query: string;
  page?: number;      // ไม่จำเป็น
  pageSize?: number;  // ไม่จำเป็น
}

// ใช้ Union กับ null/undefined เมื่อต้องการบังคับให้ระบุว่า "ไม่มีค่า" อย่างชัดเจน
interface DatabaseResult<T> {
  data: T | null;         // ต้องระบุว่า null หรือมีค่า
  error: Error | null;    // ต้องระบุว่า null หรือมี error
  loading: boolean;
}

// ใช้ Union เมื่อ property สามารถเป็นได้หลาย types
interface FormValue {
  name: string;
  value: string | number | boolean | Date;  // หลาย types
}
```

---

## 10.8 Union Types กับ Functions

```typescript
// Function Overloads ด้วย Union Types
function formatValue(value: string): string;
function formatValue(value: number, decimals?: number): string;
function formatValue(value: Date, format?: string): string;
function formatValue(
  value: string | number | Date,
  extra?: number | string
): string {
  if (typeof value === "string") {
    return value;
  } else if (typeof value === "number") {
    return value.toFixed(typeof extra === "number" ? extra : 2);
  } else {
    return value.toLocaleDateString("th-TH");
  }
}

console.log(formatValue("สวัสดี"));      // สวัสดี
console.log(formatValue(3.14159, 2));    // 3.14
console.log(formatValue(new Date()));    // วันที่ปัจจุบัน
```

```typescript
// Function ที่รับ Union Type Parameters
type EventPayload = 
  | { event: "click"; x: number; y: number }
  | { event: "keydown"; key: string; modifiers: string[] }
  | { event: "resize"; width: number; height: number }
  | { event: "scroll"; scrollTop: number; scrollLeft: number };

type EventHandler<T extends EventPayload> = (payload: T) => void;

function handleEvent(payload: EventPayload): void {
  switch (payload.event) {
    case "click":
      console.log(`คลิกที่ (${payload.x}, ${payload.y})`);
      break;
    case "keydown":
      console.log(`กดปุ่ม: ${payload.key} [${payload.modifiers.join("+")}]`);
      break;
    case "resize":
      console.log(`ขนาดหน้าต่าง: ${payload.width}x${payload.height}`);
      break;
    case "scroll":
      console.log(`เลื่อน: top=${payload.scrollTop}, left=${payload.scrollLeft}`);
      break;
  }
}

handleEvent({ event: "click", x: 100, y: 200 });
handleEvent({ event: "keydown", key: "Enter", modifiers: ["Ctrl"] });
```

```typescript
// Return Type Union
function parseValue(input: string): number | boolean | string | null {
  if (input === "true") return true;
  if (input === "false") return false;
  if (input === "null") return null;
  
  const num = Number(input);
  if (!isNaN(num)) return num;
  
  return input;
}

type ParsedValue = ReturnType<typeof parseValue>;

const results: ParsedValue[] = [
  parseValue("42"),
  parseValue("true"),
  parseValue("hello"),
  parseValue("null")
];

results.forEach(result => {
  console.log(`${result} (${typeof result})`);
});
```

---

## 10.9 Union และ Intersection กับ Generics

```typescript
// Generic Union Types
type Optional<T> = T | undefined;
type Nullable<T> = T | null;
type Maybe<T> = T | null | undefined;

function getOrDefault<T>(value: Maybe<T>, defaultValue: T): T {
  return value ?? defaultValue;
}

const name = getOrDefault<string>(null, "ไม่ระบุชื่อ");
const age = getOrDefault<number>(undefined, 0);
const isActive = getOrDefault<boolean>(true, false);

console.log(name);     // ไม่ระบุชื่อ
console.log(age);      // 0
console.log(isActive); // true
```

```typescript
// Generic Intersection Types
type WithMetadata<T> = T & {
  _metadata: {
    createdAt: Date;
    updatedAt: Date;
    version: number;
  };
};

type WithPagination<T> = T & {
  _pagination: {
    page: number;
    total: number;
    hasMore: boolean;
  };
};

type ApiListResult<T> = WithMetadata<WithPagination<{ items: T[] }>>;

const result: ApiListResult<User> = {
  items: [{ id: 1, name: "John", email: "john@e.com" }],
  _metadata: {
    createdAt: new Date(),
    updatedAt: new Date(),
    version: 1
  },
  _pagination: {
    page: 1,
    total: 100,
    hasMore: true
  }
};
```

```typescript
// Generic Type Guard
function isOfType<T>(
  value: unknown,
  check: (v: unknown) => boolean
): value is T {
  return check(value);
}

function isArrayOf<T>(
  value: unknown,
  itemCheck: (item: unknown) => item is T
): value is T[] {
  return Array.isArray(value) && value.every(itemCheck);
}

function isUserObject(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as Record<string, unknown>).id === "number" &&
    typeof (value as Record<string, unknown>).name === "string" &&
    typeof (value as Record<string, unknown>).email === "string"
  );
}

const rawData: unknown = { id: 1, name: "John", email: "john@e.com" };
if (isUserObject(rawData)) {
  console.log(`User: ${rawData.name}`); // TypeScript รู้ว่าเป็น User
}

const rawArray: unknown = [
  { id: 1, name: "John", email: "j@e.com" },
  { id: 2, name: "Jane", email: "jane@e.com" }
];
if (isArrayOf<User>(rawArray, isUserObject)) {
  console.log(`Users: ${rawArray.map(u => u.name).join(", ")}`);
}
```

---

## 10.10 Real-World Use Cases: API Responses

```typescript
// Type-safe API Response System

// Base types
type ApiSuccessResponse<T> = {
  success: true;
  statusCode: 200 | 201 | 204;
  data: T;
  message?: string;
};

type ApiErrorResponse = {
  success: false;
  statusCode: 400 | 401 | 403 | 404 | 409 | 422 | 500;
  error: {
    code: string;
    message: string;
    details?: Record<string, string | string[]>;
    stack?: string; // เฉพาะ development
  };
};

type ApiResponse<T> = ApiSuccessResponse<T> | ApiErrorResponse;

// Paginated response
type PaginatedApiResponse<T> = ApiSuccessResponse<{
  items: T[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPrevPage: boolean;
  };
}>;

// Helper functions
function isApiSuccess<T>(
  response: ApiResponse<T>
): response is ApiSuccessResponse<T> {
  return response.success === true;
}

function isApiError(
  response: ApiResponse<unknown>
): response is ApiErrorResponse {
  return response.success === false;
}

// การใช้งาน
async function fetchUserData(userId: number): Promise<ApiResponse<User>> {
  try {
    // จำลอง API call
    if (userId <= 0) {
      return {
        success: false,
        statusCode: 400,
        error: {
          code: "INVALID_ID",
          message: "User ID ต้องเป็นตัวเลขบวก"
        }
      };
    }
    
    return {
      success: true,
      statusCode: 200,
      data: {
        id: userId,
        name: "สมชาย ใจดี",
        email: "somchai@example.com"
      },
      message: "ดึงข้อมูลสำเร็จ"
    };
  } catch (error) {
    return {
      success: false,
      statusCode: 500,
      error: {
        code: "INTERNAL_ERROR",
        message: "เกิดข้อผิดพลาดภายในระบบ"
      }
    };
  }
}

// การใช้งาน
async function displayUser(userId: number): Promise<void> {
  const response = await fetchUserData(userId);
  
  if (isApiSuccess(response)) {
    console.log(`User: ${response.data.name}`);
    console.log(`Email: ${response.data.email}`);
  } else {
    console.error(`Error ${response.statusCode}: ${response.error.message}`);
  }
}
```

---

## 10.11 Real-World Use Cases: State Management

```typescript
// Type-safe State Management Pattern

// Action Types
type Action<Type extends string, Payload = void> = Payload extends void
  ? { type: Type }
  : { type: Type; payload: Payload };

// Cart Actions
type CartItem = {
  productId: number;
  name: string;
  price: number;
  quantity: number;
};

type CartAction =
  | Action<"ADD_ITEM", CartItem>
  | Action<"REMOVE_ITEM", { productId: number }>
  | Action<"UPDATE_QUANTITY", { productId: number; quantity: number }>
  | Action<"CLEAR_CART">
  | Action<"APPLY_COUPON", { code: string; discount: number }>
  | Action<"REMOVE_COUPON">;

type CartState = {
  items: CartItem[];
  coupon: { code: string; discount: number } | null;
  total: number;
  itemCount: number;
};

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find(i => i.productId === action.payload.productId);
      const items = existing
        ? state.items.map(i =>
            i.productId === action.payload.productId
              ? { ...i, quantity: i.quantity + action.payload.quantity }
              : i
          )
        : [...state.items, action.payload];
      
      const total = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      const discounted = state.coupon ? total * (1 - state.coupon.discount / 100) : total;
      
      return {
        ...state,
        items,
        total: discounted,
        itemCount: items.reduce((sum, i) => sum + i.quantity, 0)
      };
    }
    
    case "REMOVE_ITEM": {
      const items = state.items.filter(i => i.productId !== action.payload.productId);
      const total = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      
      return {
        ...state,
        items,
        total,
        itemCount: items.reduce((sum, i) => sum + i.quantity, 0)
      };
    }
    
    case "UPDATE_QUANTITY": {
      const items = state.items.map(i =>
        i.productId === action.payload.productId
          ? { ...i, quantity: Math.max(0, action.payload.quantity) }
          : i
      ).filter(i => i.quantity > 0);
      
      const total = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      
      return {
        ...state,
        items,
        total,
        itemCount: items.reduce((sum, i) => sum + i.quantity, 0)
      };
    }
    
    case "CLEAR_CART":
      return { items: [], coupon: null, total: 0, itemCount: 0 };
    
    case "APPLY_COUPON": {
      const rawTotal = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      const discounted = rawTotal * (1 - action.payload.discount / 100);
      return { ...state, coupon: action.payload, total: discounted };
    }
    
    case "REMOVE_COUPON": {
      const total = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      return { ...state, coupon: null, total };
    }
  }
}
```

---

## 10.12 Advanced Union Patterns

```typescript
// Branded Types ป้องกัน type confusion
type Brand<K, T> = K & { readonly __brand: T };

type UserId = Brand<number, "UserId">;
type ProductId = Brand<number, "ProductId">;
type OrderId = Brand<string, "OrderId">;

function createUserId(id: number): UserId {
  return id as UserId;
}

function createProductId(id: number): ProductId {
  return id as ProductId;
}

function getUserById(id: UserId): User | null {
  // ไม่สามารถส่ง ProductId เข้ามาได้โดยบังเอิญ
  return null;
}

const userId = createUserId(1);
const productId = createProductId(1);

getUserById(userId);   // OK
// getUserById(productId); // Error! ProductId ไม่ใช่ UserId
// getUserById(1);          // Error! number ไม่ใช่ UserId
```

```typescript
// Opaque Types Pattern
type Opaque<Type, Token extends string> = Type & {
  readonly ["__opaque__"]: Token;
};

type EmailAddress = Opaque<string, "EmailAddress">;
type PhoneNumber = Opaque<string, "PhoneNumber">;
type HashedPassword = Opaque<string, "HashedPassword">;

function createEmailAddress(email: string): EmailAddress {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    throw new Error(`"${email}" ไม่ใช่รูปแบบอีเมลที่ถูกต้อง`);
  }
  return email as EmailAddress;
}

function sendEmail(to: EmailAddress, subject: string, body: string): void {
  console.log(`ส่งอีเมลถึง: ${to}`);
  console.log(`หัวข้อ: ${subject}`);
}

const email = createEmailAddress("user@example.com");
sendEmail(email, "ยืนยันการสมัคร", "กรุณายืนยันอีเมลของคุณ");
// sendEmail("random string", "...", "..."); // Error!
```

```typescript
// Template Union Types
type EventName<T extends string> = `${T}Changed` | `${T}Loaded` | `${T}Error`;
type UserEvents = EventName<"user">;
// "userChanged" | "userLoaded" | "userError"

type CRUDAction<T extends string> = 
  | `create${Capitalize<T>}`
  | `read${Capitalize<T>}`
  | `update${Capitalize<T>}`
  | `delete${Capitalize<T>}`;

type UserActions = CRUDAction<"user">;
// "createUser" | "readUser" | "updateUser" | "deleteUser"
```

---

## 10.13 Union Types สำหรับ Configuration

```typescript
// Plugin System ด้วย Union Types
type PluginConfig =
  | {
      type: "logger";
      level: "debug" | "info" | "warn" | "error";
      output: "console" | "file" | "remote";
      filePath?: string;
    }
  | {
      type: "cache";
      provider: "memory" | "redis" | "memcached";
      ttl: number;
      maxSize?: number;
      redisUrl?: string;
    }
  | {
      type: "auth";
      strategy: "jwt" | "session" | "oauth";
      secret?: string;
      expiresIn?: string;
      providers?: string[];
    }
  | {
      type: "storage";
      provider: "local" | "s3" | "gcs";
      bucket?: string;
      region?: string;
      basePath?: string;
    };

function configurePlugin(config: PluginConfig): void {
  switch (config.type) {
    case "logger":
      console.log(`Logger: level=${config.level}, output=${config.output}`);
      break;
    case "cache":
      console.log(`Cache: provider=${config.provider}, ttl=${config.ttl}s`);
      break;
    case "auth":
      console.log(`Auth: strategy=${config.strategy}`);
      break;
    case "storage":
      console.log(`Storage: provider=${config.provider}`);
      break;
  }
}

const plugins: PluginConfig[] = [
  { type: "logger", level: "info", output: "console" },
  { type: "cache", provider: "redis", ttl: 3600, redisUrl: "redis://localhost" },
  { type: "auth", strategy: "jwt", secret: "my-secret", expiresIn: "24h" },
  { type: "storage", provider: "s3", bucket: "my-bucket", region: "ap-southeast-1" }
];

plugins.forEach(configurePlugin);
```

---

## 10.14 สรุปบทที่ 10

ในบทนี้เราได้เรียนรู้:

1. **Union Types** (`|`) - ค่าสามารถเป็นได้หนึ่งใน types
2. **Intersection Types** (`&`) - รวม types เข้าด้วยกัน
3. **Discriminated Unions** - ใช้ discriminant property เพื่อแยกแยะ types
4. **Type Narrowing** - typeof, instanceof, in, type predicates
5. **Exhaustiveness Checking** - ใช้ never เพื่อตรวจสอบว่าครอบคลุมทุก case
6. **Union vs Optional** - ความแตกต่างและเมื่อไหร่ควรใช้อะไร
7. **Functions กับ Union** - overloads, parameter types, return types
8. **Generic Union/Intersection** - Maybe, Optional, Nullable patterns
9. **Real-world: API Responses** - type-safe response handling
10. **Real-world: State Management** - Redux-style action types
11. **Branded Types** - ป้องกัน type confusion

---

## แบบฝึกหัดบทที่ 10

**แบบฝึกหัดที่ 1:** สร้าง Discriminated Union สำหรับ payment method ที่รองรับ Credit Card, Bank Transfer, PromptPay และ Cryptocurrency

**แบบฝึกหัดที่ 2:** ใช้ Intersection Types สร้าง Mixin สำหรับ Auditable (ติดตามการเปลี่ยนแปลง), Cacheable, และ Exportable

**แบบฝึกหัดที่ 3:** สร้าง Type Guard functions สำหรับ validate API response จาก external service

**แบบฝึกหัดที่ 4:** ออกแบบ type-safe Event System ด้วย Discriminated Union ที่รองรับ event ต่างๆ ในแอพ

---

*จบบทที่ 10: Union & Intersection Types*
*บทต่อไป: Part 11 - Classes & OOP*
