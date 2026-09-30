# ส่วนที่ 8: Interfaces ใน TypeScript

## บทนำ

Interface เป็นหนึ่งในคุณสมบัติที่สำคัญที่สุดของ TypeScript ที่ช่วยให้เราสามารถกำหนดรูปแบบ (shape) ของ object ได้อย่างชัดเจน Interface ทำหน้าที่เป็น "สัญญา" (contract) ที่บอกว่า object หรือ class จะต้องมี properties และ methods อะไรบ้าง

ในบทนี้เราจะเรียนรู้:
- การประกาศ Interface
- Optional properties
- Readonly properties
- Function types ใน Interface
- Indexable types
- Class implementing interfaces
- การขยาย Interface (Extending)
- Interface merging
- Generic interfaces
- ตัวอย่างการใช้งานจริง

---

## 8.1 การประกาศ Interface พื้นฐาน

Interface ประกาศด้วยคีย์เวิร์ด `interface` ตามด้วยชื่อและ body ที่บอก properties

```typescript
// การประกาศ Interface พื้นฐาน
interface User {
  id: number;
  name: string;
  email: string;
}

// การใช้งาน Interface
const user: User = {
  id: 1,
  name: "สมชาย ใจดี",
  email: "somchai@example.com"
};

console.log(user.name); // สมชาย ใจดี
```

```typescript
// ตัวอย่างที่ 2: Interface สำหรับสินค้า
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

const laptop: Product = {
  id: 101,
  name: "MacBook Pro",
  price: 59900,
  category: "Electronics"
};

console.log(`${laptop.name} ราคา ${laptop.price} บาท`);
```

```typescript
// ตัวอย่างที่ 3: Interface สำหรับที่อยู่
interface Address {
  street: string;
  city: string;
  province: string;
  zipCode: string;
  country: string;
}

const address: Address = {
  street: "123 ถนนสุขุมวิท",
  city: "กรุงเทพมหานคร",
  province: "กรุงเทพมหานคร",
  zipCode: "10110",
  country: "Thailand"
};

console.log(`${address.street}, ${address.city}`);
```

```typescript
// ตัวอย่างที่ 4: Interface ซ้อนกัน (Nested Interface)
interface ContactInfo {
  phone: string;
  email: string;
  address: Address; // ใช้ Interface อื่น
}

interface Person {
  id: number;
  firstName: string;
  lastName: string;
  contact: ContactInfo;
}

const person: Person = {
  id: 1,
  firstName: "สมหญิง",
  lastName: "รักเรียน",
  contact: {
    phone: "081-234-5678",
    email: "somying@example.com",
    address: {
      street: "456 ถนนพระราม 9",
      city: "กรุงเทพมหานคร",
      province: "กรุงเทพมหานคร",
      zipCode: "10310",
      country: "Thailand"
    }
  }
};
```

---

## 8.2 Optional Properties (Properties ที่ไม่บังคับ)

ใช้เครื่องหมาย `?` หลังชื่อ property เพื่อบอกว่าเป็น optional

```typescript
// Optional properties ด้วย ?
interface UserProfile {
  id: number;
  username: string;
  email: string;
  bio?: string;        // optional
  avatar?: string;     // optional
  website?: string;    // optional
}

// สร้าง user โดยไม่ใส่ optional properties
const user1: UserProfile = {
  id: 1,
  username: "john_doe",
  email: "john@example.com"
};

// สร้าง user พร้อม optional properties
const user2: UserProfile = {
  id: 2,
  username: "jane_doe",
  email: "jane@example.com",
  bio: "นักพัฒนา Full Stack",
  avatar: "https://example.com/avatar.jpg",
  website: "https://jane.dev"
};

console.log(user1.bio); // undefined
console.log(user2.bio); // นักพัฒนา Full Stack
```

```typescript
// ตัวอย่างที่ 2: Interface สำหรับการตั้งค่าแอพ
interface AppConfig {
  apiUrl: string;
  timeout: number;
  retries?: number;     // optional - default คือ 3
  debugMode?: boolean;  // optional - default คือ false
  logLevel?: "error" | "warn" | "info" | "debug"; // optional
}

function createApp(config: AppConfig) {
  const retries = config.retries ?? 3;
  const debugMode = config.debugMode ?? false;
  const logLevel = config.logLevel ?? "error";
  
  console.log(`เชื่อมต่อกับ ${config.apiUrl}`);
  console.log(`Timeout: ${config.timeout}ms`);
  console.log(`Retries: ${retries}`);
  console.log(`Debug: ${debugMode}`);
  console.log(`Log Level: ${logLevel}`);
}

createApp({
  apiUrl: "https://api.example.com",
  timeout: 5000
});
```

```typescript
// ตัวอย่างที่ 3: Interface สำหรับ Search Options
interface SearchOptions {
  query: string;
  page?: number;
  pageSize?: number;
  sortBy?: string;
  sortOrder?: "asc" | "desc";
  filters?: Record<string, string>;
}

function search(options: SearchOptions): void {
  const page = options.page ?? 1;
  const pageSize = options.pageSize ?? 10;
  const sortBy = options.sortBy ?? "createdAt";
  const sortOrder = options.sortOrder ?? "desc";
  
  console.log(`ค้นหา: "${options.query}"`);
  console.log(`หน้า: ${page}, จำนวนต่อหน้า: ${pageSize}`);
  console.log(`เรียงตาม: ${sortBy} (${sortOrder})`);
}

search({ query: "TypeScript tutorial" });
search({ query: "React hooks", page: 2, pageSize: 20, sortBy: "rating" });
```

---

## 8.3 Readonly Properties

ใช้ `readonly` เพื่อป้องกันการแก้ไข property หลังจากสร้าง object แล้ว

```typescript
// Readonly properties
interface Point {
  readonly x: number;
  readonly y: number;
}

const origin: Point = { x: 0, y: 0 };
// origin.x = 10; // Error! Cannot assign to 'x' because it is a read-only property

const point: Point = { x: 3, y: 4 };
console.log(`Point: (${point.x}, ${point.y})`);
```

```typescript
// ตัวอย่างที่ 2: Database Record Interface
interface DatabaseRecord {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
  data: Record<string, unknown>;
}

const record: DatabaseRecord = {
  id: 1,
  createdAt: new Date("2024-01-01"),
  updatedAt: new Date("2024-01-01"),
  data: { name: "test" }
};

// สามารถแก้ไขได้
record.updatedAt = new Date();
record.data = { name: "updated" };

// ไม่สามารถแก้ไขได้
// record.id = 2; // Error!
// record.createdAt = new Date(); // Error!
```

```typescript
// ตัวอย่างที่ 3: Configuration Interface พร้อม Readonly
interface ServerConfig {
  readonly host: string;
  readonly port: number;
  readonly protocol: "http" | "https";
  maxConnections: number;
  timeout: number;
}

const serverConfig: ServerConfig = {
  host: "localhost",
  port: 3000,
  protocol: "http",
  maxConnections: 100,
  timeout: 30000
};

// แก้ไข mutable properties ได้
serverConfig.maxConnections = 200;
serverConfig.timeout = 60000;

// แก้ไข readonly ไม่ได้
// serverConfig.host = "production.example.com"; // Error!
// serverConfig.port = 443; // Error!
```

```typescript
// ตัวอย่างที่ 4: ReadonlyArray ใน Interface
interface Catalog {
  readonly items: ReadonlyArray<Product>;
  readonly lastUpdated: Date;
  version: number;
}

const catalog: Catalog = {
  items: [
    { id: 1, name: "สินค้า A", price: 100, category: "A" },
    { id: 2, name: "สินค้า B", price: 200, category: "B" }
  ],
  lastUpdated: new Date(),
  version: 1
};

// catalog.items.push({...}); // Error! - ReadonlyArray
// catalog.items[0] = {...}; // Error! - ReadonlyArray
catalog.version++; // OK
```

---

## 8.4 Function Types ใน Interface

Interface สามารถกำหนด function signature ได้

```typescript
// Function type ใน Interface
interface Greeter {
  greet(name: string): string;
  farewell(name: string): void;
}

const greeter: Greeter = {
  greet(name: string): string {
    return `สวัสดี, ${name}!`;
  },
  farewell(name: string): void {
    console.log(`ลาก่อน, ${name}!`);
  }
};

console.log(greeter.greet("สมชาย")); // สวัสดี, สมชาย!
greeter.farewell("สมชาย"); // ลาก่อน, สมชาย!
```

```typescript
// ตัวอย่างที่ 2: Interface สำหรับ Calculator
interface Calculator {
  add(a: number, b: number): number;
  subtract(a: number, b: number): number;
  multiply(a: number, b: number): number;
  divide(a: number, b: number): number | never;
  reset(): void;
}

const calculator: Calculator = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b,
  divide: (a, b) => {
    if (b === 0) throw new Error("ไม่สามารถหารด้วยศูนย์ได้");
    return a / b;
  },
  reset: () => console.log("รีเซ็ตเครื่องคิดเลข")
};

console.log(calculator.add(10, 5));      // 15
console.log(calculator.multiply(4, 3));  // 12
```

```typescript
// ตัวอย่างที่ 3: Interface สำหรับ Event Handler
interface EventHandler {
  onClick(event: MouseEvent): void;
  onKeyDown(event: KeyboardEvent): void;
  onChange(value: string): void;
  onSubmit(data: Record<string, string>): Promise<void>;
}

// ตัวอย่างการ implement
const formHandler: Partial<EventHandler> = {
  onClick: (event) => {
    console.log(`คลิกที่ตำแหน่ง: (${event.clientX}, ${event.clientY})`);
  },
  onChange: (value) => {
    console.log(`ค่าเปลี่ยนเป็น: ${value}`);
  }
};
```

```typescript
// ตัวอย่างที่ 4: Interface สำหรับ Validator
interface Validator {
  validate(value: unknown): boolean;
  getMessage(): string;
  reset(): void;
}

interface FieldValidators {
  required: Validator;
  email: Validator;
  minLength: (min: number) => Validator;
  maxLength: (max: number) => Validator;
}

// การสร้าง Validators
const createRequiredValidator = (): Validator => ({
  validate: (value) => value !== null && value !== undefined && value !== "",
  getMessage: () => "จำเป็นต้องกรอกข้อมูล",
  reset: () => {}
});

const createEmailValidator = (): Validator => ({
  validate: (value) => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(String(value));
  },
  getMessage: () => "รูปแบบอีเมลไม่ถูกต้อง",
  reset: () => {}
});
```

---

## 8.5 Interface สำหรับ Function (Call Signature)

```typescript
// Call Signature ใน Interface
interface StringTransformer {
  (input: string): string;
}

const toUpperCase: StringTransformer = (input) => input.toUpperCase();
const toLowerCase: StringTransformer = (input) => input.toLowerCase();
const trim: StringTransformer = (input) => input.trim();

console.log(toUpperCase("hello world")); // HELLO WORLD
console.log(toLowerCase("HELLO WORLD")); // hello world
```

```typescript
// ตัวอย่างที่ 2: Interface ที่มีทั้ง call signature และ properties
interface Counter {
  (start: number): string;
  interval: number;
  reset(): void;
}

function makeCounter(): Counter {
  let count = 0;
  
  const counter: Counter = function(start: number) {
    count = start;
    return `เริ่มนับที่ ${start}`;
  };
  
  counter.interval = 1;
  counter.reset = function() {
    count = 0;
    console.log("รีเซ็ตตัวนับ");
  };
  
  return counter;
}

const myCounter = makeCounter();
console.log(myCounter(10)); // เริ่มนับที่ 10
myCounter.reset();
```

---

## 8.6 Indexable Types

Interface สามารถกำหนด index signature ได้ เพื่อรองรับ object ที่มี dynamic keys

```typescript
// String Index Signature
interface StringDictionary {
  [key: string]: string;
}

const translations: StringDictionary = {
  hello: "สวัสดี",
  goodbye: "ลาก่อน",
  thank_you: "ขอบคุณ",
  please: "กรุณา"
};

console.log(translations["hello"]); // สวัสดี
console.log(translations["goodbye"]); // ลาก่อน
```

```typescript
// Number Index Signature
interface NumberedList {
  [index: number]: string;
}

const weekdays: NumberedList = {
  0: "อาทิตย์",
  1: "จันทร์",
  2: "อังคาร",
  3: "พุธ",
  4: "พฤหัสบดี",
  5: "ศุกร์",
  6: "เสาร์"
};

console.log(weekdays[1]); // จันทร์
console.log(weekdays[5]); // ศุกร์
```

```typescript
// Index Signature พร้อม Properties อื่น
interface UserCache {
  [userId: string]: User;
  size: number; // ต้องเป็น type ที่ compatible กับ index value
}

// หมายเหตุ: size ต้องเป็น type ที่ compatible
// ในกรณีนี้จะเกิด error เพราะ number ไม่ compatible กับ User
// ต้องใช้ User | number แทน

interface FlexibleCache {
  [key: string]: User | number;
  size: number;
}

const cache: FlexibleCache = {
  size: 0
};

cache["user1"] = { id: 1, name: "สมชาย", email: "s@e.com" };
cache.size = 1;
```

```typescript
// ตัวอย่าง Readonly Index Signature
interface ReadonlyStringMap {
  readonly [key: string]: string;
}

const config: ReadonlyStringMap = {
  dbHost: "localhost",
  dbPort: "5432",
  apiKey: "secret123"
};

console.log(config.dbHost); // localhost
// config.dbHost = "production"; // Error! Cannot assign
```

```typescript
// ตัวอย่างใช้งานจริง: Cache System
interface CacheEntry<T> {
  value: T;
  timestamp: number;
  ttl: number; // time-to-live in seconds
}

interface Cache<T> {
  [key: string]: CacheEntry<T>;
}

function createCache<T>() {
  const store: Cache<T> = {};
  
  return {
    set(key: string, value: T, ttl: number = 3600): void {
      store[key] = {
        value,
        timestamp: Date.now(),
        ttl
      };
    },
    
    get(key: string): T | null {
      const entry = store[key];
      if (!entry) return null;
      
      const age = (Date.now() - entry.timestamp) / 1000;
      if (age > entry.ttl) {
        delete store[key];
        return null;
      }
      
      return entry.value;
    },
    
    has(key: string): boolean {
      return this.get(key) !== null;
    }
  };
}

const userCache = createCache<User>();
userCache.set("user1", { id: 1, name: "สมชาย", email: "s@e.com" });
console.log(userCache.get("user1")); // { id: 1, name: 'สมชาย', ... }
```

---

## 8.7 Class Implementing Interface

Class สามารถ implement interface ได้ด้วยคีย์เวิร์ด `implements`

```typescript
// Interface สำหรับ Animal
interface Animal {
  name: string;
  sound(): string;
  move(distance: number): void;
}

// Class implement Interface
class Dog implements Animal {
  name: string;
  
  constructor(name: string) {
    this.name = name;
  }
  
  sound(): string {
    return "โฮ่ง!";
  }
  
  move(distance: number): void {
    console.log(`${this.name} วิ่งไป ${distance} เมตร`);
  }
}

class Cat implements Animal {
  name: string;
  
  constructor(name: string) {
    this.name = name;
  }
  
  sound(): string {
    return "เมี้ยว!";
  }
  
  move(distance: number): void {
    console.log(`${this.name} เดินไป ${distance} เมตร`);
  }
}

const dog = new Dog("บัดดี้");
const cat = new Cat("วิสกี้");

console.log(dog.sound()); // โฮ่ง!
dog.move(10);             // บัดดี้ วิ่งไป 10 เมตร
console.log(cat.sound()); // เมี้ยว!
```

```typescript
// ตัวอย่างที่ 2: Interface สำหรับ Payment Processor
interface PaymentProcessor {
  process(amount: number, currency: string): Promise<boolean>;
  refund(transactionId: string, amount: number): Promise<boolean>;
  getBalance(): Promise<number>;
}

class StripeProcessor implements PaymentProcessor {
  private apiKey: string;
  
  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }
  
  async process(amount: number, currency: string): Promise<boolean> {
    console.log(`Stripe: ประมวลผลการชำระเงิน ${amount} ${currency}`);
    // จำลองการเรียก API
    return true;
  }
  
  async refund(transactionId: string, amount: number): Promise<boolean> {
    console.log(`Stripe: คืนเงิน ${amount} สำหรับ transaction ${transactionId}`);
    return true;
  }
  
  async getBalance(): Promise<number> {
    return 100000; // จำลองยอดคงเหลือ
  }
}

class PayPalProcessor implements PaymentProcessor {
  private clientId: string;
  
  constructor(clientId: string) {
    this.clientId = clientId;
  }
  
  async process(amount: number, currency: string): Promise<boolean> {
    console.log(`PayPal: ประมวลผลการชำระเงิน ${amount} ${currency}`);
    return true;
  }
  
  async refund(transactionId: string, amount: number): Promise<boolean> {
    console.log(`PayPal: คืนเงิน ${amount} สำหรับ transaction ${transactionId}`);
    return true;
  }
  
  async getBalance(): Promise<number> {
    return 50000;
  }
}

// ฟังก์ชันที่รับ PaymentProcessor ใดก็ได้
async function checkout(
  processor: PaymentProcessor,
  amount: number
): Promise<void> {
  const success = await processor.process(amount, "THB");
  if (success) {
    console.log("การชำระเงินสำเร็จ!");
  }
}
```

```typescript
// ตัวอย่างที่ 3: Interface สำหรับ Repository Pattern
interface Repository<T> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  update(id: number, data: Partial<T>): Promise<T | null>;
  delete(id: number): Promise<boolean>;
}

interface UserEntity {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
}

class InMemoryUserRepository implements Repository<UserEntity> {
  private users: UserEntity[] = [];
  private nextId = 1;
  
  async findById(id: number): Promise<UserEntity | null> {
    return this.users.find(u => u.id === id) ?? null;
  }
  
  async findAll(): Promise<UserEntity[]> {
    return [...this.users];
  }
  
  async save(entity: Omit<UserEntity, "id" | "createdAt">): Promise<UserEntity> {
    const user: UserEntity = {
      ...entity as UserEntity,
      id: this.nextId++,
      createdAt: new Date()
    };
    this.users.push(user);
    return user;
  }
  
  async update(id: number, data: Partial<UserEntity>): Promise<UserEntity | null> {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) return null;
    
    this.users[index] = { ...this.users[index], ...data };
    return this.users[index];
  }
  
  async delete(id: number): Promise<boolean> {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) return false;
    
    this.users.splice(index, 1);
    return true;
  }
}
```

---

## 8.8 Extending Interfaces (การขยาย Interface)

Interface สามารถ extend interface อื่นได้

```typescript
// Interface พื้นฐาน
interface Shape {
  color: string;
  area(): number;
  perimeter(): number;
}

// Extending Interface
interface ColoredShape extends Shape {
  borderColor: string;
  borderWidth: number;
}

// Implementing Extended Interface
class Rectangle implements ColoredShape {
  constructor(
    public color: string,
    public borderColor: string,
    public borderWidth: number,
    private width: number,
    private height: number
  ) {}
  
  area(): number {
    return this.width * this.height;
  }
  
  perimeter(): number {
    return 2 * (this.width + this.height);
  }
}

const rect = new Rectangle("blue", "black", 2, 10, 5);
console.log(`พื้นที่: ${rect.area()}`);       // 50
console.log(`เส้นรอบวง: ${rect.perimeter()}`); // 30
```

```typescript
// ตัวอย่าง Multiple Interface Extension
interface Timestamps {
  createdAt: Date;
  updatedAt: Date;
}

interface SoftDelete {
  deletedAt?: Date;
  isDeleted: boolean;
}

interface Entity extends Timestamps, SoftDelete {
  id: number;
}

interface UserModel extends Entity {
  username: string;
  email: string;
  passwordHash: string;
}

const newUser: UserModel = {
  id: 1,
  username: "john_doe",
  email: "john@example.com",
  passwordHash: "hashed_password",
  createdAt: new Date(),
  updatedAt: new Date(),
  isDeleted: false
};

console.log(`User: ${newUser.username}, Created: ${newUser.createdAt}`);
```

```typescript
// ตัวอย่างที่ 3: Interface Hierarchy
interface Vehicle {
  brand: string;
  model: string;
  year: number;
  startEngine(): void;
  stopEngine(): void;
}

interface ElectricVehicle extends Vehicle {
  batteryCapacity: number; // kWh
  currentCharge: number;   // percentage
  charge(percentage: number): void;
  getRange(): number;
}

interface HybridVehicle extends Vehicle {
  fuelCapacity: number;
  batteryCapacity: number;
  getFuelLevel(): number;
}

class Tesla implements ElectricVehicle {
  brand = "Tesla";
  model: string;
  year: number;
  batteryCapacity: number;
  currentCharge: number;
  
  constructor(model: string, year: number, batteryCapacity: number) {
    this.model = model;
    this.year = year;
    this.batteryCapacity = batteryCapacity;
    this.currentCharge = 100;
  }
  
  startEngine(): void {
    console.log(`${this.brand} ${this.model} พร้อมใช้งาน (ไฟฟ้า)`);
  }
  
  stopEngine(): void {
    console.log(`${this.brand} ${this.model} ปิดระบบ`);
  }
  
  charge(percentage: number): void {
    this.currentCharge = Math.min(100, this.currentCharge + percentage);
    console.log(`ชาร์จแบตเตอรี่: ${this.currentCharge}%`);
  }
  
  getRange(): number {
    // คำนวณระยะทางจากแบตเตอรี่ที่เหลือ
    return (this.currentCharge / 100) * this.batteryCapacity * 5;
  }
}

const model3 = new Tesla("Model 3", 2023, 75);
model3.startEngine();
console.log(`ระยะทางที่ไปได้: ${model3.getRange()} กม.`);
```

---

## 8.9 Interface Merging (Declaration Merging)

TypeScript อนุญาตให้ประกาศ Interface ที่มีชื่อเดียวกันหลายครั้ง และจะรวมกันโดยอัตโนมัติ

```typescript
// Declaration Merging - ประกาศ Interface ชื่อเดียวกันหลายครั้ง
interface Window {
  myCustomProperty: string;
}

interface Window {
  anotherProperty: number;
}

// ผลลัพธ์: Window มีทั้ง myCustomProperty และ anotherProperty
// window.myCustomProperty = "hello"; // ทำงานได้
// window.anotherProperty = 42;      // ทำงานได้
```

```typescript
// ตัวอย่างการใช้งาน Declaration Merging จริง
// สมมติมี Library ที่ประกาศ Interface นี้
interface Request {
  url: string;
  method: string;
}

// เราสามารถเพิ่ม properties เข้าไปได้
interface Request {
  userId?: string;       // เพิ่ม authentication info
  correlationId?: string; // เพิ่ม tracing info
}

// ตอนนี้ Request มีทั้ง properties ดั้งเดิมและที่เพิ่มมา
function processRequest(req: Request): void {
  console.log(`${req.method} ${req.url}`);
  if (req.userId) {
    console.log(`User: ${req.userId}`);
  }
  if (req.correlationId) {
    console.log(`Correlation: ${req.correlationId}`);
  }
}
```

```typescript
// ตัวอย่างที่ 3: Declaration Merging สำหรับ Global Types
// ไฟล์ global.d.ts
interface ProcessEnv {
  NODE_ENV: "development" | "production" | "test";
  PORT: string;
}

// ไฟล์ app.d.ts (เพิ่ม environment variables)
interface ProcessEnv {
  DATABASE_URL: string;
  API_KEY: string;
  JWT_SECRET: string;
}

// ตอนนี้ ProcessEnv มีทุก properties จากทั้งสองการประกาศ
```

```typescript
// Declaration Merging กับ Function Overloads
interface StringUtils {
  format(value: string): string;
}

interface StringUtils {
  format(value: string, locale: string): string;
}

interface StringUtils {
  format(value: number, decimals: number): string;
}

// Implementation รับได้หลาย signatures
const utils: StringUtils = {
  format: (value: string | number, extra?: string | number): string => {
    if (typeof value === "number" && typeof extra === "number") {
      return value.toFixed(extra);
    }
    if (typeof value === "string" && typeof extra === "string") {
      return new Intl.NumberFormat(extra).format(parseFloat(value));
    }
    return String(value);
  }
};
```

---

## 8.10 Interface vs Type Alias

การเปรียบเทียบระหว่าง Interface และ Type Alias

```typescript
// === Interface ===
interface PersonInterface {
  name: string;
  age: number;
}

// === Type Alias ===
type PersonType = {
  name: string;
  age: number;
};

// ทั้งสองใช้งานได้เหมือนกันสำหรับ object type พื้นฐาน
const p1: PersonInterface = { name: "สมชาย", age: 30 };
const p2: PersonType = { name: "สมหญิง", age: 25 };
```

```typescript
// ข้อแตกต่าง 1: Declaration Merging
// Interface รองรับ Declaration Merging
interface Car {
  brand: string;
}
interface Car {
  model: string;
}
// Car ตอนนี้มีทั้ง brand และ model

// Type Alias ไม่รองรับ - จะ Error
// type Car = { brand: string; }; // Error: Duplicate identifier
// type Car = { model: string; };
```

```typescript
// ข้อแตกต่าง 2: Extending
// Interface ใช้ extends
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
}

// Type ใช้ & (intersection)
type AnimalType = {
  name: string;
};
type DogType = AnimalType & {
  breed: string;
};

const myDog: Dog = { name: "Rex", breed: "Golden Retriever" };
const myDog2: DogType = { name: "Max", breed: "Labrador" };
```

```typescript
// ข้อแตกต่าง 3: Type Alias ทำได้มากกว่า
// Type Alias สามารถใช้กับ primitive, union, tuple
type StringOrNumber = string | number;
type Point2D = [number, number];
type Callback = (data: string) => void;

// Interface ทำแบบนี้ไม่ได้
// interface StringOrNumber = string | number; // Error!
```

```typescript
// แนวทางการเลือกใช้:
// ใช้ Interface เมื่อ:
// - กำหนด shape ของ object หรือ class
// - ต้องการ extends หรือ implements
// - ต้องการ declaration merging

// ใช้ Type Alias เมื่อ:
// - union types, intersection types
// - primitive types
// - tuple types
// - function types ที่ซับซ้อน
// - computed types (conditional types, mapped types)

interface ApiResponse {
  success: boolean;
  data: unknown;
  message: string;
}

type ApiResponseCode = 200 | 201 | 400 | 401 | 403 | 404 | 500;
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
```

---

## 8.11 Generic Interfaces

Interface สามารถมี type parameters ได้

```typescript
// Generic Interface พื้นฐาน
interface Container<T> {
  value: T;
  getValue(): T;
  setValue(value: T): void;
}

// Implementation
class Box<T> implements Container<T> {
  value: T;
  
  constructor(value: T) {
    this.value = value;
  }
  
  getValue(): T {
    return this.value;
  }
  
  setValue(value: T): void {
    this.value = value;
  }
}

const numberBox = new Box<number>(42);
const stringBox = new Box<string>("Hello");

console.log(numberBox.getValue()); // 42
console.log(stringBox.getValue()); // Hello
```

```typescript
// Generic Interface สำหรับ API Response
interface ApiResponse<T> {
  success: boolean;
  data: T;
  message: string;
  statusCode: number;
  timestamp: Date;
}

interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
  };
}

// การใช้งาน
async function fetchUser(id: number): Promise<ApiResponse<UserEntity>> {
  // จำลอง API call
  return {
    success: true,
    data: {
      id,
      name: "สมชาย ใจดี",
      email: "somchai@example.com",
      createdAt: new Date(),
      updatedAt: new Date(),
      isDeleted: false
    },
    message: "ดึงข้อมูล user สำเร็จ",
    statusCode: 200,
    timestamp: new Date()
  };
}

async function fetchUsers(): Promise<PaginatedResponse<UserEntity>> {
  return {
    success: true,
    data: [],
    message: "ดึงข้อมูล users สำเร็จ",
    statusCode: 200,
    timestamp: new Date(),
    pagination: {
      page: 1,
      pageSize: 10,
      total: 100,
      totalPages: 10
    }
  };
}
```

```typescript
// Generic Interface สำหรับ Repository
interface GenericRepository<T, ID> {
  findById(id: ID): Promise<T | null>;
  findAll(filters?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;
  update(id: ID, data: Partial<T>): Promise<T | null>;
  delete(id: ID): Promise<boolean>;
  count(filters?: Partial<T>): Promise<number>;
}

interface OrderItem {
  id: number;
  orderId: number;
  productId: number;
  quantity: number;
  price: number;
}

// Class ที่ implement Generic Repository
class OrderItemRepository implements GenericRepository<OrderItem, number> {
  private items: OrderItem[] = [];
  private nextId = 1;
  
  async findById(id: number): Promise<OrderItem | null> {
    return this.items.find(item => item.id === id) ?? null;
  }
  
  async findAll(filters?: Partial<OrderItem>): Promise<OrderItem[]> {
    if (!filters) return [...this.items];
    
    return this.items.filter(item => {
      return Object.entries(filters).every(([key, value]) => {
        return (item as Record<string, unknown>)[key] === value;
      });
    });
  }
  
  async create(data: Omit<OrderItem, "id">): Promise<OrderItem> {
    const item: OrderItem = { ...data, id: this.nextId++ };
    this.items.push(item);
    return item;
  }
  
  async update(id: number, data: Partial<OrderItem>): Promise<OrderItem | null> {
    const index = this.items.findIndex(item => item.id === id);
    if (index === -1) return null;
    this.items[index] = { ...this.items[index], ...data };
    return this.items[index];
  }
  
  async delete(id: number): Promise<boolean> {
    const index = this.items.findIndex(item => item.id === id);
    if (index === -1) return false;
    this.items.splice(index, 1);
    return true;
  }
  
  async count(filters?: Partial<OrderItem>): Promise<number> {
    const all = await this.findAll(filters);
    return all.length;
  }
}
```

```typescript
// Generic Interface สำหรับ Event System
interface EventEmitter<Events extends Record<string, unknown[]>> {
  on<K extends keyof Events>(event: K, listener: (...args: Events[K]) => void): void;
  off<K extends keyof Events>(event: K, listener: (...args: Events[K]) => void): void;
  emit<K extends keyof Events>(event: K, ...args: Events[K]): void;
}

interface AppEvents {
  userLoggedIn: [userId: string, timestamp: Date];
  userLoggedOut: [userId: string];
  errorOccurred: [error: Error, context: string];
  dataLoaded: [count: number, duration: number];
}

class TypedEventEmitter<Events extends Record<string, unknown[]>> 
  implements EventEmitter<Events> {
  
  private listeners: Map<keyof Events, Array<(...args: unknown[]) => void>> = new Map();
  
  on<K extends keyof Events>(event: K, listener: (...args: Events[K]) => void): void {
    const existing = this.listeners.get(event) ?? [];
    existing.push(listener as (...args: unknown[]) => void);
    this.listeners.set(event, existing);
  }
  
  off<K extends keyof Events>(event: K, listener: (...args: Events[K]) => void): void {
    const existing = this.listeners.get(event) ?? [];
    const filtered = existing.filter(l => l !== listener);
    this.listeners.set(event, filtered);
  }
  
  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    const eventListeners = this.listeners.get(event) ?? [];
    eventListeners.forEach(listener => listener(...args));
  }
}

const emitter = new TypedEventEmitter<AppEvents>();

emitter.on("userLoggedIn", (userId, timestamp) => {
  console.log(`User ${userId} เข้าสู่ระบบเมื่อ ${timestamp}`);
});

emitter.emit("userLoggedIn", "user123", new Date());
```

---

## 8.12 ตัวอย่างจริง: API Response Types

```typescript
// ตัวอย่างระบบจัดการร้านค้า E-commerce

// Base Interfaces
interface BaseEntity {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
}

// Product Interface
interface IProduct extends BaseEntity {
  name: string;
  description: string;
  price: number;
  stock: number;
  category: ICategory;
  images: string[];
  isActive: boolean;
}

interface ICategory extends BaseEntity {
  name: string;
  slug: string;
  parentId?: number;
}

// Order Interfaces
interface IOrderItem {
  product: IProduct;
  quantity: number;
  unitPrice: number;
  totalPrice: number;
}

interface IOrder extends BaseEntity {
  orderNumber: string;
  customer: ICustomer;
  items: IOrderItem[];
  subtotal: number;
  tax: number;
  shipping: number;
  total: number;
  status: OrderStatus;
  shippingAddress: IAddress;
}

type OrderStatus = 
  | "pending"
  | "confirmed"
  | "processing"
  | "shipped"
  | "delivered"
  | "cancelled"
  | "refunded";

interface ICustomer extends BaseEntity {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  addresses: IAddress[];
}

interface IAddress {
  id?: number;
  type: "billing" | "shipping" | "both";
  street: string;
  city: string;
  province: string;
  zipCode: string;
  country: string;
  isDefault?: boolean;
}

// API Response Interfaces
interface ApiSuccess<T> {
  success: true;
  data: T;
  message: string;
  statusCode: 200 | 201 | 204;
}

interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, string[]>;
  };
  statusCode: 400 | 401 | 403 | 404 | 422 | 500;
}

type ApiResult<T> = ApiSuccess<T> | ApiError;

// ตัวอย่างการใช้งาน
async function getOrder(orderId: number): Promise<ApiResult<IOrder>> {
  try {
    // จำลองการเรียก API
    const order: IOrder = {
      id: orderId,
      orderNumber: `ORD-${orderId.toString().padStart(6, "0")}`,
      createdAt: new Date(),
      updatedAt: new Date(),
      customer: {
        id: 1,
        firstName: "สมชาย",
        lastName: "ใจดี",
        email: "somchai@example.com",
        phone: "081-234-5678",
        addresses: [],
        createdAt: new Date(),
        updatedAt: new Date()
      },
      items: [],
      subtotal: 1000,
      tax: 70,
      shipping: 50,
      total: 1120,
      status: "confirmed",
      shippingAddress: {
        type: "shipping",
        street: "123 ถนนสุขุมวิท",
        city: "กรุงเทพมหานคร",
        province: "กรุงเทพมหานคร",
        zipCode: "10110",
        country: "Thailand"
      }
    };
    
    return {
      success: true,
      data: order,
      message: "ดึงข้อมูล order สำเร็จ",
      statusCode: 200
    };
  } catch (error) {
    return {
      success: false,
      error: {
        code: "ORDER_NOT_FOUND",
        message: `ไม่พบ Order หมายเลข ${orderId}`
      },
      statusCode: 404
    };
  }
}
```

---

## 8.13 ตัวอย่างจริง: Config Objects

```typescript
// Configuration System
interface DatabaseConfig {
  readonly host: string;
  readonly port: number;
  readonly database: string;
  readonly username: string;
  readonly password: string;
  readonly ssl: boolean;
  pool?: {
    min: number;
    max: number;
    idle: number;
  };
}

interface RedisConfig {
  readonly host: string;
  readonly port: number;
  readonly password?: string;
  readonly db: number;
  readonly ttl: number;
}

interface EmailConfig {
  readonly provider: "smtp" | "sendgrid" | "mailgun";
  readonly from: string;
  readonly apiKey?: string;
  smtp?: {
    host: string;
    port: number;
    secure: boolean;
  };
}

interface LoggingConfig {
  level: "debug" | "info" | "warn" | "error";
  outputs: Array<"console" | "file" | "remote">;
  filePath?: string;
  remoteUrl?: string;
}

interface AppConfiguration {
  readonly environment: "development" | "staging" | "production";
  readonly port: number;
  readonly host: string;
  readonly database: DatabaseConfig;
  readonly redis?: RedisConfig;
  readonly email: EmailConfig;
  readonly logging: LoggingConfig;
  readonly features: Record<string, boolean>;
}

// Default Configuration
const defaultConfig: AppConfiguration = {
  environment: "development",
  port: 3000,
  host: "localhost",
  database: {
    host: "localhost",
    port: 5432,
    database: "myapp_dev",
    username: "postgres",
    password: "",
    ssl: false,
    pool: {
      min: 2,
      max: 10,
      idle: 10000
    }
  },
  email: {
    provider: "smtp",
    from: "noreply@example.com",
    smtp: {
      host: "smtp.example.com",
      port: 587,
      secure: false
    }
  },
  logging: {
    level: "debug",
    outputs: ["console"]
  },
  features: {
    newDashboard: true,
    betaFeature: false
  }
};

function getConfig(): AppConfiguration {
  return defaultConfig;
}

const config = getConfig();
console.log(`Server รันที่ ${config.host}:${config.port}`);
console.log(`Database: ${config.database.host}:${config.database.port}/${config.database.database}`);
```

---

## 8.14 ตัวอย่างจริง: Event Handlers

```typescript
// Event Handler System
interface DOMEventMap {
  click: MouseEvent;
  keydown: KeyboardEvent;
  keyup: KeyboardEvent;
  submit: SubmitEvent;
  change: Event;
  input: InputEvent;
  focus: FocusEvent;
  blur: FocusEvent;
  resize: UIEvent;
  scroll: Event;
}

interface EventListenerOptions {
  once?: boolean;
  passive?: boolean;
  capture?: boolean;
}

interface TypedEventTarget<EventMap extends Record<string, Event>> {
  addEventListener<K extends keyof EventMap>(
    type: K,
    listener: (event: EventMap[K]) => void,
    options?: EventListenerOptions
  ): void;
  
  removeEventListener<K extends keyof EventMap>(
    type: K,
    listener: (event: EventMap[K]) => void
  ): void;
}

// Custom Event System
interface CustomEvents {
  "user:login": { userId: string; timestamp: Date };
  "user:logout": { userId: string };
  "cart:add": { productId: number; quantity: number };
  "cart:remove": { productId: number };
  "order:created": { orderId: number; total: number };
}

interface CustomEventEmitter<Events extends Record<string, object>> {
  on<K extends keyof Events>(
    event: K,
    handler: (data: Events[K]) => void
  ): () => void; // return unsubscribe function
  
  emit<K extends keyof Events>(event: K, data: Events[K]): void;
  
  once<K extends keyof Events>(
    event: K,
    handler: (data: Events[K]) => void
  ): void;
}

// React-like Component Interface
interface ComponentProps {
  children?: unknown;
  className?: string;
  style?: Record<string, string | number>;
  "data-testid"?: string;
}

interface ButtonProps extends ComponentProps {
  onClick?: (event: MouseEvent) => void;
  disabled?: boolean;
  variant?: "primary" | "secondary" | "danger";
  size?: "sm" | "md" | "lg";
  loading?: boolean;
  type?: "button" | "submit" | "reset";
}

interface InputProps extends ComponentProps {
  value?: string;
  defaultValue?: string;
  onChange?: (event: InputEvent) => void;
  onBlur?: (event: FocusEvent) => void;
  onFocus?: (event: FocusEvent) => void;
  placeholder?: string;
  disabled?: boolean;
  readOnly?: boolean;
  type?: "text" | "email" | "password" | "number" | "tel";
  name?: string;
  id?: string;
  required?: boolean;
  autoComplete?: string;
}

// Form Handler Interface
interface FormHandlers<T extends Record<string, unknown>> {
  handleChange: (field: keyof T, value: T[keyof T]) => void;
  handleSubmit: (event: SubmitEvent) => Promise<void>;
  handleReset: () => void;
  setFieldError: (field: keyof T, error: string) => void;
  clearFieldError: (field: keyof T) => void;
  clearAllErrors: () => void;
}

interface FormState<T extends Record<string, unknown>> {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  isSubmitting: boolean;
  isValid: boolean;
}
```

---

## 8.15 Interface สำหรับ Design Patterns

```typescript
// Observer Pattern
interface Observer<T> {
  update(data: T): void;
}

interface Observable<T> {
  subscribe(observer: Observer<T>): void;
  unsubscribe(observer: Observer<T>): void;
  notify(data: T): void;
}

// Strategy Pattern
interface SortStrategy<T> {
  sort(data: T[], compareFn: (a: T, b: T) => number): T[];
}

interface SearchStrategy<T, K> {
  search(data: T[], key: K): T[];
}

// Factory Pattern
interface ProductFactory<T> {
  create(config: Record<string, unknown>): T;
  canCreate(type: string): boolean;
}

// Decorator Pattern
interface TextTransformer {
  transform(text: string): string;
}

class UpperCaseTransformer implements TextTransformer {
  transform(text: string): string {
    return text.toUpperCase();
  }
}

class TrimTransformer implements TextTransformer {
  private inner: TextTransformer;
  
  constructor(inner: TextTransformer) {
    this.inner = inner;
  }
  
  transform(text: string): string {
    return this.inner.transform(text).trim();
  }
}

class PrefixTransformer implements TextTransformer {
  private inner: TextTransformer;
  private prefix: string;
  
  constructor(inner: TextTransformer, prefix: string) {
    this.inner = inner;
    this.prefix = prefix;
  }
  
  transform(text: string): string {
    return `${this.prefix}${this.inner.transform(text)}`;
  }
}

// การใช้งาน Decorator
const transformer = new PrefixTransformer(
  new TrimTransformer(new UpperCaseTransformer()),
  "[LOG] "
);

console.log(transformer.transform("  hello world  "));
// [LOG] HELLO WORLD
```

---

## 8.16 Interface สำหรับ Utility/Helper Types

```typescript
// Pagination Interface
interface PaginationMeta {
  page: number;
  pageSize: number;
  total: number;
  totalPages: number;
  hasNextPage: boolean;
  hasPrevPage: boolean;
}

interface PaginatedData<T> {
  items: T[];
  meta: PaginationMeta;
}

function createPagination<T>(
  data: T[],
  page: number,
  pageSize: number
): PaginatedData<T> {
  const total = data.length;
  const totalPages = Math.ceil(total / pageSize);
  const startIndex = (page - 1) * pageSize;
  const items = data.slice(startIndex, startIndex + pageSize);
  
  return {
    items,
    meta: {
      page,
      pageSize,
      total,
      totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1
    }
  };
}

// Sorting Interface
interface SortConfig {
  field: string;
  direction: "asc" | "desc";
}

interface FilterConfig {
  field: string;
  operator: "eq" | "ne" | "gt" | "gte" | "lt" | "lte" | "like" | "in";
  value: unknown;
}

interface QueryOptions {
  pagination?: {
    page: number;
    pageSize: number;
  };
  sorting?: SortConfig[];
  filters?: FilterConfig[];
  search?: string;
  fields?: string[]; // fields to select
  includes?: string[]; // relations to include
}
```

---

## 8.17 สรุปบทที่ 8

ในบทนี้เราได้เรียนรู้:

1. **Interface Declaration** - การประกาศ Interface พื้นฐาน
2. **Optional Properties** (`?`) - properties ที่ไม่บังคับ
3. **Readonly Properties** - properties ที่ไม่สามารถแก้ไขได้
4. **Function Types** - การกำหนด method signatures ใน interface
5. **Indexable Types** - index signature สำหรับ dynamic keys
6. **Class Implementation** - การใช้ `implements` กับ class
7. **Interface Extension** - การขยาย interface ด้วย `extends`
8. **Declaration Merging** - การรวม interface ที่มีชื่อเดียวกัน
9. **Generic Interfaces** - interface ที่มี type parameters
10. **Real-world Examples** - ตัวอย่างจริงจากระบบจริง

### คำแนะนำการใช้งาน Interface

```typescript
// ✅ ดี: ใช้ Interface สำหรับ object types
interface UserDTO {
  id: number;
  name: string;
  email: string;
}

// ✅ ดี: ใช้ Interface สำหรับ class contracts
interface Service {
  initialize(): Promise<void>;
  shutdown(): Promise<void>;
}

// ✅ ดี: ใช้ Generic Interface สำหรับ reusable patterns
interface Repository<T> {
  findById(id: number): Promise<T | null>;
  save(entity: T): Promise<T>;
}

// ✅ ดี: ใช้ Optional properties เมื่อ properties ไม่จำเป็น
interface Config {
  required: string;
  optional?: string;
}

// ✅ ดี: ใช้ Readonly สำหรับ immutable values
interface Constants {
  readonly MAX_RETRY: number;
  readonly TIMEOUT_MS: number;
}
```

---

## แบบฝึกหัดบทที่ 8

**แบบฝึกหัดที่ 1:** สร้าง Interface สำหรับระบบห้องสมุด (Library System) ที่มี Book, Author, Member และ BorrowRecord

**แบบฝึกหัดที่ 2:** สร้าง Generic Interface `Stack<T>` สำหรับโครงสร้างข้อมูล Stack พร้อม push, pop, peek, isEmpty, size

**แบบฝึกหัดที่ 3:** สร้าง Interface สำหรับ REST API client ที่รองรับ GET, POST, PUT, DELETE พร้อม Generic type สำหรับ request/response body

**แบบฝึกหัดที่ 4:** ออกแบบ Interface hierarchy สำหรับระบบแจ้งเตือน (Notification System) ที่รองรับ Email, SMS, Push notification

---

*จบบทที่ 8: Interfaces*
*บทต่อไป: Part 9 - Type Aliases*
