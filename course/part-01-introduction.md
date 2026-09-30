# บทที่ 1: บทนำสู่ TypeScript

> เริ่มต้นการเดินทางสู่โลกของ TypeScript - ภาษาโปรแกรมที่เพิ่มพลังให้กับ JavaScript

---

## สารบัญบทนี้

1. [TypeScript คืออะไร?](#1-typescript-คืออะไร)
2. [ทำไมต้องใช้ TypeScript?](#2-ทำไมต้องใช้-typescript)
3. [ประวัติและระบบนิเวศ](#3-ประวัติและระบบนิเวศ)
4. [กระบวนการ Compilation](#4-กระบวนการ-compilation)
5. [โปรแกรม TypeScript แรกของคุณ](#5-โปรแกรม-typescript-แรกของคุณ)
6. [ภาพรวมของ Type System](#6-ภาพรวมของ-type-system)
7. [การเปรียบเทียบ JS vs TS](#7-การเปรียบเทียบ-js-vs-ts)
8. [สรุปบทเรียน](#8-สรุปบทเรียน)

---

## 1. TypeScript คืออะไร?

### คำนิยามพื้นฐาน

**TypeScript** คือภาษาโปรแกรมที่พัฒนาโดย Microsoft ซึ่งเป็น **superset** ของ JavaScript หมายความว่า โค้ด JavaScript ที่เขียนอยู่แล้วทุกบรรทัดเป็น TypeScript ที่ถูกต้อง แต่ TypeScript เพิ่มฟีเจอร์เพิ่มเติมที่สำคัญมากๆ คือ **Static Type System**

ลองนึกภาพว่า JavaScript คือรถยนต์ธรรมดา และ TypeScript คือรถยนต์รุ่นเดียวกันแต่มีระบบนำทาง GPS ที่ช่วยให้คุณไม่หลงทาง

```typescript
// JavaScript - ไม่มี Type System
function greet(name) {
    return "สวัสดี " + name;
}

greet("สมชาย");   // ✅ ถูกต้อง
greet(42);        // ✅ JavaScript ยอมรับ แต่อาจเกิดบัก!

// TypeScript - มี Type System
function greetTyped(name: string): string {
    return "สวัสดี " + name;
}

greetTyped("สมชาย");   // ✅ ถูกต้อง
greetTyped(42);          // ❌ Error! Argument of type 'number' is not assignable to parameter of type 'string'
```

### TypeScript ไม่ใช่ภาษาใหม่ทั้งหมด

สิ่งสำคัญที่ต้องเข้าใจคือ TypeScript ไม่ได้แทนที่ JavaScript แต่เป็นเครื่องมือที่ช่วยให้คุณเขียน JavaScript ได้ดีขึ้น

```
TypeScript Code (.ts)
       ↓
TypeScript Compiler (tsc)
       ↓
JavaScript Code (.js)
       ↓
Browser / Node.js
```

เมื่อคุณรันโค้ด TypeScript ตัว compiler จะแปลง (compile) โค้ดเป็น JavaScript ก่อน แล้วจึงรัน JavaScript นั้น

### คุณสมบัติหลักของ TypeScript

**1. Static Type Checking**
```typescript
// ตรวจสอบ Type ตอน Compile Time ไม่ใช่ Runtime
let age: number = 25;
age = "ยี่สิบห้า"; // ❌ Error ทันที! ก่อนรันโค้ด
```

**2. Type Inference**
```typescript
// TypeScript เดา Type ให้เองโดยอัตโนมัติ
let city = "กรุงเทพ"; // TypeScript รู้ว่าเป็น string
city = 10; // ❌ Error! เพราะ TypeScript รู้ว่า city ควรเป็น string
```

**3. Interfaces และ Types**
```typescript
// กำหนดรูปแบบโครงสร้างข้อมูล
interface Person {
    name: string;
    age: number;
    email?: string; // Optional property
}

const person: Person = {
    name: "สมชาย",
    age: 30
    // email ไม่ต้องใส่ก็ได้ เพราะเป็น optional
};
```

**4. Enhanced IDE Support**
```typescript
// VS Code สามารถ auto-complete และแสดง error ได้ดีขึ้น
const arr = [1, 2, 3];
arr.         // VS Code จะแสดง method ที่ใช้ได้ทั้งหมด เช่น push, pop, map, filter
```

**5. Modern JavaScript Features**
```typescript
// รองรับ ES2015+ features ทั้งหมด
const greet = (name: string): string => `สวัสดี ${name}`;
const [first, ...rest] = [1, 2, 3, 4, 5];
const { x, y, ...others } = { x: 1, y: 2, z: 3, w: 4 };
```

---

## 2. ทำไมต้องใช้ TypeScript?

### ปัญหาของ JavaScript

JavaScript เป็นภาษาที่ยืดหยุ่นมาก แต่ความยืดหยุ่นนี้บางครั้งก็เป็นดาบสองคม:

```javascript
// ปัญหาที่ 1: Type Coercion ที่ไม่คาดคิด
console.log(1 + "2");     // "12" ไม่ใช่ 3!
console.log(true + true); // 2
console.log([] + {});     // "[object Object]"
console.log({} + []);     // 0 (ในบาง browser!)

// ปัญหาที่ 2: Runtime Errors
function getUserAge(user) {
    return user.profile.age; // อาจ crash ถ้า user หรือ profile เป็น null/undefined
}

// ปัญหาที่ 3: Typos ที่หาได้ยาก
const config = { apiUrl: "https://api.example.com" };
console.log(config.apiurl); // undefined! ไม่มี Error แต่ผิด
```

### วิธีที่ TypeScript แก้ปัญหา

```typescript
// แก้ปัญหาที่ 1: Type System ป้องกัน Type Coercion ที่ผิดพลาด
const num: number = 1;
const str: string = "2";
// num + str; // ❌ TypeScript จะ warn เรา

// แก้ปัญหาที่ 2: Null Safety
interface User {
    profile?: {
        age?: number;
    };
}

function getUserAge(user: User): number | undefined {
    return user.profile?.age; // Optional chaining ปลอดภัย
}

// แก้ปัญหาที่ 3: Type Checking บน Object Properties
const config = { apiUrl: "https://api.example.com" };
// config.apiurl; // ❌ Error! Property 'apiurl' does not exist
console.log(config.apiUrl); // ✅ ถูกต้อง
```

### ประโยชน์ในทีมพัฒนา

**1. Self-Documenting Code**
```typescript
// TypeScript code บอก intent ของโค้ดได้ชัดเจน
// ไม่ต้องอ่าน JSDoc หรือ comment เยอะๆ

// JavaScript - ไม่รู้ว่าฟังก์ชันต้องการ parameter แบบไหน
function processOrder(order, options) {
    // ต้องอ่าน implementation เพื่อรู้ว่า order และ options มีอะไรบ้าง
}

// TypeScript - ชัดเจนทันที
interface Order {
    id: string;
    customerId: string;
    items: OrderItem[];
    totalAmount: number;
}

interface ProcessOptions {
    sendEmail?: boolean;
    discountCode?: string;
}

function processOrder(order: Order, options: ProcessOptions = {}): Promise<void> {
    // ชัดเจนว่าต้องการอะไร และคืนค่าอะไร
}
```

**2. Refactoring ที่ปลอดภัย**
```typescript
// เมื่อ rename property เราจะรู้ทันทีว่ามีที่ไหนต้องแก้บ้าง
interface Product {
    // เปลี่ยน productName เป็น name
    name: string; // ถ้าเปลี่ยนชื่อ TypeScript จะ error ทุกที่ที่ใช้ productName
    price: number;
}

// ทุกที่ที่ใช้ productName จะ error ทันที บังคับให้แก้ทั้งหมด
```

**3. Better Collaboration**
```typescript
// Type Definitions เป็นเหมือน Contract ระหว่าง team members
// Frontend รู้ว่า API จะส่ง format อะไรมา
// Backend รู้ว่าต้องส่ง data format อะไร

interface ApiResponse<T> {
    data: T;
    status: "success" | "error";
    message?: string;
    timestamp: string;
}

interface UserData {
    id: number;
    username: string;
    email: string;
    createdAt: string;
}

// Frontend ทราบ type ของ response ชัดเจน
async function fetchUser(id: number): Promise<ApiResponse<UserData>> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
}
```

### สถิติที่น่าสนใจ

- **TypeScript** ติดอันดับ 1 ใน "Most Wanted Language" ของ Stack Overflow Survey หลายปีติดต่อกัน
- บริษัทใหญ่อย่าง Google, Microsoft, Airbnb, Slack ใช้ TypeScript ในโปรเจกต์หลัก
- **Angular** เขียนด้วย TypeScript 100%
- **Next.js** รองรับ TypeScript ตั้งแต่แรก
- ในปี 2023 TypeScript เป็นภาษาที่ถูกกล่าวถึงมากที่สุดใน GitHub

---

## 3. ประวัติและระบบนิเวศ

### ประวัติการพัฒนา

**2012 - The Beginning**
- Microsoft เปิดตัว TypeScript เวอร์ชัน 0.8 ในเดือนตุลาคม 2012
- Anders Hejlsberg (ผู้สร้าง C# และ Delphi) เป็น lead architect
- เป้าหมายหลัก: ทำให้การพัฒนาแอปขนาดใหญ่ด้วย JavaScript เป็นไปได้

```typescript
// TypeScript 0.8 - เริ่มต้นด้วยฟีเจอร์พื้นฐาน
class Greeter {
    greeting: string;
    
    constructor(message: string) {
        this.greeting = message;
    }
    
    greet() {
        return "สวัสดี " + this.greeting;
    }
}
```

**2014 - TypeScript 1.0**
- เปิดตัว TypeScript 1.0 อย่างเป็นทางการ
- Visual Studio รองรับ TypeScript เป็น first-class
- Generics เข้ามาใน TypeScript

**2016 - TypeScript 2.0**
- Non-nullable types (ป้องกัน null/undefined bugs)
- Strictness options ใน tsconfig.json
- Tagged Template Literals

**2018 - TypeScript 3.0**
- Project References สำหรับ Monorepos
- Tuples ที่ยืดหยุ่นขึ้น
- Unknown Type

**2021 - TypeScript 4.0+**
- Variadic Tuple Types
- Template Literal Types
- Type-only imports

**2023 - TypeScript 5.0+**
- Decorators ใหม่ (TC39 Stage 3)
- const Type Parameters
- Speed improvements

### ระบบนิเวศ TypeScript

```
TypeScript Ecosystem
├── Core
│   ├── TypeScript Compiler (tsc)
│   ├── Language Server Protocol (LSP)
│   └── TypeScript Language Service
├── Package Types
│   ├── @types/* packages (DefinitelyTyped)
│   ├── Built-in types (lib.*.d.ts)
│   └── Custom .d.ts files
├── Build Tools
│   ├── tsc (TypeScript Compiler)
│   ├── ts-node (Run TS directly)
│   ├── esbuild (Fast bundler)
│   ├── webpack + ts-loader
│   ├── Vite (Modern bundler)
│   └── rollup
├── Frameworks & Libraries
│   ├── Angular (TypeScript-first)
│   ├── NestJS (Node.js framework)
│   ├── Next.js (React framework)
│   ├── Nuxt 3 (Vue framework)
│   └── Deno (TypeScript runtime)
├── Testing
│   ├── Jest + ts-jest
│   ├── Vitest
│   └── Playwright
└── Code Quality
    ├── ESLint + @typescript-eslint
    ├── Prettier
    └── TypeScript strict mode
```

---

## 4. กระบวนการ Compilation

### ขั้นตอนการ Compile

TypeScript ทำงานผ่าน 3 ขั้นตอนหลัก:

```
Phase 1: Parsing
TypeScript Source (.ts) → AST (Abstract Syntax Tree)

Phase 2: Type Checking
AST → Type Analysis → Error Report (ถ้ามี)

Phase 3: Code Generation
AST → JavaScript Output (.js)
```

### ตัวอย่างจริง: TypeScript → JavaScript

**TypeScript Input:**
```typescript
// user.ts
interface User {
    id: number;
    name: string;
    email: string;
}

class UserService {
    private users: User[] = [];
    
    addUser(user: User): void {
        this.users.push(user);
    }
    
    findById(id: number): User | undefined {
        return this.users.find(u => u.id === id);
    }
    
    getAllUsers(): User[] {
        return [...this.users];
    }
}

const service = new UserService();
service.addUser({ id: 1, name: "สมชาย", email: "somchai@example.com" });
const user = service.findById(1);
console.log(user?.name);
```

**JavaScript Output (ES2015):**
```javascript
// user.js (Generated)
class UserService {
    constructor() {
        this.users = [];
    }
    
    addUser(user) {
        this.users.push(user);
    }
    
    findById(id) {
        return this.users.find(u => u.id === id);
    }
    
    getAllUsers() {
        return [...this.users];
    }
}

const service = new UserService();
service.addUser({ id: 1, name: "สมชาย", email: "somchai@example.com" });
const user = service.findById(1);
console.log(user?.name);
```

สังเกตว่า:
- Interface `User` หายไปทั้งหมด (มีไว้แค่ตอน compile time)
- Type annotations (`: User[]`, `: void`, `: number`) ถูกลบออก
- JavaScript ที่ได้ออกมาเป็นโค้ดที่ clean และอ่านง่าย

### TypeScript Configuration (tsconfig.json)

```json
{
    "compilerOptions": {
        "target": "ES2020",          // JavaScript version ที่ output
        "module": "commonjs",         // Module system
        "strict": true,               // เปิด strict type checking ทั้งหมด
        "outDir": "./dist",           // ที่เก็บ JavaScript output
        "rootDir": "./src",           // ที่เก็บ TypeScript source
        "declaration": true,          // สร้าง .d.ts files
        "sourceMap": true             // สร้าง source map สำหรับ debugging
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "dist"]
}
```

### ความแตกต่าง Type Checking vs Runtime

```typescript
// Type Checking เกิดขึ้นตอน Compile Time
// Runtime เกิดขึ้นตอนรันโปรแกรม

// ตัวอย่างที่ 1: Type Checking ป้องกัน Error ก่อน Runtime
function divide(a: number, b: number): number {
    return a / b;
}

// divide("10", 2); // ❌ TypeScript Error ตอน compile
// แต่ถ้าเปลี่ยน type เป็น any แล้วส่ง string มา
// มันยังคง divide ได้ (NaN) แต่ TypeScript จะ warn ก่อน

// ตัวอย่างที่ 2: Type Erasure
// ตอน Runtime ไม่มี Type information เลย
const value: string = "hello";
// typeof value === "string" // ✅ มีแค่ typeof ของ JavaScript
// value instanceof String   // ❌ String object ไม่ใช่ string primitive
```

---

## 5. โปรแกรม TypeScript แรกของคุณ

### Hello World

```typescript
// hello.ts

// ตัวอย่างที่ 1: Simple Hello World
const greeting: string = "สวัสดีโลก!";
console.log(greeting);

// ตัวอย่างที่ 2: Function ที่มี Type
function sayHello(name: string): string {
    return `สวัสดี ${name}!`;
}

const message = sayHello("ไทยแลนด์");
console.log(message); // สวัสดี ไทยแลนด์!
```

### โปรแกรมที่สมบูรณ์มากขึ้น

```typescript
// calculator.ts

// Type สำหรับ operations ที่รองรับ
type Operation = "add" | "subtract" | "multiply" | "divide";

// Interface สำหรับผลลัพธ์การคำนวณ
interface CalculationResult {
    operation: Operation;
    operand1: number;
    operand2: number;
    result: number;
    timestamp: Date;
}

// ฟังก์ชันหลัก
function calculate(a: number, b: number, op: Operation): CalculationResult {
    let result: number;
    
    switch (op) {
        case "add":
            result = a + b;
            break;
        case "subtract":
            result = a - b;
            break;
        case "multiply":
            result = a * b;
            break;
        case "divide":
            if (b === 0) {
                throw new Error("หารด้วยศูนย์ไม่ได้!");
            }
            result = a / b;
            break;
        default:
            // TypeScript รู้ว่า op เป็น never ที่นี่
            // เพราะเราจัดการทุก case แล้ว
            const _exhaustiveCheck: never = op;
            throw new Error(`ไม่รู้จัก operation: ${_exhaustiveCheck}`);
    }
    
    return {
        operation: op,
        operand1: a,
        operand2: b,
        result,
        timestamp: new Date()
    };
}

// ทดสอบ
const addResult = calculate(10, 5, "add");
console.log(`${addResult.operand1} + ${addResult.operand2} = ${addResult.result}`);

const divResult = calculate(20, 4, "divide");
console.log(`${divResult.operand1} ÷ ${divResult.operand2} = ${divResult.result}`);

try {
    const badDiv = calculate(10, 0, "divide");
} catch (error) {
    if (error instanceof Error) {
        console.log(`Error: ${error.message}`);
    }
}
```

### โปรแกรม Todo App เบื้องต้น

```typescript
// todo.ts

// กำหนด Type สำหรับ Todo item
interface Todo {
    id: number;
    title: string;
    completed: boolean;
    createdAt: Date;
    completedAt?: Date; // Optional - ใส่เมื่อทำเสร็จ
}

// Class สำหรับจัดการ Todo list
class TodoManager {
    private todos: Todo[] = [];
    private nextId: number = 1;
    
    // เพิ่ม Todo ใหม่
    addTodo(title: string): Todo {
        const todo: Todo = {
            id: this.nextId++,
            title,
            completed: false,
            createdAt: new Date()
        };
        
        this.todos.push(todo);
        console.log(`✅ เพิ่มงาน: "${title}"`);
        return todo;
    }
    
    // ทำเครื่องหมายว่าเสร็จแล้ว
    completeTodo(id: number): void {
        const todo = this.todos.find(t => t.id === id);
        
        if (!todo) {
            throw new Error(`ไม่พบ Todo ID: ${id}`);
        }
        
        todo.completed = true;
        todo.completedAt = new Date();
        console.log(`🎉 เสร็จงาน: "${todo.title}"`);
    }
    
    // ลบ Todo
    deleteTodo(id: number): void {
        const index = this.todos.findIndex(t => t.id === id);
        
        if (index === -1) {
            throw new Error(`ไม่พบ Todo ID: ${id}`);
        }
        
        const deleted = this.todos.splice(index, 1)[0];
        console.log(`🗑️ ลบงาน: "${deleted.title}"`);
    }
    
    // ดู Todo ที่ยังไม่เสร็จ
    getPendingTodos(): Todo[] {
        return this.todos.filter(t => !t.completed);
    }
    
    // ดู Todo ที่เสร็จแล้ว
    getCompletedTodos(): Todo[] {
        return this.todos.filter(t => t.completed);
    }
    
    // แสดงสถานะทั้งหมด
    printStatus(): void {
        const pending = this.getPendingTodos().length;
        const completed = this.getCompletedTodos().length;
        const total = this.todos.length;
        
        console.log("\n📋 สรุปงาน:");
        console.log(`   รวมทั้งหมด: ${total} งาน`);
        console.log(`   ยังไม่เสร็จ: ${pending} งาน`);
        console.log(`   เสร็จแล้ว: ${completed} งาน`);
        
        if (this.todos.length > 0) {
            console.log("\nรายการงาน:");
            this.todos.forEach(todo => {
                const status = todo.completed ? "✅" : "⬜";
                console.log(`   ${status} [${todo.id}] ${todo.title}`);
            });
        }
    }
}

// การใช้งาน
const manager = new TodoManager();

manager.addTodo("เรียน TypeScript");
manager.addTodo("ทำโปรเจกต์ Portfolio");
manager.addTodo("อ่านหนังสือ Clean Code");
manager.addTodo("ออกกำลังกาย");

manager.completeTodo(1); // เสร็จ TypeScript
manager.completeTodo(4); // เสร็จออกกำลังกาย

manager.printStatus();
```

---

## 6. ภาพรวมของ Type System

### Type System ของ TypeScript

TypeScript ใช้ **Structural Type System** (หรือ "duck typing") ซึ่งแตกต่างจาก **Nominal Type System** ที่ใช้ใน Java หรือ C#

```typescript
// Structural Typing - ดูที่ "รูปร่าง" ของ object ไม่ใช่ชื่อ

interface Flyable {
    fly(): void;
}

class Bird {
    fly(): void {
        console.log("บินด้วยปีก");
    }
}

class Airplane {
    fly(): void {
        console.log("บินด้วยเครื่องยนต์");
    }
    land(): void {
        console.log("ลงจอด");
    }
}

// ทั้งสองสามารถใช้ได้ที่ไหนก็ตามที่ต้องการ Flyable
function makeItFly(flyable: Flyable): void {
    flyable.fly();
}

makeItFly(new Bird());     // ✅ Bird มี fly() method
makeItFly(new Airplane()); // ✅ Airplane มี fly() method ด้วย

// แม้แต่ object literal ก็ใช้ได้
makeItFly({
    fly() { console.log("บินแบบไม่รู้จัก!"); }
}); // ✅ มี fly() method ก็พอ
```

### ชนิดของ Types ใน TypeScript

```typescript
// 1. Primitive Types
let str: string = "สวัสดี";
let num: number = 42;
let bool: boolean = true;
let nullVal: null = null;
let undefinedVal: undefined = undefined;

// 2. Object Types
let obj: object = { key: "value" };
let arr: number[] = [1, 2, 3];
let tuple: [string, number] = ["age", 25];

// 3. Special Types
let any: any = "ใส่อะไรก็ได้";
let unknown: unknown = "ต้องตรวจสอบก่อนใช้";
let never: never; // ไม่มีค่าเลย
let voidFn: void; // ค่าส่งคืนของฟังก์ชันที่ไม่คืนค่า

// 4. Union Types
let id: string | number = "abc123";
id = 456; // ✅ สลับ type ได้

// 5. Intersection Types
interface HasName { name: string; }
interface HasAge { age: number; }
type Person = HasName & HasAge; // มีทั้ง name และ age
const person: Person = { name: "สมชาย", age: 30 };

// 6. Literal Types
let direction: "north" | "south" | "east" | "west";
direction = "north"; // ✅
// direction = "up";  // ❌ Error!

// 7. Template Literal Types
type EventName = `on${string}`; // onCLick, onChange, onSubmit
```

### Type vs Interface

```typescript
// Interface - ดีสำหรับ Object shapes
interface Animal {
    name: string;
    sound(): string;
}

// Type Alias - ยืดหยุ่นกว่า ใช้กับ Union, Intersection ได้
type StringOrNumber = string | number;
type Nullable<T> = T | null;
type Callback = (error: Error | null, result: string) => void;

// ความแตกต่างหลัก: Interface สามารถ extends และ merge ได้
interface Config {
    apiUrl: string;
}
interface Config {
    timeout: number; // Interface merging - เพิ่ม property ได้
}
// ตอนนี้ Config มีทั้ง apiUrl และ timeout

// Type ไม่สามารถ merge ได้
type Config2 = { apiUrl: string; };
// type Config2 = { timeout: number; }; // ❌ Error! Duplicate identifier
```

---

## 7. การเปรียบเทียบ JS vs TS

### ตัวอย่างที่ 1: ฟังก์ชันพื้นฐาน

```javascript
// JavaScript
function calculateDiscount(price, discountPercent) {
    return price - (price * discountPercent / 100);
}

// ปัญหา: ไม่รู้ว่า price และ discountPercent เป็น type อะไร
calculateDiscount("100", 10); // "1000" - ผิดพลาดเพราะ type coercion!
calculateDiscount(100, "10%"); // NaN - crash ไม่มี error ชัดเจน
```

```typescript
// TypeScript
function calculateDiscount(price: number, discountPercent: number): number {
    if (discountPercent < 0 || discountPercent > 100) {
        throw new Error("ส่วนลดต้องอยู่ระหว่าง 0-100%");
    }
    return price - (price * discountPercent / 100);
}

// ✅ ชัดเจน ปลอดภัย
const finalPrice = calculateDiscount(100, 10); // 90
// calculateDiscount("100", 10); // ❌ Error ตอน compile!
```

### ตัวอย่างที่ 2: การจัดการ API Response

```javascript
// JavaScript - ไม่รู้ว่า data มีอะไรบ้าง
async function fetchUser(id) {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();
    return data; // data เป็นอะไร? ต้องไปดู API docs
}

async function displayUser(userId) {
    const user = await fetchUser(userId);
    console.log(user.usrname); // Typo! แต่ JavaScript ไม่บอก
}
```

```typescript
// TypeScript - ชัดเจนและปลอดภัย
interface User {
    id: number;
    username: string;
    email: string;
    createdAt: string;
}

async function fetchUser(id: number): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json() as Promise<User>;
}

async function displayUser(userId: number): Promise<void> {
    const user = await fetchUser(userId);
    console.log(user.username); // ✅ TypeScript รู้ว่า property ชื่ออะไร
    // console.log(user.usrname); // ❌ Error! Typo ถูกจับได้
}
```

### ตัวอย่างที่ 3: Class-based Programming

```javascript
// JavaScript Class
class ShoppingCart {
    constructor() {
        this.items = [];
        this.discount = 0;
    }
    
    addItem(item) {
        this.items.push(item);
    }
    
    calculateTotal() {
        const subtotal = this.items.reduce((sum, item) => {
            return sum + item.price * item.quantity;
        }, 0);
        return subtotal * (1 - this.discount / 100);
    }
}

// ไม่รู้ว่า item ต้องมี properties อะไร
const cart = new ShoppingCart();
cart.addItem({ nam: "สินค้า", pric: 100, qty: 2 }); // Typos แต่ไม่มี error
```

```typescript
// TypeScript Class
interface CartItem {
    id: string;
    name: string;
    price: number;
    quantity: number;
}

class ShoppingCart {
    private items: CartItem[] = [];
    private discount: number = 0;
    
    addItem(item: CartItem): void {
        const existingItem = this.items.find(i => i.id === item.id);
        
        if (existingItem) {
            existingItem.quantity += item.quantity;
        } else {
            this.items.push({ ...item });
        }
    }
    
    removeItem(id: string): boolean {
        const index = this.items.findIndex(i => i.id === id);
        if (index === -1) return false;
        this.items.splice(index, 1);
        return true;
    }
    
    setDiscount(percent: number): void {
        if (percent < 0 || percent > 100) {
            throw new Error("ส่วนลดต้องอยู่ระหว่าง 0-100");
        }
        this.discount = percent;
    }
    
    calculateTotal(): number {
        const subtotal = this.items.reduce((sum, item) => {
            return sum + item.price * item.quantity;
        }, 0);
        return subtotal * (1 - this.discount / 100);
    }
    
    getItemCount(): number {
        return this.items.reduce((sum, item) => sum + item.quantity, 0);
    }
    
    printSummary(): void {
        console.log("🛒 ตะกร้าสินค้า:");
        this.items.forEach(item => {
            console.log(`   ${item.name}: ${item.quantity} x ฿${item.price} = ฿${item.price * item.quantity}`);
        });
        console.log(`   ส่วนลด: ${this.discount}%`);
        console.log(`   รวมทั้งหมด: ฿${this.calculateTotal().toFixed(2)}`);
    }
}

// การใช้งาน
const cart = new ShoppingCart();

// ✅ TypeScript บอก error ถ้า properties ไม่ถูกต้อง
cart.addItem({ id: "1", name: "เสื้อ", price: 299, quantity: 2 });
cart.addItem({ id: "2", name: "กางเกง", price: 499, quantity: 1 });
// cart.addItem({ id: "3", nam: "หมวก", price: 199, quantity: 1 }); // ❌ Error!

cart.setDiscount(10);
cart.printSummary();
```

### เปรียบเทียบข้อดีข้อเสีย

| หัวข้อ | JavaScript | TypeScript |
|--------|-----------|-----------|
| **เรียนรู้** | ง่ายกว่า | มี learning curve สูงกว่า |
| **ความเร็วในการเขียนโค้ด** | เร็วกว่าในช่วงแรก | ช้ากว่าเล็กน้อยแต่ดีกว่าในระยะยาว |
| **Bug Detection** | Runtime เท่านั้น | Compile time + Runtime |
| **IDE Support** | ดี | ดีมาก (autocomplete, refactoring) |
| **Team Collaboration** | อาจสับสน | ชัดเจน มี contract |
| **Refactoring** | เสี่ยง | ปลอดภัย |
| **Documentation** | ต้องเขียนเอง | Type เป็น documentation |
| **Performance** | เร็ว | เท่ากัน (หลัง compile) |
| **Ecosystem** | ใหญ่มาก | ใหญ่มาก (subset ของ JS) |
| **Browser Support** | รันได้เลย | ต้อง compile ก่อน |

---

## 8. สรุปบทเรียน

### สิ่งที่ได้เรียนรู้ในบทนี้

1. **TypeScript คือ superset ของ JavaScript** - ทุก JavaScript code เป็น TypeScript ที่ถูกต้อง
2. **Static Type System** - ช่วยตรวจหาข้อผิดพลาดก่อนรันโปรแกรม
3. **Type Inference** - TypeScript เดา Type ให้เองในหลายกรณี
4. **Structural Typing** - ดูที่ "รูปร่าง" ของ object ไม่ใช่ชื่อ class
5. **Compilation Process** - TypeScript → Compile → JavaScript → Run

### ข้อคิดสำคัญ

```typescript
// TypeScript ไม่ใช่แค่ "JavaScript with Types"
// มันเป็นเครื่องมือที่ช่วยให้คุณ:

// 1. คิดให้ชัดเจนขึ้นเกี่ยวกับ Data
interface ProductData {
    id: string;
    name: string;
    price: number;
    stock: number;
    category: "electronics" | "clothing" | "food";
}

// 2. สื่อสารกับ Team ได้ดีขึ้น
function processPayment(
    amount: number,
    currency: "THB" | "USD" | "EUR",
    method: "credit" | "debit" | "promptpay"
): Promise<{ success: boolean; transactionId: string }> {
    // ใครก็รู้ว่าฟังก์ชันนี้รับอะไร คืนอะไร
    return Promise.resolve({ success: true, transactionId: "TX123" });
}

// 3. Refactor โค้ดได้อย่างมั่นใจ
// เมื่อเปลี่ยน type หรือ rename - TypeScript บอกทุกที่ที่ต้องแก้
```

### Preview บทถัดไป

ในบทที่ 2 เราจะ:
- ติดตั้ง Node.js และ npm
- ติดตั้ง TypeScript
- ตั้งค่า VS Code สำหรับ TypeScript
- สร้าง tsconfig.json
- รัน TypeScript program แรกของเรา

---

### แบบฝึกหัดบทที่ 1

**ข้อ 1:** เขียนฟังก์ชัน `greetUser` ที่:
- รับ parameter `name: string` และ `age: number`
- คืนค่า string เช่น "สวัสดี สมชาย! คุณอายุ 25 ปี"

```typescript
// เขียนโค้ดของคุณที่นี่
function greetUser(name: string, age: number): string {
    // ...
}
```

**ข้อ 2:** สร้าง interface `Product` ที่มี:
- `id: number`
- `name: string`
- `price: number`
- `inStock: boolean`
- `description?: string` (optional)

แล้วสร้าง array ของ `Product[]` ที่มีสินค้า 3 รายการ

**ข้อ 3:** เขียนฟังก์ชัน `filterByPrice` ที่:
- รับ `products: Product[]` และ `maxPrice: number`
- คืนค่า `Product[]` ที่ราคาไม่เกิน maxPrice

---

*บทต่อไป: [Part 02 - การติดตั้งและตั้งค่า](part-02-installation-setup.md)*
