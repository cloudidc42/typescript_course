# ส่วนที่ 12: Generics พื้นฐาน (Generics Basics)

## บทนำ

Generics เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ TypeScript ช่วยให้เราเขียนโค้ดที่ยืดหยุ่น นำกลับมาใช้ใหม่ได้ และยังคงความปลอดภัยของ type ไว้ได้

---

## 12.1 Generics คืออะไร และทำไมต้องใช้?

### ปัญหาก่อนใช้ Generics

ลองนึกภาพว่าเราต้องการฟังก์ชันที่รับค่าใดๆ และคืนค่าเดิมกลับมา (identity function)

```typescript
// แบบไม่ใช้ Generics - ต้องเขียนหลายฟังก์ชัน
function identityString(arg: string): string {
  return arg;
}

function identityNumber(arg: number): number {
  return arg;
}

function identityBoolean(arg: boolean): boolean {
  return arg;
}

// ใช้งาน
const str = identityString("สวัสดี"); // type: string
const num = identityNumber(42);       // type: number
const bool = identityBoolean(true);   // type: boolean
```

ปัญหาคือเราต้องเขียนฟังก์ชันเดิมซ้ำหลายครั้ง หรือถ้าใช้ `any` ก็จะเสียการตรวจสอบ type:

```typescript
// ใช้ any - ไม่ดี เพราะเสีย type safety
function identityAny(arg: any): any {
  return arg;
}

const result = identityAny("สวัสดี");
// result มี type เป็น any - TypeScript ไม่รู้ว่าเป็น string
console.log(result.toUpperCase()); // ไม่มี error แต่อาจพัง runtime
```

### วิธีแก้ด้วย Generics

```typescript
// ใช้ Generic - เขียนครั้งเดียว ใช้ได้กับทุก type
function identity<T>(arg: T): T {
  return arg;
}

// TypeScript รู้ type อัตโนมัติ
const str = identity("สวัสดี");    // type: string
const num = identity(42);           // type: number
const bool = identity(true);        // type: boolean
const arr = identity([1, 2, 3]);    // type: number[]

console.log(str.toUpperCase());     // OK - TypeScript รู้ว่าเป็น string
console.log(num.toFixed(2));        // OK - TypeScript รู้ว่าเป็น number
```

### ประโยชน์ของ Generics

1. **นำกลับมาใช้ใหม่ได้** - เขียนครั้งเดียว ใช้ได้หลาย type
2. **Type Safety** - TypeScript ยังคงตรวจสอบ type ได้อย่างถูกต้อง
3. **ลดการเขียนโค้ดซ้ำ** - ไม่ต้องสร้างฟังก์ชันแยกสำหรับแต่ละ type
4. **IDE Support** - IntelliSense ยังทำงานได้ดี

```typescript
// ตัวอย่างเพิ่มเติม: ฟังก์ชันที่รับ array และคืนค่าแรก
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const firstNum = first([1, 2, 3]);      // type: number | undefined
const firstStr = first(["ก", "ข", "ค"]); // type: string | undefined
const firstBool = first([true, false]);  // type: boolean | undefined

// TypeScript รู้ type ที่แน่นอน
if (firstNum !== undefined) {
  console.log(firstNum.toFixed(2)); // OK
}
```

---

## 12.2 Generic Functions

### รูปแบบพื้นฐาน

```typescript
// รูปแบบ: function name<TypeParameter>(param: TypeParameter): TypeParameter
function wrap<T>(value: T): { value: T } {
  return { value };
}

const wrappedStr = wrap("TypeScript");    // { value: string }
const wrappedNum = wrap(2024);            // { value: number }
const wrappedObj = wrap({ name: "สมชาย" }); // { value: { name: string } }

console.log(wrappedStr.value.toUpperCase()); // "TYPESCRIPT"
console.log(wrappedNum.value.toFixed(0));    // "2024"
```

### การระบุ Type Parameter ตรงๆ

```typescript
// TypeScript อาจ infer ไม่ได้ในบางกรณี - ต้องระบุเอง
function createArray<T>(length: number, value: T): T[] {
  return Array(length).fill(value);
}

// TypeScript infer จาก value
const nums = createArray(3, 0);        // number[]
const strs = createArray(3, "");       // string[]

// ระบุ type parameter ตรงๆ
const mixedArr = createArray<string | number>(3, "test"); // (string | number)[]
```

### Arrow Function Generic

```typescript
// Arrow function กับ Generics
const getFirst = <T>(arr: T[]): T | undefined => arr[0];

const getLast = <T>(arr: T[]): T | undefined => arr[arr.length - 1];

// ใน .tsx files ต้องเพิ่ม trailing comma เพื่อไม่ให้ TypeScript สับสนกับ JSX
const identity = <T,>(value: T): T => value;

const nums = [1, 2, 3, 4, 5];
console.log(getFirst(nums)); // 1
console.log(getLast(nums));  // 5
```

### Generic Function ที่ซับซ้อนขึ้น

```typescript
// ฟังก์ชันสลับค่าใน tuple
function swap<T, U>(tuple: [T, U]): [U, T] {
  return [tuple[1], tuple[0]];
}

const swapped = swap([1, "hello"]); // ["hello", 1] - type: [string, number]
console.log(swapped); // ["hello", 1]

// ฟังก์ชัน zip สองอาร์เรย์
function zip<T, U>(arr1: T[], arr2: U[]): [T, U][] {
  const length = Math.min(arr1.length, arr2.length);
  return Array.from({ length }, (_, i) => [arr1[i], arr2[i]]);
}

const names = ["สมชาย", "สมหญิง", "สมปอง"];
const ages = [25, 30, 28];
const zipped = zip(names, ages); // [string, number][]
// [["สมชาย", 25], ["สมหญิง", 30], ["สมปอง", 28]]
```

### Generic ใน Method

```typescript
class Transformer {
  // Generic method ในคลาสปกติ
  transform<T, U>(value: T, fn: (val: T) => U): U {
    return fn(value);
  }

  // Generic method ที่รับ array
  transformAll<T, U>(values: T[], fn: (val: T) => U): U[] {
    return values.map(fn);
  }
}

const transformer = new Transformer();

const length = transformer.transform("TypeScript", (s) => s.length); // number
const doubled = transformer.transformAll([1, 2, 3], (n) => n * 2);    // number[]
const upper = transformer.transformAll(["hello", "world"], (s) => s.toUpperCase()); // string[]
```

---

## 12.3 Generic Interfaces

### Interface ที่ใช้ Generic

```typescript
// Interface พื้นฐาน
interface Box<T> {
  value: T;
  label?: string;
}

const numberBox: Box<number> = { value: 42 };
const stringBox: Box<string> = { value: "TypeScript", label: "ชื่อ" };
const boolBox: Box<boolean> = { value: true, label: "สถานะ" };

// ฟังก์ชันที่รับ Box<T>
function openBox<T>(box: Box<T>): T {
  return box.value;
}

const val = openBox(numberBox); // type: number
```

### Interface สำหรับ Pair

```typescript
interface Pair<T, U> {
  first: T;
  second: U;
}

const numStrPair: Pair<number, string> = {
  first: 1,
  second: "หนึ่ง"
};

const strBoolPair: Pair<string, boolean> = {
  first: "active",
  second: true
};

function makePair<T, U>(first: T, second: U): Pair<T, U> {
  return { first, second };
}

const pair = makePair(42, "สวัสดี"); // Pair<number, string>
```

### Generic Interface สำหรับ Response

```typescript
// รูปแบบที่ใช้บ่อยใน API
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: Date;
}

interface User {
  id: number;
  name: string;
  email: string;
}

interface Product {
  id: number;
  name: string;
  price: number;
}

// ฟังก์ชัน mock API
async function fetchUser(id: number): Promise<ApiResponse<User>> {
  // จำลองการเรียก API
  return {
    data: { id, name: "สมชาย ใจดี", email: "somchai@example.com" },
    status: 200,
    message: "สำเร็จ",
    timestamp: new Date()
  };
}

async function fetchProduct(id: number): Promise<ApiResponse<Product>> {
  return {
    data: { id, name: "สินค้า A", price: 199.99 },
    status: 200,
    message: "สำเร็จ",
    timestamp: new Date()
  };
}

// ใช้งาน
async function main() {
  const userResponse = await fetchUser(1);
  console.log(userResponse.data.name); // type: string - ถูกต้อง

  const productResponse = await fetchProduct(1);
  console.log(productResponse.data.price); // type: number - ถูกต้อง
}
```

### Generic Interface สำหรับ Collection

```typescript
interface Collection<T> {
  items: T[];
  add(item: T): void;
  remove(item: T): void;
  find(predicate: (item: T) => boolean): T | undefined;
  filter(predicate: (item: T) => boolean): T[];
  count(): number;
}

class SimpleCollection<T> implements Collection<T> {
  items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  remove(item: T): void {
    const index = this.items.indexOf(item);
    if (index > -1) {
      this.items.splice(index, 1);
    }
  }

  find(predicate: (item: T) => boolean): T | undefined {
    return this.items.find(predicate);
  }

  filter(predicate: (item: T) => boolean): T[] {
    return this.items.filter(predicate);
  }

  count(): number {
    return this.items.length;
  }
}

// ใช้งาน
const numbers = new SimpleCollection<number>();
numbers.add(1);
numbers.add(2);
numbers.add(3);

const found = numbers.find(n => n > 1); // type: number | undefined
const filtered = numbers.filter(n => n % 2 === 0); // type: number[]

const strings = new SimpleCollection<string>();
strings.add("สวัสดี");
strings.add("โลก");
```

---

## 12.4 Generic Classes

### คลาส Generic พื้นฐาน

```typescript
class Container<T> {
  private value: T;

  constructor(value: T) {
    this.value = value;
  }

  getValue(): T {
    return this.value;
  }

  setValue(newValue: T): void {
    this.value = newValue;
  }

  map<U>(fn: (value: T) => U): Container<U> {
    return new Container(fn(this.value));
  }
}

const numContainer = new Container(42);
const strContainer = new Container("TypeScript");

const doubledContainer = numContainer.map(n => n * 2); // Container<number>
const lengthContainer = strContainer.map(s => s.length); // Container<number>

console.log(numContainer.getValue());     // 42
console.log(strContainer.getValue());     // "TypeScript"
console.log(doubledContainer.getValue()); // 84
console.log(lengthContainer.getValue());  // 10
```

### คลาส Generic หลาย Type Parameters

```typescript
class KeyValuePair<K, V> {
  constructor(
    private key: K,
    private value: V
  ) {}

  getKey(): K {
    return this.key;
  }

  getValue(): V {
    return this.value;
  }

  swap(): KeyValuePair<V, K> {
    return new KeyValuePair(this.value, this.key);
  }

  toString(): string {
    return `${this.key}: ${this.value}`;
  }
}

const kvp = new KeyValuePair("name", "สมชาย");
console.log(kvp.getKey());    // "name" - type: string
console.log(kvp.getValue());  // "สมชาย" - type: string

const numKvp = new KeyValuePair(1, true);
const swapped = numKvp.swap(); // KeyValuePair<boolean, number>
```

---

## 12.5 Generic Constraints (extends)

### ทำไมต้องใช้ Constraints?

```typescript
// ปัญหา: ไม่รู้ว่า T มี property length หรือเปล่า
function getLength<T>(arg: T): number {
  // Error! Property 'length' does not exist on type 'T'
  // return arg.length;
  return 0; // ต้องทำอย่างนี้แทน
}

// แก้ด้วย constraint
function getLengthFixed<T extends { length: number }>(arg: T): number {
  return arg.length; // OK ตอนนี้
}

console.log(getLengthFixed("สวัสดี"));   // 6
console.log(getLengthFixed([1, 2, 3])); // 3
// Error! number ไม่มี length
// getLengthFixed(42);
```

### Constraint พื้นฐาน

```typescript
// Constraint ด้วย interface
interface HasName {
  name: string;
}

function greet<T extends HasName>(person: T): string {
  return `สวัสดี ${person.name}!`;
}

const user = { name: "สมชาย", age: 25 };
console.log(greet(user)); // "สวัสดี สมชาย!"

// Error! object ไม่มี name
// greet({ age: 25 });
```

### Constraint กับ Type

```typescript
// T ต้องเป็น string หรือ number
function compareValues<T extends string | number>(a: T, b: T): number {
  if (a < b) return -1;
  if (a > b) return 1;
  return 0;
}

console.log(compareValues(1, 2));       // -1
console.log(compareValues("a", "b"));  // -1
console.log(compareValues(5, 5));      // 0

// Error! boolean ไม่ใช่ string | number
// compareValues(true, false);
```

### Constraint ซับซ้อน

```typescript
interface Printable {
  print(): void;
}

interface Serializable {
  serialize(): string;
}

// T ต้องมีทั้ง print และ serialize
function processItem<T extends Printable & Serializable>(item: T): string {
  item.print();
  return item.serialize();
}

class Document implements Printable, Serializable {
  constructor(private content: string) {}

  print(): void {
    console.log(this.content);
  }

  serialize(): string {
    return JSON.stringify({ content: this.content });
  }
}

const doc = new Document("เนื้อหาเอกสาร");
const serialized = processItem(doc); // OK
```

### keyof Constraint

```typescript
// K ต้องเป็น key ของ T
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = {
  name: "สมชาย",
  age: 25,
  email: "somchai@example.com"
};

const name = getProperty(person, "name");  // type: string
const age = getProperty(person, "age");    // type: number

// Error! "phone" ไม่ใช่ key ของ person
// getProperty(person, "phone");

// ฟังก์ชัน setProperty ที่ type-safe
function setProperty<T, K extends keyof T>(obj: T, key: K, value: T[K]): void {
  obj[key] = value;
}

setProperty(person, "name", "สมหญิง");  // OK
setProperty(person, "age", 30);          // OK
// Error! ค่าไม่ตรง type
// setProperty(person, "age", "30");
```

---

## 12.6 Multiple Type Parameters

### ฟังก์ชันที่มีหลาย Type Parameters

```typescript
// ฟังก์ชัน merge objects
function merge<T, U>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 } as T & U;
}

const personInfo = { name: "สมชาย", age: 25 };
const contactInfo = { email: "somchai@example.com", phone: "081-234-5678" };

const fullInfo = merge(personInfo, contactInfo);
// type: { name: string; age: number; } & { email: string; phone: string; }

console.log(fullInfo.name);  // "สมชาย"
console.log(fullInfo.email); // "somchai@example.com"
```

### Map ที่ Type-Safe

```typescript
function mapObject<K extends string, T, U>(
  obj: Record<K, T>,
  fn: (value: T, key: K) => U
): Record<K, U> {
  const result = {} as Record<K, U>;
  for (const key in obj) {
    result[key] = fn(obj[key], key);
  }
  return result;
}

const prices = {
  apple: 10,
  banana: 5,
  cherry: 20
};

const discountedPrices = mapObject(prices, (price) => price * 0.9);
// type: Record<"apple" | "banana" | "cherry", number>
console.log(discountedPrices); // { apple: 9, banana: 4.5, cherry: 18 }
```

### Chaining Generic Functions

```typescript
// ฟังก์ชัน pipeline
function pipe<A, B, C>(
  value: A,
  fn1: (a: A) => B,
  fn2: (b: B) => C
): C {
  return fn2(fn1(value));
}

function pipe3<A, B, C, D>(
  value: A,
  fn1: (a: A) => B,
  fn2: (b: B) => C,
  fn3: (c: C) => D
): D {
  return fn3(fn2(fn1(value)));
}

const result = pipe(
  "TypeScript",
  (s) => s.length,     // string -> number
  (n) => n * 2         // number -> number
); // type: number
console.log(result); // 20

const result2 = pipe3(
  [1, 2, 3, 4, 5],
  (arr) => arr.filter(n => n % 2 === 0),  // number[] -> number[]
  (arr) => arr.map(n => n.toString()),      // number[] -> string[]
  (arr) => arr.join(", ")                   // string[] -> string
); // type: string
console.log(result2); // "2, 4"
```

---

## 12.7 Default Type Parameters

### Type Parameter ที่มีค่า Default

```typescript
// Default type parameter
interface PaginatedList<T, Meta = Record<string, unknown>> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  meta?: Meta;
}

// ใช้งานโดยไม่ระบุ Meta - ใช้ default
const userList: PaginatedList<User> = {
  items: [{ id: 1, name: "สมชาย", email: "a@b.com" }],
  total: 100,
  page: 1,
  pageSize: 10
};

// ระบุ Meta เอง
interface UserListMeta {
  adminCount: number;
  activeCount: number;
}

const userListWithMeta: PaginatedList<User, UserListMeta> = {
  items: [],
  total: 50,
  page: 1,
  pageSize: 10,
  meta: { adminCount: 5, activeCount: 45 }
};
```

### Default ที่ขึ้นกับ Parameter อื่น

```typescript
// Default ขึ้นกับ parameter ก่อนหน้า
interface Converter<From, To = From> {
  convert(value: From): To;
}

// ใช้ default - แปลง string -> string
const stringConverter: Converter<string> = {
  convert: (s) => s.toUpperCase()
};

// ระบุเอง - แปลง string -> number
const lengthConverter: Converter<string, number> = {
  convert: (s) => s.length
};

console.log(stringConverter.convert("hello")); // "HELLO"
console.log(lengthConverter.convert("hello")); // 5
```

### Generic Class กับ Default

```typescript
class EventEmitter<
  Events extends Record<string, unknown> = Record<string, unknown>
> {
  private listeners: Partial<{
    [K in keyof Events]: Array<(data: Events[K]) => void>;
  }> = {};

  on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): void {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners[event]?.forEach(listener => listener(data));
  }
}

// ใช้กับ type เฉพาะ
interface AppEvents {
  userLogin: { userId: number; timestamp: Date };
  userLogout: { userId: number };
  messageReceived: { from: string; content: string };
}

const emitter = new EventEmitter<AppEvents>();

emitter.on("userLogin", (data) => {
  console.log(`ผู้ใช้ ${data.userId} เข้าสู่ระบบเมื่อ ${data.timestamp}`);
});

emitter.emit("userLogin", { userId: 1, timestamp: new Date() });
```

---

## 12.8 Generic Utility Functions

### ฟังก์ชัน Array Utilities

```typescript
// groupBy: จัดกลุ่มสมาชิกใน array
function groupBy<T, K extends string | number>(
  arr: T[],
  getKey: (item: T) => K
): Record<K, T[]> {
  return arr.reduce((groups, item) => {
    const key = getKey(item);
    if (!groups[key]) {
      groups[key] = [];
    }
    groups[key].push(item);
    return groups;
  }, {} as Record<K, T[]>);
}

interface Student {
  name: string;
  grade: "A" | "B" | "C" | "D" | "F";
  score: number;
}

const students: Student[] = [
  { name: "สมชาย", grade: "A", score: 95 },
  { name: "สมหญิง", grade: "B", score: 82 },
  { name: "สมปอง", grade: "A", score: 91 },
  { name: "สมศรี", grade: "C", score: 71 }
];

const byGrade = groupBy(students, s => s.grade);
// { A: [...], B: [...], C: [...] }
```

### ฟังก์ชัน Object Utilities

```typescript
// pick: เลือกเฉพาะ properties ที่ต้องการ
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(key => {
    result[key] = obj[key];
  });
  return result;
}

// omit: ตัด properties ที่ไม่ต้องการออก
function omit<T, K extends keyof T>(obj: T, keys: K[]): Omit<T, K> {
  const result = { ...obj };
  keys.forEach(key => delete result[key]);
  return result as Omit<T, K>;
}

const user = {
  id: 1,
  name: "สมชาย",
  email: "somchai@example.com",
  password: "secret123",
  createdAt: new Date()
};

const publicInfo = pick(user, ["id", "name", "email"]);
// { id: 1, name: "สมชาย", email: "somchai@example.com" }

const safeUser = omit(user, ["password"]);
// ไม่มี password
```

### Async Utility Functions

```typescript
// retry: ลองใหม่อัตโนมัติเมื่อล้มเหลว
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  delay: number = 1000
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      console.log(`พยายามครั้งที่ ${attempt} ล้มเหลว: ${lastError.message}`);
      
      if (attempt < maxAttempts) {
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  
  throw lastError!;
}

// timeout: จำกัดเวลาการทำงาน
async function withTimeout<T>(
  fn: () => Promise<T>,
  timeoutMs: number
): Promise<T> {
  return Promise.race([
    fn(),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`หมดเวลา ${timeoutMs}ms`)), timeoutMs)
    )
  ]);
}

// ใช้งาน
async function fetchData(): Promise<string> {
  // สมมติเรียก API
  return "ข้อมูล";
}

// ลอง 3 ครั้ง timeout 5 วินาที
const data = await withTimeout(
  () => retry(fetchData, 3),
  5000
);
```

---

## 12.9 ตัวอย่างปฏิบัติ: Stack

### Stack Data Structure

```typescript
class Stack<T> {
  private items: T[] = [];

  // เพิ่มข้อมูลด้านบน
  push(item: T): void {
    this.items.push(item);
  }

  // นำข้อมูลออกจากด้านบน
  pop(): T | undefined {
    return this.items.pop();
  }

  // ดูข้อมูลด้านบนโดยไม่เอาออก
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  // ตรวจสอบว่าว่างเปล่า
  isEmpty(): boolean {
    return this.items.length === 0;
  }

  // จำนวนข้อมูล
  size(): number {
    return this.items.length;
  }

  // ล้างข้อมูล
  clear(): void {
    this.items = [];
  }

  // แปลงเป็น array
  toArray(): T[] {
    return [...this.items];
  }

  // ตรวจสอบว่ามีข้อมูลหรือไม่
  contains(item: T): boolean {
    return this.items.includes(item);
  }
}

// ใช้งาน Stack<number>
const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
numStack.push(3);

console.log(numStack.peek()); // 3
console.log(numStack.pop());  // 3
console.log(numStack.size()); // 2

// ใช้งาน Stack<string>
const strStack = new Stack<string>();
strStack.push("TypeScript");
strStack.push("JavaScript");
strStack.push("Python");

console.log(strStack.toArray()); // ["TypeScript", "JavaScript", "Python"]

// ใช้ Stack ตรวจสอบ bracket matching
function isBalanced(expression: string): boolean {
  const stack = new Stack<string>();
  const pairs: Record<string, string> = {
    ")": "(",
    "]": "[",
    "}": "{"
  };
  const openBrackets = new Set(["(", "[", "{"]);
  
  for (const char of expression) {
    if (openBrackets.has(char)) {
      stack.push(char);
    } else if (char in pairs) {
      if (stack.pop() !== pairs[char]) {
        return false;
      }
    }
  }
  
  return stack.isEmpty();
}

console.log(isBalanced("(()[]{})")); // true
console.log(isBalanced("([)]"));     // false
console.log(isBalanced("{[()]}"));   // true
```

---

## 12.10 ตัวอย่างปฏิบัติ: Queue

### Queue Data Structure

```typescript
class Queue<T> {
  private items: T[] = [];

  // เพิ่มข้อมูลด้านหลัง
  enqueue(item: T): void {
    this.items.push(item);
  }

  // นำข้อมูลออกจากด้านหน้า
  dequeue(): T | undefined {
    return this.items.shift();
  }

  // ดูข้อมูลด้านหน้าโดยไม่เอาออก
  front(): T | undefined {
    return this.items[0];
  }

  // ดูข้อมูลด้านหลัง
  back(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }

  clear(): void {
    this.items = [];
  }

  toArray(): T[] {
    return [...this.items];
  }
}

// Priority Queue
class PriorityQueue<T> {
  private items: Array<{ item: T; priority: number }> = [];

  enqueue(item: T, priority: number): void {
    const newItem = { item, priority };
    
    // แทรกในตำแหน่งที่ถูกต้องตาม priority
    let added = false;
    for (let i = 0; i < this.items.length; i++) {
      if (priority > this.items[i].priority) {
        this.items.splice(i, 0, newItem);
        added = true;
        break;
      }
    }
    
    if (!added) {
      this.items.push(newItem);
    }
  }

  dequeue(): T | undefined {
    return this.items.shift()?.item;
  }

  peek(): T | undefined {
    return this.items[0]?.item;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }
}

// ใช้งาน Queue
const taskQueue = new Queue<string>();
taskQueue.enqueue("งานที่ 1");
taskQueue.enqueue("งานที่ 2");
taskQueue.enqueue("งานที่ 3");

console.log(taskQueue.dequeue()); // "งานที่ 1"
console.log(taskQueue.front());   // "งานที่ 2"

// ใช้งาน Priority Queue
interface Task {
  name: string;
  urgency: string;
}

const priorityQueue = new PriorityQueue<Task>();
priorityQueue.enqueue({ name: "งานด่วนน้อย", urgency: "low" }, 1);
priorityQueue.enqueue({ name: "งานด่วนมาก", urgency: "high" }, 10);
priorityQueue.enqueue({ name: "งานปานกลาง", urgency: "medium" }, 5);

console.log(priorityQueue.dequeue()?.name); // "งานด่วนมาก"
console.log(priorityQueue.dequeue()?.name); // "งานปานกลาง"
```

---

## 12.11 ตัวอย่างปฏิบัติ: Generic Repository

### Repository Pattern

```typescript
// Interface พื้นฐาน
interface Entity {
  id: number;
}

// Generic Repository Interface
interface Repository<T extends Entity> {
  findById(id: number): Promise<T | undefined>;
  findAll(): Promise<T[]>;
  findBy(predicate: (item: T) => boolean): Promise<T[]>;
  create(item: Omit<T, "id">): Promise<T>;
  update(id: number, updates: Partial<Omit<T, "id">>): Promise<T | undefined>;
  delete(id: number): Promise<boolean>;
  count(): Promise<number>;
}

// In-memory Repository implementation
class InMemoryRepository<T extends Entity> implements Repository<T> {
  protected items: T[] = [];
  private nextId = 1;

  async findById(id: number): Promise<T | undefined> {
    return this.items.find(item => item.id === id);
  }

  async findAll(): Promise<T[]> {
    return [...this.items];
  }

  async findBy(predicate: (item: T) => boolean): Promise<T[]> {
    return this.items.filter(predicate);
  }

  async create(data: Omit<T, "id">): Promise<T> {
    const item = { ...data, id: this.nextId++ } as T;
    this.items.push(item);
    return item;
  }

  async update(id: number, updates: Partial<Omit<T, "id">>): Promise<T | undefined> {
    const index = this.items.findIndex(item => item.id === id);
    if (index === -1) return undefined;
    
    this.items[index] = { ...this.items[index], ...updates };
    return this.items[index];
  }

  async delete(id: number): Promise<boolean> {
    const index = this.items.findIndex(item => item.id === id);
    if (index === -1) return false;
    
    this.items.splice(index, 1);
    return true;
  }

  async count(): Promise<number> {
    return this.items.length;
  }
}

// Domain entities
interface UserEntity extends Entity {
  name: string;
  email: string;
  role: "admin" | "user" | "guest";
  createdAt: Date;
}

interface ProductEntity extends Entity {
  name: string;
  price: number;
  category: string;
  stock: number;
}

// Specific repositories
class UserRepository extends InMemoryRepository<UserEntity> {
  async findByEmail(email: string): Promise<UserEntity | undefined> {
    return this.items.find(user => user.email === email);
  }

  async findByRole(role: UserEntity["role"]): Promise<UserEntity[]> {
    return this.items.filter(user => user.role === role);
  }
}

class ProductRepository extends InMemoryRepository<ProductEntity> {
  async findByCategory(category: string): Promise<ProductEntity[]> {
    return this.items.filter(p => p.category === category);
  }

  async findInStock(): Promise<ProductEntity[]> {
    return this.items.filter(p => p.stock > 0);
  }

  async updateStock(id: number, quantity: number): Promise<boolean> {
    const product = await this.findById(id);
    if (!product) return false;
    
    await this.update(id, { stock: product.stock + quantity });
    return true;
  }
}

// ใช้งาน
async function demonstrateRepository() {
  const userRepo = new UserRepository();
  const productRepo = new ProductRepository();

  // สร้าง users
  const user1 = await userRepo.create({
    name: "สมชาย ใจดี",
    email: "somchai@example.com",
    role: "user",
    createdAt: new Date()
  });

  const admin = await userRepo.create({
    name: "ผู้ดูแล",
    email: "admin@example.com",
    role: "admin",
    createdAt: new Date()
  });

  // สร้าง products
  const product = await productRepo.create({
    name: "สินค้า A",
    price: 199.99,
    category: "อิเล็กทรอนิกส์",
    stock: 100
  });

  // ค้นหา
  const foundUser = await userRepo.findByEmail("somchai@example.com");
  console.log(`พบผู้ใช้: ${foundUser?.name}`);

  const admins = await userRepo.findByRole("admin");
  console.log(`จำนวน admin: ${admins.length}`);

  // อัปเดต
  await userRepo.update(user1.id, { name: "สมชาย แก้ไข" });

  // ลบ
  const deleted = await userRepo.delete(user1.id);
  console.log(`ลบสำเร็จ: ${deleted}`);

  // นับ
  const total = await userRepo.count();
  console.log(`จำนวนผู้ใช้ทั้งหมด: ${total}`);
}

demonstrateRepository();
```

---

## 12.12 Generic ใน Real-World Scenarios

### Form Validation

```typescript
type ValidationResult<T> = {
  valid: boolean;
  data?: T;
  errors: Partial<Record<keyof T, string>>;
};

type Validator<T> = {
  [K in keyof T]?: (value: T[K]) => string | null;
};

function validate<T extends object>(
  data: T,
  validators: Validator<T>
): ValidationResult<T> {
  const errors: Partial<Record<keyof T, string>> = {};
  let valid = true;

  for (const key in validators) {
    const validator = validators[key];
    if (validator) {
      const error = validator(data[key]);
      if (error) {
        errors[key] = error;
        valid = false;
      }
    }
  }

  return {
    valid,
    data: valid ? data : undefined,
    errors
  };
}

// ใช้งาน
interface RegistrationForm {
  username: string;
  email: string;
  password: string;
  age: number;
}

const formData: RegistrationForm = {
  username: "somchai",
  email: "invalid-email",
  password: "123",
  age: 17
};

const result = validate(formData, {
  username: (v) => v.length < 3 ? "ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร" : null,
  email: (v) => !v.includes("@") ? "อีเมลไม่ถูกต้อง" : null,
  password: (v) => v.length < 8 ? "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร" : null,
  age: (v) => v < 18 ? "ต้องมีอายุ 18 ปีขึ้นไป" : null
});

console.log(result.valid);        // false
console.log(result.errors.email); // "อีเมลไม่ถูกต้อง"
```

### State Machine

```typescript
type Transition<States extends string, Events extends string> = {
  [S in States]?: {
    [E in Events]?: States;
  };
};

class StateMachine<States extends string, Events extends string> {
  private currentState: States;
  private transitions: Transition<States, Events>;
  private listeners: Array<(from: States, event: Events, to: States) => void> = [];

  constructor(
    initialState: States,
    transitions: Transition<States, Events>
  ) {
    this.currentState = initialState;
    this.transitions = transitions;
  }

  getState(): States {
    return this.currentState;
  }

  dispatch(event: Events): boolean {
    const stateTransitions = this.transitions[this.currentState];
    if (!stateTransitions) return false;

    const nextState = stateTransitions[event];
    if (!nextState) return false;

    const from = this.currentState;
    this.currentState = nextState;
    this.listeners.forEach(l => l(from, event, nextState));
    return true;
  }

  onTransition(listener: (from: States, event: Events, to: States) => void): void {
    this.listeners.push(listener);
  }
}

// ตัวอย่าง: Order State Machine
type OrderState = "pending" | "confirmed" | "processing" | "shipped" | "delivered" | "cancelled";
type OrderEvent = "confirm" | "process" | "ship" | "deliver" | "cancel";

const orderMachine = new StateMachine<OrderState, OrderEvent>(
  "pending",
  {
    pending: { confirm: "confirmed", cancel: "cancelled" },
    confirmed: { process: "processing", cancel: "cancelled" },
    processing: { ship: "shipped" },
    shipped: { deliver: "delivered" }
  }
);

orderMachine.onTransition((from, event, to) => {
  console.log(`สถานะเปลี่ยน: ${from} --[${event}]--> ${to}`);
});

orderMachine.dispatch("confirm");  // pending -> confirmed
orderMachine.dispatch("process"); // confirmed -> processing
orderMachine.dispatch("ship");    // processing -> shipped
orderMachine.dispatch("deliver"); // shipped -> delivered
console.log(orderMachine.getState()); // "delivered"
```

---

## 12.13 สรุปและแนวทางปฏิบัติที่ดี

### เมื่อควรใช้ Generics

```typescript
// ✅ ดี - ใช้ Generics เมื่อ logic เหมือนกันแต่ต่าง type
function sortArray<T>(arr: T[], compareFn: (a: T, b: T) => number): T[] {
  return [...arr].sort(compareFn);
}

// ✅ ดี - ใช้ Generics เพื่อรักษา type relationship
function mapValues<T, U>(obj: Record<string, T>, fn: (val: T) => U): Record<string, U> {
  const result: Record<string, U> = {};
  for (const key in obj) {
    result[key] = fn(obj[key]);
  }
  return result;
}

// ❌ ไม่ดี - ใช้ Generics โดยไม่จำเป็น
function printValue<T>(value: T): void { // ควรใช้ unknown หรือ any แทน
  console.log(value);
}

// ✅ ดีกว่า
function printValue(value: unknown): void {
  console.log(value);
}
```

### ชื่อ Type Parameter ที่แนะนำ

```typescript
// T - Type ทั่วไป
// K - Key
// V - Value
// E - Element
// R - Return/Result
// S - State
// U, W - Type เพิ่มเติม

// ตัวอย่าง
function transform<T, R>(value: T, fn: (val: T) => R): R {
  return fn(value);
}

interface Dictionary<K extends string, V> {
  [key: string]: V;
  get(key: K): V | undefined;
  set(key: K, value: V): void;
}
```

### Constraint ที่ดี

```typescript
// ✅ ระบุ constraint ที่ชัดเจน
function processArray<T extends { id: number; name: string }>(items: T[]): string[] {
  return items.map(item => `${item.id}: ${item.name}`);
}

// ✅ ใช้ keyof ให้ type-safe
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map(item => item[key]);
}

const users = [
  { id: 1, name: "สมชาย", age: 25 },
  { id: 2, name: "สมหญิง", age: 30 }
];

const names = pluck(users, "name"); // string[]
const ages = pluck(users, "age");   // number[]
// Error: pluck(users, "phone"); // ไม่มี property "phone"
```

---

## แบบฝึกหัด

1. สร้าง Generic `LinkedList<T>` class ที่มี method `push`, `pop`, `shift`, `unshift`, `toArray`
2. สร้าง `Cache<K, V>` class ที่มี TTL (time-to-live) และ method `get`, `set`, `has`, `delete`
3. สร้างฟังก์ชัน `deepClone<T>(obj: T): T` ที่ clone object อย่าง deep
4. สร้าง Generic `Result<T, E>` type (คล้าย Rust's Result) พร้อม method `map`, `mapError`, `unwrap`
5. สร้าง `Observable<T>` class พื้นฐานที่ใช้ subscriber pattern

---

## สรุปบทนี้

ในบทนี้เราได้เรียนรู้:

- **Generics** คือการสร้างโค้ดที่ทำงานกับหลาย type โดยรักษา type safety
- **Generic Functions** รับ type parameter แล้วใช้ภายในฟังก์ชัน
- **Generic Interfaces** กำหนด contract ที่ใช้ได้กับหลาย type
- **Generic Classes** สร้าง class ที่ยืดหยุ่นและนำกลับมาใช้ได้
- **Constraints** จำกัดว่า type parameter ต้องมี property หรือ method อะไร
- **Multiple Type Parameters** ใช้เมื่อต้องการ type หลายตัว
- **Default Type Parameters** กำหนดค่า default เมื่อไม่ระบุ type

บทถัดไปจะเรียนรู้ Advanced Generics เช่น Mapped Types, Conditional Types และ Utility Types
