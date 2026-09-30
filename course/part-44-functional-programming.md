# ตอนที่ 44: Functional Programming ใน TypeScript

## บทนำ

Functional Programming (FP) คือ paradigm การเขียนโปรแกรมที่เน้นการใช้ฟังก์ชันเป็นหลัก โดยหลีกเลี่ยงการเปลี่ยนแปลง state และ side effects TypeScript รองรับ FP ได้อย่างยอดเยี่ยมเนื่องจากมีระบบ type ที่แข็งแกร่ง ทำให้เราสามารถเขียนโค้ดที่ปลอดภัยและคาดเดาได้

---

## 44.1 Pure Functions และ Immutability

### Pure Functions คืออะไร

Pure Function คือฟังก์ชันที่:
1. ให้ผลลัพธ์เดิมเสมอเมื่อได้รับ input เดิม
2. ไม่มี side effects (ไม่เปลี่ยนแปลง state ภายนอก)

```typescript
// ❌ Impure Function - มี side effect
let counter = 0;
function incrementCounter(): number {
  counter++; // เปลี่ยนแปลง external state
  return counter;
}

// ✅ Pure Function - ไม่มี side effect
function add(a: number, b: number): number {
  return a + b;
}

// ✅ Pure Function อีกตัวอย่าง
function multiply(a: number, b: number): number {
  return a * b;
}

console.log(add(2, 3));      // 5 เสมอ
console.log(multiply(4, 5)); // 20 เสมอ
```

### ตัวอย่างการทดสอบ Pure Function

```typescript
// Pure function ทดสอบได้ง่าย
function formatName(firstName: string, lastName: string): string {
  return `${firstName} ${lastName}`.trim();
}

// ทดสอบ
console.log(formatName("สมชาย", "ใจดี"));   // "สมชาย ใจดี"
console.log(formatName("สมหญิง", ""));       // "สมหญิง"
console.log(formatName("", "นามสกุล"));     // "นามสกุล"

// ทดสอบซ้ำ - ได้ผลเหมือนเดิม
console.log(formatName("สมชาย", "ใจดี"));   // "สมชาย ใจดี" เสมอ
```

### Immutability ใน TypeScript

```typescript
// ❌ Mutable approach
const user = {
  name: "สมชาย",
  age: 25
};
user.age = 26; // เปลี่ยนแปลง object โดยตรง

// ✅ Immutable approach
const originalUser = {
  name: "สมชาย",
  age: 25
};

// สร้าง object ใหม่แทนการแก้ไขของเดิม
const updatedUser = { ...originalUser, age: 26 };

console.log(originalUser); // { name: "สมชาย", age: 25 } ไม่เปลี่ยน
console.log(updatedUser);  // { name: "สมชาย", age: 26 }
```

### readonly กับ Immutability

```typescript
// ใช้ readonly เพื่อป้องกันการแก้ไข
interface User {
  readonly id: number;
  readonly name: string;
  readonly email: string;
}

const user: User = {
  id: 1,
  name: "สมชาย",
  email: "somchai@example.com"
};

// user.name = "สมหญิง"; // Error: Cannot assign to 'name'

// Readonly Array
const numbers: readonly number[] = [1, 2, 3, 4, 5];
// numbers.push(6); // Error: Property 'push' does not exist on type 'readonly number[]'

// ใช้ Readonly utility type
type ReadonlyUser = Readonly<{
  id: number;
  name: string;
  scores: number[];
}>;

const readonlyUser: ReadonlyUser = {
  id: 1,
  name: "สมชาย",
  scores: [95, 87, 92]
};
// readonlyUser.name = "test"; // Error
```

### Object.freeze สำหรับ Deep Immutability

```typescript
// Object.freeze สำหรับ runtime immutability
function deepFreeze<T>(obj: T): Readonly<T> {
  Object.getOwnPropertyNames(obj).forEach(name => {
    const value = (obj as any)[name];
    if (typeof value === 'object' && value !== null) {
      deepFreeze(value);
    }
  });
  return Object.freeze(obj);
}

interface Config {
  database: {
    host: string;
    port: number;
  };
  api: {
    key: string;
    timeout: number;
  };
}

const config: Config = deepFreeze({
  database: {
    host: "localhost",
    port: 5432
  },
  api: {
    key: "secret-key",
    timeout: 3000
  }
});

// config.database.host = "newhost"; // จะ throw error ใน runtime
```

### Immutable Array Operations

```typescript
// ❌ Mutable array operations
const arr1 = [1, 2, 3];
arr1.push(4);    // แก้ไข arr1 โดยตรง
arr1.splice(0, 1); // แก้ไข arr1 โดยตรง

// ✅ Immutable array operations
const original = [1, 2, 3];

// เพิ่มต่อท้าย
const withFour = [...original, 4];

// เพิ่มต้น
const withZero = [0, ...original];

// ลบ element ที่ index 1
const withoutSecond = original.filter((_, index) => index !== 1);

// อัปเดต element
const updated = original.map((item, index) => 
  index === 1 ? item * 10 : item
);

console.log(original);     // [1, 2, 3] ไม่เปลี่ยน
console.log(withFour);     // [1, 2, 3, 4]
console.log(withZero);     // [0, 1, 2, 3]
console.log(withoutSecond);// [1, 3]
console.log(updated);      // [1, 20, 3]
```

---

## 44.2 Higher-Order Functions

Higher-order function คือฟังก์ชันที่รับฟังก์ชันเป็น argument หรือ return ฟังก์ชัน

### การรับฟังก์ชันเป็น Argument

```typescript
// ฟังก์ชันที่รับ callback
function applyOperation(
  numbers: number[], 
  operation: (n: number) => number
): number[] {
  return numbers.map(operation);
}

const double = (n: number) => n * 2;
const square = (n: number) => n * n;
const addTen = (n: number) => n + 10;

const numbers = [1, 2, 3, 4, 5];
console.log(applyOperation(numbers, double));  // [2, 4, 6, 8, 10]
console.log(applyOperation(numbers, square));  // [1, 4, 9, 16, 25]
console.log(applyOperation(numbers, addTen));  // [11, 12, 13, 14, 15]
```

### การ Return ฟังก์ชัน

```typescript
// ฟังก์ชันที่ return ฟังก์ชัน
function createMultiplier(factor: number): (n: number) => number {
  return (n: number) => n * factor;
}

const triple = createMultiplier(3);
const quintuple = createMultiplier(5);

console.log(triple(4));    // 12
console.log(quintuple(4)); // 20

// ตัวอย่างที่ซับซ้อนขึ้น
function createGreeter(greeting: string): (name: string) => string {
  return (name: string) => `${greeting}, ${name}!`;
}

const sayHello = createGreeter("สวัสดี");
const sayHi = createGreeter("หวัดดี");

console.log(sayHello("สมชาย")); // "สวัสดี, สมชาย!"
console.log(sayHi("สมหญิง"));   // "หวัดดี, สมหญิง!"
```

### Built-in Higher-Order Functions

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

const products: Product[] = [
  { id: 1, name: "โทรศัพท์", price: 15000, category: "อิเล็กทรอนิกส์", inStock: true },
  { id: 2, name: "เสื้อ", price: 500, category: "เสื้อผ้า", inStock: true },
  { id: 3, name: "แล็ปท็อป", price: 35000, category: "อิเล็กทรอนิกส์", inStock: false },
  { id: 4, name: "กางเกง", price: 700, category: "เสื้อผ้า", inStock: true },
  { id: 5, name: "หูฟัง", price: 2000, category: "อิเล็กทรอนิกส์", inStock: true },
];

// map - แปลงข้อมูล
const productNames = products.map(p => p.name);
console.log(productNames); // ["โทรศัพท์", "เสื้อ", ...]

// filter - กรองข้อมูล
const inStockProducts = products.filter(p => p.inStock);
console.log(inStockProducts.length); // 4

// reduce - รวมข้อมูล
const totalValue = products
  .filter(p => p.inStock)
  .reduce((sum, p) => sum + p.price, 0);
console.log(totalValue); // 18200

// find - ค้นหาตัวแรก
const laptop = products.find(p => p.name === "แล็ปท็อป");
console.log(laptop?.price); // 35000

// some - มีอย่างน้อยหนึ่งตัวที่ตรงเงื่อนไข
const hasExpensive = products.some(p => p.price > 30000);
console.log(hasExpensive); // true

// every - ทุกตัวตรงเงื่อนไข
const allInStock = products.every(p => p.inStock);
console.log(allInStock); // false

// flatMap - map แล้ว flatten
const tags = [
  { product: "โทรศัพท์", tags: ["mobile", "tech"] },
  { product: "เสื้อ", tags: ["fashion", "clothing"] },
];
const allTags = tags.flatMap(t => t.tags);
console.log(allTags); // ["mobile", "tech", "fashion", "clothing"]
```

### Custom Higher-Order Functions

```typescript
// การสร้าง memoize function
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();
  
  return ((...args: Parameters<T>) => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      console.log("Cache hit!");
      return cache.get(key);
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

function expensiveCalculation(n: number): number {
  console.log(`Computing for ${n}...`);
  // จำลองการคำนวณที่ใช้เวลานาน
  let result = 0;
  for (let i = 0; i <= n; i++) {
    result += i;
  }
  return result;
}

const memoizedCalc = memoize(expensiveCalculation);
console.log(memoizedCalc(100)); // Computing for 100... 5050
console.log(memoizedCalc(100)); // Cache hit! 5050
console.log(memoizedCalc(200)); // Computing for 200... 20100
```

---

## 44.3 Function Composition

Function Composition คือการรวมฟังก์ชันหลายๆ ตัวเข้าด้วยกัน

### Compose และ Pipe

```typescript
// compose - ทำงานจากขวาไปซ้าย
function compose<T>(...fns: Array<(arg: T) => T>): (arg: T) => T {
  return (arg: T) => fns.reduceRight((acc, fn) => fn(acc), arg);
}

// pipe - ทำงานจากซ้ายไปขวา
function pipe<T>(...fns: Array<(arg: T) => T>): (arg: T) => T {
  return (arg: T) => fns.reduce((acc, fn) => fn(acc), arg);
}

// ฟังก์ชันพื้นฐาน
const double = (n: number) => n * 2;
const addOne = (n: number) => n + 1;
const square = (n: number) => n * n;

// compose: square(addOne(double(3))) = square(addOne(6)) = square(7) = 49
const composedFn = compose(square, addOne, double);
console.log(composedFn(3)); // 49

// pipe: square(addOne(double(3))) แต่เขียนตามลำดับ
const pipedFn = pipe(double, addOne, square);
console.log(pipedFn(3)); // 49
```

### Generic Compose Function

```typescript
// Generic compose ที่รองรับ type ต่างกัน
type Fn<A, B> = (a: A) => B;

function composeTwo<A, B, C>(f: Fn<B, C>, g: Fn<A, B>): Fn<A, C> {
  return (a: A) => f(g(a));
}

// ตัวอย่างการใช้งาน
const toString = (n: number): string => n.toString();
const addExclamation = (s: string): string => s + "!";
const getLength = (s: string): number => s.length;

const numberToLength = composeTwo(getLength, toString);
console.log(numberToLength(12345)); // 5

const numberToExclamation = composeTwo(addExclamation, toString);
console.log(numberToExclamation(42)); // "42!"
```

### การใช้ Composition กับ Data Transformation

```typescript
interface RawData {
  name: string;
  email: string;
  age: string; // string จาก API
}

interface ProcessedData {
  name: string;
  email: string;
  age: number;
  isAdult: boolean;
  displayName: string;
}

// ฟังก์ชันแต่ละขั้นตอน
const parseName = (data: RawData): RawData & { name: string } => ({
  ...data,
  name: data.name.trim().toLowerCase()
});

const parseAge = (data: RawData): RawData & { parsedAge: number } => ({
  ...data,
  parsedAge: parseInt(data.age, 10)
});

const checkAdult = (data: any): any => ({
  ...data,
  isAdult: data.parsedAge >= 18
});

const formatDisplayName = (data: any): any => ({
  ...data,
  displayName: data.name.charAt(0).toUpperCase() + data.name.slice(1)
});

// รวมฟังก์ชันทั้งหมด
const processData = pipe(
  parseName,
  parseAge,
  checkAdult,
  formatDisplayName
);

const rawData: RawData = {
  name: "  สมชาย  ",
  email: "somchai@example.com",
  age: "25"
};

const result = processData(rawData);
console.log(result);
```

---

## 44.4 Currying

Currying คือการแปลงฟังก์ชันที่รับหลาย argument เป็นฟังก์ชันที่รับ argument ทีละตัว

### Basic Currying

```typescript
// ฟังก์ชันปกติ
function add(a: number, b: number): number {
  return a + b;
}

// Curried version
function curriedAdd(a: number): (b: number) => number {
  return (b: number) => a + b;
}

const addFive = curriedAdd(5);
const addTen = curriedAdd(10);

console.log(addFive(3));  // 8
console.log(addFive(7));  // 12
console.log(addTen(5));   // 15

// สามารถเรียกแบบ partial application
console.log(curriedAdd(2)(3)); // 5
```

### Generic Curry Function

```typescript
// Type สำหรับ curried function
type Curry<T extends (...args: any[]) => any> = 
  Parameters<T> extends [infer First, ...infer Rest]
    ? Rest extends []
      ? T
      : (arg: First) => Curry<(...args: Rest) => ReturnType<T>>
    : never;

// Curry implementation
function curry<T extends (...args: any[]) => any>(fn: T): Curry<T> {
  return function curried(...args: any[]): any {
    if (args.length >= fn.length) {
      return fn(...args);
    }
    return function(...moreArgs: any[]) {
      return curried(...args, ...moreArgs);
    };
  } as Curry<T>;
}

// ตัวอย่างการใช้งาน
function volume(width: number, height: number, depth: number): number {
  return width * height * depth;
}

const curriedVolume = curry(volume);

// เรียกแบบต่างๆ
console.log(curriedVolume(2)(3)(4));    // 24
console.log(curriedVolume(2, 3)(4));    // 24
console.log(curriedVolume(2)(3, 4));    // 24
console.log(curriedVolume(2, 3, 4));    // 24

const baseArea = curriedVolume(5)(10);  // รอรับ depth
console.log(baseArea(3));  // 150
```

### Practical Currying Examples

```typescript
// Curried filter
const filter = curry(
  <T>(predicate: (item: T) => boolean, arr: T[]): T[] => 
    arr.filter(predicate)
);

// Curried map
const map = curry(
  <T, U>(fn: (item: T) => U, arr: T[]): U[] => 
    arr.map(fn)
);

// Curried reduce
const reduce = curry(
  <T, U>(fn: (acc: U, item: T) => U, initial: U, arr: T[]): U =>
    arr.reduce(fn, initial)
);

// การใช้งาน
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

const isEven = (n: number) => n % 2 === 0;
const filterEven = filter(isEven);

const double = (n: number) => n * 2;
const mapDouble = map(double);

const sum = (acc: number, n: number) => acc + n;
const sumAll = reduce(sum, 0);

console.log(filterEven(numbers));      // [2, 4, 6, 8, 10]
console.log(mapDouble([1, 2, 3]));     // [2, 4, 6]
console.log(sumAll([1, 2, 3, 4, 5])); // 15

// ใช้ร่วมกัน
const sumOfEvenDoubles = pipe(
  filterEven,
  mapDouble,
  sumAll
)(numbers);
console.log(sumOfEvenDoubles); // 60
```

---

## 44.5 Partial Application

Partial Application คือการสร้างฟังก์ชันใหม่โดยกำหนด argument บางตัวไว้ก่อน

```typescript
// partial function
function partial<T extends (...args: any[]) => any>(
  fn: T,
  ...partialArgs: Partial<Parameters<T>>
): (...remainingArgs: any[]) => ReturnType<T> {
  return (...remainingArgs: any[]) => {
    const allArgs = [...partialArgs, ...remainingArgs] as Parameters<T>;
    return fn(...allArgs);
  };
}

// ตัวอย่าง
function createUrl(protocol: string, domain: string, path: string): string {
  return `${protocol}://${domain}/${path}`;
}

const createHttpUrl = partial(createUrl, "http");
const createHttpsUrl = partial(createUrl, "https");
const createLocalUrl = partial(createUrl, "http", "localhost:3000");

console.log(createHttpUrl("example.com", "users"));      // http://example.com/users
console.log(createHttpsUrl("api.example.com", "data"));   // https://api.example.com/data
console.log(createLocalUrl("api/users"));                  // http://localhost:3000/api/users
```

### Partial Application กับ Event Handlers

```typescript
// ตัวอย่างกับ DOM events
type EventHandler<T> = (event: T, data: string) => void;

function createHandler<T>(handler: EventHandler<T>, data: string): (event: T) => void {
  return (event: T) => handler(event, data);
}

const logEvent: EventHandler<MouseEvent> = (event, data) => {
  console.log(`Event: ${event.type}, Data: ${data}`);
};

// สร้าง specific handlers
const handleLogin = createHandler(logEvent, "user-login");
const handleLogout = createHandler(logEvent, "user-logout");

// handleLogin และ handleLogout พร้อมใช้เป็น event listener
```

---

## 44.6 Functors

Functor คือ container ที่มี map function ที่รักษา structure ของ container ไว้

### Custom Functor

```typescript
// Box Functor
class Box<T> {
  constructor(private readonly value: T) {}
  
  static of<T>(value: T): Box<T> {
    return new Box(value);
  }
  
  map<U>(fn: (value: T) => U): Box<U> {
    return new Box(fn(this.value));
  }
  
  fold<U>(fn: (value: T) => U): U {
    return fn(this.value);
  }
  
  getOrElse(defaultValue: T): T {
    return this.value ?? defaultValue;
  }
  
  toString(): string {
    return `Box(${this.value})`;
  }
}

// การใช้งาน
const result = Box.of(5)
  .map(n => n * 2)        // Box(10)
  .map(n => n + 1)        // Box(11)
  .map(n => n.toString()) // Box("11")
  .fold(s => s + "!");    // "11!"

console.log(result); // "11!"

// ตัวอย่างกับ string
const formattedName = Box.of("  สมชาย ใจดี  ")
  .map(s => s.trim())
  .map(s => s.toUpperCase())
  .fold(s => s);

console.log(formattedName); // "สมชาย ใจดี"
```

### Array เป็น Functor

```typescript
// Array ก็เป็น Functor ที่มี map
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(n => n * 2);
const strings = doubled.map(n => `Number: ${n}`);

console.log(strings);
// ["Number: 2", "Number: 4", "Number: 6", "Number: 8", "Number: 10"]

// Functor laws
// 1. Identity law: fa.map(x => x) === fa
const identity = <T>(x: T): T => x;
console.log(JSON.stringify(numbers.map(identity)) === JSON.stringify(numbers)); // true

// 2. Composition law: fa.map(f).map(g) === fa.map(x => g(f(x)))
const addOne = (n: number) => n + 1;
const double = (n: number) => n * 2;

const result1 = numbers.map(addOne).map(double);
const result2 = numbers.map(n => double(addOne(n)));
console.log(JSON.stringify(result1) === JSON.stringify(result2)); // true
```

---

## 44.7 Monads

Monad เป็น pattern สำหรับจัดการ computation ที่มี context เช่น null safety, error handling, async operations

### Maybe Monad

```typescript
// Maybe Monad สำหรับ null safety
abstract class Maybe<T> {
  abstract isNothing(): boolean;
  abstract map<U>(fn: (value: T) => U): Maybe<U>;
  abstract flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U>;
  abstract getOrElse(defaultValue: T): T;
  abstract fold<U>(onNothing: () => U, onJust: (value: T) => U): U;
  
  static of<T>(value: T | null | undefined): Maybe<T> {
    if (value === null || value === undefined) {
      return new Nothing<T>();
    }
    return new Just<T>(value);
  }
  
  static fromNullable<T>(value: T | null | undefined): Maybe<T> {
    return Maybe.of(value);
  }
}

class Just<T> extends Maybe<T> {
  constructor(private readonly value: T) {
    super();
  }
  
  isNothing(): boolean {
    return false;
  }
  
  map<U>(fn: (value: T) => U): Maybe<U> {
    return Maybe.of(fn(this.value));
  }
  
  flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U> {
    return fn(this.value);
  }
  
  getOrElse(_defaultValue: T): T {
    return this.value;
  }
  
  fold<U>(_onNothing: () => U, onJust: (value: T) => U): U {
    return onJust(this.value);
  }
  
  toString(): string {
    return `Just(${this.value})`;
  }
}

class Nothing<T> extends Maybe<T> {
  isNothing(): boolean {
    return true;
  }
  
  map<U>(_fn: (value: T) => U): Maybe<U> {
    return new Nothing<U>();
  }
  
  flatMap<U>(_fn: (value: T) => Maybe<U>): Maybe<U> {
    return new Nothing<U>();
  }
  
  getOrElse(defaultValue: T): T {
    return defaultValue;
  }
  
  fold<U>(onNothing: () => U, _onJust: (value: T) => U): U {
    return onNothing();
  }
  
  toString(): string {
    return "Nothing";
  }
}

// ตัวอย่างการใช้งาน Maybe
interface UserProfile {
  id: number;
  name: string;
  address?: {
    city?: string;
    zipCode?: string;
  };
}

function getUserCity(user: UserProfile | null): string {
  return Maybe.of(user)
    .flatMap(u => Maybe.of(u.address))
    .flatMap(addr => Maybe.of(addr.city))
    .getOrElse("ไม่ระบุเมือง");
}

const userWithCity: UserProfile = {
  id: 1,
  name: "สมชาย",
  address: { city: "กรุงเทพ" }
};

const userWithoutCity: UserProfile = {
  id: 2,
  name: "สมหญิง"
};

console.log(getUserCity(userWithCity));    // "กรุงเทพ"
console.log(getUserCity(userWithoutCity)); // "ไม่ระบุเมือง"
console.log(getUserCity(null));            // "ไม่ระบุเมือง"
```

### Either Monad

```typescript
// Either Monad สำหรับ error handling
abstract class Either<L, R> {
  abstract isLeft(): boolean;
  abstract isRight(): boolean;
  abstract map<U>(fn: (value: R) => U): Either<L, U>;
  abstract flatMap<U>(fn: (value: R) => Either<L, U>): Either<L, U>;
  abstract getOrElse(defaultValue: R): R;
  abstract fold<U>(onLeft: (error: L) => U, onRight: (value: R) => U): U;
  
  static left<L, R>(value: L): Either<L, R> {
    return new Left<L, R>(value);
  }
  
  static right<L, R>(value: R): Either<L, R> {
    return new Right<L, R>(value);
  }
  
  static tryCatch<L, R>(
    fn: () => R,
    onError: (error: unknown) => L
  ): Either<L, R> {
    try {
      return Either.right(fn());
    } catch (error) {
      return Either.left(onError(error));
    }
  }
}

class Left<L, R> extends Either<L, R> {
  constructor(private readonly error: L) {
    super();
  }
  
  isLeft(): boolean { return true; }
  isRight(): boolean { return false; }
  
  map<U>(_fn: (value: R) => U): Either<L, U> {
    return new Left<L, U>(this.error);
  }
  
  flatMap<U>(_fn: (value: R) => Either<L, U>): Either<L, U> {
    return new Left<L, U>(this.error);
  }
  
  getOrElse(defaultValue: R): R {
    return defaultValue;
  }
  
  fold<U>(onLeft: (error: L) => U, _onRight: (value: R) => U): U {
    return onLeft(this.error);
  }
  
  toString(): string {
    return `Left(${this.error})`;
  }
}

class Right<L, R> extends Either<L, R> {
  constructor(private readonly value: R) {
    super();
  }
  
  isLeft(): boolean { return false; }
  isRight(): boolean { return true; }
  
  map<U>(fn: (value: R) => U): Either<L, U> {
    return new Right<L, U>(fn(this.value));
  }
  
  flatMap<U>(fn: (value: R) => Either<L, U>): Either<L, U> {
    return fn(this.value);
  }
  
  getOrElse(_defaultValue: R): R {
    return this.value;
  }
  
  fold<U>(_onLeft: (error: L) => U, onRight: (value: R) => U): U {
    return onRight(this.value);
  }
  
  toString(): string {
    return `Right(${this.value})`;
  }
}

// ตัวอย่างการใช้งาน Either
type ValidationError = string;

function parseAge(ageStr: string): Either<ValidationError, number> {
  const age = parseInt(ageStr, 10);
  
  if (isNaN(age)) {
    return Either.left(`"${ageStr}" ไม่ใช่ตัวเลข`);
  }
  
  if (age < 0) {
    return Either.left("อายุต้องไม่ต่ำกว่า 0");
  }
  
  if (age > 150) {
    return Either.left("อายุต้องไม่เกิน 150");
  }
  
  return Either.right(age);
}

function isAdult(age: number): Either<ValidationError, string> {
  if (age >= 18) {
    return Either.right(`ผู้ใหญ่ (อายุ ${age} ปี)`);
  }
  return Either.left(`ผู้เยาว์ (อายุ ${age} ปี)`);
}

// ใช้งาน
const validAge = parseAge("25").flatMap(isAdult);
const negativeAge = parseAge("-5");
const invalidAge = parseAge("abc");
const youngAge = parseAge("15").flatMap(isAdult);

console.log(validAge.fold(err => `Error: ${err}`, val => val));   // "ผู้ใหญ่ (อายุ 25 ปี)"
console.log(negativeAge.fold(err => `Error: ${err}`, val => `${val}`)); // "Error: อายุต้องไม่ต่ำกว่า 0"
console.log(invalidAge.fold(err => `Error: ${err}`, val => `${val}`));  // "Error: "abc" ไม่ใช่ตัวเลข"
console.log(youngAge.fold(err => `Error: ${err}`, val => val));   // "Error: ผู้เยาว์ (อายุ 15 ปี)"
```

### IO Monad

```typescript
// IO Monad สำหรับ side effects
class IO<T> {
  constructor(private readonly effect: () => T) {}
  
  static of<T>(value: T): IO<T> {
    return new IO(() => value);
  }
  
  static from<T>(effect: () => T): IO<T> {
    return new IO(effect);
  }
  
  map<U>(fn: (value: T) => U): IO<U> {
    return new IO(() => fn(this.effect()));
  }
  
  flatMap<U>(fn: (value: T) => IO<U>): IO<U> {
    return new IO(() => fn(this.effect()).run());
  }
  
  run(): T {
    return this.effect();
  }
}

// ตัวอย่างการใช้งาน IO
const getDate = IO.from(() => new Date());
const formatDate = (date: Date): string => 
  date.toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });

const getCurrentDateString = getDate.map(formatDate);
console.log(getCurrentDateString.run()); // วันที่ปัจจุบัน

// การจัดการ side effects ด้วย IO
const readEnv = (key: string): IO<string | undefined> => 
  IO.from(() => process.env[key]);

const logMessage = (message: string): IO<void> =>
  IO.from(() => { console.log(message); });

const program = readEnv("NODE_ENV")
  .map(env => env ?? "development")
  .flatMap(env => logMessage(`Environment: ${env}`));

program.run(); // ทำงาน side effect
```

---

## 44.8 fp-ts Library Overview

fp-ts เป็น library ที่ทรงพลังสำหรับ Functional Programming ใน TypeScript

### การติดตั้ง

```bash
npm install fp-ts
```

### Option Type

```typescript
import { Option, some, none, map, getOrElse, chain } from 'fp-ts/Option';
import { pipe } from 'fp-ts/function';

// สร้าง Option values
const someValue: Option<number> = some(42);
const noValue: Option<number> = none;

// ใช้ pipe กับ Option
const result = pipe(
  someValue,
  map(n => n * 2),      // some(84)
  map(n => n + 1),      // some(85)
  getOrElse(() => 0)    // 85
);

console.log(result); // 85

// ตัวอย่างกับ none
const result2 = pipe(
  noValue,
  map(n => n * 2),   // none
  map(n => n + 1),   // none
  getOrElse(() => 0) // 0 (default)
);

console.log(result2); // 0

// fromNullable
import { fromNullable } from 'fp-ts/Option';

const maybeUser = fromNullable<string>("สมชาย");
const maybeNull = fromNullable<string>(null);

const greeting = pipe(
  maybeUser,
  map(name => `สวัสดี ${name}!`),
  getOrElse(() => "ไม่มีผู้ใช้")
);

console.log(greeting); // "สวัสดี สมชาย!"
```

### Either Type ใน fp-ts

```typescript
import { Either, left, right, map, mapLeft, chain, fold } from 'fp-ts/Either';
import { pipe } from 'fp-ts/function';

type AppError = {
  code: string;
  message: string;
};

function validateEmail(email: string): Either<AppError, string> {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (emailRegex.test(email)) {
    return right(email);
  }
  return left({
    code: "INVALID_EMAIL",
    message: `${email} ไม่ใช่ email ที่ถูกต้อง`
  });
}

function validatePassword(password: string): Either<AppError, string> {
  if (password.length >= 8) {
    return right(password);
  }
  return left({
    code: "WEAK_PASSWORD",
    message: "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"
  });
}

// ตัวอย่างการใช้งาน
const emailResult = pipe(
  validateEmail("test@example.com"),
  map(email => email.toLowerCase()),
  fold(
    (error) => `ข้อผิดพลาด: ${error.message}`,
    (email) => `Email ถูกต้อง: ${email}`
  )
);

console.log(emailResult); // "Email ถูกต้อง: test@example.com"
```

### Task สำหรับ Async Operations

```typescript
import { Task } from 'fp-ts/Task';
import { TaskEither, tryCatchK, map, chain } from 'fp-ts/TaskEither';
import { pipe } from 'fp-ts/function';

// Task เป็น async computation ที่ไม่มี error
const delay = (ms: number): Task<void> =>
  () => new Promise(resolve => setTimeout(resolve, ms));

const fetchUserId: Task<number> = async () => {
  await new Promise(resolve => setTimeout(resolve, 100));
  return 42;
};

const fetchUserName = (id: number): Task<string> => async () => {
  await new Promise(resolve => setTimeout(resolve, 100));
  return `ผู้ใช้หมายเลข ${id}`;
};

// TaskEither สำหรับ async computation ที่อาจมี error
type FetchError = { message: string };

const fetchUser = (id: number): TaskEither<FetchError, { id: number; name: string }> =>
  tryCatchK(
    async (userId: number) => {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return response.json();
    },
    (error): FetchError => ({
      message: error instanceof Error ? error.message : "Unknown error"
    })
  )(id);
```

---

## 44.9 Pipe และ Flow Operators

```typescript
import { pipe, flow } from 'fp-ts/function';

// pipe - รับ value แล้วส่งผ่านฟังก์ชันต่างๆ
const result = pipe(
  5,
  (n) => n * 2,        // 10
  (n) => n + 1,        // 11
  (n) => n.toString(), // "11"
  (s) => `Result: ${s}` // "Result: 11"
);

console.log(result); // "Result: 11"

// flow - สร้างฟังก์ชันจากการรวมฟังก์ชันหลายตัว
const processNumber = flow(
  (n: number) => n * 2,
  (n) => n + 1,
  (n) => n.toString(),
  (s) => `Result: ${s}`
);

console.log(processNumber(5));  // "Result: 11"
console.log(processNumber(10)); // "Result: 21"
console.log(processNumber(15)); // "Result: 31"
```

### pipe กับ Array Operations

```typescript
import { pipe } from 'fp-ts/function';
import * as A from 'fp-ts/Array';
import * as O from 'fp-ts/Option';

interface Student {
  name: string;
  grade: number;
  subject: string;
}

const students: Student[] = [
  { name: "สมชาย", grade: 85, subject: "คณิตศาสตร์" },
  { name: "สมหญิง", grade: 92, subject: "วิทยาศาสตร์" },
  { name: "มานะ", grade: 78, subject: "คณิตศาสตร์" },
  { name: "มานี", grade: 95, subject: "ภาษาไทย" },
  { name: "ปิติ", grade: 60, subject: "วิทยาศาสตร์" },
];

// หาคะแนนเฉลี่ยของนักเรียนที่ผ่านเกณฑ์ (>= 70)
const averagePassingGrade = pipe(
  students,
  A.filter(s => s.grade >= 70),
  A.map(s => s.grade),
  (grades) => grades.length > 0 
    ? O.some(grades.reduce((a, b) => a + b, 0) / grades.length)
    : O.none,
  O.getOrElse(() => 0)
);

console.log(averagePassingGrade); // 87.5
```

---

## 44.10 Practical FP Patterns ใน TypeScript

### Validation Pattern

```typescript
type ValidationResult<T> = 
  | { success: true; data: T }
  | { success: false; errors: string[] };

function validate<T>(
  value: T,
  ...validators: Array<(v: T) => string | null>
): ValidationResult<T> {
  const errors = validators
    .map(validator => validator(value))
    .filter((result): result is string => result !== null);
  
  if (errors.length === 0) {
    return { success: true, data: value };
  }
  
  return { success: false, errors };
}

// Validators
const notEmpty = (value: string): string | null =>
  value.trim() === "" ? "ห้ามว่างเปล่า" : null;

const minLength = (min: number) => (value: string): string | null =>
  value.length < min ? `ต้องมีอย่างน้อย ${min} ตัวอักษร` : null;

const maxLength = (max: number) => (value: string): string | null =>
  value.length > max ? `ต้องไม่เกิน ${max} ตัวอักษร` : null;

const isEmail = (value: string): string | null => {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(value) ? null : "รูปแบบ email ไม่ถูกต้อง";
};

// การใช้งาน
const usernameResult = validate(
  "ab",
  notEmpty,
  minLength(3),
  maxLength(20)
);

console.log(usernameResult);
// { success: false, errors: ["ต้องมีอย่างน้อย 3 ตัวอักษร"] }

const emailResult = validate(
  "test@example.com",
  notEmpty,
  isEmail
);

console.log(emailResult);
// { success: true, data: "test@example.com" }
```

### Repository Pattern กับ FP

```typescript
import { TaskEither, tryCatch, map, chain } from 'fp-ts/TaskEither';
import { pipe } from 'fp-ts/function';

type DatabaseError = { type: "DB_ERROR"; message: string };
type NotFoundError = { type: "NOT_FOUND"; id: string };
type AppError = DatabaseError | NotFoundError;

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserRepository {
  findById: (id: string) => TaskEither<AppError, User>;
  save: (user: User) => TaskEither<AppError, User>;
  delete: (id: string) => TaskEither<AppError, void>;
}

// Mock implementation
const createUserRepository = (): UserRepository => {
  const users: Map<string, User> = new Map([
    ["1", { id: "1", name: "สมชาย", email: "somchai@example.com" }],
    ["2", { id: "2", name: "สมหญิง", email: "somying@example.com" }],
  ]);
  
  return {
    findById: (id) => async () => {
      const user = users.get(id);
      if (!user) {
        return { _tag: "Left" as const, left: { type: "NOT_FOUND" as const, id } };
      }
      return { _tag: "Right" as const, right: user };
    },
    
    save: (user) => async () => {
      users.set(user.id, user);
      return { _tag: "Right" as const, right: user };
    },
    
    delete: (id) => async () => {
      users.delete(id);
      return { _tag: "Right" as const, right: undefined };
    }
  };
};
```

### Pipeline สำหรับ Data Processing

```typescript
// สร้าง pipeline สำหรับประมวลผลข้อมูล
type Transformer<T, U> = (input: T) => U;
type AsyncTransformer<T, U> = (input: T) => Promise<U>;

class Pipeline<T> {
  constructor(private readonly value: T) {}
  
  static of<T>(value: T): Pipeline<T> {
    return new Pipeline(value);
  }
  
  map<U>(fn: Transformer<T, U>): Pipeline<U> {
    return new Pipeline(fn(this.value));
  }
  
  async mapAsync<U>(fn: AsyncTransformer<T, U>): Promise<Pipeline<U>> {
    const result = await fn(this.value);
    return new Pipeline(result);
  }
  
  filter(predicate: (value: T) => boolean, defaultValue: T): Pipeline<T> {
    return predicate(this.value) ? this : new Pipeline(defaultValue);
  }
  
  tap(fn: (value: T) => void): Pipeline<T> {
    fn(this.value);
    return this;
  }
  
  get(): T {
    return this.value;
  }
}

// ตัวอย่างการใช้งาน
interface OrderData {
  id: string;
  customerId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  discount?: number;
}

interface ProcessedOrder {
  id: string;
  customerId: string;
  subtotal: number;
  discount: number;
  total: number;
  itemCount: number;
}

const processOrder = (order: OrderData): ProcessedOrder => {
  return Pipeline.of(order)
    .map(o => ({
      ...o,
      subtotal: o.items.reduce((sum, item) => sum + item.quantity * item.price, 0)
    }))
    .map(o => ({
      ...o,
      discount: o.subtotal * (o.discount ?? 0)
    }))
    .map(o => ({
      id: o.id,
      customerId: o.customerId,
      subtotal: o.subtotal,
      discount: o.discount,
      total: o.subtotal - o.discount,
      itemCount: o.items.reduce((sum, item) => sum + item.quantity, 0)
    }))
    .tap(o => console.log(`Processing order ${o.id}, total: ${o.total}`))
    .get();
};

const order: OrderData = {
  id: "ORD-001",
  customerId: "CUST-001",
  items: [
    { productId: "P1", quantity: 2, price: 500 },
    { productId: "P2", quantity: 1, price: 1200 },
  ],
  discount: 0.1
};

const processed = processOrder(order);
console.log(processed);
// { id: "ORD-001", subtotal: 2200, discount: 220, total: 1980, itemCount: 3 }
```

---

## 44.11 Performance Considerations

### Lazy Evaluation

```typescript
// Lazy evaluation ด้วย generators
function* lazyMap<T, U>(
  iterable: Iterable<T>,
  fn: (item: T) => U
): Generator<U> {
  for (const item of iterable) {
    yield fn(item);
  }
}

function* lazyFilter<T>(
  iterable: Iterable<T>,
  predicate: (item: T) => boolean
): Generator<T> {
  for (const item of iterable) {
    if (predicate(item)) {
      yield item;
    }
  }
}

function* lazyTake<T>(
  iterable: Iterable<T>,
  count: number
): Generator<T> {
  let taken = 0;
  for (const item of iterable) {
    if (taken >= count) break;
    yield item;
    taken++;
  }
}

function toArray<T>(iterable: Iterable<T>): T[] {
  return [...iterable];
}

// สร้าง infinite sequence
function* range(start: number, step = 1): Generator<number> {
  let current = start;
  while (true) {
    yield current;
    current += step;
  }
}

// ใช้ lazy evaluation - ไม่คำนวณทั้งหมดจนกว่าจะต้องการ
const first10EvenSquares = toArray(
  lazyTake(
    lazyFilter(
      lazyMap(range(1), n => n * n),  // squares: 1, 4, 9, 16, ...
      n => n % 2 === 0                  // even squares: 4, 16, 36, ...
    ),
    5                                   // แค่ 5 ตัว
  )
);

console.log(first10EvenSquares); // [4, 16, 36, 64, 100]
```

### Memoization สำหรับ Performance

```typescript
// Memoization พร้อม LRU Cache
class LRUCache<K, V> {
  private cache: Map<K, V> = new Map();
  
  constructor(private readonly maxSize: number) {}
  
  get(key: K): V | undefined {
    if (this.cache.has(key)) {
      // Move to end (most recently used)
      const value = this.cache.get(key)!;
      this.cache.delete(key);
      this.cache.set(key, value);
      return value;
    }
    return undefined;
  }
  
  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove least recently used
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
  
  get size(): number {
    return this.cache.size;
  }
}

function memoizeWithLRU<T extends (...args: any[]) => any>(
  fn: T,
  maxSize: number = 100
): T {
  const cache = new LRUCache<string, ReturnType<T>>(maxSize);
  
  return ((...args: Parameters<T>) => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);
    
    if (cached !== undefined) {
      return cached;
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// ตัวอย่าง: Fibonacci ที่รวดเร็ว
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const memoFib = memoizeWithLRU(fibonacci);
console.time("fib(40)");
console.log(memoFib(40)); // 102334155
console.timeEnd("fib(40)");
```

---

## 44.12 ตัวอย่าง FP Pattern ที่ใช้งานจริง

### State Management Pattern

```typescript
// Immutable State Management
interface AppState {
  users: User[];
  loading: boolean;
  error: string | null;
  selectedUserId: string | null;
}

type Action =
  | { type: "SET_LOADING"; payload: boolean }
  | { type: "SET_USERS"; payload: User[] }
  | { type: "SET_ERROR"; payload: string }
  | { type: "SELECT_USER"; payload: string }
  | { type: "CLEAR_ERROR" };

// Pure reducer function
function reducer(state: AppState, action: Action): AppState {
  switch (action.type) {
    case "SET_LOADING":
      return { ...state, loading: action.payload };
    
    case "SET_USERS":
      return { ...state, users: action.payload, loading: false, error: null };
    
    case "SET_ERROR":
      return { ...state, error: action.payload, loading: false };
    
    case "SELECT_USER":
      return { ...state, selectedUserId: action.payload };
    
    case "CLEAR_ERROR":
      return { ...state, error: null };
    
    default:
      return state;
  }
}

// การใช้งาน
const initialState: AppState = {
  users: [],
  loading: false,
  error: null,
  selectedUserId: null
};

let state = initialState;

// Dispatch actions
state = reducer(state, { type: "SET_LOADING", payload: true });
console.log(state.loading); // true

state = reducer(state, {
  type: "SET_USERS",
  payload: [
    { id: "1", name: "สมชาย", email: "somchai@example.com" },
    { id: "2", name: "สมหญิง", email: "somying@example.com" }
  ]
});
console.log(state.users.length); // 2
console.log(state.loading);      // false
```

### Event Sourcing Pattern

```typescript
// Event Sourcing Pattern
interface Event {
  id: string;
  type: string;
  timestamp: Date;
  payload: unknown;
}

interface BankAccount {
  id: string;
  balance: number;
  transactions: Array<{ amount: number; description: string; date: Date }>;
}

type BankEvent =
  | { type: "ACCOUNT_CREATED"; accountId: string }
  | { type: "MONEY_DEPOSITED"; amount: number; description: string }
  | { type: "MONEY_WITHDRAWN"; amount: number; description: string };

function applyEvent(account: BankAccount, event: BankEvent): BankAccount {
  switch (event.type) {
    case "ACCOUNT_CREATED":
      return { ...account, id: event.accountId };
    
    case "MONEY_DEPOSITED":
      return {
        ...account,
        balance: account.balance + event.amount,
        transactions: [
          ...account.transactions,
          { amount: event.amount, description: event.description, date: new Date() }
        ]
      };
    
    case "MONEY_WITHDRAWN":
      if (account.balance < event.amount) {
        throw new Error("ยอดเงินไม่เพียงพอ");
      }
      return {
        ...account,
        balance: account.balance - event.amount,
        transactions: [
          ...account.transactions,
          { amount: -event.amount, description: event.description, date: new Date() }
        ]
      };
  }
}

function replayEvents(events: BankEvent[]): BankAccount {
  const initialAccount: BankAccount = {
    id: "",
    balance: 0,
    transactions: []
  };
  
  return events.reduce(applyEvent, initialAccount);
}

// ทดสอบ
const events: BankEvent[] = [
  { type: "ACCOUNT_CREATED", accountId: "ACC-001" },
  { type: "MONEY_DEPOSITED", amount: 10000, description: "เงินเดือน" },
  { type: "MONEY_WITHDRAWN", amount: 2000, description: "ค่าอาหาร" },
  { type: "MONEY_DEPOSITED", amount: 5000, description: "โบนัส" },
  { type: "MONEY_WITHDRAWN", amount: 1500, description: "ค่าไฟ" },
];

const finalAccount = replayEvents(events);
console.log(`ยอดเงินคงเหลือ: ${finalAccount.balance} บาท`); // 11500 บาท
```

---

## สรุปบทที่ 44

ในบทนี้เราได้เรียนรู้:

1. **Pure Functions และ Immutability** - หลักการพื้นฐานของ FP
2. **Higher-Order Functions** - การใช้ฟังก์ชันเป็น first-class citizen
3. **Function Composition** - การรวมฟังก์ชันเข้าด้วยกัน
4. **Currying** - การแปลงฟังก์ชันเพื่อ partial application
5. **Functors** - Container ที่มี map
6. **Monads** - Maybe, Either, IO สำหรับจัดการ context
7. **fp-ts** - Library สำหรับ FP ใน TypeScript
8. **Pipe และ Flow** - การเชื่อมฟังก์ชัน
9. **Performance** - Lazy evaluation และ Memoization
10. **Practical Patterns** - State management, Event sourcing

Functional Programming ช่วยให้โค้ดของเรา:
- ทดสอบได้ง่ายขึ้น
- คาดเดาผลลัพธ์ได้
- Compose ได้ดี
- ลด bugs จาก side effects
