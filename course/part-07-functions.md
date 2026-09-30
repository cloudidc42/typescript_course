# Part 7: Functions ใน TypeScript

## บทนำ

Functions เป็นหัวใจสำคัญของการเขียนโปรแกรม TypeScript เพิ่มระบบ Type ที่แข็งแกร่งให้กับ Functions ทำให้เราสามารถกำหนดชนิดของ parameters, return values, callbacks และฟังก์ชันที่ซับซ้อนได้อย่างแม่นยำ ส่งผลให้โค้ดปลอดภัย อ่านง่าย และบำรุงรักษาได้ดียิ่งขึ้น

---

## 7.1 Function Types พื้นฐาน

### 7.1.1 Function Declarations

```typescript
// ตัวอย่างที่ 1: Function Declaration พื้นฐาน
function greet(name: string): string {
    return `สวัสดี, ${name}!`;
}

function add(a: number, b: number): number {
    return a + b;
}

function printMessage(message: string): void {
    console.log(message);
}

console.log(greet("สมชาย"));  // สวัสดี, สมชาย!
console.log(add(5, 3));        // 8
printMessage("Hello, World!"); // Hello, World!
```

```typescript
// ตัวอย่างที่ 2: Function กับ Object Parameters
interface UserInfo {
    firstName: string;
    lastName: string;
    age: number;
    email: string;
}

function createUserProfile(user: UserInfo): string {
    return `${user.firstName} ${user.lastName} (อายุ ${user.age}) - ${user.email}`;
}

const user: UserInfo = {
    firstName: "สมชาย",
    lastName: "ใจดี",
    age: 30,
    email: "somchai@example.com"
};

console.log(createUserProfile(user));
```

### 7.1.2 Function Type Annotations

```typescript
// ตัวอย่างที่ 3: กำหนด Type ให้ Function Variable
let calculator: (a: number, b: number) => number;

calculator = (a, b) => a + b;
console.log(calculator(10, 5));  // 15

calculator = (a, b) => a * b;
console.log(calculator(10, 5));  // 50
```

```typescript
// ตัวอย่างที่ 4: Type Alias สำหรับ Function
type StringTransformer = (input: string) => string;
type NumberValidator = (value: number) => boolean;
type AsyncFetcher<T> = (url: string) => Promise<T>;

const toUpperCase: StringTransformer = (s) => s.toUpperCase();
const isPositive: NumberValidator = (n) => n > 0;

console.log(toUpperCase("hello"));    // HELLO
console.log(isPositive(5));           // true
console.log(isPositive(-3));          // false
```

```typescript
// ตัวอย่างที่ 5: Interface สำหรับ Function Type
interface MathOperation {
    (a: number, b: number): number;
    operationName: string;
}

// TypeScript ไม่อนุญาตโดยตรง แต่เราสามารถทำแบบนี้
type Operation = {
    (a: number, b: number): number;
    description: string;
};

const multiply: Operation = Object.assign(
    (a: number, b: number): number => a * b,
    { description: "คูณตัวเลขสองตัว" }
);

console.log(multiply(4, 5));           // 20
console.log(multiply.description);    // คูณตัวเลขสองตัว
```

---

## 7.2 Optional Parameters

```typescript
// ตัวอย่างที่ 6: Optional Parameters พื้นฐาน
function buildAddress(street: string, city: string, province?: string): string {
    if (province) {
        return `${street}, ${city}, ${province}`;
    }
    return `${street}, ${city}`;
}

console.log(buildAddress("123 ถนนสุขุมวิท", "กรุงเทพฯ", "กรุงเทพมหานคร"));
console.log(buildAddress("456 ถนนนิมมาน", "เชียงใหม่"));
```

```typescript
// ตัวอย่างที่ 7: หลาย Optional Parameters
function sendEmail(
    to: string,
    subject: string,
    body: string,
    cc?: string,
    bcc?: string,
    replyTo?: string
): { success: boolean; message: string } {
    const emailInfo: string[] = [`ถึง: ${to}`, `หัวข้อ: ${subject}`];
    if (cc) emailInfo.push(`CC: ${cc}`);
    if (bcc) emailInfo.push(`BCC: ${bcc}`);
    if (replyTo) emailInfo.push(`ตอบกลับ: ${replyTo}`);
    
    console.log("ส่งอีเมล:");
    emailInfo.forEach(info => console.log(`  ${info}`));
    
    return { success: true, message: "ส่งอีเมลสำเร็จ" };
}

sendEmail("user@example.com", "ยืนยันคำสั่งซื้อ", "คำสั่งซื้อของคุณได้รับการยืนยัน");
sendEmail("admin@example.com", "รายงานประจำวัน", "สรุปยอดขาย", "manager@example.com");
```

```typescript
// ตัวอย่างที่ 8: Optional กับ Type Guard
function formatName(firstName: string, lastName?: string): string {
    // lastName อาจเป็น string หรือ undefined
    if (lastName !== undefined) {
        return `${firstName} ${lastName}`;
    }
    return firstName;
}

console.log(formatName("สมชาย", "ใจดี"));  // สมชาย ใจดี
console.log(formatName("สมหญิง"));          // สมหญิง
```

```typescript
// ตัวอย่างที่ 9: Optional กับ Nullish Coalescing
function getDisplayName(firstName: string, lastName?: string, nickname?: string): string {
    return nickname ?? `${firstName}${lastName ? ' ' + lastName : ''}`;
}

console.log(getDisplayName("สมชาย", "ใจดี", "ชาย"));  // ชาย
console.log(getDisplayName("สมหญิง", "มีโชค"));         // สมหญิง มีโชค
console.log(getDisplayName("ประสงค์"));                  // ประสงค์
```

---

## 7.3 Default Parameters

```typescript
// ตัวอย่างที่ 10: Default Parameters พื้นฐาน
function createProduct(
    name: string,
    price: number,
    category: string = "ทั่วไป",
    inStock: boolean = true,
    discount: number = 0
): object {
    return {
        name,
        price: price * (1 - discount),
        originalPrice: price,
        category,
        inStock,
        hasDiscount: discount > 0
    };
}

console.log(createProduct("แล็ปท็อป", 25000));
console.log(createProduct("เมาส์", 500, "อุปกรณ์คอมพิวเตอร์", true, 0.1));
console.log(createProduct("คีย์บอร์ด", 1200, "อุปกรณ์คอมพิวเตอร์"));
```

```typescript
// ตัวอย่างที่ 11: Default Parameters กับ Object
interface PaginationOptions {
    page?: number;
    pageSize?: number;
    sortBy?: string;
    sortOrder?: "asc" | "desc";
}

function fetchData(
    endpoint: string,
    options: PaginationOptions = {}
): string {
    const {
        page = 1,
        pageSize = 20,
        sortBy = "createdAt",
        sortOrder = "desc"
    } = options;
    
    return `GET ${endpoint}?page=${page}&size=${pageSize}&sort=${sortBy}&order=${sortOrder}`;
}

console.log(fetchData("/api/products"));
console.log(fetchData("/api/users", { page: 2, pageSize: 10 }));
console.log(fetchData("/api/orders", { sortBy: "amount", sortOrder: "asc" }));
```

```typescript
// ตัวอย่างที่ 12: Default Parameters กับ Function
function processItems<T>(
    items: T[],
    processor: (item: T) => T = (item) => item,
    filter: (item: T) => boolean = () => true
): T[] {
    return items.filter(filter).map(processor);
}

const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const doubled = processItems(numbers, n => n * 2);
const evenDoubled = processItems(numbers, n => n * 2, n => n % 2 === 0);

console.log("Double ทั้งหมด:", doubled);
console.log("Double เฉพาะคู่:", evenDoubled);
```

---

## 7.4 Rest Parameters

```typescript
// ตัวอย่างที่ 13: Rest Parameters พื้นฐาน
function sumAll(...numbers: number[]): number {
    return numbers.reduce((total, n) => total + n, 0);
}

console.log(sumAll(1, 2, 3));             // 6
console.log(sumAll(10, 20, 30, 40, 50));  // 150
console.log(sumAll());                    // 0

function combineStrings(separator: string, ...parts: string[]): string {
    return parts.join(separator);
}

console.log(combineStrings(", ", "แอปเปิ้ล", "กล้วย", "ส้ม"));  // แอปเปิ้ล, กล้วย, ส้ม
console.log(combineStrings(" - ", "กรุงเทพฯ", "เชียงใหม่"));      // กรุงเทพฯ - เชียงใหม่
```

```typescript
// ตัวอย่างที่ 14: Rest Parameters กับ Typed Tuples
function first<T>(first: T, ...rest: T[]): T {
    return first;
}

function last<T>(...items: T[]): T {
    return items[items.length - 1];
}

console.log(first(1, 2, 3, 4));                     // 1
console.log(last("a", "b", "c", "d"));               // "d"
console.log(first("สมชาย", "สมหญิง", "สมศักดิ์")); // สมชาย
```

```typescript
// ตัวอย่างที่ 15: Rest Parameters ในสถานการณ์จริง
interface LogEntry {
    timestamp: Date;
    level: "info" | "warn" | "error";
    message: string;
    tags: string[];
}

function createLog(
    level: "info" | "warn" | "error",
    message: string,
    ...tags: string[]
): LogEntry {
    return {
        timestamp: new Date(),
        level,
        message,
        tags
    };
}

const log1 = createLog("info", "ผู้ใช้เข้าสู่ระบบ", "auth", "user");
const log2 = createLog("error", "การเชื่อมต่อล้มเหลว", "network", "critical", "database");
const log3 = createLog("warn", "หน่วยความจำใกล้เต็ม");

console.log(log1);
console.log(log2);
console.log(log3);
```

---

## 7.5 Function Overloads

Function Overloads ช่วยให้ฟังก์ชันรับ parameter ได้หลายรูปแบบพร้อมระบุ return type ที่แตกต่างกัน

```typescript
// ตัวอย่างที่ 16: Function Overloads พื้นฐาน
function formatValue(value: string): string;
function formatValue(value: number): string;
function formatValue(value: boolean): string;
function formatValue(value: string | number | boolean): string {
    if (typeof value === "string") {
        return `"${value}"`;
    } else if (typeof value === "number") {
        return value.toLocaleString("th-TH");
    } else {
        return value ? "ใช่" : "ไม่";
    }
}

console.log(formatValue("สวัสดี"));   // "สวัสดี"
console.log(formatValue(1234567));    // 1,234,567
console.log(formatValue(true));       // ใช่
console.log(formatValue(false));      // ไม่
```

```typescript
// ตัวอย่างที่ 17: Overloads กับ Return Type ต่างกัน
function parse(input: string): number;
function parse(input: number): string;
function parse(input: string | number): number | string {
    if (typeof input === "string") {
        return parseFloat(input);
    } else {
        return input.toString();
    }
}

const num: number = parse("3.14");    // 3.14 (number)
const str: string = parse(42);        // "42" (string)
console.log(typeof num, num);
console.log(typeof str, str);
```

```typescript
// ตัวอย่างที่ 18: Overloads กับ Optional Parameters
function createElement(tag: string): HTMLElement;
function createElement(tag: string, content: string): HTMLElement;
function createElement(tag: string, content?: string): HTMLElement {
    // จำลองสร้าง HTML Element
    const element = { tagName: tag, innerHTML: content || "", children: [] } as unknown as HTMLElement;
    return element;
}
```

```typescript
// ตัวอย่างที่ 19: Real-world Overloads - Database Query
interface User {
    id: number;
    name: string;
    email: string;
}

// Overload signatures
function findUser(id: number): User | undefined;
function findUser(email: string): User | undefined;
function findUser(criteria: { name: string }): User[];

// Implementation
function findUser(
    criteria: number | string | { name: string }
): User | User[] | undefined {
    const users: User[] = [
        { id: 1, name: "สมชาย ใจดี", email: "somchai@example.com" },
        { id: 2, name: "สมหญิง มีโชค", email: "somying@example.com" },
        { id: 3, name: "สมศักดิ์ ดีงาม", email: "somsak@example.com" }
    ];
    
    if (typeof criteria === "number") {
        return users.find(u => u.id === criteria);
    } else if (typeof criteria === "string") {
        return users.find(u => u.email === criteria);
    } else {
        return users.filter(u => u.name.includes(criteria.name));
    }
}

const byId = findUser(1);
const byEmail = findUser("somying@example.com");
const byName = findUser({ name: "สม" });

console.log("By ID:", byId);
console.log("By Email:", byEmail);
console.log("By Name:", byName);
```

---

## 7.6 this Parameter

```typescript
// ตัวอย่างที่ 20: this Parameter ใน Methods
interface Button {
    text: string;
    isEnabled: boolean;
    click(this: Button): void;
    toggle(this: Button): void;
}

const submitButton: Button = {
    text: "ส่งข้อมูล",
    isEnabled: true,
    click(this: Button): void {
        if (this.isEnabled) {
            console.log(`คลิกปุ่ม: ${this.text}`);
        } else {
            console.log(`ปุ่ม ${this.text} ถูกปิดใช้งาน`);
        }
    },
    toggle(this: Button): void {
        this.isEnabled = !this.isEnabled;
        console.log(`ปุ่ม ${this.text}: ${this.isEnabled ? "เปิด" : "ปิด"}`);
    }
};

submitButton.click();
submitButton.toggle();
submitButton.click();
```

```typescript
// ตัวอย่างที่ 21: this ใน Class Methods
class Counter {
    private count: number = 0;
    private readonly name: string;
    
    constructor(name: string) {
        this.name = name;
    }
    
    increment(this: Counter, amount: number = 1): Counter {
        this.count += amount;
        return this; // Method chaining
    }
    
    decrement(this: Counter, amount: number = 1): Counter {
        this.count = Math.max(0, this.count - amount);
        return this;
    }
    
    reset(this: Counter): Counter {
        this.count = 0;
        return this;
    }
    
    getValue(this: Counter): number {
        return this.count;
    }
    
    display(this: Counter): void {
        console.log(`${this.name}: ${this.count}`);
    }
}

const scoreCounter = new Counter("คะแนน");
scoreCounter
    .increment(10)
    .increment(5)
    .increment(3)
    .decrement(2);

scoreCounter.display(); // คะแนน: 16
```

```typescript
// ตัวอย่างที่ 22: this กับ Event Handlers
class EventEmitter {
    private listeners: Map<string, ((this: EventEmitter, data: unknown) => void)[]> = new Map();
    
    on(event: string, listener: (this: EventEmitter, data: unknown) => void): this {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event)!.push(listener);
        return this; // Fluent interface
    }
    
    emit(event: string, data: unknown): void {
        const eventListeners = this.listeners.get(event) || [];
        eventListeners.forEach(listener => listener.call(this, data));
    }
}

const emitter = new EventEmitter();
emitter
    .on("data", function(this: EventEmitter, data) {
        console.log("ได้รับข้อมูล:", data);
    })
    .on("error", function(this: EventEmitter, error) {
        console.error("เกิดข้อผิดพลาด:", error);
    });

emitter.emit("data", { userId: 1, action: "login" });
emitter.emit("error", "เชื่อมต่อล้มเหลว");
```

---

## 7.7 Arrow Functions

```typescript
// ตัวอย่างที่ 23: Arrow Functions พื้นฐาน
const square = (n: number): number => n * n;
const greetUser = (name: string): string => `สวัสดี, ${name}!`;
const isAdult = (age: number): boolean => age >= 18;

console.log(square(5));         // 25
console.log(greetUser("สมชาย")); // สวัสดี, สมชาย!
console.log(isAdult(20));       // true
console.log(isAdult(16));       // false
```

```typescript
// ตัวอย่างที่ 24: Arrow Functions กับ Object Return
const createUser = (name: string, age: number): { name: string; age: number } => ({
    name,
    age
});

// ต้องใช้ () ครอบเมื่อ return object literal
const users = [
    createUser("สมชาย", 30),
    createUser("สมหญิง", 25),
    createUser("สมศักดิ์", 35)
];

console.log(users);
```

```typescript
// ตัวอย่างที่ 25: Arrow Functions กับ Generic Types
const identity = <T>(value: T): T => value;
const firstItem = <T>(arr: T[]): T | undefined => arr[0];
const lastItem = <T>(arr: T[]): T | undefined => arr[arr.length - 1];

console.log(identity(42));           // 42
console.log(identity("สวัสดี"));      // สวัสดี
console.log(firstItem([1, 2, 3]));   // 1
console.log(lastItem(["a", "b"]));   // "b"
```

```typescript
// ตัวอย่างที่ 26: Arrow Functions กับ this Binding
class Timer {
    private seconds: number = 0;
    private intervalId?: ReturnType<typeof setInterval>;
    
    start(): void {
        // Arrow function จะ bind this จาก outer scope
        this.intervalId = setInterval(() => {
            this.seconds++;
            console.log(`เวลา: ${this.seconds} วินาที`);
            if (this.seconds >= 5) {
                this.stop();
            }
        }, 1000);
    }
    
    stop(): void {
        if (this.intervalId) {
            clearInterval(this.intervalId);
            console.log("หยุดนับเวลา");
        }
    }
    
    reset(): void {
        this.stop();
        this.seconds = 0;
    }
}
```

---

## 7.8 Callback Types

```typescript
// ตัวอย่างที่ 27: Callback Types พื้นฐาน
type Callback = () => void;
type ErrorCallback = (error: Error) => void;
type DataCallback<T> = (data: T) => void;
type TransformCallback<T, R> = (item: T) => R;

function executeWithCallback(callback: Callback): void {
    console.log("กำลังดำเนินการ...");
    callback();
    console.log("เสร็จสิ้น");
}

function loadData<T>(
    url: string,
    onSuccess: DataCallback<T>,
    onError: ErrorCallback
): void {
    // จำลองการโหลดข้อมูล
    try {
        const data = { message: "ข้อมูลจาก " + url } as unknown as T;
        onSuccess(data);
    } catch (error) {
        onError(error as Error);
    }
}

executeWithCallback(() => console.log("Callback ถูกเรียก!"));
```

```typescript
// ตัวอย่างที่ 28: Async Callback Pattern
type AsyncCallback<T> = (error: Error | null, result: T | null) => void;

function fetchUserById(
    userId: number,
    callback: AsyncCallback<{ id: number; name: string }>
): void {
    // จำลอง async operation
    setTimeout(() => {
        if (userId > 0) {
            callback(null, { id: userId, name: `ผู้ใช้ ${userId}` });
        } else {
            callback(new Error("รหัสผู้ใช้ไม่ถูกต้อง"), null);
        }
    }, 100);
}

fetchUserById(1, (error, user) => {
    if (error) {
        console.error("ข้อผิดพลาด:", error.message);
    } else if (user) {
        console.log("พบผู้ใช้:", user.name);
    }
});
```

```typescript
// ตัวอย่างที่ 29: Event Handler Callbacks
type EventHandler<T = void> = (event: T) => void;
type MouseEventHandler = EventHandler<{ x: number; y: number }>;
type KeyEventHandler = EventHandler<{ key: string; code: string }>;
type FormSubmitHandler = EventHandler<{ data: Record<string, string> }>;

function handleMouseClick(handler: MouseEventHandler): void {
    // จำลอง mouse event
    handler({ x: 100, y: 200 });
}

handleMouseClick(({ x, y }) => {
    console.log(`คลิกที่ตำแหน่ง: (${x}, ${y})`);
});
```

---

## 7.9 Higher-Order Functions

```typescript
// ตัวอย่างที่ 30: Higher-Order Functions พื้นฐาน
function multiplyBy(factor: number): (n: number) => number {
    return (n: number): number => n * factor;
}

const double = multiplyBy(2);
const triple = multiplyBy(3);
const tenTimes = multiplyBy(10);

console.log(double(5));    // 10
console.log(triple(5));    // 15
console.log(tenTimes(5));  // 50
```

```typescript
// ตัวอย่างที่ 31: Function Composition
type Transformer<T> = (value: T) => T;

function compose<T>(...fns: Transformer<T>[]): Transformer<T> {
    return (value: T): T => fns.reduceRight((acc, fn) => fn(acc), value);
}

function pipe<T>(...fns: Transformer<T>[]): Transformer<T> {
    return (value: T): T => fns.reduce((acc, fn) => fn(acc), value);
}

const addTax = (price: number): number => price * 1.07;
const applyDiscount = (price: number): number => price * 0.9;
const roundUp = (price: number): number => Math.ceil(price);

const calculateFinalPrice = pipe(addTax, applyDiscount, roundUp);

console.log(calculateFinalPrice(1000)); // 1000 * 1.07 * 0.9 = 963 -> 963
console.log(calculateFinalPrice(5000)); // 5000 * 1.07 * 0.9 = 4815 -> 4815
```

```typescript
// ตัวอย่างที่ 32: Currying
function curry<A, B, C>(fn: (a: A, b: B) => C): (a: A) => (b: B) => C {
    return (a: A) => (b: B) => fn(a, b);
}

function add(a: number, b: number): number {
    return a + b;
}

const curriedAdd = curry(add);
const addFive = curriedAdd(5);
const addTen = curriedAdd(10);

console.log(addFive(3));   // 8
console.log(addFive(7));   // 12
console.log(addTen(20));   // 30
```

```typescript
// ตัวอย่างที่ 33: Memoization
function memoize<Args extends unknown[], R>(
    fn: (...args: Args) => R
): (...args: Args) => R {
    const cache = new Map<string, R>();
    
    return (...args: Args): R => {
        const key = JSON.stringify(args);
        
        if (cache.has(key)) {
            console.log(`ใช้ cache สำหรับ ${key}`);
            return cache.get(key)!;
        }
        
        const result = fn(...args);
        cache.set(key, result);
        return result;
    };
}

function expensiveCalculation(n: number): number {
    console.log(`คำนวณสำหรับ ${n}...`);
    // จำลองการคำนวณที่ใช้เวลา
    return n * n * n;
}

const memoizedCalc = memoize(expensiveCalculation);
console.log(memoizedCalc(5));  // คำนวณสำหรับ 5... -> 125
console.log(memoizedCalc(5));  // ใช้ cache -> 125
console.log(memoizedCalc(10)); // คำนวณสำหรับ 10... -> 1000
console.log(memoizedCalc(5));  // ใช้ cache -> 125
```

---

## 7.10 void vs never Return Types

```typescript
// ตัวอย่างที่ 34: void - ฟังก์ชันที่ไม่ return ค่า
function logMessage(message: string): void {
    console.log(message);
    // ไม่ return หรือ return undefined ได้
}

function clearScreen(): void {
    // ล้างหน้าจอ
    console.clear();
    return; // return undefined
}

// void function สามารถ return undefined ได้
const result: void = logMessage("สวัสดี");
console.log(result); // undefined
```

```typescript
// ตัวอย่างที่ 35: never - ฟังก์ชันที่ไม่มีทางสิ้นสุดปกติ
function throwError(message: string): never {
    throw new Error(message);
}

function infiniteLoop(): never {
    while (true) {
        // loop ไม่มีสิ้นสุด
    }
}

// never ใช้ใน Exhaustive Type Checking
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape, size: number): number {
    switch (shape) {
        case "circle":
            return Math.PI * size * size;
        case "square":
            return size * size;
        case "triangle":
            return 0.5 * size * size;
        default:
            // TypeScript รู้ว่า shape ต้องเป็น never ที่นี่
            const exhaustiveCheck: never = shape;
            throw new Error(`Shape ไม่รู้จัก: ${exhaustiveCheck}`);
    }
}
```

```typescript
// ตัวอย่างที่ 36: void vs undefined
function returnsVoid(): void {
    return; // OK
    // return undefined; // OK
    // return 42; // Error!
}

function returnsUndefined(): undefined {
    return undefined; // ต้อง return undefined เท่านั้น
    // return; // OK
    // return 42; // Error!
}

// void ใช้เมื่อไม่สนใจ return value
// undefined ใช้เมื่อต้องการระบุว่า return undefined จริงๆ

type VoidCallback = () => void;
type UndefinedCallback = () => undefined;

const voidFn: VoidCallback = () => {
    console.log("void function");
    // ไม่ต้อง return
};

const undefinedFn: UndefinedCallback = () => {
    return undefined; // ต้อง return undefined
};
```

---

## 7.11 Async Functions กับ TypeScript

```typescript
// ตัวอย่างที่ 37: Async/Await พื้นฐาน
async function fetchUserData(userId: number): Promise<{ id: number; name: string }> {
    // จำลองการดึงข้อมูลจาก API
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (userId > 0) {
                resolve({ id: userId, name: `ผู้ใช้ ${userId}` });
            } else {
                reject(new Error("รหัสผู้ใช้ไม่ถูกต้อง"));
            }
        }, 100);
    });
}

async function main(): Promise<void> {
    try {
        const user = await fetchUserData(1);
        console.log(`พบผู้ใช้: ${user.name}`);
    } catch (error) {
        if (error instanceof Error) {
            console.error(`ข้อผิดพลาด: ${error.message}`);
        }
    }
}

main();
```

```typescript
// ตัวอย่างที่ 38: Async กับ Array Operations
interface Product {
    id: number;
    name: string;
    price: number;
    categoryId: number;
}

interface Category {
    id: number;
    name: string;
}

async function getProducts(): Promise<Product[]> {
    return [
        { id: 1, name: "แล็ปท็อป", price: 25000, categoryId: 1 },
        { id: 2, name: "เมาส์", price: 500, categoryId: 1 },
        { id: 3, name: "เสื้อยืด", price: 299, categoryId: 2 }
    ];
}

async function getCategoryById(id: number): Promise<Category | null> {
    const categories: Category[] = [
        { id: 1, name: "อิเล็กทรอนิกส์" },
        { id: 2, name: "เสื้อผ้า" }
    ];
    return categories.find(c => c.id === id) || null;
}

async function getProductsWithCategories(): Promise<(Product & { categoryName: string })[]> {
    const products = await getProducts();
    
    // Parallel requests
    const productsWithCategories = await Promise.all(
        products.map(async (product) => {
            const category = await getCategoryById(product.categoryId);
            return {
                ...product,
                categoryName: category?.name || "ไม่ระบุ"
            };
        })
    );
    
    return productsWithCategories;
}

async function runExample(): Promise<void> {
    const products = await getProductsWithCategories();
    products.forEach(p => {
        console.log(`[${p.categoryName}] ${p.name}: ฿${p.price.toLocaleString()}`);
    });
}

runExample();
```

```typescript
// ตัวอย่างที่ 39: Promise.all, Promise.race, Promise.allSettled
async function fetchMultipleAPIs(): Promise<void> {
    const urls: string[] = [
        "https://api.example.com/users",
        "https://api.example.com/products",
        "https://api.example.com/orders"
    ];
    
    // จำลอง API calls
    const mockFetch = (url: string): Promise<{ url: string; data: string }> =>
        new Promise(resolve => 
            setTimeout(() => resolve({ url, data: `ข้อมูลจาก ${url}` }), Math.random() * 1000)
        );
    
    // ทั้งหมดต้องสำเร็จ
    const results = await Promise.all(urls.map(mockFetch));
    console.log("Promise.all:", results.length, "results");
    
    // ใครเสร็จก่อนชนะ
    const fastest = await Promise.race(urls.map(mockFetch));
    console.log("Promise.race:", fastest.url);
    
    // รวมผลทั้งหมดแม้บางอันล้มเหลว
    const allResults = await Promise.allSettled(urls.map(mockFetch));
    allResults.forEach(result => {
        if (result.status === "fulfilled") {
            console.log("สำเร็จ:", result.value.url);
        } else {
            console.log("ล้มเหลว:", result.reason);
        }
    });
}
```

```typescript
// ตัวอย่างที่ 40: Async Error Handling
type AsyncResult<T> = Promise<{ data: T; error: null } | { data: null; error: string }>;

async function safeAsync<T>(promise: Promise<T>): AsyncResult<T> {
    try {
        const data = await promise;
        return { data, error: null };
    } catch (err) {
        const errorMessage = err instanceof Error ? err.message : "เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ";
        return { data: null, error: errorMessage };
    }
}

async function riskyOperation(): Promise<number> {
    if (Math.random() < 0.5) {
        throw new Error("การดำเนินการล้มเหลว");
    }
    return 42;
}

async function demonstrateSafeAsync(): Promise<void> {
    const result = await safeAsync(riskyOperation());
    
    if (result.error) {
        console.log("จัดการข้อผิดพลาด:", result.error);
    } else {
        console.log("ผลลัพธ์:", result.data);
    }
}

demonstrateSafeAsync();
```

---

## 7.12 Generator Functions

```typescript
// ตัวอย่างที่ 41: Generator Function พื้นฐาน
function* numberGenerator(): Generator<number, void, undefined> {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
    yield 5;
}

const gen = numberGenerator();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }

// ใช้ใน for...of
for (const num of numberGenerator()) {
    console.log(num);
}
```

```typescript
// ตัวอย่างที่ 42: Infinite Generator
function* idGenerator(start: number = 1): Generator<number> {
    let current = start;
    while (true) {
        yield current++;
    }
}

const getId = idGenerator(1000);
console.log(getId.next().value); // 1000
console.log(getId.next().value); // 1001
console.log(getId.next().value); // 1002

// สร้าง IDs สำหรับ orders
function* orderIdGenerator(): Generator<string> {
    let num = 1;
    while (true) {
        yield `ORD-${new Date().getFullYear()}-${String(num++).padStart(5, '0')}`;
    }
}

const orderIds = orderIdGenerator();
console.log(orderIds.next().value); // ORD-2024-00001
console.log(orderIds.next().value); // ORD-2024-00002
```

```typescript
// ตัวอย่างที่ 43: Generator สำหรับ Lazy Evaluation
function* rangeGenerator(
    start: number,
    end: number,
    step: number = 1
): Generator<number> {
    for (let i = start; i <= end; i += step) {
        yield i;
    }
}

// Lazy - คำนวณเฉพาะเมื่อต้องการ
for (const n of rangeGenerator(1, 100, 5)) {
    if (n > 25) break; // หยุดเมื่อพอ
    console.log(n);
}

function* filterGenerator<T>(
    source: Iterable<T>,
    predicate: (value: T) => boolean
): Generator<T> {
    for (const value of source) {
        if (predicate(value)) {
            yield value;
        }
    }
}

const evenNumbers = filterGenerator(rangeGenerator(1, 20), n => n % 2 === 0);
const result: number[] = [];
for (const n of evenNumbers) {
    result.push(n);
}
console.log("เลขคู่:", result);
```

```typescript
// ตัวอย่างที่ 44: Async Generator
async function* asyncDataFetcher(
    ids: number[]
): AsyncGenerator<{ id: number; data: string }, void, undefined> {
    for (const id of ids) {
        // จำลองการดึงข้อมูล async
        await new Promise(resolve => setTimeout(resolve, 100));
        yield { id, data: `ข้อมูลสำหรับ ID ${id}` };
    }
}

async function processAsyncGenerator(): Promise<void> {
    const ids = [1, 2, 3, 4, 5];
    
    for await (const item of asyncDataFetcher(ids)) {
        console.log(`รับข้อมูล: ${item.data}`);
    }
}

processAsyncGenerator();
```

---

## 7.13 Generic Functions

```typescript
// ตัวอย่างที่ 45: Generic Functions พื้นฐาน
function identity<T>(value: T): T {
    return value;
}

function pair<T, U>(first: T, second: U): [T, U] {
    return [first, second];
}

function swap<T, U>(pair: [T, U]): [U, T] {
    return [pair[1], pair[0]];
}

console.log(identity(42));             // 42
console.log(identity("สวัสดี"));        // สวัสดี
console.log(pair("ชื่อ", 25));          // ["ชื่อ", 25]
console.log(swap(["a", 1]));           // [1, "a"]
```

```typescript
// ตัวอย่างที่ 46: Generic Functions กับ Constraints
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

interface Car {
    brand: string;
    model: string;
    year: number;
    price: number;
}

const myCar: Car = {
    brand: "Toyota",
    model: "Camry",
    year: 2024,
    price: 1200000
};

const brand = getProperty(myCar, "brand"); // type: string
const year = getProperty(myCar, "year");   // type: number
const price = getProperty(myCar, "price"); // type: number

console.log(`${brand} ${myCar.model} ปี ${year}: ฿${price.toLocaleString()}`);
// getProperty(myCar, "color"); // Error! "color" ไม่ใช่ key ของ Car
```

```typescript
// ตัวอย่างที่ 47: Generic Utility Functions
function groupBy<T, K extends string | number>(
    items: T[],
    keyFn: (item: T) => K
): Record<K, T[]> {
    return items.reduce((groups, item) => {
        const key = keyFn(item);
        if (!groups[key]) {
            groups[key] = [];
        }
        groups[key].push(item);
        return groups;
    }, {} as Record<K, T[]>);
}

function unique<T>(items: T[], keyFn?: (item: T) => unknown): T[] {
    if (!keyFn) {
        return [...new Set(items)];
    }
    const seen = new Set<unknown>();
    return items.filter(item => {
        const key = keyFn(item);
        if (seen.has(key)) return false;
        seen.add(key);
        return true;
    });
}

interface SaleItem {
    product: string;
    category: string;
    amount: number;
}

const sales: SaleItem[] = [
    { product: "แล็ปท็อป", category: "อิเล็กทรอนิกส์", amount: 25000 },
    { product: "เมาส์", category: "อิเล็กทรอนิกส์", amount: 500 },
    { product: "เสื้อยืด", category: "เสื้อผ้า", amount: 299 },
    { product: "กางเกง", category: "เสื้อผ้า", amount: 599 },
    { product: "หนังสือ", category: "สื่อ", amount: 250 }
];

const grouped = groupBy(sales, item => item.category);
console.log("จัดกลุ่มตาม category:");
Object.entries(grouped).forEach(([category, items]) => {
    const total = items.reduce((sum, item) => sum + item.amount, 0);
    console.log(`  ${category}: ${items.length} รายการ, รวม ฿${total.toLocaleString()}`);
});
```

---

## 7.14 Function Patterns ขั้นสูง

```typescript
// ตัวอย่างที่ 48: Builder Pattern ด้วย Functions
interface QueryBuilder {
    select(fields: string[]): QueryBuilder;
    from(table: string): QueryBuilder;
    where(condition: string): QueryBuilder;
    orderBy(field: string, direction?: "ASC" | "DESC"): QueryBuilder;
    limit(count: number): QueryBuilder;
    build(): string;
}

function createQueryBuilder(): QueryBuilder {
    const state = {
        fields: ["*"] as string[],
        table: "",
        conditions: [] as string[],
        orderFields: [] as string[],
        limitCount: 0
    };
    
    const builder: QueryBuilder = {
        select(fields: string[]): QueryBuilder {
            state.fields = fields;
            return builder;
        },
        from(table: string): QueryBuilder {
            state.table = table;
            return builder;
        },
        where(condition: string): QueryBuilder {
            state.conditions.push(condition);
            return builder;
        },
        orderBy(field: string, direction: "ASC" | "DESC" = "ASC"): QueryBuilder {
            state.orderFields.push(`${field} ${direction}`);
            return builder;
        },
        limit(count: number): QueryBuilder {
            state.limitCount = count;
            return builder;
        },
        build(): string {
            let query = `SELECT ${state.fields.join(", ")} FROM ${state.table}`;
            if (state.conditions.length > 0) {
                query += ` WHERE ${state.conditions.join(" AND ")}`;
            }
            if (state.orderFields.length > 0) {
                query += ` ORDER BY ${state.orderFields.join(", ")}`;
            }
            if (state.limitCount > 0) {
                query += ` LIMIT ${state.limitCount}`;
            }
            return query;
        }
    };
    
    return builder;
}

const query = createQueryBuilder()
    .select(["id", "name", "email", "department"])
    .from("employees")
    .where("department = 'IT'")
    .where("salary > 50000")
    .orderBy("name", "ASC")
    .limit(10)
    .build();

console.log(query);
// SELECT id, name, email, department FROM employees WHERE department = 'IT' AND salary > 50000 ORDER BY name ASC LIMIT 10
```

```typescript
// ตัวอย่างที่ 49: Middleware Pattern
type Middleware<T> = (context: T, next: () => Promise<void>) => Promise<void>;

interface RequestContext {
    path: string;
    method: string;
    userId?: number;
    startTime: number;
    log: string[];
}

function createMiddlewareRunner<T>(middlewares: Middleware<T>[]) {
    return async (context: T): Promise<void> => {
        let index = 0;
        
        const next = async (): Promise<void> => {
            if (index < middlewares.length) {
                const middleware = middlewares[index++];
                await middleware(context, next);
            }
        };
        
        await next();
    };
}

// Middleware functions
const loggingMiddleware: Middleware<RequestContext> = async (ctx, next) => {
    ctx.log.push(`[${new Date().toISOString()}] ${ctx.method} ${ctx.path}`);
    await next();
    const duration = Date.now() - ctx.startTime;
    ctx.log.push(`ใช้เวลา: ${duration}ms`);
};

const authMiddleware: Middleware<RequestContext> = async (ctx, next) => {
    ctx.userId = 1; // จำลอง auth
    ctx.log.push(`ยืนยันตัวตน: userId=${ctx.userId}`);
    await next();
};

const requestHandler: Middleware<RequestContext> = async (ctx, next) => {
    ctx.log.push(`จัดการ request: ${ctx.path}`);
    await next();
};

const runMiddlewares = createMiddlewareRunner([
    loggingMiddleware,
    authMiddleware,
    requestHandler
]);

const context: RequestContext = {
    path: "/api/users",
    method: "GET",
    startTime: Date.now(),
    log: []
};

runMiddlewares(context).then(() => {
    console.log("Middleware Log:");
    context.log.forEach(log => console.log("  " + log));
});
```

---

## 7.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Function Overloads

```typescript
// โจทย์: สร้าง function ที่รับข้อมูลหลายรูปแบบและคำนวณยอดรวม

interface CartItem {
    name: string;
    price: number;
    quantity: number;
}

// Overload 1: รับ array ของ CartItem
function calculateTotal(items: CartItem[]): number;
// Overload 2: รับแต่ละ item แยก
function calculateTotal(price: number, quantity: number): number;
// Overload 3: รับ total โดยตรง
function calculateTotal(total: number): number;

// Implementation
function calculateTotal(
    arg1: CartItem[] | number,
    arg2?: number
): number {
    if (Array.isArray(arg1)) {
        return arg1.reduce((sum, item) => sum + item.price * item.quantity, 0);
    } else if (arg2 !== undefined) {
        return arg1 * arg2;
    } else {
        return arg1;
    }
}

const cart: CartItem[] = [
    { name: "แล็ปท็อป", price: 25000, quantity: 1 },
    { name: "เมาส์", price: 500, quantity: 2 },
    { name: "คีย์บอร์ด", price: 1200, quantity: 1 }
];

console.log("ยอดรวม Cart:", calculateTotal(cart));            // 27200
console.log("ราคา x ปริมาณ:", calculateTotal(500, 3));        // 1500
console.log("ยอดตรง:", calculateTotal(9999));                  // 9999
```

### แบบฝึกหัดที่ 2: Higher-Order Functions

```typescript
// โจทย์: สร้าง validation system ด้วย Higher-Order Functions

type Validator<T> = (value: T) => { valid: boolean; message: string };

function required<T>(fieldName: string): Validator<T> {
    return (value: T) => {
        const valid = value !== null && value !== undefined && value !== "";
        return { valid, message: valid ? "" : `${fieldName} จำเป็นต้องกรอก` };
    };
}

function minLength(fieldName: string, min: number): Validator<string> {
    return (value: string) => {
        const valid = value.length >= min;
        return { valid, message: valid ? "" : `${fieldName} ต้องมีอย่างน้อย ${min} ตัวอักษร` };
    };
}

function maxLength(fieldName: string, max: number): Validator<string> {
    return (value: string) => {
        const valid = value.length <= max;
        return { valid, message: valid ? "" : `${fieldName} ต้องไม่เกิน ${max} ตัวอักษร` };
    };
}

function pattern(fieldName: string, regex: RegExp, errorMessage: string): Validator<string> {
    return (value: string) => {
        const valid = regex.test(value);
        return { valid, message: valid ? "" : `${fieldName}: ${errorMessage}` };
    };
}

function range(fieldName: string, min: number, max: number): Validator<number> {
    return (value: number) => {
        const valid = value >= min && value <= max;
        return { valid, message: valid ? "" : `${fieldName} ต้องอยู่ระหว่าง ${min} ถึง ${max}` };
    };
}

function combineValidators<T>(...validators: Validator<T>[]): Validator<T> {
    return (value: T) => {
        for (const validator of validators) {
            const result = validator(value);
            if (!result.valid) return result;
        }
        return { valid: true, message: "" };
    };
}

// สร้าง validators สำหรับฟอร์ม
const nameValidator = combineValidators<string>(
    required("ชื่อ"),
    minLength("ชื่อ", 2),
    maxLength("ชื่อ", 50)
);

const emailValidator = combineValidators<string>(
    required("อีเมล"),
    pattern("อีเมล", /^[^\s@]+@[^\s@]+\.[^\s@]+$/, "รูปแบบอีเมลไม่ถูกต้อง")
);

const ageValidator = combineValidators<number>(
    range("อายุ", 18, 100)
);

// ทดสอบ
const testData = [
    { name: "สมชาย ใจดี", email: "somchai@example.com", age: 25 },
    { name: "", email: "test", age: 15 },
    { name: "ก", email: "valid@email.com", age: 30 }
];

testData.forEach((data, i) => {
    console.log(`\nทดสอบชุดที่ ${i + 1}:`);
    const nameResult = nameValidator(data.name);
    const emailResult = emailValidator(data.email);
    const ageResult = ageValidator(data.age);
    
    if (!nameResult.valid) console.log(`  ❌ ${nameResult.message}`);
    if (!emailResult.valid) console.log(`  ❌ ${emailResult.message}`);
    if (!ageResult.valid) console.log(`  ❌ ${ageResult.message}`);
    if (nameResult.valid && emailResult.valid && ageResult.valid) {
        console.log(`  ✓ ข้อมูลถูกต้องทั้งหมด`);
    }
});
```

### แบบฝึกหัดที่ 3: Async Functions

```typescript
// โจทย์: สร้าง Data Fetching Service

interface ApiResponse<T> {
    data: T | null;
    error: string | null;
    statusCode: number;
    timestamp: Date;
}

interface PaginatedResponse<T> {
    items: T[];
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
}

interface Product {
    id: number;
    name: string;
    price: number;
    category: string;
    inStock: boolean;
}

// Mock data
const mockProducts: Product[] = Array.from({ length: 50 }, (_, i) => ({
    id: i + 1,
    name: `สินค้า ${i + 1}`,
    price: Math.floor(Math.random() * 10000) + 100,
    category: ["อิเล็กทรอนิกส์", "เสื้อผ้า", "อาหาร", "หนังสือ"][i % 4],
    inStock: Math.random() > 0.2
}));

async function fetchProducts(
    page: number = 1,
    pageSize: number = 10,
    category?: string
): Promise<ApiResponse<PaginatedResponse<Product>>> {
    try {
        await new Promise(resolve => setTimeout(resolve, 50)); // จำลอง delay
        
        let filtered = category
            ? mockProducts.filter(p => p.category === category)
            : mockProducts;
        
        const total = filtered.length;
        const start = (page - 1) * pageSize;
        const items = filtered.slice(start, start + pageSize);
        
        return {
            data: {
                items,
                total,
                page,
                pageSize,
                totalPages: Math.ceil(total / pageSize)
            },
            error: null,
            statusCode: 200,
            timestamp: new Date()
        };
    } catch (err) {
        return {
            data: null,
            error: err instanceof Error ? err.message : "เกิดข้อผิดพลาด",
            statusCode: 500,
            timestamp: new Date()
        };
    }
}

async function fetchInStockProducts(category?: string): Promise<Product[]> {
    const response = await fetchProducts(1, 100, category);
    if (!response.data) return [];
    return response.data.items.filter(p => p.inStock);
}

async function fetchProductsByPriceRange(
    minPrice: number,
    maxPrice: number
): Promise<Product[]> {
    const allProducts: Product[] = [];
    let page = 1;
    let hasMore = true;
    
    while (hasMore) {
        const response = await fetchProducts(page, 20);
        if (!response.data) break;
        
        allProducts.push(...response.data.items);
        hasMore = page < response.data.totalPages;
        page++;
    }
    
    return allProducts.filter(p => p.price >= minPrice && p.price <= maxPrice);
}

async function demonstrateDataFetching(): Promise<void> {
    console.log("=== ดึงข้อมูลสินค้า ===\n");
    
    // ดึงหน้าแรก
    const firstPage = await fetchProducts(1, 5);
    if (firstPage.data) {
        console.log(`รวมสินค้า: ${firstPage.data.total} รายการ`);
        console.log(`หน้า 1 จาก ${firstPage.data.totalPages}:`);
        firstPage.data.items.forEach(p => {
            console.log(`  ${p.name}: ฿${p.price.toLocaleString()} - ${p.inStock ? "มีสินค้า" : "สินค้าหมด"}`);
        });
    }
    
    // ดึงสินค้าที่มีในสต็อกในหมวดอิเล็กทรอนิกส์
    console.log("\nสินค้าอิเล็กทรอนิกส์ที่มีในสต็อก:");
    const electronics = await fetchInStockProducts("อิเล็กทรอนิกส์");
    electronics.slice(0, 3).forEach(p => {
        console.log(`  ${p.name}: ฿${p.price.toLocaleString()}`);
    });
    
    // ดึงสินค้าในช่วงราคา
    console.log("\nสินค้าราคา 1,000-5,000 บาท:");
    const priceRange = await fetchProductsByPriceRange(1000, 5000);
    console.log(`  พบ ${priceRange.length} รายการ`);
    priceRange.slice(0, 3).forEach(p => {
        console.log(`  ${p.name}: ฿${p.price.toLocaleString()}`);
    });
}

demonstrateDataFetching();
```

---

## สรุปบทที่ 7

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Function Types | กำหนด type ของ parameters และ return value |
| Optional Parameters | `param?: type` - ใส่หรือไม่ก็ได้ |
| Default Parameters | `param: type = defaultValue` |
| Rest Parameters | `...params: type[]` - รับได้หลายค่า |
| Function Overloads | รับ parameter ได้หลายรูปแบบ |
| this Parameter | กำหนด type ของ this ใน function |
| Arrow Functions | `(params) => returnValue` - กระชับกว่า |
| Callback Types | `(param: type) => returnType` |
| Higher-Order Functions | รับหรือส่งคืน function |
| void | ฟังก์ชันที่ไม่ return ค่าสำคัญ |
| never | ฟังก์ชันที่ไม่มีทางสิ้นสุดปกติ |
| async/await | ใช้กับ Promise สำหรับ async operations |
| Generator | `function*` - yield ค่าทีละค่า |
| Generic Functions | `<T>` - รับ type เป็น parameter |

**Best Practices:**
1. ระบุ return type ชัดเจนเสมอสำหรับ public functions
2. ใช้ Arrow Functions สำหรับ callbacks และ higher-order functions
3. ใช้ Generic เมื่อต้องการ reusable functions
4. หลีกเลี่ยงการใช้ `any` ใน function signatures
5. ใช้ Function Overloads เมื่อ function มีหลาย behavior
6. ใช้ `async/await` แทน callbacks ที่ซับซ้อน

ยินดีด้วย! คุณผ่านส่วนที่ 4-7 ของคอร์ส TypeScript แล้ว ในบทถัดไปเราจะเข้าสู่โลกของ **Classes และ Object-Oriented Programming** ใน TypeScript
