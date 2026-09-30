# บทที่ 3: Types พื้นฐานใน TypeScript

> เรียนรู้ระบบ Type ของ TypeScript อย่างลึกซึ้ง - ฐานรากสำคัญที่ทุกอย่างสร้างขึ้นมา

---

## สารบัญบทนี้

1. [Primitive Types](#1-primitive-types)
2. [Special Types](#2-special-types)
3. [Type Annotations](#3-type-annotations)
4. [Type Inference](#4-type-inference)
5. [Literal Types](#5-literal-types)
6. [Type Assertions](#6-type-assertions)
7. [ตารางเปรียบเทียบ Types](#7-ตารางเปรียบเทียบ-types)
8. [ตัวอย่างการใช้งานจริง 50+ ตัวอย่าง](#8-ตัวอย่างการใช้งานจริง)

---

## 1. Primitive Types

### string

Type สำหรับข้อความทุกรูปแบบ

```typescript
// ============================================
// string - ตัวอย่างพื้นฐาน
// ============================================

// การประกาศตัวแปร string
let firstName: string = "สมชาย";
let lastName: string = "ใจดี";
let greeting: string = `สวัสดี ${firstName} ${lastName}`;

console.log(greeting); // สวัสดี สมชาย ใจดี

// String methods ที่ใช้บ่อย
const message = "Hello, TypeScript!";
console.log(message.toUpperCase());    // HELLO, TYPESCRIPT!
console.log(message.toLowerCase());    // hello, typescript!
console.log(message.length);           // 18
console.log(message.includes("Type")); // true
console.log(message.replace("Hello", "สวัสดี")); // สวัสดี, TypeScript!
console.log(message.split(", "));     // ["Hello", "TypeScript!"]

// Template literals
const name = "TypeScript";
const version = 5;
const info = `ยินดีต้อนรับสู่ ${name} เวอร์ชัน ${version}!`;
console.log(info); // ยินดีต้อนรับสู่ TypeScript เวอร์ชัน 5!

// Multi-line strings
const poem = `
    บางครั้งการเขียนโค้ด
    ก็เหมือนกับการเขียนกลอน
    ต้องใส่ใจในทุกตัวอักษร
`;

// String concatenation
const part1 = "Hello";
const part2 = " World";
const combined = part1 + part2; // "Hello World"

// Type checking
function formatName(first: string, last: string): string {
    return `${last} ${first}`; // ไทย: นามสกุล ชื่อจริง
}

const fullName = formatName("สมชาย", "ใจดี");
console.log(fullName); // ใจดี สมชาย
```

### number

TypeScript ใช้ `number` สำหรับตัวเลขทุกประเภท (integer, float, negative)

```typescript
// ============================================
// number - ตัวอย่างพื้นฐาน
// ============================================

// Integer
let age: number = 25;
let year: number = 2024;

// Float
let price: number = 299.99;
let pi: number = 3.14159;

// Negative
let temperature: number = -5;
let debt: number = -50000;

// Special values
let infinity: number = Infinity;
let negInfinity: number = -Infinity;
let notANumber: number = NaN;

// Number systems
let decimal: number = 100;
let binary: number = 0b1100100;    // 100 ในระบบ binary
let octal: number = 0o144;         // 100 ในระบบ octal
let hex: number = 0x64;            // 100 ในระบบ hexadecimal
let bigNumber: number = 1_000_000; // ใช้ _ คั่นหลักล้าน

console.log(decimal === binary);  // true - ทั้งหมดเท่ากับ 100
console.log(bigNumber);           // 1000000

// Math operations
const a: number = 10;
const b: number = 3;

console.log(a + b);    // 13
console.log(a - b);    // 7
console.log(a * b);    // 30
console.log(a / b);    // 3.3333...
console.log(a % b);    // 1 (remainder)
console.log(a ** b);   // 1000 (power)
console.log(Math.floor(a / b)); // 3

// Number methods
const num = 3.14159;
console.log(num.toFixed(2));      // "3.14"
console.log(num.toPrecision(4));  // "3.142"
console.log(num.toString());      // "3.14159"
console.log(Number.isInteger(3)); // true
console.log(Number.isNaN(NaN));   // true
console.log(Number.isFinite(Infinity)); // false

// Practical example: คำนวณราคา
function calculateTax(price: number, taxRate: number): number {
    return Math.round(price * taxRate * 100) / 100;
}

function calculateTotal(price: number, taxRate: number): number {
    const tax = calculateTax(price, taxRate);
    return price + tax;
}

const itemPrice = 1000;
const vatRate = 0.07; // 7% VAT
console.log(`ราคา: ฿${itemPrice}`);
console.log(`ภาษี: ฿${calculateTax(itemPrice, vatRate)}`);
console.log(`รวม: ฿${calculateTotal(itemPrice, vatRate)}`);
```

### boolean

```typescript
// ============================================
// boolean - true หรือ false เท่านั้น
// ============================================

let isActive: boolean = true;
let isDeleted: boolean = false;
let isLoggedIn: boolean = true;

// Boolean operations
console.log(true && false);  // false (AND)
console.log(true || false);  // true (OR)
console.log(!true);          // false (NOT)
console.log(true && true);   // true
console.log(false || false); // false

// Comparison operators return boolean
const x = 10;
console.log(x > 5);   // true
console.log(x < 5);   // false
console.log(x === 10); // true (strict equality)
console.log(x !== 10); // false
console.log(x >= 10);  // true
console.log(x <= 10);  // true

// Practical example: ระบบ permissions
interface UserPermissions {
    canRead: boolean;
    canWrite: boolean;
    canDelete: boolean;
    isAdmin: boolean;
}

function checkAccess(permissions: UserPermissions, action: "read" | "write" | "delete"): boolean {
    if (permissions.isAdmin) return true; // Admin ทำได้ทุกอย่าง
    
    switch (action) {
        case "read":    return permissions.canRead;
        case "write":   return permissions.canWrite;
        case "delete":  return permissions.canDelete;
    }
}

const user: UserPermissions = {
    canRead: true,
    canWrite: true,
    canDelete: false,
    isAdmin: false
};

console.log(checkAccess(user, "read"));   // true
console.log(checkAccess(user, "delete")); // false
```

### null และ undefined

```typescript
// ============================================
// null และ undefined
// ============================================

// null = ค่าที่ตั้งใจให้เป็นว่างเปล่า (intentional absence)
let userData: string | null = null;
userData = "สมชาย"; // ใส่ข้อมูลได้ภายหลัง

// undefined = ค่าที่ไม่ถูก assign (unintentional absence)
let username: string | undefined;
console.log(username); // undefined

// ความแตกต่างระหว่าง null และ undefined
console.log(null == undefined);   // true  (loose equality)
console.log(null === undefined);  // false (strict equality)
console.log(typeof null);        // "object" (bug ใน JavaScript!)
console.log(typeof undefined);   // "undefined"

// Strict Null Checks (เมื่อเปิด strictNullChecks)
function printLength(str: string | null): void {
    if (str === null) {
        console.log("String เป็น null");
        return;
    }
    console.log(`ความยาว: ${str.length}`);
}

printLength("สวัสดี"); // ความยาว: 6
printLength(null);      // String เป็น null

// Optional chaining (?.) - ป้องกัน null/undefined error
interface Address {
    city?: string;
    zipCode?: string;
}

interface Person {
    name: string;
    address?: Address;
}

const person: Person = { name: "สมชาย" };
console.log(person.address?.city);        // undefined (ไม่ crash)
console.log(person.address?.city ?? "ไม่ระบุ"); // "ไม่ระบุ"

// Nullish coalescing (??) - ใช้ค่า default เมื่อเป็น null/undefined
const value1: string | null = null;
const value2: string | undefined = undefined;
const value3: string = "";

console.log(value1 ?? "default"); // "default"
console.log(value2 ?? "default"); // "default"
console.log(value3 ?? "default"); // "" (string ว่างไม่ใช่ null/undefined)
console.log(value3 || "default"); // "default" (|| ถือว่า empty string เป็น falsy)

// Non-null assertion operator (!) - บอก TypeScript ว่าแน่ใจว่าไม่ null
function findElement(id: string): HTMLElement | null {
    return document.getElementById(id);
}

// ถ้ามั่นใจว่า element มีอยู่จริง
// const element = findElement("myId")!; // บอก TS ว่าไม่ null
// element.classList.add("active"); // ✅ TypeScript ไม่ error
```

---

## 2. Special Types

### any

```typescript
// ============================================
// any - ปิด Type Checking (ใช้เท่าที่จำเป็น!)
// ============================================

// any คือการบอก TypeScript ว่า "ไม่ต้องตรวจสอบ type นี้"
let anything: any = "สวัสดี";
anything = 42;        // ✅ ได้
anything = true;      // ✅ ได้
anything = { x: 1 };  // ✅ ได้
anything = [1, 2, 3]; // ✅ ได้

// เมื่อใช้ any สามารถเรียก method อะไรก็ได้ (แต่อาจ crash ตอน runtime!)
anything.foo();         // ✅ TypeScript ไม่ error แต่ runtime อาจ crash
anything.bar.baz.qux;   // ✅ TypeScript ไม่ error

// กรณีที่อาจต้องใช้ any:
// 1. Migration จาก JavaScript
function legacyFunction(data: any): any {
    // โค้ดเก่าที่ยังไม่ได้แก้ type
    return data.value * 2;
}

// 2. Library ที่ไม่มี Type Definitions
// const result: any = someOldLibrary.doSomething();

// 3. Dynamic properties
const dynamicConfig: { [key: string]: any } = {};
dynamicConfig.apiUrl = "https://api.example.com";
dynamicConfig.timeout = 3000;
dynamicConfig.isProduction = true;

// ❌ ปัญหาของ any
function addNumbers(a: any, b: any): any {
    return a + b;
}

console.log(addNumbers(1, 2));       // 3 ✅
console.log(addNumbers("1", 2));     // "12" ❌ Type coercion!
console.log(addNumbers([], {}));     // "[object Object]" ❌
// ไม่มี error ตอน compile แต่ผลลัพธ์ผิด
```

### unknown

```typescript
// ============================================
// unknown - Type-safe any
// ============================================

// unknown คล้าย any แต่ปลอดภัยกว่า
// ต้องตรวจสอบ type ก่อนใช้งาน

let unknownValue: unknown = "สวัสดี";
unknownValue = 42;
unknownValue = { name: "สมชาย" };

// ❌ ใช้งานตรงๆ ไม่ได้
// unknownValue.toUpperCase(); // Error! Object is of type 'unknown'
// unknownValue.length;        // Error!

// ✅ ต้องตรวจสอบ type ก่อน
if (typeof unknownValue === "string") {
    console.log(unknownValue.toUpperCase()); // ✅ ใช้ได้เพราะ TypeScript รู้ว่าเป็น string
}

if (typeof unknownValue === "number") {
    console.log(unknownValue.toFixed(2)); // ✅
}

// ตัวอย่างการใช้งานจริง: รับ data จาก API
async function fetchData(url: string): Promise<unknown> {
    const response = await fetch(url);
    return response.json();
}

// ต้องตรวจสอบก่อนใช้
interface ApiUser {
    id: number;
    name: string;
    email: string;
}

function isApiUser(data: unknown): data is ApiUser {
    return (
        typeof data === "object" &&
        data !== null &&
        "id" in data &&
        "name" in data &&
        "email" in data &&
        typeof (data as ApiUser).id === "number" &&
        typeof (data as ApiUser).name === "string" &&
        typeof (data as ApiUser).email === "string"
    );
}

async function getUser(id: number): Promise<ApiUser | null> {
    try {
        const data = await fetchData(`/api/users/${id}`);
        
        if (isApiUser(data)) {
            return data; // ✅ TypeScript รู้ว่าเป็น ApiUser แล้ว
        }
        
        return null;
    } catch {
        return null;
    }
}
```

### never

```typescript
// ============================================
// never - ค่าที่ไม่มีทางเกิดขึ้น
// ============================================

// never ใช้ใน 2 กรณีหลัก:
// 1. ฟังก์ชันที่ไม่มีทาง return (throw หรือ infinite loop)
// 2. ใน type narrowing ที่ทุก case ถูก handle แล้ว

// กรณีที่ 1: ฟังก์ชันที่ throw เสมอ
function throwError(message: string): never {
    throw new Error(message);
}

function infiniteLoop(): never {
    while (true) {
        // วนซ้ำตลอดไป
    }
}

// กรณีที่ 2: Exhaustive type checking
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape, size: number): number {
    switch (shape) {
        case "circle":
            return Math.PI * size * size;
        case "square":
            return size * size;
        case "triangle":
            return (size * size) / 2;
        default:
            // ถ้า TypeScript เตือนที่นี่ แสดงว่าเรา handle ทุก case แล้ว
            // ถ้าเพิ่ม shape ใหม่ใน type แต่ไม่เพิ่ม case จะ Error ตรงนี้!
            const exhaustiveCheck: never = shape;
            throw new Error(`Unhandled shape: ${exhaustiveCheck}`);
    }
}

console.log(getArea("circle", 5));   // 78.539...
console.log(getArea("square", 4));   // 16
console.log(getArea("triangle", 3)); // 4.5

// ตัวอย่างขั้นสูง: Error handling ที่ดี
type Result<T> =
    | { success: true; data: T }
    | { success: false; error: string };

function processResult<T>(result: Result<T>): T {
    if (result.success) {
        return result.data;
    } else {
        throw new Error(result.error);
    }
    // TypeScript รู้ว่า code ที่นี่จะไม่มีทางถึง (unreachable)
}
```

### void

```typescript
// ============================================
// void - ฟังก์ชันที่ไม่คืนค่า
// ============================================

// void ใช้บอกว่าฟังก์ชันไม่คืนค่า
function printMessage(message: string): void {
    console.log(message);
    // ไม่มี return statement หรือ return; เฉยๆ
}

function logError(error: Error): void {
    console.error(`Error: ${error.message}`);
    // console.error เป็น void ไม่คืนค่า
}

// void vs undefined
// void: ฟังก์ชันไม่ควรคืนค่า (แต่ยังสามารถ return undefined ได้)
// undefined: ค่าตัวแปรที่เป็น undefined

function voidFunction(): void {
    return; // ✅ return ได้
    // return undefined; // ✅ return undefined ได้
    // return 42; // ❌ Error! Type '42' is not assignable to type 'void'
}

// Event handlers มักเป็น void
type EventHandler = (event: Event) => void;

const handleClick: EventHandler = (event) => {
    console.log("คลิกที่:", event.target);
    // ไม่ต้อง return อะไร
};

// Callbacks ที่เป็น void
const numbers = [1, 2, 3, 4, 5];
numbers.forEach((num): void => {
    console.log(num);
});
```

---

## 3. Type Annotations

### การเพิ่ม Type Annotations

```typescript
// ============================================
// Type Annotations - การกำหนด type อย่างชัดเจน
// ============================================

// ตัวแปร
let count: number = 0;
let name: string = "TypeScript";
let isEnabled: boolean = true;
let data: null = null;
let value: undefined = undefined;

// อาร์เรย์
let numbers: number[] = [1, 2, 3];
let strings: string[] = ["a", "b", "c"];
let mixed: (number | string)[] = [1, "two", 3];

// ฟังก์ชัน
function add(a: number, b: number): number {
    return a + b;
}

// Arrow function
const multiply = (a: number, b: number): number => a * b;

// Function with optional parameters
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "สวัสดี"} ${name}`;
}

// Function with default values
function createUser(name: string, role: string = "user"): { name: string; role: string } {
    return { name, role };
}

// Object
let person: { name: string; age: number } = {
    name: "สมชาย",
    age: 30
};

// Nested object
let company: {
    name: string;
    address: {
        street: string;
        city: string;
        country: string;
    };
    employees: number;
} = {
    name: "บริษัท TypeScript จำกัด",
    address: {
        street: "ถนนสุขุมวิท",
        city: "กรุงเทพฯ",
        country: "ไทย"
    },
    employees: 50
};
```

### Annotations กับ Arrays

```typescript
// ============================================
// Type Annotations กับ Arrays
// ============================================

// Syntax สองรูปแบบ (เหมือนกัน)
let arr1: number[] = [1, 2, 3];
let arr2: Array<number> = [1, 2, 3];

// Array ของ Objects
interface Product {
    id: number;
    name: string;
    price: number;
}

let products: Product[] = [
    { id: 1, name: "เสื้อ", price: 299 },
    { id: 2, name: "กางเกง", price: 499 },
];

// Readonly array
const frozenArray: readonly number[] = [1, 2, 3];
// frozenArray.push(4); // ❌ Error! Cannot add to readonly array

// Tuple - Array ที่กำหนด length และ type ของแต่ละ element
let tuple: [string, number, boolean] = ["สมชาย", 25, true];

// ใช้ Tuple เพื่อคืนหลายค่า
function getCoordinates(): [number, number] {
    return [13.7563, 100.5018]; // lat, lng ของกรุงเทพฯ
}

const [lat, lng] = getCoordinates();
console.log(`Latitude: ${lat}, Longitude: ${lng}`);

// Named tuples (TypeScript 4.0+)
type RGB = [red: number, green: number, blue: number];
const color: RGB = [255, 128, 0];
const [r, g, b] = color;
```

---

## 4. Type Inference

### TypeScript เดา Type ได้เอง

```typescript
// ============================================
// Type Inference - TypeScript เดา Type ให้
// ============================================

// ตัวอย่างที่ 1: Variable declarations
let inferredString = "สวัสดี"; // TypeScript รู้ว่าเป็น string
let inferredNumber = 42;        // TypeScript รู้ว่าเป็น number
let inferredBoolean = true;     // TypeScript รู้ว่าเป็น boolean

// inferredString = 123; // ❌ Error! แม้ไม่ได้ระบุ type ก็รู้ว่าผิด

// ตัวอย่างที่ 2: Return types
function multiply(a: number, b: number) { // ไม่ต้องระบุ `: number`
    return a * b; // TypeScript รู้ว่า return number
}

const result = multiply(3, 4); // result เป็น number โดย inference

// ตัวอย่างที่ 3: Arrays
const fruits = ["apple", "banana", "mango"]; // string[]
const scores = [100, 90, 85, 95];            // number[]
const mixed = [1, "two", true];              // (number | string | boolean)[]

// ตัวอย่างที่ 4: Objects
const config = {
    apiUrl: "https://api.example.com",  // string
    timeout: 5000,                        // number
    debug: false                          // boolean
};
// TypeScript infer ว่า config: { apiUrl: string; timeout: number; debug: boolean; }

// config.apiUrl = 123; // ❌ Error!
// config.newProp = "test"; // ❌ Error! Property ไม่มีใน type

// ตัวอย่างที่ 5: Function callbacks
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(n => n * 2); // n ถูก infer เป็น number
const evens = numbers.filter(n => n % 2 === 0); // boolean return

// ตัวอย่างที่ 6: Contextual typing
type Handler = (event: MouseEvent) => void;
const clickHandler: Handler = (event) => {
    // event ถูก infer เป็น MouseEvent จาก context
    console.log(event.clientX, event.clientY);
};

// ตัวอย่างที่ 7: Destructuring with inference
const point = { x: 10, y: 20 };
const { x, y } = point; // x: number, y: number

const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first: number, second: number, rest: number[]
```

### เมื่อควรระบุ Type เองและเมื่อควรใช้ Inference

```typescript
// ✅ ใช้ Inference เมื่อชัดเจน
const userId = 123;
const userName = "สมชาย";
const isActive = true;

// ✅ ระบุ Type เองเมื่อ
// 1. ตัวแปรที่ประกาศแต่ยังไม่ assign ค่า
let userAge: number;
userAge = 25;

// 2. Function parameters (TypeScript บังคับ)
function processData(input: string, count: number): boolean {
    return input.length > count;
}

// 3. ต้องการ type ที่กว้างกว่า inference
const value: string | null = null; // ถ้าไม่ระบุ จะเป็น null type เท่านั้น
// value = "hello"; // จะ Error ถ้าไม่ระบุ string | null

// 4. Object ที่ต้องการ interface
interface Config {
    host: string;
    port: number;
}

const serverConfig: Config = { // ระบุ type ให้ชัดเจน
    host: "localhost",
    port: 3000
};
```

---

## 5. Literal Types

### Literal Types คืออะไร?

```typescript
// ============================================
// Literal Types - ค่าที่แน่นอน
// ============================================

// String Literals
type Direction = "north" | "south" | "east" | "west";
let heading: Direction = "north";
// heading = "up"; // ❌ Error! ไม่อยู่ใน type

// Number Literals
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
let roll: DiceRoll = 4;
// roll = 7; // ❌ Error!

// Boolean Literals (มีประโยชน์ใน Discriminated Unions)
type AlwaysTrue = true;
type AlwaysFalse = false;

// Literal Types ใน Objects
const config = {
    mode: "production" as const, // "production" literal ไม่ใช่ string
    version: 3 as const,          // 3 literal ไม่ใช่ number
};

// const assertion
const directions = ["north", "south", "east", "west"] as const;
// directions ถูก infer เป็น readonly ["north", "south", "east", "west"]
// ไม่ใช่ string[]

// ตัวอย่างจริง: Status codes
type HttpStatus = 200 | 201 | 400 | 401 | 403 | 404 | 500;

function handleResponse(status: HttpStatus): string {
    switch (status) {
        case 200: return "สำเร็จ";
        case 201: return "สร้างสำเร็จ";
        case 400: return "คำขอไม่ถูกต้อง";
        case 401: return "ไม่ได้รับอนุญาต";
        case 403: return "ห้ามเข้าถึง";
        case 404: return "ไม่พบข้อมูล";
        case 500: return "เกิดข้อผิดพลาดในเซิร์ฟเวอร์";
    }
}

// Template Literal Types (TypeScript 4.1+)
type EventName = `on${Capitalize<string>}`;
// onClick, onChange, onSubmit, etc.

type CSSProperty = `--${string}`; // CSS Custom Properties
const myColor: CSSProperty = "--primary-color";

type Endpoint = `/api/${string}`;
const usersEndpoint: Endpoint = "/api/users";

// Practical: สร้าง flexible string types
type Position = "top" | "bottom" | "left" | "right";
type Size = "small" | "medium" | "large";
type Variant = "primary" | "secondary" | "danger";

interface ButtonProps {
    label: string;
    size: Size;
    variant: Variant;
    disabled?: boolean;
}

function renderButton(props: ButtonProps): string {
    return `<button class="${props.size} ${props.variant}" ${props.disabled ? "disabled" : ""}>${props.label}</button>`;
}

const button = renderButton({
    label: "คลิกที่นี่",
    size: "medium",
    variant: "primary"
});

console.log(button);
```

---

## 6. Type Assertions

### การ Assert Type

```typescript
// ============================================
// Type Assertions - บอก TypeScript ว่ารู้ type ดีกว่า
// ============================================

// Syntax สองรูปแบบ:
// 1. as keyword (แนะนำ)
// 2. angle bracket (ใช้ใน JSX ไม่ได้)

// as syntax
const someValue: unknown = "Hello TypeScript";
const strLength = (someValue as string).length;
console.log(strLength); // 16

// angle bracket syntax (ไม่ใช้ใน .tsx files)
// const strLength2 = (<string>someValue).length;

// ตัวอย่างที่ 1: DOM Manipulation
// TypeScript ไม่รู้ว่า element เป็น HTMLInputElement
const input = document.getElementById("myInput") as HTMLInputElement;
// ถ้าไม่ assert: input.value จะ Error เพราะ HTMLElement ไม่มี value
// input.value = "new value"; // ✅ หลัง assert

// ตัวอย่างที่ 2: JSON parsing
interface UserData {
    id: number;
    name: string;
}

const jsonString = '{"id": 1, "name": "สมชาย"}';
const parsed = JSON.parse(jsonString) as UserData; // ✅ assert type
console.log(parsed.name); // สมชาย

// ตัวอย่างที่ 3: Narrowing type
function processInput(input: string | number): void {
    if (typeof input === "string") {
        // TypeScript รู้ว่า input เป็น string ใน block นี้
        console.log(input.toUpperCase());
    } else {
        // TypeScript รู้ว่า input เป็น number ที่นี่
        console.log(input.toFixed(2));
    }
}

// ตัวอย่างที่ 4: satisfies operator (TypeScript 4.9+)
type Colors = {
    [key: string]: string | [number, number, number];
};

const palette = {
    red: [255, 0, 0],
    green: "#00ff00",
    blue: [0, 0, 255],
} satisfies Colors;

// ✅ TypeScript รู้ว่า palette.red เป็น number[] ไม่ใช่ string
// palette.red.map(v => v); // ✅

// Double assertion (อันตราย! ใช้เมื่อจำเป็นจริงๆ)
const element = document.getElementById("myDiv") as unknown as HTMLInputElement;
// ใช้เมื่อ TypeScript ไม่ยอมให้ assert โดยตรง

// as const assertion
const apiConfig = {
    url: "https://api.example.com",
    version: "v2",
    timeout: 5000
} as const;

// apiConfig.url เป็น "https://api.example.com" (literal type)
// ไม่ใช่ string ทั่วไป
// apiConfig.url = "other"; // ❌ Error! readonly

type ApiVersion = typeof apiConfig.version; // "v2"
```

---

## 7. ตารางเปรียบเทียบ Types

### ตารางสรุป Primitive Types

| Type | ค่าตัวอย่าง | typeof | Use Case |
|------|------------|--------|----------|
| `string` | `"hello"`, `'world'`, `` `template` `` | `"string"` | ข้อความทุกรูปแบบ |
| `number` | `42`, `3.14`, `NaN`, `Infinity` | `"number"` | ตัวเลขทุกประเภท |
| `boolean` | `true`, `false` | `"boolean"` | ค่า true/false |
| `null` | `null` | `"object"` | ค่าว่างที่ตั้งใจ |
| `undefined` | `undefined` | `"undefined"` | ค่าที่ไม่ได้ assign |
| `bigint` | `9007199254740991n` | `"bigint"` | ตัวเลขขนาดใหญ่มาก |
| `symbol` | `Symbol("id")` | `"symbol"` | Unique identifiers |

### ตารางสรุป Special Types

| Type | คำอธิบาย | เมื่อใช้ | ข้อควรระวัง |
|------|---------|---------|------------|
| `any` | ปิด type checking | Migration, dynamic data | หลีกเลี่ยงถ้าทำได้ |
| `unknown` | Type-safe any | รับ input ที่ไม่รู้ type | ต้องตรวจสอบก่อนใช้ |
| `never` | ไม่มีค่าเลย | Exhaustive checks, throw | ใช้เพื่อความถูกต้อง |
| `void` | ไม่คืนค่า | Function return type | ไม่ควรใช้กับตัวแปร |
| `object` | Non-primitive | รับ object ทั่วไป | ไม่รู้ properties |

### เปรียบเทียบ null, undefined, never, void

```typescript
// ============================================
// เปรียบเทียบ null, undefined, never, void
// ============================================

// null - ค่าว่างที่ตั้งใจ
const emptyUser: string | null = null;
// ใช้เมื่อ: ยังไม่มีข้อมูล หรือ ไม่มีผลลัพธ์

// undefined - ยังไม่ถูก assign
let notAssigned: string | undefined;
// ใช้เมื่อ: optional properties, ยังไม่ได้ตั้งค่า

// void - ฟังก์ชันไม่คืนค่า
function logMessage(msg: string): void {
    console.log(msg);
    // ไม่มี return
}

// never - ไม่มีทางเกิดขึ้น
function assertNever(x: never): never {
    throw new Error("Unexpected value: " + x);
}

// ตัวอย่าง: เมื่อใช้ผสมกัน
type ApiResponse =
    | { status: "success"; data: string }
    | { status: "error"; message: string }
    | { status: "loading" };

function handleApiResponse(response: ApiResponse): string | null {
    switch (response.status) {
        case "success":
            return response.data;
        case "error":
            console.error(response.message);
            return null; // คืน null เมื่อ error
        case "loading":
            return null; // คืน null เมื่อ loading
        default:
            // ถ้าเพิ่ม status ใหม่โดยไม่ handle จะ error ที่นี่
            assertNever(response); // never
    }
}
```

---

## 8. ตัวอย่างการใช้งานจริง

### ตัวอย่างที่ 1-10: ระบบจัดการผู้ใช้

```typescript
// ============================================
// ตัวอย่าง 1: User Registration Form
// ============================================

interface RegistrationForm {
    username: string;
    email: string;
    password: string;
    confirmPassword: string;
    dateOfBirth: string;
    acceptTerms: boolean;
}

interface ValidationResult {
    isValid: boolean;
    errors: Record<string, string>;
}

function validateRegistration(form: RegistrationForm): ValidationResult {
    const errors: Record<string, string> = {};
    
    // ตรวจสอบ username
    if (form.username.length < 3) {
        errors.username = "Username ต้องมีอย่างน้อย 3 ตัวอักษร";
    }
    if (!/^[a-zA-Z0-9_]+$/.test(form.username)) {
        errors.username = "Username ใช้ได้เฉพาะ a-z, 0-9, และ _";
    }
    
    // ตรวจสอบ email
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email)) {
        errors.email = "อีเมลไม่ถูกต้อง";
    }
    
    // ตรวจสอบ password
    if (form.password.length < 8) {
        errors.password = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร";
    }
    if (form.password !== form.confirmPassword) {
        errors.confirmPassword = "รหัสผ่านไม่ตรงกัน";
    }
    
    // ตรวจสอบ terms
    if (!form.acceptTerms) {
        errors.acceptTerms = "กรุณายอมรับข้อกำหนด";
    }
    
    return {
        isValid: Object.keys(errors).length === 0,
        errors
    };
}

// ทดสอบ
const formData: RegistrationForm = {
    username: "somchai123",
    email: "somchai@example.com",
    password: "SecurePass123",
    confirmPassword: "SecurePass123",
    dateOfBirth: "1990-01-01",
    acceptTerms: true
};

const result = validateRegistration(formData);
console.log("Valid:", result.isValid);
console.log("Errors:", result.errors);
```

```typescript
// ============================================
// ตัวอย่าง 2: Product Inventory System
// ============================================

type Category = "electronics" | "clothing" | "food" | "books" | "other";
type StockStatus = "in-stock" | "low-stock" | "out-of-stock";

interface Product {
    id: string;
    name: string;
    description: string;
    price: number;
    stock: number;
    category: Category;
    tags: string[];
    createdAt: Date;
    updatedAt: Date;
}

function getStockStatus(stock: number): StockStatus {
    if (stock === 0) return "out-of-stock";
    if (stock <= 5) return "low-stock";
    return "in-stock";
}

function formatPrice(amount: number, currency: string = "THB"): string {
    return new Intl.NumberFormat("th-TH", {
        style: "currency",
        currency
    }).format(amount);
}

class Inventory {
    private products: Map<string, Product> = new Map();
    
    addProduct(product: Omit<Product, "id" | "createdAt" | "updatedAt">): Product {
        const now = new Date();
        const newProduct: Product = {
            ...product,
            id: Math.random().toString(36).substr(2, 9),
            createdAt: now,
            updatedAt: now
        };
        this.products.set(newProduct.id, newProduct);
        return newProduct;
    }
    
    updateStock(id: string, quantity: number): void {
        const product = this.products.get(id);
        if (!product) throw new Error(`ไม่พบสินค้า ID: ${id}`);
        product.stock += quantity;
        product.updatedAt = new Date();
    }
    
    getLowStockProducts(): Product[] {
        return Array.from(this.products.values())
            .filter(p => p.stock <= 5 && p.stock > 0);
    }
    
    searchByCategory(category: Category): Product[] {
        return Array.from(this.products.values())
            .filter(p => p.category === category);
    }
    
    printReport(): void {
        console.log("\n📦 รายงานสินค้าคงคลัง:");
        this.products.forEach(product => {
            const status = getStockStatus(product.stock);
            const statusIcon = {
                "in-stock": "✅",
                "low-stock": "⚠️",
                "out-of-stock": "❌"
            }[status];
            
            console.log(`${statusIcon} ${product.name}`);
            console.log(`   ราคา: ${formatPrice(product.price)}`);
            console.log(`   คงเหลือ: ${product.stock} ชิ้น`);
        });
    }
}

const inventory = new Inventory();

inventory.addProduct({
    name: "iPhone 15",
    description: "สมาร์ทโฟนรุ่นล่าสุด",
    price: 32900,
    stock: 15,
    category: "electronics",
    tags: ["smartphone", "apple", "5g"]
});

inventory.addProduct({
    name: "เสื้อยืด Basic",
    description: "เสื้อยืดผ้าฝ้าย",
    price: 199,
    stock: 3, // low stock!
    category: "clothing",
    tags: ["shirt", "basic", "cotton"]
});

inventory.printReport();
```

```typescript
// ============================================
// ตัวอย่าง 3: Banking System
// ============================================

type TransactionType = "deposit" | "withdrawal" | "transfer";
type TransactionStatus = "pending" | "completed" | "failed" | "cancelled";

interface Transaction {
    id: string;
    type: TransactionType;
    amount: number;
    status: TransactionStatus;
    description: string;
    createdAt: Date;
    fromAccountId?: string;
    toAccountId?: string;
}

interface BankAccount {
    id: string;
    ownerName: string;
    accountNumber: string;
    balance: number;
    isActive: boolean;
    transactions: Transaction[];
}

class Bank {
    private accounts: Map<string, BankAccount> = new Map();
    
    createAccount(ownerName: string): BankAccount {
        const account: BankAccount = {
            id: this.generateId(),
            ownerName,
            accountNumber: this.generateAccountNumber(),
            balance: 0,
            isActive: true,
            transactions: []
        };
        this.accounts.set(account.id, account);
        return account;
    }
    
    deposit(accountId: string, amount: number, description: string = "ฝากเงิน"): void {
        if (amount <= 0) throw new Error("จำนวนเงินต้องมากกว่า 0");
        
        const account = this.getAccount(accountId);
        account.balance += amount;
        
        account.transactions.push({
            id: this.generateId(),
            type: "deposit",
            amount,
            status: "completed",
            description,
            createdAt: new Date()
        });
        
        console.log(`✅ ฝากเงิน ฿${amount.toLocaleString()} สำเร็จ`);
        console.log(`   ยอดคงเหลือ: ฿${account.balance.toLocaleString()}`);
    }
    
    withdraw(accountId: string, amount: number): void {
        if (amount <= 0) throw new Error("จำนวนเงินต้องมากกว่า 0");
        
        const account = this.getAccount(accountId);
        
        if (account.balance < amount) {
            throw new Error("ยอดเงินไม่เพียงพอ");
        }
        
        account.balance -= amount;
        account.transactions.push({
            id: this.generateId(),
            type: "withdrawal",
            amount,
            status: "completed",
            description: "ถอนเงิน",
            createdAt: new Date()
        });
        
        console.log(`✅ ถอนเงิน ฿${amount.toLocaleString()} สำเร็จ`);
        console.log(`   ยอดคงเหลือ: ฿${account.balance.toLocaleString()}`);
    }
    
    transfer(fromId: string, toId: string, amount: number): void {
        const from = this.getAccount(fromId);
        const to = this.getAccount(toId);
        
        if (from.balance < amount) throw new Error("ยอดเงินไม่เพียงพอ");
        
        from.balance -= amount;
        to.balance += amount;
        
        const transactionId = this.generateId();
        const now = new Date();
        
        from.transactions.push({
            id: transactionId,
            type: "transfer",
            amount,
            status: "completed",
            description: `โอนให้ ${to.ownerName}`,
            createdAt: now,
            toAccountId: toId
        });
        
        to.transactions.push({
            id: transactionId,
            type: "transfer",
            amount,
            status: "completed",
            description: `รับจาก ${from.ownerName}`,
            createdAt: now,
            fromAccountId: fromId
        });
        
        console.log(`✅ โอนเงิน ฿${amount.toLocaleString()} สำเร็จ`);
    }
    
    getBalance(accountId: string): number {
        return this.getAccount(accountId).balance;
    }
    
    private getAccount(id: string): BankAccount {
        const account = this.accounts.get(id);
        if (!account) throw new Error(`ไม่พบบัญชี ID: ${id}`);
        if (!account.isActive) throw new Error("บัญชีนี้ถูกปิดแล้ว");
        return account;
    }
    
    private generateId(): string {
        return Math.random().toString(36).substr(2, 9).toUpperCase();
    }
    
    private generateAccountNumber(): string {
        return Array.from({ length: 10 }, () => Math.floor(Math.random() * 10)).join("");
    }
}

// ทดสอบ
const bank = new Bank();
const account1 = bank.createAccount("สมชาย ใจดี");
const account2 = bank.createAccount("สมหญิง รักดี");

bank.deposit(account1.id, 50000, "เงินเดือน");
bank.deposit(account2.id, 30000, "เงินเดือน");
bank.transfer(account1.id, account2.id, 10000);
console.log(`ยอดของสมชาย: ฿${bank.getBalance(account1.id).toLocaleString()}`);
console.log(`ยอดของสมหญิง: ฿${bank.getBalance(account2.id).toLocaleString()}`);
```

```typescript
// ============================================
// ตัวอย่าง 4-6: Weather App Types
// ============================================

type WindDirection = "N" | "NE" | "E" | "SE" | "S" | "SW" | "W" | "NW";
type WeatherCondition = 
    | "sunny" 
    | "cloudy" 
    | "rainy" 
    | "stormy" 
    | "foggy" 
    | "snowy";

interface WeatherData {
    location: string;
    temperature: {
        celsius: number;
        fahrenheit: number;
    };
    humidity: number;       // 0-100 percent
    windSpeed: number;      // km/h
    windDirection: WindDirection;
    condition: WeatherCondition;
    feelsLike: number;      // Celsius
    pressure: number;       // hPa
    visibility: number;     // km
    timestamp: Date;
}

function celsiusToFahrenheit(celsius: number): number {
    return (celsius * 9/5) + 32;
}

function getWeatherEmoji(condition: WeatherCondition): string {
    const emojis: Record<WeatherCondition, string> = {
        sunny: "☀️",
        cloudy: "☁️",
        rainy: "🌧️",
        stormy: "⛈️",
        foggy: "🌫️",
        snowy: "❄️"
    };
    return emojis[condition];
}

function formatWeatherReport(data: WeatherData): string {
    const emoji = getWeatherEmoji(data.condition);
    return `
${emoji} สภาพอากาศ ${data.location}
━━━━━━━━━━━━━━━━━━━━
อุณหภูมิ:      ${data.temperature.celsius}°C (${data.temperature.fahrenheit}°F)
รู้สึกเหมือน:  ${data.feelsLike}°C
ความชื้น:      ${data.humidity}%
ลม:           ${data.windSpeed} km/h ${data.windDirection}
ความกดอากาศ:  ${data.pressure} hPa
ทัศนวิสัย:   ${data.visibility} km
    `.trim();
}

const bangkokWeather: WeatherData = {
    location: "กรุงเทพมหานคร",
    temperature: {
        celsius: 32,
        fahrenheit: celsiusToFahrenheit(32)
    },
    humidity: 75,
    windSpeed: 15,
    windDirection: "SE",
    condition: "cloudy",
    feelsLike: 38,
    pressure: 1013,
    visibility: 8,
    timestamp: new Date()
};

console.log(formatWeatherReport(bangkokWeather));
```

```typescript
// ============================================
// ตัวอย่าง 7-9: Restaurant Order System
// ============================================

type MenuCategory = "appetizer" | "main" | "dessert" | "beverage";
type OrderStatus = "placed" | "confirmed" | "preparing" | "ready" | "delivered" | "cancelled";
type PaymentMethod = "cash" | "credit-card" | "promptpay" | "true-money";

interface MenuItem {
    id: string;
    name: string;
    nameEn: string;
    description: string;
    price: number;
    category: MenuCategory;
    isAvailable: boolean;
    isVegetarian: boolean;
    allergens: string[];
    calories?: number;
}

interface OrderItem {
    menuItem: MenuItem;
    quantity: number;
    specialRequest?: string;
    totalPrice: number;
}

interface Order {
    id: string;
    tableNumber: number;
    items: OrderItem[];
    status: OrderStatus;
    subtotal: number;
    tax: number;
    total: number;
    paymentMethod?: PaymentMethod;
    isPaid: boolean;
    createdAt: Date;
    estimatedReadyAt?: Date;
}

function createOrder(tableNumber: number, items: { item: MenuItem; quantity: number; note?: string }[]): Order {
    const orderItems: OrderItem[] = items.map(({ item, quantity, note }) => ({
        menuItem: item,
        quantity,
        specialRequest: note,
        totalPrice: item.price * quantity
    }));
    
    const subtotal = orderItems.reduce((sum, i) => sum + i.totalPrice, 0);
    const tax = subtotal * 0.07; // VAT 7%
    
    return {
        id: `ORD-${Date.now()}`,
        tableNumber,
        items: orderItems,
        status: "placed",
        subtotal,
        tax,
        total: subtotal + tax,
        isPaid: false,
        createdAt: new Date()
    };
}

// ตัวอย่าง Menu Items
const menu: MenuItem[] = [
    {
        id: "M001",
        name: "ผัดไทยกุ้งสด",
        nameEn: "Pad Thai with Fresh Shrimp",
        description: "ผัดไทยต้นตำรับ ใส่กุ้งสด",
        price: 180,
        category: "main",
        isAvailable: true,
        isVegetarian: false,
        allergens: ["shellfish", "peanuts", "gluten"],
        calories: 520
    },
    {
        id: "M002",
        name: "ต้มยำกุ้ง",
        nameEn: "Tom Yum Goong",
        description: "ต้มยำกุ้งน้ำข้นรสแซบ",
        price: 220,
        category: "main",
        isAvailable: true,
        isVegetarian: false,
        allergens: ["shellfish"],
        calories: 320
    },
    {
        id: "D001",
        name: "ชาไทย",
        nameEn: "Thai Iced Tea",
        description: "ชาไทยหวานมัน",
        price: 60,
        category: "beverage",
        isAvailable: true,
        isVegetarian: true,
        allergens: ["dairy"],
        calories: 150
    }
];

// สร้าง order
const order = createOrder(5, [
    { item: menu[0], quantity: 2 },
    { item: menu[1], quantity: 1 },
    { item: menu[2], quantity: 3, note: "หวานน้อย" }
]);

console.log("\n🍽️ ใบสั่งอาหาร");
console.log(`โต๊ะ: ${order.tableNumber}`);
console.log(`หมายเลขออเดอร์: ${order.id}`);
console.log("\nรายการอาหาร:");
order.items.forEach(item => {
    console.log(`   ${item.menuItem.name} x${item.quantity} = ฿${item.totalPrice}`);
    if (item.specialRequest) {
        console.log(`   (หมายเหตุ: ${item.specialRequest})`);
    }
});
console.log(`\nราคาก่อนภาษี: ฿${order.subtotal}`);
console.log(`VAT 7%: ฿${order.tax.toFixed(2)}`);
console.log(`รวมทั้งหมด: ฿${order.total.toFixed(2)}`);
```

```typescript
// ============================================
// ตัวอย่าง 10: Type Guards และ Narrowing
// ============================================

// Type Guard functions
function isString(value: unknown): value is string {
    return typeof value === "string";
}

function isNumber(value: unknown): value is number {
    return typeof value === "number" && !isNaN(value);
}

function isDate(value: unknown): value is Date {
    return value instanceof Date && !isNaN(value.getTime());
}

function isNonEmptyArray<T>(value: unknown): value is T[] {
    return Array.isArray(value) && value.length > 0;
}

// ใช้งาน
function processValue(value: unknown): string {
    if (isString(value)) {
        return `String: "${value}" (length: ${value.length})`;
    }
    if (isNumber(value)) {
        return `Number: ${value.toFixed(2)}`;
    }
    if (isDate(value)) {
        return `Date: ${value.toLocaleDateString("th-TH")}`;
    }
    if (isNonEmptyArray(value)) {
        return `Array: [${value.join(", ")}] (${value.length} items)`;
    }
    if (value === null) {
        return "null";
    }
    if (value === undefined) {
        return "undefined";
    }
    return `Object: ${JSON.stringify(value)}`;
}

console.log(processValue("สวัสดี"));
console.log(processValue(42.567));
console.log(processValue(new Date()));
console.log(processValue([1, 2, 3]));
console.log(processValue(null));
```

### ตัวอย่างที่ 11-20: Advanced Type Patterns

```typescript
// ============================================
// ตัวอย่าง 11: Utility Types กับ Basic Types
// ============================================

interface UserProfile {
    id: number;
    username: string;
    email: string;
    password: string;  // sensitive - ไม่ควรส่งไป client
    createdAt: Date;
    lastLogin: Date;
    isActive: boolean;
    role: "admin" | "user" | "moderator";
}

// Partial - ทำทุก field เป็น optional
type PartialUser = Partial<UserProfile>;
const updateUser: PartialUser = { email: "new@example.com" }; // ✅

// Required - ทำทุก field เป็น required
type RequiredUser = Required<UserProfile>;

// Pick - เลือกเฉพาะ fields ที่ต้องการ
type PublicUser = Pick<UserProfile, "id" | "username" | "role" | "createdAt">;
// ไม่มี password ในนี้ ปลอดภัย!

// Omit - ยกเว้น fields ที่ไม่ต้องการ
type UserWithoutPassword = Omit<UserProfile, "password">;

// Readonly - ทำทุก field เป็น readonly
type ImmutableUser = Readonly<UserProfile>;

// Record - สร้าง type จาก keys และ values
type UserRoles = Record<"admin" | "user" | "moderator", {
    permissions: string[];
    label: string;
}>;

const roles: UserRoles = {
    admin: { permissions: ["read", "write", "delete", "manage"], label: "ผู้ดูแลระบบ" },
    user: { permissions: ["read"], label: "ผู้ใช้ทั่วไป" },
    moderator: { permissions: ["read", "write", "moderate"], label: "ผู้ดูแล" }
};
```

```typescript
// ============================================
// ตัวอย่าง 12-15: Real-world Form Handling
// ============================================

// Generic form state
interface FormField<T> {
    value: T;
    error?: string;
    touched: boolean;
    isValid: boolean;
}

interface LoginForm {
    email: FormField<string>;
    password: FormField<string>;
    rememberMe: FormField<boolean>;
}

function createField<T>(initialValue: T): FormField<T> {
    return {
        value: initialValue,
        error: undefined,
        touched: false,
        isValid: false
    };
}

function validateEmail(email: string): string | undefined {
    if (!email) return "กรุณากรอกอีเมล";
    if (!/\S+@\S+\.\S+/.test(email)) return "รูปแบบอีเมลไม่ถูกต้อง";
    return undefined;
}

function validatePassword(password: string): string | undefined {
    if (!password) return "กรุณากรอกรหัสผ่าน";
    if (password.length < 8) return "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร";
    return undefined;
}

// Form state
const loginForm: LoginForm = {
    email: createField(""),
    password: createField(""),
    rememberMe: createField(false)
};

// Simulate form input
loginForm.email.value = "user@example.com";
loginForm.email.touched = true;
loginForm.email.error = validateEmail(loginForm.email.value);
loginForm.email.isValid = !loginForm.email.error;

console.log("Email field:", loginForm.email);
```

```typescript
// ============================================
// ตัวอย่าง 16-20: Data Transformation
// ============================================

// แปลง snake_case เป็น camelCase types
type SnakeToCamel<S extends string> = S extends `${infer P}_${infer Q}${infer R}`
    ? `${P}${Uppercase<Q>}${SnakeToCamel<R>}`
    : S;

// Generic mapper
function mapObject<T extends Record<string, unknown>, U>(
    obj: T,
    mapper: (value: T[keyof T], key: keyof T) => U
): Record<keyof T, U> {
    const result = {} as Record<keyof T, U>;
    for (const key in obj) {
        result[key] = mapper(obj[key], key);
    }
    return result;
}

// ตัวอย่าง: แปลงข้อมูลจาก API
interface ApiProduct {
    product_id: number;
    product_name: string;
    sale_price: number;
    in_stock: boolean;
}

interface AppProduct {
    productId: number;
    productName: string;
    salePrice: number;
    inStock: boolean;
}

function transformProduct(api: ApiProduct): AppProduct {
    return {
        productId: api.product_id,
        productName: api.product_name,
        salePrice: api.sale_price,
        inStock: api.in_stock
    };
}

const apiData: ApiProduct = {
    product_id: 1,
    product_name: "Laptop",
    sale_price: 29999,
    in_stock: true
};

const appData = transformProduct(apiData);
console.log("Transformed:", appData);

// Array transformation
const apiProducts: ApiProduct[] = [
    { product_id: 1, product_name: "Laptop", sale_price: 29999, in_stock: true },
    { product_id: 2, product_name: "Mouse", sale_price: 799, in_stock: false }
];

const appProducts: AppProduct[] = apiProducts.map(transformProduct);
console.log("All products:", appProducts);
```

---

### สรุปบทเรียน

ในบทนี้คุณได้เรียนรู้:

1. **Primitive Types** - string, number, boolean, null, undefined
2. **Special Types** - any, unknown, never, void และความแตกต่าง
3. **Type Annotations** - วิธีระบุ type ให้ตัวแปรและฟังก์ชัน
4. **Type Inference** - TypeScript เดา type ให้เองอัตโนมัติ
5. **Literal Types** - ค่าที่แน่นอน เช่น "north" | "south"
6. **Type Assertions** - บอก TypeScript ว่ารู้ type ดีกว่า
7. **ตัวอย่าง 20+ กรณีการใช้งานจริง** ในโปรเจกต์ต่างๆ

### Preview บทถัดไป

ในบทที่ 4 เราจะเรียนรู้เรื่อง:
- Arrays และ Tuple Types
- Array methods กับ TypeScript
- ReadonlyArray และ readonly modifiers
- Destructuring กับ Types

---

### แบบฝึกหัดบทที่ 3

**ข้อ 1:** สร้าง type สำหรับ traffic light system
```typescript
// เขียนโค้ดที่นี่
type TrafficLight = // "red" | "yellow" | "green"
function getAction(light: TrafficLight): string {
    // คืนค่า "หยุด", "เตรียมตัว", หรือ "ไป"
}
```

**ข้อ 2:** สร้างฟังก์ชัน `safeParseNumber` ที่:
- รับ `input: unknown`
- คืน `number | null`
- ถ้า input เป็น number ที่ valid ให้คืน number
- ถ้าไม่ใช่ให้คืน null

**ข้อ 3:** สร้าง interface สำหรับ library book system:
- `Book` interface พร้อม type-safe properties
- Function `checkoutBook(book: Book, dueDate: Date): void`
- Function `returnBook(book: Book): boolean` (คืน true ถ้าคืนในเวลา)

---

*บทต่อไป: [Part 04 - Arrays และ Tuples](part-04-arrays-tuples.md)*
