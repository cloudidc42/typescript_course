# Part 56: TypeScript Interview Questions (คำถามสัมภาษณ์ TypeScript)

## บทนำ

ในบทนี้เราจะรวบรวมคำถามสัมภาษณ์ TypeScript ที่พบบ่อยกว่า 100 ข้อ พร้อมคำตอบที่ละเอียด ครอบคลุมทุกระดับตั้งแต่ผู้เริ่มต้นจนถึงผู้เชี่ยวชาญ

---

## ส่วนที่ 1: คำถามระดับเริ่มต้น (Beginner Level)

### คำถามที่ 1: TypeScript คืออะไร และแตกต่างจาก JavaScript อย่างไร?

**คำตอบ:**

TypeScript คือ Superset ของ JavaScript ที่พัฒนาโดย Microsoft ซึ่งเพิ่มระบบ Static Type เข้าไป

ความแตกต่างหลัก:

```typescript
// JavaScript - ไม่มี type checking
function add(a, b) {
  return a + b;
}
add("1", 2); // ไม่มี error แต่ผลลัพธ์คือ "12"

// TypeScript - มี type checking
function add(a: number, b: number): number {
  return a + b;
}
add("1", 2); // Error: Argument of type 'string' is not assignable to parameter of type 'number'
```

ข้อดีของ TypeScript:
1. **Type Safety**: ตรวจจับ error ก่อน runtime
2. **Better IDE Support**: Autocomplete, refactoring ที่ดีกว่า
3. **Readable Code**: โค้ดอ่านง่ายและเข้าใจง่าย
4. **Large-scale Development**: เหมาะสำหรับโปรเจกต์ขนาดใหญ่

---

### คำถามที่ 2: Types พื้นฐานใน TypeScript มีอะไรบ้าง?

**คำตอบ:**

```typescript
// Primitive Types
let name: string = "John";
let age: number = 30;
let isActive: boolean = true;
let value: undefined = undefined;
let nothing: null = null;

// Special Types
let anything: any = "สามารถเป็นอะไรก็ได้";
let unknownValue: unknown = 42;
let neverReturns: never; // ฟังก์ชันที่ไม่มีทางคืนค่าได้

// Object Types
let person: object = { name: "John" };

// Array Types
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b", "c"];

// Tuple Types
let tuple: [string, number] = ["John", 30];

// Void Type
function greet(): void {
  console.log("Hello!");
}
```

---

### คำถามที่ 3: Interface กับ Type Alias ต่างกันอย่างไร?

**คำตอบ:**

```typescript
// Interface
interface User {
  name: string;
  age: number;
}

// Type Alias
type User = {
  name: string;
  age: number;
};

// ความแตกต่างสำคัญ:

// 1. Interface สามารถ extend ได้
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
}

// 2. Type สามารถ union/intersection ได้
type StringOrNumber = string | number;
type AdminUser = User & { role: string };

// 3. Interface สามารถ merge ได้ (Declaration Merging)
interface Window {
  myCustomProp: string;
}
interface Window {
  anotherProp: number;
}
// ทั้งสองถูก merge เข้ากัน

// 4. Type ไม่สามารถ merge ได้
type Config = { debug: boolean };
// type Config = { verbose: boolean }; // Error!

// แนะนำ: ใช้ Interface สำหรับ object shapes, ใช้ Type สำหรับ unions/intersections
```

---

### คำถามที่ 4: Optional Properties และ Readonly คืออะไร?

**คำตอบ:**

```typescript
interface Config {
  host: string;
  port?: number; // optional property
  readonly apiKey: string; // readonly property
}

const config: Config = {
  host: "localhost",
  apiKey: "secret123"
};

// config.apiKey = "new-secret"; // Error: Cannot assign to 'apiKey' because it is read-only

// Optional chaining
const port = config.port ?? 3000; // ใช้ 3000 ถ้า port เป็น undefined

// Readonly Arrays
const arr: readonly number[] = [1, 2, 3];
// arr.push(4); // Error!
// arr[0] = 10; // Error!
```

---

### คำถามที่ 5: Union Types และ Intersection Types คืออะไร?

**คำตอบ:**

```typescript
// Union Types - เป็น A หรือ B
type StringOrNumber = string | number;
type Status = "active" | "inactive" | "pending";

function formatId(id: string | number): string {
  if (typeof id === "string") {
    return id.toUpperCase();
  }
  return id.toString();
}

// Intersection Types - เป็นทั้ง A และ B
interface Named {
  name: string;
}

interface Aged {
  age: number;
}

type Person = Named & Aged;

const person: Person = {
  name: "John",
  age: 30
};

// ตัวอย่างใช้งานจริง
type AdminUser = User & {
  role: "admin";
  permissions: string[];
};
```

---

### คำถามที่ 6: Type Assertion คืออะไร?

**คำตอบ:**

```typescript
// Type Assertion บอก TypeScript ว่า type คืออะไร
const input = document.getElementById("myInput") as HTMLInputElement;
input.value = "Hello";

// หรือใช้ angle bracket syntax (ไม่แนะนำใน JSX)
const input2 = <HTMLInputElement>document.getElementById("myInput");

// Non-null assertion operator
const element = document.getElementById("myDiv")!; // บอกว่าไม่ null

// ระวัง: Type assertion ไม่ใช่ Type conversion
const num = "123" as unknown as number; // ทำได้แต่อันตราย
// ไม่เหมือน Number("123") ที่แปลงค่าจริงๆ

// ตัวอย่างที่ดี
interface ApiResponse {
  data: unknown;
}

function processData(response: ApiResponse) {
  const data = response.data as { name: string; age: number };
  console.log(data.name);
}
```

---

### คำถามที่ 7: Generics คืออะไร? ทำไมถึงสำคัญ?

**คำตอบ:**

```typescript
// ปัญหาโดยไม่ใช้ Generics
function getFirstItem(arr: any[]): any {
  return arr[0];
}
const first = getFirstItem([1, 2, 3]); // type เป็น any ไม่มี type safety

// แก้ด้วย Generics
function getFirstItem<T>(arr: T[]): T {
  return arr[0];
}

const firstNum = getFirstItem<number>([1, 2, 3]); // type เป็น number
const firstStr = getFirstItem(["a", "b", "c"]); // TypeScript อนุมาน type เอง

// Generic Interface
interface Repository<T> {
  findById(id: string): T | null;
  findAll(): T[];
  save(entity: T): void;
  delete(id: string): void;
}

// Generic Class
class Stack<T> {
  private items: T[] = [];
  
  push(item: T): void {
    this.items.push(item);
  }
  
  pop(): T | undefined {
    return this.items.pop();
  }
  
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }
  
  get size(): number {
    return this.items.length;
  }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
console.log(numStack.pop()); // 2
```

---

### คำถามที่ 8: Enum คืออะไร มีกี่ประเภท?

**คำตอบ:**

```typescript
// Numeric Enum (default)
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}

// String Enum
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
  Pending = "PENDING"
}

// Const Enum (inlined at compile time)
const enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE"
}

// Heterogeneous Enum (ไม่แนะนำ)
enum Mixed {
  No = 0,
  Yes = "YES"
}

// ตัวอย่างการใช้งาน
function getDirection(dir: Direction): string {
  switch (dir) {
    case Direction.Up: return "going up";
    case Direction.Down: return "going down";
    default: return "going sideways";
  }
}

// Enum ใน runtime
console.log(Direction.Up); // 0
console.log(Direction[0]); // "Up" (reverse mapping สำหรับ numeric enum)
console.log(Status.Active); // "ACTIVE"
```

---

### คำถามที่ 9: Type Guards คืออะไร?

**คำตอบ:**

```typescript
// typeof Type Guard
function processValue(value: string | number) {
  if (typeof value === "string") {
    // value เป็น string ที่นี่
    return value.toUpperCase();
  }
  // value เป็น number ที่นี่
  return value.toFixed(2);
}

// instanceof Type Guard
class Dog {
  bark() { return "Woof!"; }
}
class Cat {
  meow() { return "Meow!"; }
}

function makeSound(animal: Dog | Cat) {
  if (animal instanceof Dog) {
    return animal.bark();
  }
  return animal.meow();
}

// in Type Guard
interface Fish {
  swim(): void;
}
interface Bird {
  fly(): void;
}

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim();
  } else {
    animal.fly();
  }
}

// Custom Type Guard (Type Predicate)
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function processUnknown(value: unknown) {
  if (isString(value)) {
    console.log(value.toUpperCase()); // value เป็น string ที่นี่
  }
}
```

---

### คำถามที่ 10: Decorators ใน TypeScript คืออะไร?

**คำตอบ:**

```typescript
// ต้องเปิด experimentalDecorators ใน tsconfig.json

// Class Decorator
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class BugReport {
  type = "report";
  title: string;
  
  constructor(t: string) {
    this.title = t;
  }
}

// Method Decorator
function log(target: any, key: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with`, args);
    return originalMethod.apply(this, args);
  };
  return descriptor;
}

class Calculator {
  @log
  add(a: number, b: number) {
    return a + b;
  }
}

// Property Decorator
function readonly(target: any, key: string) {
  const descriptor: PropertyDescriptor = {
    get() { return "readonly value"; },
    enumerable: true,
    configurable: false
  };
  Object.defineProperty(target, key, descriptor);
}
```

---

### คำถามที่ 11: tsconfig.json ที่สำคัญมีอะไรบ้าง?

**คำตอบ:**

```json
{
  "compilerOptions": {
    "target": "ES2020",          // JavaScript version ที่ compile ออกมา
    "module": "commonjs",         // Module system
    "lib": ["ES2020", "DOM"],     // Built-in type definitions
    "outDir": "./dist",           // Output directory
    "rootDir": "./src",           // Source directory
    "strict": true,               // เปิด strict mode ทั้งหมด
    "noImplicitAny": true,        // ห้าม implicit any
    "strictNullChecks": true,     // null/undefined ต้องตรวจสอบ
    "strictFunctionTypes": true,  // Function type checking ที่เข้มงวด
    "noUnusedLocals": true,       // Error ถ้ามี unused variables
    "noUnusedParameters": true,   // Error ถ้ามี unused parameters
    "noImplicitReturns": true,    // ต้อง return ในทุก code path
    "esModuleInterop": true,      // CommonJS/ES module compatibility
    "experimentalDecorators": true, // เปิดใช้ Decorators
    "emitDecoratorMetadata": true,  // Decorator metadata
    "resolveJsonModule": true,    // Import JSON files
    "declaration": true,          // Generate .d.ts files
    "sourceMap": true             // Generate source maps
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

### คำถามที่ 12: Utility Types คืออะไร มีอะไรบ้าง?

**คำตอบ:**

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

// Partial - ทุก property เป็น optional
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; age?: number; }

// Required - ทุก property เป็น required
type RequiredUser = Required<PartialUser>;

// Readonly - ทุก property เป็น readonly
type ReadonlyUser = Readonly<User>;

// Pick - เลือก properties บางส่วน
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string; }

// Omit - ลบ properties บางส่วน
type UserWithoutId = Omit<User, "id">;
// { name: string; email: string; age: number; }

// Record - สร้าง object type
type UserMap = Record<string, User>;
// { [key: string]: User }

// Exclude - ลบ types ออกจาก union
type NotString = Exclude<string | number | boolean, string>;
// number | boolean

// Extract - เอาเฉพาะ types ที่อยู่ใน union
type OnlyString = Extract<string | number | boolean, string>;
// string

// NonNullable - ลบ null และ undefined
type SafeString = NonNullable<string | null | undefined>;
// string

// ReturnType - ดึง return type ของ function
function getUser(): User { return {} as User; }
type UserReturnType = ReturnType<typeof getUser>; // User

// Parameters - ดึง parameter types ของ function
function createUser(name: string, age: number): void {}
type CreateUserParams = Parameters<typeof createUser>; // [string, number]

// InstanceType - ดึง type ของ class instance
class MyClass { value = 42; }
type MyInstance = InstanceType<typeof MyClass>; // MyClass
```

---

### คำถามที่ 13: Conditional Types คืออะไร?

**คำตอบ:**

```typescript
// ไวยากรณ์: T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Conditional Types กับ Generics
type Flatten<T> = T extends Array<infer Item> ? Item : T;

type Str = Flatten<string[]>;   // string
type Num = Flatten<number>;     // number

// infer keyword
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function greet(name: string): string {
  return `Hello, ${name}!`;
}

type GreetReturn = ReturnType<typeof greet>; // string

// Distributive Conditional Types
type NonNullable<T> = T extends null | undefined ? never : T;

type Clean = NonNullable<string | null | undefined | number>;
// string | number
```

---

### คำถามที่ 14: Mapped Types คืออะไร?

**คำตอบ:**

```typescript
// Mapped Types สร้าง type ใหม่จาก type เดิม
type Optional<T> = {
  [K in keyof T]?: T[K];
};

type ReadOnly<T> = {
  readonly [K in keyof T]: T[K];
};

// ตัวอย่างการใช้งาน
interface Config {
  host: string;
  port: number;
  debug: boolean;
}

type OptionalConfig = Optional<Config>;
// { host?: string; port?: number; debug?: boolean; }

// + and - modifiers
type Mutable<T> = {
  -readonly [K in keyof T]: T[K]; // ลบ readonly
};

type AllRequired<T> = {
  [K in keyof T]-?: T[K]; // ลบ optional
};

// Key remapping (TypeScript 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number; }
```

---

### คำถามที่ 15: Template Literal Types คืออะไร?

**คำตอบ:**

```typescript
// Template Literal Types (TypeScript 4.1+)
type EventName = "click" | "focus" | "blur";
type EventHandler = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// ตัวอย่างการใช้งาน
type CSSProperty = "margin" | "padding";
type CSSDirection = "Top" | "Right" | "Bottom" | "Left";
type CSSShorthand = `${CSSProperty}${CSSDirection}`;
// "marginTop" | "marginRight" | ... | "paddingLeft"

// กับ API endpoints
type HttpMethod = "get" | "post" | "put" | "delete";
type ApiPath = "/users" | "/posts" | "/comments";
type ApiEndpoint = `${Uppercase<HttpMethod>} ${ApiPath}`;
// "GET /users" | "GET /posts" | ... | "DELETE /comments"

// Dynamic object keys
type EventMap<T extends string> = {
  [K in T as `on${Capitalize<K>}`]: () => void;
};

type ClickEvents = EventMap<"click" | "focus">;
// { onClick: () => void; onFocus: () => void; }
```

---

## ส่วนที่ 2: คำถามระดับกลาง (Intermediate Level)

### คำถามที่ 16: Discriminated Unions คืออะไร?

**คำตอบ:**

```typescript
// Discriminated Unions ใช้ literal type เป็น discriminant
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
    default:
      // Exhaustiveness check
      const _exhaustive: never = shape;
      throw new Error(`Unknown shape: ${JSON.stringify(_exhaustive)}`);
  }
}

// Result type pattern
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { success: false, error: "Division by zero" };
  }
  return { success: true, data: a / b };
}

const result = divide(10, 2);
if (result.success) {
  console.log(result.data); // number
} else {
  console.log(result.error); // string
}
```

---

### คำถามที่ 17: Function Overloads คืออะไร?

**คำตอบ:**

```typescript
// Function Overloads
function processInput(input: string): string;
function processInput(input: number): number;
function processInput(input: string | number): string | number {
  if (typeof input === "string") {
    return input.toUpperCase();
  }
  return input * 2;
}

const result1 = processInput("hello"); // type: string
const result2 = processInput(42);      // type: number

// Overloads กับ Optional Parameters
function createElement(tag: "a"): HTMLAnchorElement;
function createElement(tag: "canvas"): HTMLCanvasElement;
function createElement(tag: "table"): HTMLTableElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const anchor = createElement("a");    // HTMLAnchorElement
const canvas = createElement("canvas"); // HTMLCanvasElement

// Method Overloads ใน Class
class Formatter {
  format(value: string): string;
  format(value: number, decimals: number): string;
  format(value: string | number, decimals?: number): string {
    if (typeof value === "string") {
      return value.trim();
    }
    return value.toFixed(decimals ?? 2);
  }
}
```

---

### คำถามที่ 18: Abstract Classes คืออะไร?

**คำตอบ:**

```typescript
// Abstract Class - ไม่สามารถสร้าง instance โดยตรงได้
abstract class Animal {
  abstract name: string;
  abstract makeSound(): string;
  
  // Concrete method
  describe(): string {
    return `I am ${this.name} and I say ${this.makeSound()}`;
  }
}

class Dog extends Animal {
  name = "Dog";
  
  makeSound(): string {
    return "Woof!";
  }
}

class Cat extends Animal {
  name = "Cat";
  
  makeSound(): string {
    return "Meow!";
  }
}

// const animal = new Animal(); // Error!
const dog = new Dog();
console.log(dog.describe()); // "I am Dog and I say Woof!"

// Abstract กับ Generic
abstract class Repository<T> {
  protected items: T[] = [];
  
  abstract findById(id: string): T | undefined;
  
  findAll(): T[] {
    return this.items;
  }
  
  count(): number {
    return this.items.length;
  }
}
```

---

### คำถามที่ 19: Index Signatures คืออะไร?

**คำตอบ:**

```typescript
// Index Signatures
interface StringMap {
  [key: string]: string;
}

const translations: StringMap = {
  hello: "สวัสดี",
  goodbye: "ลาก่อน",
  thanks: "ขอบคุณ"
};

// Mixed Index Signature
interface Config {
  [key: string]: string | number | boolean;
  host: string;  // ต้องเป็น type ที่ compatible กับ index signature
  port: number;
}

// Readonly Index Signature
interface ReadonlyStringMap {
  readonly [key: string]: string;
}

// Number Index Signature
interface NumberMap {
  [index: number]: string;
}

const arr: NumberMap = ["a", "b", "c"];

// ตัวอย่างการใช้งานจริง
interface Translations {
  [locale: string]: {
    [key: string]: string;
  };
}

const i18n: Translations = {
  en: { hello: "Hello", goodbye: "Goodbye" },
  th: { hello: "สวัสดี", goodbye: "ลาก่อน" }
};
```

---

### คำถามที่ 20: Namespace vs Modules ต่างกันอย่างไร?

**คำตอบ:**

```typescript
// Namespaces (internal modules)
namespace Validation {
  export interface StringValidator {
    isAcceptable(s: string): boolean;
  }
  
  export class LettersOnlyValidator implements StringValidator {
    isAcceptable(s: string) {
      return /^[A-Za-z]+$/.test(s);
    }
  }
  
  export class ZipCodeValidator implements StringValidator {
    isAcceptable(s: string) {
      return s.length === 5 && /^[0-9]+$/.test(s);
    }
  }
}

const validator = new Validation.LettersOnlyValidator();

// ES Modules (recommended)
// utils.ts
export function formatDate(date: Date): string {
  return date.toISOString();
}

export interface FormattedDate {
  date: string;
  time: string;
}

// main.ts
import { formatDate, FormattedDate } from "./utils";

// ความแตกต่าง:
// - Modules: ไฟล์แต่ละไฟล์เป็น module, ใช้ import/export
// - Namespaces: ใช้ใน global scope, เหมาะสำหรับ browser (script tags)
// - Modern TypeScript: ใช้ ES Modules เป็นหลัก
```

---

### คำถามที่ 21: Declaration Files (.d.ts) คืออะไร?

**คำตอบ:**

```typescript
// .d.ts files - บอก TypeScript เกี่ยวกับ type ของ JavaScript libraries

// lodash.d.ts (simplified)
declare module "lodash" {
  export function chunk<T>(array: T[], size?: number): T[][];
  export function flatten<T>(array: T[][]): T[];
  export function groupBy<T>(
    collection: T[],
    iteratee: keyof T | ((item: T) => string)
  ): Record<string, T[]>;
}

// Global declarations
declare const __DEV__: boolean;
declare const __VERSION__: string;

declare function require(module: string): any;

// Ambient declarations
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: "development" | "production" | "test";
    PORT?: string;
    DATABASE_URL: string;
  }
}

// ใน code จะใช้ได้เลย
const isDev = __DEV__;
const dbUrl = process.env.DATABASE_URL; // type: string
const port = process.env.PORT; // type: string | undefined
```

---

### คำถามที่ 22: Covariance และ Contravariance คืออะไร?

**คำตอบ:**

```typescript
// Covariance - ประเภทลูกสามารถใช้แทนประเภทแม่ได้
class Animal {
  breathe() {}
}

class Dog extends Animal {
  bark() {}
}

// Array is covariant
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs; // OK - Dog[] ใช้แทน Animal[] ได้

// Function return types are covariant
type GetAnimal = () => Animal;
type GetDog = () => Dog;

const getDog: GetDog = () => new Dog();
const getAnimal: GetAnimal = getDog; // OK

// Contravariance - function parameters
type HandleAnimal = (animal: Animal) => void;
type HandleDog = (dog: Dog) => void;

const handleAnimal: HandleAnimal = (a: Animal) => a.breathe();
// const handleDog: HandleDog = handleAnimal; // ไม่ OK โดยค่าเริ่มต้น

// strictFunctionTypes ทำให้ function parameters เป็น contravariant
// นั่นคือ HandleAnimal ไม่สามารถใช้แทน HandleDog ได้

// Bivariance (เดิม - อนุญาตทั้งสองทิศทาง)
interface LegacyHandler {
  handle(animal: Animal): void; // bivariant
}
```

---

### คำถามที่ 23: Symbols ใน TypeScript คืออะไร?

**คำตอบ:**

```typescript
// Symbols - unique identifiers
const sym1 = Symbol("description");
const sym2 = Symbol("description");

console.log(sym1 === sym2); // false - แต่ละ Symbol ไม่ซ้ำกัน

// Symbol เป็น type
type SymbolType = typeof sym1;

// Symbols เป็น object keys
const KEY = Symbol("key");
const obj = {
  [KEY]: "value",
  name: "John"
};

console.log(obj[KEY]); // "value"

// Well-known Symbols
class MyArray {
  static [Symbol.hasInstance](instance: any) {
    return Array.isArray(instance);
  }
}

// unique symbol
const uniqueSym: unique symbol = Symbol();
// type ของ uniqueSym เป็น typeof uniqueSym ไม่ใช่ symbol

// Symbols ใน enum-like patterns
const Permission = {
  READ: Symbol("read"),
  WRITE: Symbol("write"),
  DELETE: Symbol("delete")
} as const;
```

---

### คำถามที่ 24: Type Narrowing เทคนิคต่างๆ คืออะไร?

**คำตอบ:**

```typescript
// 1. typeof narrowing
function processValue(value: string | number | boolean) {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else if (typeof value === "number") {
    console.log(value.toFixed(2));
  } else {
    console.log(value ? "yes" : "no");
  }
}

// 2. Truthiness narrowing
function greet(name: string | null | undefined) {
  if (name) {
    console.log(`Hello, ${name}!`);
  } else {
    console.log("Hello, stranger!");
  }
}

// 3. Equality narrowing
function compare(a: string | number, b: string | boolean) {
  if (a === b) {
    // a และ b ต้องเป็น string
    console.log(a.toUpperCase());
  }
}

// 4. in operator narrowing
interface Admin { role: "admin"; permissions: string[] }
interface User { role: "user"; email: string }

function processUser(user: Admin | User) {
  if ("permissions" in user) {
    console.log(user.permissions); // Admin
  } else {
    console.log(user.email); // User
  }
}

// 5. instanceof narrowing
function handleError(error: Error | string) {
  if (error instanceof Error) {
    console.log(error.message);
  } else {
    console.log(error);
  }
}

// 6. Control flow analysis
function example(x: string | number | boolean) {
  if (typeof x !== "boolean") {
    // x เป็น string | number
    if (typeof x === "string") {
      // x เป็น string
      return x.length;
    }
    // x เป็น number ที่นี่
    return x;
  }
  return x;
}

// 7. Assertion functions
function assert(condition: boolean, msg?: string): asserts condition {
  if (!condition) {
    throw new Error(msg ?? "Assertion failed");
  }
}

function processId(id: string | undefined) {
  assert(id !== undefined, "id must be defined");
  // id เป็น string ที่นี่
  return id.toUpperCase();
}
```

---

### คำถามที่ 25: Excess Property Checking คืออะไร?

**คำตอบ:**

```typescript
interface Point {
  x: number;
  y: number;
}

// Excess property checking เมื่อ assign ตรง
const p1: Point = { x: 1, y: 2, z: 3 }; // Error: Object literal may only specify known properties

// แต่ไม่ตรวจเมื่อ assign ผ่าน intermediate variable
const temp = { x: 1, y: 2, z: 3 };
const p2: Point = temp; // OK! (structural typing)

// และไม่ตรวจเมื่อ pass เป็น argument ผ่าน variable
function getDistance(p: Point): number {
  return Math.sqrt(p.x ** 2 + p.y ** 2);
}

getDistance({ x: 1, y: 2, z: 3 }); // Error: Object literal
getDistance(temp); // OK!

// ทำไมถึงมี excess property checking?
// ป้องกัน typo ใน object literals
interface Config {
  host: string;
  port: number;
}

const config: Config = {
  host: "localhost",
  port: 3000,
  prot: 3001 // Error: 'prot' ไม่มีใน Config (อาจเป็น typo ของ port)
};
```

---

### คำถามที่ 26: Class Access Modifiers คืออะไร?

**คำตอบ:**

```typescript
class BankAccount {
  // public - เข้าถึงได้จากทุกที่ (default)
  public accountNumber: string;
  
  // private - เข้าถึงได้เฉพาะภายใน class เท่านั้น
  private balance: number;
  
  // protected - เข้าถึงได้ภายใน class และ subclasses
  protected owner: string;
  
  // readonly - ตั้งค่าได้ครั้งเดียว
  readonly createdAt: Date;
  
  constructor(accountNumber: string, owner: string, initialBalance: number) {
    this.accountNumber = accountNumber;
    this.owner = owner;
    this.balance = initialBalance;
    this.createdAt = new Date();
  }
  
  // Private method
  private validateAmount(amount: number): boolean {
    return amount > 0 && amount <= this.balance;
  }
  
  withdraw(amount: number): boolean {
    if (this.validateAmount(amount)) {
      this.balance -= amount;
      return true;
    }
    return false;
  }
  
  getBalance(): number {
    return this.balance;
  }
}

class SavingsAccount extends BankAccount {
  private interestRate: number;
  
  constructor(accountNumber: string, owner: string, balance: number, rate: number) {
    super(accountNumber, owner, balance);
    this.interestRate = rate;
  }
  
  addInterest(): void {
    // สามารถเข้าถึง protected ได้
    console.log(`Adding interest for ${this.owner}`);
  }
}

// ECMAScript private fields (แตกต่างจาก TypeScript private)
class ModernClass {
  #privateField: string = "truly private"; // ซ่อนใน runtime ด้วย
  
  getField() {
    return this.#privateField;
  }
}
```

---

### คำถามที่ 27: Mixins ใน TypeScript คืออะไร?

**คำตอบ:**

```typescript
// Mixins - เพิ่ม behaviors ให้กับ classes

// Mixin constructor type
type Constructor<T = {}> = new (...args: any[]) => T;

// Timestamped Mixin
function Timestamped<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();
    
    touch() {
      this.updatedAt = new Date();
    }
  };
}

// Activatable Mixin
function Activatable<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    isActive = false;
    
    activate() {
      this.isActive = true;
    }
    
    deactivate() {
      this.isActive = false;
    }
  };
}

// Base class
class User {
  constructor(public name: string) {}
}

// Apply Mixins
const TimestampedUser = Timestamped(User);
const TimestampedActivatableUser = Activatable(Timestamped(User));

const user = new TimestampedActivatableUser("John");
user.activate();
console.log(user.isActive);  // true
console.log(user.createdAt); // Date
user.touch();
```

---

### คำถามที่ 28: Module Augmentation คืออะไร?

**คำตอบ:**

```typescript
// เพิ่ม types ให้กับ existing modules

// ตัวอย่าง: เพิ่ม methods ให้กับ Express Request
// types/express.d.ts
import "express";

declare module "express-serve-static-core" {
  interface Request {
    user?: {
      id: string;
      email: string;
      role: string;
    };
    session: {
      token?: string;
    };
  }
}

// ตอนใช้งาน
import express from "express";

const app = express();

app.get("/profile", (req, res) => {
  if (req.user) { // ไม่ Error เพราะเรา augment แล้ว
    res.json(req.user);
  }
});

// Global Augmentation
// ใส่ใน .d.ts file
declare global {
  interface Array<T> {
    last(): T | undefined;
    first(): T | undefined;
  }
}

// Implementation
Array.prototype.last = function() {
  return this[this.length - 1];
};

Array.prototype.first = function() {
  return this[0];
};
```

---

### คำถามที่ 29: Variance ใน Type Parameters คืออะไร?

**คำตอบ:**

```typescript
// TypeScript 4.7+ Variance Annotations

// in - contravariant (เขียนเข้า)
interface Sink<in T> {
  write(value: T): void;
}

// out - covariant (อ่านออก)
interface Source<out T> {
  read(): T;
}

// in out - invariant (ทั้งอ่านและเขียน)
interface Container<in out T> {
  get(): T;
  set(value: T): void;
}

// ตัวอย่างที่ชัดเจน
// Covariant: Producer<Dog> ใช้แทน Producer<Animal> ได้
interface Producer<out T> {
  produce(): T;
}

const dogProducer: Producer<Dog> = { produce: () => new Dog() };
const animalProducer: Producer<Animal> = dogProducer; // OK

// Contravariant: Consumer<Animal> ใช้แทน Consumer<Dog> ได้
interface Consumer<in T> {
  consume(value: T): void;
}

const animalConsumer: Consumer<Animal> = { consume: (a: Animal) => {} };
const dogConsumer: Consumer<Dog> = animalConsumer; // OK
```

---

### คำถามที่ 30: Higher-order Types คืออะไร?

**คำตอบ:**

```typescript
// Higher-order Types - types ที่รับ types เป็น argument

// Generic Type Alias (higher-order type)
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Maybe<T> = T | null | undefined;

// Type transformers
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Type composition
type Compose<F extends (x: any) => any, G extends (x: any) => any> =
  F extends (x: infer A) => infer B
    ? G extends (x: B) => infer C
      ? (x: A) => C
      : never
    : never;

// ตัวอย่างการใช้งาน
interface DeepConfig {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
}

type PartialConfig = DeepPartial<DeepConfig>;
// ทุก field เป็น optional รวมถึง nested ด้วย
```

---

## ส่วนที่ 3: คำถามระดับขั้นสูง (Advanced Level)

### คำถามที่ 31: Type-level Programming คืออะไร?

**คำตอบ:**

```typescript
// Type-level Programming - เขียน logic ด้วย Types

// Type-level equality check
type IsEqual<A, B> = A extends B ? B extends A ? true : false : false;

type Test1 = IsEqual<string, string>; // true
type Test2 = IsEqual<string, number>; // false

// Type-level conditional
type If<Condition extends boolean, Then, Else> =
  Condition extends true ? Then : Else;

type Result = If<IsEqual<1, 1>, "equal", "not equal">; // "equal"

// Type-level recursion
type BuildArray<N extends number, T = unknown, Acc extends T[] = []> =
  Acc["length"] extends N
    ? Acc
    : BuildArray<N, T, [...Acc, T]>;

type Array5 = BuildArray<5>; // [unknown, unknown, unknown, unknown, unknown]

// Type-level arithmetic (addition)
type Add<A extends number, B extends number> =
  [...BuildArray<A>, ...BuildArray<B>]["length"];

type Sum = Add<3, 4>; // 7

// Type-level string operations
type Split<S extends string, D extends string> =
  S extends `${infer T}${D}${infer U}` ? [T, ...Split<U, D>] : [S];

type Parts = Split<"a,b,c", ",">; // ["a", "b", "c"]
```

---

### คำถามที่ 32: Branded Types คืออะไร?

**คำตอบ:**

```typescript
// Branded Types - เพิ่ม nominal typing ให้กับ TypeScript

// แบบเดิม - structural typing อาจทำให้สับสน
function createUser(id: string, email: string) {
  return { id, email };
}
createUser(email, id); // TypeScript ไม่ error แต่ logic ผิด!

// Branded Types แก้ปัญหานี้
declare const __brand: unique symbol;
type Brand<T, B> = T & { [__brand]: B };

type UserId = Brand<string, "UserId">;
type Email = Brand<string, "Email">;

function createUserId(id: string): UserId {
  return id as UserId;
}

function createEmail(email: string): Email {
  return email as Email;
}

function createUser(id: UserId, email: Email) {
  return { id, email };
}

const userId = createUserId("user-123");
const email = createEmail("john@example.com");

createUser(userId, email); // OK
// createUser(email, userId); // Error! Email ไม่ใช่ UserId

// ตัวอย่างเพิ่มเติม
type Kilometers = Brand<number, "Kilometers">;
type Miles = Brand<number, "Miles">;

function kmToMiles(km: Kilometers): Miles {
  return (km * 0.621371) as Miles;
}

const distance: Kilometers = 100 as Kilometers;
const inMiles = kmToMiles(distance); // OK
// const wrong = kmToMiles(100); // Error! 100 ไม่ใช่ Kilometers
```

---

### คำถามที่ 33: Phantom Types คืออะไร?

**คำตอบ:**

```typescript
// Phantom Types - type parameter ที่ไม่ปรากฏใน runtime

// ตัวอย่าง: State machine
type State = "pending" | "active" | "inactive";

interface Entity<TState extends State> {
  id: string;
  state: TState; // phantom type
}

type PendingEntity = Entity<"pending">;
type ActiveEntity = Entity<"active">;

function activate(entity: PendingEntity): ActiveEntity {
  return { ...entity, state: "active" as const } as ActiveEntity;
}

function deactivate(entity: ActiveEntity): Entity<"inactive"> {
  return { ...entity, state: "inactive" as const };
}

// เพิ่มความปลอดภัย
const pending: PendingEntity = { id: "1", state: "pending" };
const active = activate(pending);
// activate(active); // Error! ActiveEntity ไม่ใช่ PendingEntity

// Type-safe Units
type Meter<T extends number = number> = T & { _unit: "meter" };
type Second<T extends number = number> = T & { _unit: "second" };

function calculateSpeed(distance: Meter, time: Second): number {
  return distance / time;
}
```

---

### คำถามที่ 34: Opaque Types คืออะไร?

**คำตอบ:**

```typescript
// Opaque Types - ซ่อน implementation details

// Pattern 1: ใช้ unique symbol
declare const _opaque: unique symbol;

type Opaque<Type, Token> = Type & {
  readonly [_opaque]: Token;
};

// ใช้ in specific module only
type Password = Opaque<string, "Password">;
type HashedPassword = Opaque<string, "HashedPassword">;

// ฟังก์ชันเท่านั้นที่สร้างได้
function hashPassword(password: Password): HashedPassword {
  // bcrypt hash
  return "hashed_" + password as unknown as HashedPassword;
}

// ฟังก์ชันรับเฉพาะ HashedPassword
function verifyPassword(hashed: HashedPassword, attempt: Password): boolean {
  return hashed === "hashed_" + attempt;
}

// Pattern 2: Class-based opaque
class UserId {
  private constructor(private readonly value: string) {}
  
  static create(value: string): UserId {
    if (!value.startsWith("usr_")) {
      throw new Error("Invalid UserId format");
    }
    return new UserId(value);
  }
  
  toString(): string {
    return this.value;
  }
  
  equals(other: UserId): boolean {
    return this.value === other.value;
  }
}
```

---

### คำถามที่ 35: Type-safe Event Emitter คืออะไร?

**คำตอบ:**

```typescript
// Type-safe Event Emitter

type EventMap = {
  userCreated: { userId: string; email: string };
  userDeleted: { userId: string };
  orderPlaced: { orderId: string; total: number };
  paymentReceived: { orderId: string; amount: number };
};

class TypedEventEmitter<Events extends Record<string, unknown>> {
  private listeners: {
    [K in keyof Events]?: Array<(payload: Events[K]) => void>;
  } = {};
  
  on<K extends keyof Events>(
    event: K,
    listener: (payload: Events[K]) => void
  ): this {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
    return this;
  }
  
  off<K extends keyof Events>(
    event: K,
    listener: (payload: Events[K]) => void
  ): this {
    this.listeners[event] = this.listeners[event]?.filter(
      l => l !== listener
    );
    return this;
  }
  
  emit<K extends keyof Events>(event: K, payload: Events[K]): void {
    this.listeners[event]?.forEach(listener => listener(payload));
  }
  
  once<K extends keyof Events>(
    event: K,
    listener: (payload: Events[K]) => void
  ): this {
    const wrappedListener = (payload: Events[K]) => {
      listener(payload);
      this.off(event, wrappedListener);
    };
    return this.on(event, wrappedListener);
  }
}

// การใช้งาน
const emitter = new TypedEventEmitter<EventMap>();

emitter.on("userCreated", ({ userId, email }) => {
  console.log(`User ${userId} created with email ${email}`);
});

emitter.emit("userCreated", { userId: "123", email: "john@example.com" }); // OK
// emitter.emit("userCreated", { userId: "123" }); // Error! email ขาดอยู่
// emitter.on("unknownEvent", () => {}); // Error! event ไม่มีใน EventMap
```

---

### คำถามที่ 36: Type-safe Fetch Wrapper คืออะไร?

**คำตอบ:**

```typescript
// Type-safe Fetch Wrapper

interface ApiEndpoints {
  "GET /users": {
    request: { page?: number; limit?: number };
    response: { users: User[]; total: number };
  };
  "GET /users/:id": {
    request: { id: string };
    response: User;
  };
  "POST /users": {
    request: { name: string; email: string };
    response: User;
  };
  "DELETE /users/:id": {
    request: { id: string };
    response: { success: boolean };
  };
}

type ExtractMethod<T extends string> =
  T extends `${infer Method} ${string}` ? Method : never;

type ExtractPath<T extends string> =
  T extends `${string} ${infer Path}` ? Path : never;

async function apiFetch<
  K extends keyof ApiEndpoints,
  Method extends ExtractMethod<K>,
  Path extends ExtractPath<K>
>(
  key: K,
  params: ApiEndpoints[K]["request"]
): Promise<ApiEndpoints[K]["response"]> {
  const [method, path] = key.split(" ");
  
  // Replace path parameters
  let url = path;
  const query: Record<string, string> = {};
  
  for (const [key, value] of Object.entries(params)) {
    if (url.includes(`:${key}`)) {
      url = url.replace(`:${key}`, String(value));
    } else {
      query[key] = String(value);
    }
  }
  
  const queryString = new URLSearchParams(query).toString();
  const fullUrl = queryString ? `${url}?${queryString}` : url;
  
  const options: RequestInit = {
    method,
    headers: { "Content-Type": "application/json" }
  };
  
  if (method !== "GET") {
    options.body = JSON.stringify(params);
  }
  
  const response = await fetch(fullUrl, options);
  return response.json();
}

// การใช้งาน
const users = await apiFetch("GET /users", { page: 1, limit: 10 });
// users.users เป็น User[], users.total เป็น number

const user = await apiFetch("GET /users/:id", { id: "123" });
// user เป็น User
```

---

### คำถามที่ 37: Recursive Types ปัญหาและวิธีแก้?

**คำตอบ:**

```typescript
// Recursive Types

// JSON type
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONArray
  | JSONObject;

interface JSONArray extends Array<JSONValue> {}
interface JSONObject {
  [key: string]: JSONValue;
}

// Tree structure
interface TreeNode<T> {
  value: T;
  children?: TreeNode<T>[];
}

// Deep types
type DeepReadonly<T> =
  T extends (infer U)[]
    ? DeepReadonlyArray<U>
    : T extends object
    ? DeepReadonlyObject<T>
    : T;

interface DeepReadonlyArray<T> extends ReadonlyArray<DeepReadonly<T>> {}

type DeepReadonlyObject<T> = {
  readonly [K in keyof T]: DeepReadonly<T[K]>;
};

// ตัวอย่างการใช้งาน
const config: DeepReadonly<{
  database: {
    host: string;
    port: number;
  };
}> = {
  database: {
    host: "localhost",
    port: 5432
  }
};

// config.database.host = "newhost"; // Error!
// config.database.port = 3306;     // Error!
```

---

### คำถามที่ 38: Performance Optimization ใน TypeScript คืออะไร?

**คำตอบ:**

```typescript
// Performance tips

// 1. ใช้ const enum แทน enum ปกติ
const enum Direction {
  North = "NORTH",
  South = "SOUTH"
}
// ถูก inline ที่ compile time ไม่มี runtime object

// 2. ใช้ as const สำหรับ literal types
const config = {
  host: "localhost",
  port: 3000,
} as const;

// 3. หลีกเลี่ยง any และ unknown ที่ไม่จำเป็น
// Bad
function process(data: any) {
  return data.value;
}

// Good
interface DataShape {
  value: string;
}
function process(data: DataShape) {
  return data.value;
}

// 4. ใช้ Lazy Type Evaluation
// Bad - recursive type ที่ TypeScript evaluate ช้า
type DeepNested<T, Depth extends number = 10> =
  Depth extends 0 ? T : { value: T; nested: DeepNested<T, Depth> };

// Better - ใช้ interface (lazy evaluation)
interface DeepNested<T> {
  value: T;
  nested?: DeepNested<T>;
}

// 5. Project References สำหรับ monorepo
// tsconfig.json
// {
//   "references": [
//     { "path": "./packages/common" },
//     { "path": "./packages/api" }
//   ]
// }
```

---

### คำถามที่ 39-50: Code Challenges

**Challenge 1: Implement a type-safe pipeline function**

```typescript
// Type-safe pipeline
type Fn<A, B> = (a: A) => B;

function pipe<A, B>(f: Fn<A, B>): Fn<A, B>;
function pipe<A, B, C>(f: Fn<A, B>, g: Fn<B, C>): Fn<A, C>;
function pipe<A, B, C, D>(f: Fn<A, B>, g: Fn<B, C>, h: Fn<C, D>): Fn<A, D>;
function pipe(...fns: Fn<any, any>[]): Fn<any, any> {
  return (input: any) => fns.reduce((acc, fn) => fn(acc), input);
}

// การใช้งาน
const process = pipe(
  (x: string) => parseInt(x),
  (x: number) => x * 2,
  (x: number) => x.toString()
);

console.log(process("21")); // "42"
```

**Challenge 2: Type-safe Builder Pattern**

```typescript
// Builder Pattern ที่ type-safe
class QueryBuilder<
  Table extends string,
  Selected extends keyof any = never,
  HasWhere extends boolean = false
> {
  private query: {
    table: Table;
    columns: string[];
    conditions: string[];
    orderBy?: string;
    limit?: number;
  };
  
  constructor(table: Table) {
    this.query = { table, columns: ["*"], conditions: [] };
  }
  
  select<K extends string>(
    ...columns: K[]
  ): QueryBuilder<Table, K, HasWhere> {
    return Object.assign(
      Object.create(QueryBuilder.prototype),
      { query: { ...this.query, columns } }
    ) as QueryBuilder<Table, K, HasWhere>;
  }
  
  where(
    condition: string
  ): QueryBuilder<Table, Selected, true> {
    return Object.assign(
      Object.create(QueryBuilder.prototype),
      { query: { ...this.query, conditions: [...this.query.conditions, condition] } }
    ) as QueryBuilder<Table, Selected, true>;
  }
  
  build(): string {
    const columns = this.query.columns.join(", ");
    let sql = `SELECT ${columns} FROM ${this.query.table}`;
    if (this.query.conditions.length > 0) {
      sql += ` WHERE ${this.query.conditions.join(" AND ")}`;
    }
    return sql;
  }
}

const query = new QueryBuilder("users")
  .select("id", "name", "email")
  .where("age > 18")
  .where("active = true")
  .build();

console.log(query);
// "SELECT id, name, email FROM users WHERE age > 18 AND active = true"
```

---

## ส่วนที่ 4: TypeScript Gotchas และปัญหาที่พบบ่อย

### Gotcha 1: any ทำลาย type safety

```typescript
// อันตราย!
function processData(data: any) {
  return data.name.toUpperCase(); // runtime error ถ้า data ไม่มี name
}

// ดีกว่า
function processData(data: unknown) {
  if (typeof data === "object" && data !== null && "name" in data) {
    const name = (data as { name: unknown }).name;
    if (typeof name === "string") {
      return name.toUpperCase();
    }
  }
  throw new Error("Invalid data format");
}
```

### Gotcha 2: Type assertions ไม่ตรวจสอบ runtime

```typescript
// TypeScript เชื่อคุณ แต่ถ้าผิดจะ error ตอน runtime
const value = "not a number" as unknown as number;
console.log(value.toFixed(2)); // Runtime error!

// ใช้ type guards แทน
function ensureNumber(value: unknown): number {
  if (typeof value !== "number") {
    throw new TypeError(`Expected number, got ${typeof value}`);
  }
  return value;
}
```

### Gotcha 3: Enum ใน JavaScript runtime

```typescript
// String enum ดีกว่า numeric enum
enum Status {
  Active = 0,  // ค่าใน JS คือ 0
  Inactive = 1 // ค่าใน JS คือ 1
}

// ถ้าเปลี่ยนลำดับ bug เกิดขึ้นได้
enum Status {
  Inactive = 0, // เปลี่ยนลำดับ แต่ค่าที่บันทึก 0 หมายถึง Inactive แล้ว
  Active = 1    // แต่ต้องการ Active!
}

// ใช้ string enum แทน
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE"
}
```

### Gotcha 4: Readonly ไม่ deep

```typescript
interface Config {
  readonly server: {
    host: string;
    port: number;
  };
}

const config: Config = {
  server: { host: "localhost", port: 3000 }
};

// config.server = {}; // Error! server เป็น readonly
config.server.host = "newhost"; // OK! แต่อาจไม่ต้องการ
config.server.port = 8080;     // OK!

// ใช้ DeepReadonly แทน
type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> };

const deepConfig: DeepReadonly<Config> = {
  server: { host: "localhost", port: 3000 }
};
// deepConfig.server.host = "new"; // Error!
```

### Gotcha 5: Overloaded functions และ implementation signature

```typescript
// Implementation signature ไม่ available ภายนอก
function create(a: string): string;
function create(a: number): number;
function create(a: string | number): string | number { // implementation
  return a;
}

create("hello"); // OK
create(42);      // OK
// create(true); // Error - boolean ไม่ใช่ string หรือ number

// implementation signature ต้องกว้างพอที่จะรองรับทุก overload
```

---

## ส่วนที่ 5: Real Interview Scenarios

### Scenario 1: API Response Handling

```typescript
// คำถาม: เขียน type-safe API response handler

// Solution
type ApiError = {
  code: string;
  message: string;
  details?: Record<string, string[]>;
};

type ApiSuccess<T> = {
  data: T;
  meta?: {
    page: number;
    limit: number;
    total: number;
  };
};

type ApiResponse<T> = ApiSuccess<T> | { error: ApiError };

function isApiError<T>(
  response: ApiResponse<T>
): response is { error: ApiError } {
  return "error" in response;
}

async function fetchUsers(): Promise<ApiResponse<User[]>> {
  try {
    const response = await fetch("/api/users");
    const data = await response.json();
    
    if (!response.ok) {
      return { error: { code: "FETCH_ERROR", message: data.message } };
    }
    
    return { data };
  } catch (e) {
    return {
      error: {
        code: "NETWORK_ERROR",
        message: "Network request failed"
      }
    };
  }
}

// การใช้งาน
const response = await fetchUsers();

if (isApiError(response)) {
  console.error(response.error.message);
} else {
  console.log(response.data); // User[]
}
```

### Scenario 2: Middleware Pattern

```typescript
// Type-safe Middleware
type Middleware<Context = {}, NextContext = Context> = (
  ctx: Context,
  next: (ctx: NextContext) => Promise<void>
) => Promise<void>;

type AuthContext = { user?: { id: string; role: string } };
type AuthenticatedContext = { user: { id: string; role: string } };

const authMiddleware: Middleware<AuthContext, AuthenticatedContext> = async (
  ctx,
  next
) => {
  const token = "...; // get from header
  const user = verifyToken(token);
  if (!user) throw new Error("Unauthorized");
  
  await next({ ...ctx, user });
};

// Stack middleware types
type MiddlewareStack<T extends any[]> = T extends []
  ? {}
  : T extends [Middleware<infer C, any>, ...infer Rest]
  ? C & MiddlewareStack<Rest extends Middleware<any, any>[] ? Rest : never>
  : never;
```

### Scenario 3: Form Validation

```typescript
// Type-safe Form Validation
type ValidationRule<T> = {
  validate: (value: T) => boolean;
  message: string;
};

type FieldValidator<T> = {
  rules: ValidationRule<T>[];
};

type FormSchema = {
  [field: string]: FieldValidator<any>;
};

type FormValues<S extends FormSchema> = {
  [K in keyof S]: S[K] extends FieldValidator<infer T> ? T : never;
};

type FormErrors<S extends FormSchema> = {
  [K in keyof S]?: string[];
};

function createForm<S extends FormSchema>(schema: S) {
  return {
    validate(values: FormValues<S>): FormErrors<S> {
      const errors: FormErrors<S> = {};
      
      for (const field in schema) {
        const validator = schema[field];
        const value = values[field];
        const fieldErrors: string[] = [];
        
        for (const rule of validator.rules) {
          if (!rule.validate(value)) {
            fieldErrors.push(rule.message);
          }
        }
        
        if (fieldErrors.length > 0) {
          errors[field] = fieldErrors;
        }
      }
      
      return errors;
    }
  };
}

// การใช้งาน
const userForm = createForm({
  name: {
    rules: [
      { validate: (v: string) => v.length > 0, message: "Name is required" },
      { validate: (v: string) => v.length >= 3, message: "Name must be at least 3 characters" }
    ]
  },
  age: {
    rules: [
      { validate: (v: number) => v >= 18, message: "Must be 18 or older" }
    ]
  }
});

const errors = userForm.validate({ name: "Jo", age: 16 });
// errors.name: ["Name must be at least 3 characters"]
// errors.age: ["Must be 18 or older"]
```

---

## ส่วนที่ 6: คำถาม OOP

### คำถามที่ 51: SOLID Principles ใน TypeScript

```typescript
// Single Responsibility
class UserRepository {
  async findById(id: string): Promise<User> { /* ... */ return {} as User; }
  async save(user: User): Promise<void> { /* ... */ }
}

class EmailService {
  async sendWelcome(email: string): Promise<void> { /* ... */ }
}

class UserService {
  constructor(
    private userRepo: UserRepository,
    private emailService: EmailService
  ) {}
  
  async createUser(data: CreateUserDto): Promise<User> {
    const user = await this.userRepo.save(data as User);
    await this.emailService.sendWelcome(data.email);
    return user;
  }
}

// Open/Closed - เปิดสำหรับ extension, ปิดสำหรับ modification
abstract class Shape {
  abstract area(): number;
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  area() { return Math.PI * this.radius ** 2; }
}

class Rectangle extends Shape {
  constructor(private w: number, private h: number) { super(); }
  area() { return this.w * this.h; }
}

// Liskov Substitution - subtype ต้องใช้แทน base type ได้
function calculateTotalArea(shapes: Shape[]): number {
  return shapes.reduce((sum, s) => sum + s.area(), 0);
}

const shapes: Shape[] = [new Circle(5), new Rectangle(4, 6)];
calculateTotalArea(shapes); // ทำงานได้กับทุก Shape

// Interface Segregation
interface Readable {
  read(): string;
}

interface Writable {
  write(data: string): void;
}

interface ReadWritable extends Readable, Writable {}

// Dependency Inversion
interface UserStore {
  findById(id: string): Promise<User>;
  save(user: User): Promise<void>;
}

class PostgresUserStore implements UserStore {
  async findById(id: string): Promise<User> { return {} as User; }
  async save(user: User): Promise<void> {}
}

class UserService {
  constructor(private store: UserStore) {} // ขึ้นกับ interface ไม่ใช่ implementation
}
```

---

## สรุปเพิ่มเติม: คำถามที่ถามบ่อยใน Interview

### คำถาม 52-60: TypeScript vs JavaScript

```typescript
// 1. TypeScript compile ไปเป็น JavaScript
// TypeScript code → TS Compiler → JavaScript code

// 2. TypeScript types ไม่มีใน runtime
type User = { name: string }; // ไม่มีใน compiled JS
// ใช้ class สำหรับ runtime type checks

// 3. Duck typing vs Structural typing
interface Duck {
  quack(): void;
  walk(): void;
}

class RubberDuck {
  quack() { console.log("Squeak!"); }
  walk() { console.log("Wobble!"); }
}

// TypeScript ยอมรับ RubberDuck เป็น Duck (structural)
const duck: Duck = new RubberDuck(); // OK!

// 4. Type inference
const numbers = [1, 2, 3]; // TypeScript รู้ว่าเป็น number[]
const first = numbers[0];   // TypeScript รู้ว่าเป็น number

// 5. strictNullChecks
// ปิด: null/undefined เป็น subtype ของทุก type
// เปิด: ต้องตรวจสอบ null/undefined ก่อนใช้งาน

function getName(user: { name: string } | null): string {
  // return user.name; // Error ถ้าเปิด strictNullChecks
  return user?.name ?? "Anonymous"; // Safe
}

// 6. Optional chaining และ nullish coalescing
const user = null;
const name = user?.profile?.name ?? "Default";

// 7. Non-null assertion
const element = document.getElementById("root")!;
// เตือน: จะ crash ถ้า element เป็น null จริงๆ

// 8. satisfies operator (TypeScript 4.9+)
type Color = "red" | "green" | "blue";
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255]
} satisfies Record<Color, string | number[]>;

// palette.red เป็น number[] ไม่ใช่ string | number[]
palette.red.map(v => v * 2);
```

---

## บทสรุป

TypeScript Interview Questions ที่ควรรู้จัก:

1. **พื้นฐาน**: Types, Interfaces, Generics, Enums
2. **กลาง**: Utility Types, Conditional Types, Mapped Types
3. **ขั้นสูง**: Type-level Programming, Branded Types, Phantom Types
4. **Patterns**: Builder, Repository, Event Emitter ที่ type-safe
5. **Gotchas**: any, type assertions, enum pitfalls
6. **OOP**: SOLID Principles ด้วย TypeScript
7. **Real World**: API handlers, middleware, form validation

การเตรียมตัวสัมภาษณ์ TypeScript:
- ฝึก type exercises บน TypeScript Playground
- อ่าน TypeScript handbook อย่างละเอียด
- ทำ projects จริงๆ เพื่อเข้าใจปัญหาจริง
- เรียนรู้จาก type definitions ของ library ที่ใช้

---

*จบ Part 56 - TypeScript Interview Questions*
