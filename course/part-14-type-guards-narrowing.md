# ส่วนที่ 14: Type Guards และ Type Narrowing

## บทนำ

Type Narrowing คือกระบวนการที่ TypeScript ลดช่วง (narrow) ของ type ที่เป็นไปได้ลง โดยอาศัยการตรวจสอบต่างๆ ในโค้ด TypeScript ฉลาดพอที่จะติดตามการตรวจสอบเหล่านี้และปรับ type ให้แม่นยำขึ้น

---

## 14.1 typeof Type Guards

### การใช้ typeof

```typescript
// typeof ใช้ตรวจสอบ primitive types
function processValue(value: string | number | boolean | null | undefined): string {
  if (typeof value === "string") {
    // ในบล็อกนี้ TypeScript รู้ว่า value เป็น string
    return value.toUpperCase();
  }
  
  if (typeof value === "number") {
    // ในบล็อกนี้ TypeScript รู้ว่า value เป็น number
    return value.toFixed(2);
  }
  
  if (typeof value === "boolean") {
    // ในบล็อกนี้ TypeScript รู้ว่า value เป็น boolean
    return value ? "ใช่" : "ไม่ใช่";
  }
  
  // ที่นี่ TypeScript รู้ว่า value เป็น null | undefined
  return "ไม่มีค่า";
}

console.log(processValue("hello"));   // "HELLO"
console.log(processValue(3.14));      // "3.14"
console.log(processValue(true));      // "ใช่"
console.log(processValue(null));      // "ไม่มีค่า"
```

### typeof กับ Union Types

```typescript
type StringOrNumber = string | number;

function double(value: StringOrNumber): StringOrNumber {
  if (typeof value === "string") {
    return value.repeat(2);
  }
  return value * 2;
}

console.log(double("hello")); // "hellohello"
console.log(double(5));       // 10

// typeof ตรวจสอบได้: "string", "number", "boolean", "bigint", "symbol", "object", "function", "undefined"
function getTypeDescription(value: unknown): string {
  switch (typeof value) {
    case "string": return `ข้อความ: "${value}"`;
    case "number": return `ตัวเลข: ${value}`;
    case "boolean": return `ค่าบูลีน: ${value}`;
    case "undefined": return "ไม่มีค่า (undefined)";
    case "object":
      if (value === null) return "null";
      if (Array.isArray(value)) return `อาร์เรย์ ความยาว ${value.length}`;
      return `object: ${JSON.stringify(value)}`;
    case "function": return `ฟังก์ชัน: ${value.name}`;
    default: return `ประเภทอื่น: ${typeof value}`;
  }
}
```

---

## 14.2 instanceof Type Guards

### การใช้ instanceof

```typescript
class Dog {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  bark(): string {
    return `${this.name} โฮ่ง!`;
  }
}

class Cat {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  meow(): string {
    return `${this.name} เหมียว~`;
  }
}

type Animal = Dog | Cat;

function makeSound(animal: Animal): string {
  if (animal instanceof Dog) {
    // TypeScript รู้ว่า animal เป็น Dog
    return animal.bark();
  }
  // TypeScript รู้ว่า animal เป็น Cat (เพราะไม่ใช่ Dog)
  return animal.meow();
}

const dog = new Dog("บัดดี้");
const cat = new Cat("มิ้ว");

console.log(makeSound(dog)); // "บัดดี้ โฮ่ง!"
console.log(makeSound(cat)); // "มิ้ว เหมียว~"
```

### instanceof กับ Error Types

```typescript
class NetworkError extends Error {
  statusCode: number;
  
  constructor(message: string, statusCode: number) {
    super(message);
    this.name = "NetworkError";
    this.statusCode = statusCode;
  }
}

class ValidationError extends Error {
  field: string;
  
  constructor(message: string, field: string) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

class NotFoundError extends Error {
  resource: string;
  
  constructor(resource: string) {
    super(`ไม่พบ ${resource}`);
    this.name = "NotFoundError";
    this.resource = resource;
  }
}

function handleError(error: unknown): string {
  if (error instanceof NetworkError) {
    return `เกิดข้อผิดพลาดเครือข่าย (${error.statusCode}): ${error.message}`;
  }
  
  if (error instanceof ValidationError) {
    return `ข้อมูลไม่ถูกต้อง ที่ฟิลด์ "${error.field}": ${error.message}`;
  }
  
  if (error instanceof NotFoundError) {
    return `ไม่พบทรัพยากร: ${error.resource}`;
  }
  
  if (error instanceof Error) {
    return `เกิดข้อผิดพลาด: ${error.message}`;
  }
  
  return "เกิดข้อผิดพลาดที่ไม่รู้จัก";
}

try {
  throw new NetworkError("Connection refused", 503);
} catch (err) {
  console.log(handleError(err));
}
```

---

## 14.3 in Operator Type Guard

### การใช้ in

```typescript
interface Fish {
  swim(): void;
  breatheUnderwater: boolean;
}

interface Bird {
  fly(): void;
  wingspan: number;
}

type Pet = Fish | Bird;

function moveAnimal(pet: Pet): void {
  if ("swim" in pet) {
    // TypeScript รู้ว่า pet เป็น Fish
    pet.swim();
    console.log(`หายใจใต้น้ำ: ${pet.breatheUnderwater}`);
  } else {
    // TypeScript รู้ว่า pet เป็น Bird
    pet.fly();
    console.log(`ปีกกว้าง: ${pet.wingspan} ซม.`);
  }
}
```

### in กับ Complex Types

```typescript
interface CircleShape {
  kind: "circle";
  radius: number;
}

interface RectangleShape {
  kind: "rectangle";
  width: number;
  height: number;
}

interface TriangleShape {
  kind: "triangle";
  base: number;
  height: number;
}

type Shape = CircleShape | RectangleShape | TriangleShape;

function calculateArea(shape: Shape): number {
  if ("radius" in shape) {
    // TypeScript รู้ว่า shape เป็น CircleShape
    return Math.PI * shape.radius ** 2;
  }
  
  if ("width" in shape) {
    // TypeScript รู้ว่า shape เป็น RectangleShape
    return shape.width * shape.height;
  }
  
  // TypeScript รู้ว่า shape เป็น TriangleShape
  return (shape.base * shape.height) / 2;
}

const circle: CircleShape = { kind: "circle", radius: 5 };
const rect: RectangleShape = { kind: "rectangle", width: 4, height: 6 };
const tri: TriangleShape = { kind: "triangle", base: 3, height: 8 };

console.log(calculateArea(circle)); // 78.54...
console.log(calculateArea(rect));   // 24
console.log(calculateArea(tri));    // 12
```

---

## 14.4 Custom Type Predicates (is)

### Type Predicate พื้นฐาน

```typescript
// รูปแบบ: function name(param: Type): param is SpecificType
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNumber(value: unknown): value is number {
  return typeof value === "number" && !isNaN(value);
}

function isArray<T>(value: unknown): value is T[] {
  return Array.isArray(value);
}

// ใช้งาน
const values: unknown[] = ["hello", 42, true, null, [1, 2, 3]];

const strings = values.filter(isString); // type: string[]
const numbers = values.filter(isNumber); // type: number[]

// ฟังก์ชัน filter ที่ type-safe
function filterByType<T>(arr: unknown[], guard: (val: unknown) => val is T): T[] {
  return arr.filter(guard);
}

const onlyStrings = filterByType(values, isString); // string[]
const onlyNumbers = filterByType(values, isNumber); // number[]
```

### Type Predicate สำหรับ Objects

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

interface Admin extends User {
  permissions: string[];
  level: number;
}

// ตรวจสอบว่าเป็น User
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as User).id === "number" &&
    typeof (value as User).name === "string" &&
    typeof (value as User).email === "string"
  );
}

// ตรวจสอบว่าเป็น Admin
function isAdmin(user: User): user is Admin {
  return "permissions" in user && Array.isArray((user as Admin).permissions);
}

// ใช้งาน
function processApiResponse(data: unknown): void {
  if (!isUser(data)) {
    throw new Error("ข้อมูลไม่ถูกต้อง");
  }
  
  // ที่นี่ data เป็น User แน่นอน
  console.log(`ผู้ใช้: ${data.name}`);
  
  if (isAdmin(data)) {
    // ที่นี่ data เป็น Admin
    console.log(`สิทธิ์: ${data.permissions.join(", ")}`);
  }
}
```

### Type Predicate กับ Array Methods

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  available: boolean;
}

interface DiscountedProduct extends Product {
  discountPercent: number;
  originalPrice: number;
}

function isDiscountedProduct(p: Product): p is DiscountedProduct {
  return "discountPercent" in p;
}

const products: Product[] = [
  { id: 1, name: "สินค้า A", price: 100, available: true },
  { id: 2, name: "สินค้า B (ลด)", price: 80, available: true, 
    discountPercent: 20, originalPrice: 100 } as DiscountedProduct,
  { id: 3, name: "สินค้า C", price: 200, available: false }
];

// กรองเฉพาะสินค้าลดราคา
const discountedProducts = products.filter(isDiscountedProduct);
// type: DiscountedProduct[]

discountedProducts.forEach(p => {
  console.log(`${p.name}: ลด ${p.discountPercent}% จาก ${p.originalPrice} บาท`);
});
```

---

## 14.5 Discriminated Unions

### พื้นฐาน Discriminated Union

```typescript
// ใช้ "kind" หรือ "type" field เป็น discriminant
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number }
  | { kind: "ellipse"; radiusX: number; radiusY: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
    case "ellipse":
      return Math.PI * shape.radiusX * shape.radiusY;
  }
}

function getPerimeter(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return 2 * Math.PI * shape.radius;
    case "rectangle":
      return 2 * (shape.width + shape.height);
    case "triangle":
      // สมมติเป็น equilateral triangle
      return 3 * shape.base;
    case "ellipse":
      // ประมาณ perimeter ของ ellipse
      return 2 * Math.PI * Math.sqrt((shape.radiusX ** 2 + shape.radiusY ** 2) / 2);
  }
}
```

### Discriminated Union กับ Actions

```typescript
// Redux-style actions
type UserAction =
  | { type: "USER_LOGIN"; payload: { userId: string; token: string } }
  | { type: "USER_LOGOUT" }
  | { type: "USER_UPDATE"; payload: { name?: string; email?: string } }
  | { type: "USER_DELETE"; payload: { userId: string; reason: string } };

interface UserState {
  userId: string | null;
  token: string | null;
  name: string;
  email: string;
  isLoggedIn: boolean;
}

function userReducer(state: UserState, action: UserAction): UserState {
  switch (action.type) {
    case "USER_LOGIN":
      return {
        ...state,
        userId: action.payload.userId,
        token: action.payload.token,
        isLoggedIn: true
      };
    
    case "USER_LOGOUT":
      return {
        ...state,
        userId: null,
        token: null,
        isLoggedIn: false
      };
    
    case "USER_UPDATE":
      return {
        ...state,
        ...(action.payload.name && { name: action.payload.name }),
        ...(action.payload.email && { email: action.payload.email })
      };
    
    case "USER_DELETE":
      console.log(`ลบผู้ใช้ ${action.payload.userId}: ${action.payload.reason}`);
      return {
        ...state,
        userId: null,
        token: null,
        isLoggedIn: false
      };
  }
}
```

### Discriminated Union สำหรับ API Results

```typescript
// Result type ที่ชัดเจน
type ApiResult<T> =
  | { success: true; data: T; statusCode: 200 | 201 }
  | { success: false; error: string; statusCode: 400 | 401 | 403 | 404 | 500 };

interface Order {
  id: string;
  items: string[];
  total: number;
}

async function createOrder(items: string[]): Promise<ApiResult<Order>> {
  if (items.length === 0) {
    return {
      success: false,
      error: "ต้องเลือกสินค้าอย่างน้อย 1 รายการ",
      statusCode: 400
    };
  }
  
  return {
    success: true,
    data: {
      id: "ORD-" + Date.now(),
      items,
      total: items.length * 100
    },
    statusCode: 201
  };
}

// ใช้งาน
const result = await createOrder(["สินค้า A", "สินค้า B"]);

if (result.success) {
  // TypeScript รู้ว่า result.data มีอยู่
  console.log(`สร้างคำสั่งซื้อ: ${result.data.id}`);
  console.log(`ยอดรวม: ${result.data.total} บาท`);
} else {
  // TypeScript รู้ว่า result.error มีอยู่
  console.error(`ข้อผิดพลาด: ${result.error}`);
}
```

---

## 14.6 Truthiness Narrowing

### Falsy Values

```typescript
// TypeScript ลด type เมื่อตรวจสอบ truthy/falsy
function processName(name: string | null | undefined): string {
  if (name) {
    // name เป็น string (ไม่ใช่ null, undefined, หรือ "")
    return name.trim().toUpperCase();
  }
  return "ไม่ระบุชื่อ";
}

// ระวัง! "" ก็เป็น falsy เช่นกัน
function processAge(age: number | null | undefined): number {
  if (age) {
    // age เป็น number ที่ไม่ใช่ 0, null, หรือ undefined
    // แต่ถ้า age = 0 จะไม่เข้าบล็อกนี้!
    return age;
  }
  // ที่นี่ age อาจเป็น 0 ด้วย
  return 0;
}

// วิธีที่ดีกว่า
function processAgeSafe(age: number | null | undefined): number {
  if (age != null) {
    // ตรวจสอบแบบ loose equality กับ null ครอบคลุมทั้ง null และ undefined
    return age; // ที่นี่ age เป็น number แน่นอน
  }
  return 0;
}

console.log(processAgeSafe(0));         // 0 (ถูกต้อง)
console.log(processAgeSafe(null));      // 0
console.log(processAgeSafe(undefined)); // 0
console.log(processAgeSafe(25));        // 25
```

### Nullish Coalescing กับ Narrowing

```typescript
interface UserProfile {
  name: string;
  bio: string | null;
  age: number | undefined;
  preferences: {
    theme: "light" | "dark" | null;
    language: string | undefined;
  } | null;
}

function displayProfile(profile: UserProfile): void {
  console.log(`ชื่อ: ${profile.name}`);
  
  // Nullish coalescing
  console.log(`Bio: ${profile.bio ?? "ยังไม่ได้กรอก"}`);
  console.log(`อายุ: ${profile.age ?? "ไม่ระบุ"}`);
  
  // Optional chaining + narrowing
  const theme = profile.preferences?.theme ?? "light";
  const language = profile.preferences?.language ?? "th";
  
  console.log(`ธีม: ${theme}`);
  console.log(`ภาษา: ${language}`);
}
```

---

## 14.7 Equality Narrowing

### การตรวจสอบด้วย ===

```typescript
function compareAndProcess(a: string | number, b: string | boolean): string {
  if (a === b) {
    // a และ b ต้องเป็น string เท่านั้น (ตัดกรณี number กับ boolean ออก)
    return a.toUpperCase(); // TypeScript รู้ว่า a เป็น string
  }
  return `a=${a}, b=${b}`;
}

// Equality กับ null/undefined
function processOptional(value: string | null | undefined): string {
  if (value === null) {
    return "ค่าเป็น null";
  }
  
  if (value === undefined) {
    return "ค่าเป็น undefined";
  }
  
  // ที่นี่ value เป็น string แน่นอน
  return value.trim();
}

// ใช้ !== เพื่อ narrow
function processRequired(value: string | null | undefined): string {
  if (value !== null && value !== undefined) {
    // value เป็น string
    return value.toUpperCase();
  }
  return "";
}

// หรือใช้ != null (ครอบคลุมทั้ง null และ undefined)
function processRequired2(value: string | null | undefined): string {
  if (value != null) {
    // value เป็น string
    return value.toUpperCase();
  }
  return "";
}
```

### Switch Statement Narrowing

```typescript
type StatusCode = 200 | 201 | 400 | 401 | 403 | 404 | 500;

function getStatusMessage(code: StatusCode): string {
  switch (code) {
    case 200:
      return "สำเร็จ";
    case 201:
      return "สร้างสำเร็จ";
    case 400:
      return "คำขอไม่ถูกต้อง";
    case 401:
      return "ต้องเข้าสู่ระบบก่อน";
    case 403:
      return "ไม่มีสิทธิ์เข้าถึง";
    case 404:
      return "ไม่พบข้อมูล";
    case 500:
      return "เซิร์ฟเวอร์เกิดข้อผิดพลาด";
  }
}
```

---

## 14.8 Assignment Narrowing

### Narrowing จากการ Assign

```typescript
// TypeScript ปรับ type ตามค่าที่ assign
let value: string | number;

value = "TypeScript";
// ที่นี่ value เป็น string
console.log(value.toUpperCase()); // OK

value = 42;
// ที่นี่ value เป็น number
console.log(value.toFixed(2)); // OK

// ตัวอย่างใน function
function processInput(input: string | number | null): void {
  let result: string;
  
  if (input === null) {
    result = "ไม่มีข้อมูล";
  } else if (typeof input === "string") {
    result = input.trim();
  } else {
    result = input.toString();
  }
  
  // result เป็น string แน่นอน (ถูก narrowed จากทุก branch)
  console.log(result.toUpperCase());
}
```

---

## 14.9 Control Flow Analysis

### TypeScript ติดตาม Flow

```typescript
interface Config {
  host?: string;
  port?: number;
  ssl?: boolean;
}

function buildConnectionString(config: Config): string {
  const host = config.host ?? "localhost";
  const port = config.port ?? 3000;
  const ssl = config.ssl ?? false;
  
  // TypeScript รู้ว่าทุกตัวแปรมีค่าแน่นอนแล้ว
  const protocol = ssl ? "https" : "http";
  return `${protocol}://${host}:${port}`;
}

// Early return pattern
function processUser(userId: string | null): string {
  if (userId === null) {
    return "ไม่มี user ID"; // early return
  }
  
  // ที่นี่ userId เป็น string แน่นอน
  return `กำลังประมวลผลผู้ใช้: ${userId}`;
}

// Assertion function
function assertDefined<T>(value: T | undefined | null, name: string): asserts value is T {
  if (value === undefined || value === null) {
    throw new Error(`${name} ต้องมีค่า`);
  }
}

function processOrder(orderId: string | undefined): void {
  assertDefined(orderId, "orderId");
  // ที่นี่ TypeScript รู้ว่า orderId เป็น string
  console.log(`ประมวลผลคำสั่งซื้อ: ${orderId}`);
}
```

### Multiple Conditions

```typescript
interface Animal {
  name: string;
  canFly?: boolean;
  canSwim?: boolean;
  canClimb?: boolean;
}

function describeAnimal(animal: Animal): string {
  const abilities: string[] = [];
  
  if (animal.canFly) {
    abilities.push("บินได้");
  }
  
  if (animal.canSwim) {
    abilities.push("ว่ายน้ำได้");
  }
  
  if (animal.canClimb) {
    abilities.push("ปีนป่ายได้");
  }
  
  if (abilities.length === 0) {
    return `${animal.name}: ไม่มีความสามารถพิเศษ`;
  }
  
  return `${animal.name}: ${abilities.join(", ")}`;
}

// Narrowing ใน try-catch
async function safeParseJSON(text: string): Promise<object | null> {
  try {
    const parsed: unknown = JSON.parse(text);
    if (typeof parsed === "object" && parsed !== null) {
      return parsed;
    }
    return null;
  } catch {
    return null;
  }
}
```

---

## 14.10 Exhaustive Checking with never

### never สำหรับ Exhaustive Check

```typescript
// ใช้ never เพื่อตรวจสอบว่า handle ครบทุก case
type TrafficLight = "red" | "yellow" | "green";

function getAction(light: TrafficLight): string {
  switch (light) {
    case "red":
      return "หยุด";
    case "yellow":
      return "ระวัง";
    case "green":
      return "ไป";
    default:
      // ถ้า TypeScript บอก error ที่บรรทัดนี้ แสดงว่าเพิ่ม case ใหม่แต่ยังไม่ handle
      const exhaustiveCheck: never = light;
      throw new Error(`ไฟจราจรที่ไม่รู้จัก: ${exhaustiveCheck}`);
  }
}
```

### Exhaustive Check ใน Union Types

```typescript
type PaymentMethod =
  | { type: "creditCard"; cardNumber: string; cvv: string }
  | { type: "bankTransfer"; bankAccount: string; bankCode: string }
  | { type: "promptPay"; phoneNumber: string }
  | { type: "crypto"; walletAddress: string; currency: string };

function processPayment(payment: PaymentMethod): string {
  switch (payment.type) {
    case "creditCard":
      return `ชำระด้วยบัตรเครดิต: ****${payment.cardNumber.slice(-4)}`;
    
    case "bankTransfer":
      return `โอนเงินธนาคาร: ${payment.bankCode} ${payment.bankAccount}`;
    
    case "promptPay":
      return `PromptPay: ${payment.phoneNumber}`;
    
    case "crypto":
      return `Crypto (${payment.currency}): ${payment.walletAddress}`;
    
    default:
      // ถ้าเพิ่ม payment method ใหม่โดยไม่ handle จะเกิด TypeScript error ที่นี่
      const _exhaustive: never = payment;
      throw new Error(`วิธีชำระเงินไม่รู้จัก`);
  }
}

// Helper function สำหรับ exhaustive check
function assertNever(value: never, message = "Unhandled case"): never {
  throw new Error(`${message}: ${JSON.stringify(value)}`);
}

function getPaymentIcon(payment: PaymentMethod): string {
  switch (payment.type) {
    case "creditCard": return "💳";
    case "bankTransfer": return "🏦";
    case "promptPay": return "📱";
    case "crypto": return "🔐";
    default: return assertNever(payment, "วิธีชำระเงินไม่รู้จัก");
  }
}
```

---

## 14.11 API Data Validation ในงานจริง

### Validator Library แบบ Type-Safe

```typescript
type ValidationError = { field: string; message: string };
type ValidationResult<T> =
  | { valid: true; value: T }
  | { valid: false; errors: ValidationError[] };

// Validator type
type Validator<T> = (value: unknown) => ValidationResult<T>;

// Primitive validators
function isStringValidator(field: string): Validator<string> {
  return (value) => {
    if (typeof value === "string") {
      return { valid: true, value };
    }
    return {
      valid: false,
      errors: [{ field, message: `${field} ต้องเป็น string` }]
    };
  };
}

function isNumberValidator(field: string): Validator<number> {
  return (value) => {
    if (typeof value === "number" && !isNaN(value)) {
      return { valid: true, value };
    }
    return {
      valid: false,
      errors: [{ field, message: `${field} ต้องเป็น number` }]
    };
  };
}

// Schema-based validator
type SchemaValidator<T> = {
  [K in keyof T]: Validator<T[K]>;
};

function validateSchema<T extends object>(
  data: unknown,
  schema: SchemaValidator<T>
): ValidationResult<T> {
  if (typeof data !== "object" || data === null) {
    return {
      valid: false,
      errors: [{ field: "root", message: "ข้อมูลต้องเป็น object" }]
    };
  }

  const errors: ValidationError[] = [];
  const result = {} as T;

  for (const key in schema) {
    const validator = schema[key];
    const fieldResult = validator((data as Record<string, unknown>)[key]);

    if (fieldResult.valid) {
      result[key] = fieldResult.value;
    } else {
      errors.push(...fieldResult.errors);
    }
  }

  if (errors.length > 0) {
    return { valid: false, errors };
  }

  return { valid: true, value: result };
}

// ใช้งาน
interface CreateProductRequest {
  name: string;
  price: number;
  category: string;
  stock: number;
}

const productSchema: SchemaValidator<CreateProductRequest> = {
  name: isStringValidator("name"),
  price: isNumberValidator("price"),
  category: isStringValidator("category"),
  stock: isNumberValidator("stock")
};

function handleCreateProduct(body: unknown): void {
  const result = validateSchema(body, productSchema);

  if (!result.valid) {
    console.error("ข้อมูลไม่ถูกต้อง:");
    result.errors.forEach(e => console.error(`  ${e.field}: ${e.message}`));
    return;
  }

  // result.value เป็น CreateProductRequest แน่นอน
  console.log(`สร้างสินค้า: ${result.value.name} ราคา ${result.value.price}`);
}

// ทดสอบ
handleCreateProduct({ name: "สินค้า A", price: 100, category: "อาหาร", stock: 50 });
handleCreateProduct({ name: 123, price: "ไม่ถูก" }); // จะมี error
```

### Runtime Type Checking

```typescript
// Type guards ที่ครอบคลุม
function isNonNullObject(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

function hasProperty<T extends object, K extends PropertyKey>(
  obj: T,
  key: K
): obj is T & Record<K, unknown> {
  return key in obj;
}

function hasStringProperty<T extends object>(
  obj: T,
  key: string
): obj is T & Record<typeof key, string> {
  return hasProperty(obj, key) && typeof (obj as Record<string, unknown>)[key] === "string";
}

function hasNumberProperty<T extends object>(
  obj: T,
  key: string
): obj is T & Record<typeof key, number> {
  return hasProperty(obj, key) && typeof (obj as Record<string, unknown>)[key] === "number";
}

// ใช้ร่วมกัน
function parseWebhookPayload(payload: unknown): {
  event: string;
  timestamp: number;
  data: Record<string, unknown>;
} | null {
  if (!isNonNullObject(payload)) return null;
  if (!hasStringProperty(payload, "event")) return null;
  if (!hasNumberProperty(payload, "timestamp")) return null;
  if (!hasProperty(payload, "data")) return null;
  if (!isNonNullObject(payload.data)) return null;

  return {
    event: payload.event,
    timestamp: payload.timestamp,
    data: payload.data
  };
}

// ทดสอบ
const payload1 = {
  event: "user.created",
  timestamp: Date.now(),
  data: { userId: "123", name: "สมชาย" }
};

const parsed = parseWebhookPayload(payload1);
if (parsed) {
  console.log(`Event: ${parsed.event}`);
  console.log(`Time: ${new Date(parsed.timestamp).toLocaleString("th-TH")}`);
}
```

---

## 14.12 Advanced Narrowing Patterns

### Narrowing กับ Class Hierarchy

```typescript
abstract class Vehicle {
  abstract getSpeed(): number;
  abstract getFuelType(): string;
  
  describe(): string {
    return `ความเร็วสูงสุด: ${this.getSpeed()} กม./ชม., เชื้อเพลิง: ${this.getFuelType()}`;
  }
}

class ElectricCar extends Vehicle {
  batteryCapacity: number;
  
  constructor(batteryCapacity: number) {
    super();
    this.batteryCapacity = batteryCapacity;
  }
  
  getSpeed(): number { return 250; }
  getFuelType(): string { return "ไฟฟ้า"; }
  
  charge(): void {
    console.log(`กำลังชาร์จแบตเตอรี่ ${this.batteryCapacity} kWh`);
  }
}

class GasCar extends Vehicle {
  tankCapacity: number;
  
  constructor(tankCapacity: number) {
    super();
    this.tankCapacity = tankCapacity;
  }
  
  getSpeed(): number { return 200; }
  getFuelType(): string { return "น้ำมัน"; }
  
  refuel(): void {
    console.log(`กำลังเติมน้ำมัน ${this.tankCapacity} ลิตร`);
  }
}

class Bicycle extends Vehicle {
  gears: number;
  
  constructor(gears: number) {
    super();
    this.gears = gears;
  }
  
  getSpeed(): number { return 30; }
  getFuelType(): string { return "พลังงานมนุษย์"; }
}

// ฟังก์ชันที่ handle ทุก vehicle type
function refillVehicle(vehicle: Vehicle): void {
  if (vehicle instanceof ElectricCar) {
    vehicle.charge(); // TypeScript รู้ว่าเป็น ElectricCar
  } else if (vehicle instanceof GasCar) {
    vehicle.refuel(); // TypeScript รู้ว่าเป็น GasCar
  } else if (vehicle instanceof Bicycle) {
    console.log("จักรยานไม่ต้องเติมเชื้อเพลิง");
  }
}
```

### Type Narrowing กับ Generics

```typescript
function processArray<T>(
  arr: T[],
  handler: (item: T) => void
): void {
  arr.forEach(handler);
}

// ตัวอย่าง: ประมวลผลข้อมูล API
interface ApiUser {
  type: "user";
  id: number;
  name: string;
}

interface ApiProduct {
  type: "product";
  id: number;
  name: string;
  price: number;
}

type ApiItem = ApiUser | ApiProduct;

function isApiUser(item: ApiItem): item is ApiUser {
  return item.type === "user";
}

function isApiProduct(item: ApiItem): item is ApiProduct {
  return item.type === "product";
}

const items: ApiItem[] = [
  { type: "user", id: 1, name: "สมชาย" },
  { type: "product", id: 1, name: "สินค้า A", price: 100 },
  { type: "user", id: 2, name: "สมหญิง" }
];

const users = items.filter(isApiUser);      // ApiUser[]
const products = items.filter(isApiProduct); // ApiProduct[]

console.log(`ผู้ใช้: ${users.map(u => u.name).join(", ")}`);
console.log(`สินค้า: ${products.map(p => `${p.name} (${p.price})`).join(", ")}`);
```

---

## สรุปบทนี้

ในบทนี้เราได้เรียนรู้:

- **typeof guards** ตรวจสอบ primitive types
- **instanceof guards** ตรวจสอบ class instances
- **in operator** ตรวจสอบ property ที่มีอยู่
- **Type predicates (is)** สร้าง type guard เอง
- **Discriminated unions** ใช้ discriminant field ในการ narrow
- **Truthiness narrowing** ลด type จากการตรวจสอบ truthy/falsy
- **Equality narrowing** ลด type จากการเปรียบเทียบด้วย ===
- **Assignment narrowing** ลด type จากการ assign
- **Control flow analysis** TypeScript ติดตาม type ตาม flow
- **Exhaustive checking** ใช้ never ตรวจสอบว่า handle ครบ

บทถัดไปจะเรียนรู้ Advanced Types
