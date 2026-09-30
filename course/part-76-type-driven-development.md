# ตอนที่ 76: Type-Driven Development (การพัฒนาด้วย Type เป็นตัวนำ)

## บทนำ

Type-Driven Development (TDD ในอีกความหมาย) คือแนวทางการพัฒนาซอฟต์แวร์ที่ใช้ระบบ Type ของภาษาเป็นตัวนำในการออกแบบและเขียนโค้ด แนวคิดหลักคือ "ถ้า code compile ผ่านได้ หมายความว่า code นั้นถูกต้องในระดับ type" ซึ่งช่วยลด runtime errors ได้อย่างมาก

---

## 1. Making Illegal States Unrepresentable (ทำให้สถานะที่ไม่ถูกต้องไม่สามารถแสดงได้)

แนวคิดนี้มาจาก Haskell และ F# community: ออกแบบ type ให้ state ที่ invalid เป็นไปไม่ได้ทาง type system

### ตัวอย่างที่ 1: Form State แบบไม่ดี

```typescript
// ❌ Bad: สามารถมีสถานะที่ไม่สมเหตุสมผลได้
interface UserFormBad {
  isLoading: boolean;
  isSuccess: boolean;
  isError: boolean;
  data?: User;
  error?: string;
}

// สถานะที่ invalid เช่น isLoading=true และ isSuccess=true พร้อมกัน
const badState: UserFormBad = {
  isLoading: true,
  isSuccess: true, // ❌ ไม่สมเหตุสมผล!
  isError: false,
  data: someUser,
};
```

### ตัวอย่างที่ 2: Form State แบบดี

```typescript
// ✅ Good: ใช้ Discriminated Union
type UserFormGood =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: string };

// ตอนนี้ state ที่ invalid เป็นไปไม่ได้แล้ว!
function handleFormState(state: UserFormGood) {
  switch (state.status) {
    case "idle":
      return <div>กรุณากรอกข้อมูล</div>;
    case "loading":
      return <div>กำลังโหลด...</div>;
    case "success":
      return <div>สำเร็จ! ข้อมูล: {state.data.name}</div>;
    case "error":
      return <div>เกิดข้อผิดพลาด: {state.error}</div>;
  }
}
```

### ตัวอย่างที่ 3: Email Verification State

```typescript
// ❌ Bad: สามารถมี email ที่ยังไม่ได้ verify แต่ isVerified = true
interface UserBad {
  email: string;
  isEmailVerified: boolean;
}

// ✅ Good: Email ที่ verified จะเป็น type แยกกัน
type UnverifiedEmail = {
  readonly _brand: "UnverifiedEmail";
  value: string;
};

type VerifiedEmail = {
  readonly _brand: "VerifiedEmail";
  value: string;
};

type EmailAddress = UnverifiedEmail | VerifiedEmail;

// User ที่ยังไม่ verify email
interface UnverifiedUser {
  id: string;
  email: UnverifiedEmail;
  name: string;
}

// User ที่ verify email แล้ว
interface VerifiedUser {
  id: string;
  email: VerifiedEmail;
  name: string;
  verifiedAt: Date;
}

type User = UnverifiedUser | VerifiedUser;

function sendVerificationEmail(user: UnverifiedUser): void {
  console.log(`ส่ง email verification ไปที่ ${user.email.value}`);
}

function accessProtectedResource(user: VerifiedUser): void {
  console.log(`${user.name} เข้าถึง resource ที่ต้องการ verified email`);
}
```

### ตัวอย่างที่ 4: Traffic Light State

```typescript
// ❌ Bad: สามารถมีสถานะที่ไม่ถูกต้องได้
interface TrafficLightBad {
  isRed: boolean;
  isYellow: boolean;
  isGreen: boolean;
}

// ✅ Good: ใช้ Discriminated Union
type TrafficLight =
  | { color: "red"; nextColor: "green" }
  | { color: "yellow"; nextColor: "red" }
  | { color: "green"; nextColor: "yellow" };

function nextState(light: TrafficLight): TrafficLight {
  switch (light.color) {
    case "red":
      return { color: "green", nextColor: "yellow" };
    case "yellow":
      return { color: "red", nextColor: "green" };
    case "green":
      return { color: "yellow", nextColor: "red" };
  }
}

const redLight: TrafficLight = { color: "red", nextColor: "green" };
const greenLight = nextState(redLight); // { color: "green", nextColor: "yellow" }
```

---

## 2. Phantom Types สำหรับ Domain Modeling

Phantom Types คือ type parameters ที่ไม่ปรากฏใน runtime แต่ช่วยให้ TypeScript ตรวจสอบความถูกต้องได้

### ตัวอย่างที่ 5: Unit Types

```typescript
// Phantom type markers
declare const _unit: unique symbol;
type Unit<T extends string> = { readonly [_unit]: T };

// Length units
type Meters = Unit<"meters">;
type Kilometers = Unit<"kilometers">;
type Miles = Unit<"miles">;

type Length<U extends Unit<string>> = number & { readonly _unit: U };

function meters(value: number): Length<Meters> {
  return value as Length<Meters>;
}

function kilometers(value: number): Length<Kilometers> {
  return value as Length<Kilometers>;
}

function miles(value: number): Length<Miles> {
  return value as Length<Miles>;
}

function addLengths<U extends Unit<string>>(
  a: Length<U>,
  b: Length<U>
): Length<U> {
  return (a + b) as Length<U>;
}

const distance1 = meters(100);
const distance2 = meters(200);
const total = addLengths(distance1, distance2); // ✅ OK: 300 meters

// const wrong = addLengths(meters(100), kilometers(1)); // ❌ Error!

// แปลงหน่วย
function metersToKilometers(m: Length<Meters>): Length<Kilometers> {
  return (m / 1000) as Length<Kilometers>;
}

const km = metersToKilometers(meters(5000)); // 5 km
```

### ตัวอย่างที่ 6: Currency Types

```typescript
declare const _currency: unique symbol;
type Currency<C extends string> = { readonly [_currency]: C };

type THB = Currency<"THB">;
type USD = Currency<"USD">;
type EUR = Currency<"EUR">;

type Money<C extends Currency<string>> = {
  amount: number;
  readonly _currency: C;
};

function thb(amount: number): Money<THB> {
  return { amount, _currency: "THB" as any };
}

function usd(amount: number): Money<USD> {
  return { amount, _currency: "USD" as any };
}

function addMoney<C extends Currency<string>>(
  a: Money<C>,
  b: Money<C>
): Money<C> {
  return { ...a, amount: a.amount + b.amount };
}

// Exchange rates (simplified)
type ExchangeRate<From extends Currency<string>, To extends Currency<string>> = {
  from: From;
  to: To;
  rate: number;
};

function convert<F extends Currency<string>, T extends Currency<string>>(
  money: Money<F>,
  rate: ExchangeRate<F, T>
): Money<T> {
  return { amount: money.amount * rate.rate, _currency: rate.to as any };
}

const price1 = thb(1000);
const price2 = thb(500);
const total = addMoney(price1, price2); // 1500 THB

// const wrong = addMoney(thb(100), usd(3)); // ❌ Error! ไม่สามารถบวก THB กับ USD ได้โดยตรง
```

### ตัวอย่างที่ 7: Validated/Unvalidated Data

```typescript
declare const _validated: unique symbol;
type Validated = { readonly [_validated]: true };
type Unvalidated = { readonly [_validated]: false };

type FormData<V extends Validated | Unvalidated> = {
  name: string;
  email: string;
  age: number;
  readonly _validated: V;
};

type RawFormData = FormData<Unvalidated>;
type ValidatedFormData = FormData<Validated>;

function createRawData(data: {
  name: string;
  email: string;
  age: number;
}): RawFormData {
  return { ...data, _validated: false as any };
}

function validate(data: RawFormData): ValidatedFormData | Error {
  if (!data.name) return new Error("ชื่อต้องไม่ว่าง");
  if (!data.email.includes("@")) return new Error("อีเมลไม่ถูกต้อง");
  if (data.age < 0 || data.age > 150) return new Error("อายุไม่ถูกต้อง");

  return { ...data, _validated: true as any };
}

function saveToDatabase(data: ValidatedFormData): void {
  console.log("บันทึกข้อมูลที่ผ่านการตรวจสอบแล้ว:", data.name);
}

const raw = createRawData({
  name: "สมชาย",
  email: "somchai@example.com",
  age: 30,
});

const result = validate(raw);
if (result instanceof Error) {
  console.error(result.message);
} else {
  saveToDatabase(result); // ✅ ทำได้เพราะผ่าน validation แล้ว
}

// saveToDatabase(raw); // ❌ Error! ไม่สามารถบันทึกข้อมูลที่ยังไม่ validate
```

---

## 3. Branded Types สำหรับ Type-Safe IDs

Branded types ช่วยป้องกันการสับสนระหว่าง IDs ต่าง type กัน

### ตัวอย่างที่ 8: Basic Branded ID

```typescript
// การประกาศ Brand
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

// Branded ID types
type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;
type CommentId = Brand<string, "CommentId">;
type OrderId = Brand<string, "OrderId">;

// Constructor functions
const UserId = (id: string): UserId => id as UserId;
const PostId = (id: string): PostId => id as PostId;
const CommentId = (id: string): CommentId => id as CommentId;
const OrderId = (id: string): OrderId => id as OrderId;

// ใช้งาน
interface User {
  id: UserId;
  name: string;
}

interface Post {
  id: PostId;
  authorId: UserId;
  title: string;
  content: string;
}

interface Comment {
  id: CommentId;
  postId: PostId;
  authorId: UserId;
  content: string;
}

function getUser(id: UserId): User {
  return { id, name: "สมชาย" };
}

function getPost(id: PostId): Post {
  return {
    id,
    authorId: UserId("user-1"),
    title: "บทความแรก",
    content: "เนื้อหา...",
  };
}

const userId = UserId("user-123");
const postId = PostId("post-456");

getUser(userId); // ✅ OK
// getUser(postId); // ❌ Error! ส่ง PostId แทน UserId ไม่ได้

getPost(postId); // ✅ OK
// getPost(userId); // ❌ Error!
```

### ตัวอย่างที่ 9: Numeric Branded Types

```typescript
type PositiveNumber = Brand<number, "Positive">;
type NegativeNumber = Brand<number, "Negative">;
type NonZeroNumber = Brand<number, "NonZero">;
type Percentage = Brand<number, "Percentage">; // 0-100
type Probability = Brand<number, "Probability">; // 0-1

function toPositive(n: number): PositiveNumber | null {
  return n > 0 ? (n as PositiveNumber) : null;
}

function toPercentage(n: number): Percentage | null {
  return n >= 0 && n <= 100 ? (n as Percentage) : null;
}

function toProbability(n: number): Probability | null {
  return n >= 0 && n <= 1 ? (n as Probability) : null;
}

function divide(a: number, b: NonZeroNumber): number {
  return a / b;
}

// ต้องตรวจสอบก่อนใช้
const divisor = 5 as NonZeroNumber; // ต้องมั่นใจเองว่าไม่ใช่ 0

function safeDivide(a: number, b: number): number | null {
  if (b === 0) return null;
  return divide(a, b as NonZeroNumber);
}

// การใช้งาน Percentage
function applyDiscount(price: number, discount: Percentage): number {
  return price * (1 - discount / 100);
}

const discountPercent = toPercentage(20);
if (discountPercent !== null) {
  const finalPrice = applyDiscount(1000, discountPercent);
  console.log(`ราคาหลังหักส่วนลด: ${finalPrice} บาท`);
}
```

### ตัวอย่างที่ 10: Opaque Types Library Pattern

```typescript
// Generic Opaque Type helper
type Opaque<T, K extends string> = T & { readonly __opaque_type: K };

// Domain types
type Email = Opaque<string, "Email">;
type PhoneNumber = Opaque<string, "PhoneNumber">;
type Password = Opaque<string, "Password">;
type HashedPassword = Opaque<string, "HashedPassword">;
type Salt = Opaque<string, "Salt">;
type JWT = Opaque<string, "JWT">;

// Smart constructors (validation included)
const Email = {
  create(value: string): Email | Error {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value)) {
      return new Error(`"${value}" ไม่ใช่ email ที่ถูกต้อง`);
    }
    return value as Email;
  },
  getValue(email: Email): string {
    return email;
  },
};

const PhoneNumber = {
  create(value: string): PhoneNumber | Error {
    const cleaned = value.replace(/\D/g, "");
    if (cleaned.length !== 10) {
      return new Error(`"${value}" ไม่ใช่เบอร์โทรที่ถูกต้อง`);
    }
    return cleaned as PhoneNumber;
  },
};

const HashedPassword = {
  // ในความเป็นจริงจะใช้ bcrypt
  hash(password: Password, salt: Salt): HashedPassword {
    return `${salt}:${password}` as HashedPassword;
  },
  verify(password: Password, hash: HashedPassword, salt: Salt): boolean {
    return HashedPassword.hash(password, salt) === hash;
  },
};

// ตัวอย่างการใช้งาน
const emailResult = Email.create("user@example.com");
if (!(emailResult instanceof Error)) {
  console.log("Email ถูกต้อง:", Email.getValue(emailResult));
}

const phoneResult = PhoneNumber.create("0812345678");
if (!(phoneResult instanceof Error)) {
  console.log("เบอร์โทรถูกต้อง:", phoneResult);
}
```

---

## 4. State Machines with Types

### ตัวอย่างที่ 11: Order State Machine

```typescript
// States
type OrderState =
  | "pending"
  | "confirmed"
  | "preparing"
  | "ready"
  | "delivering"
  | "delivered"
  | "cancelled";

// Events
type OrderEvent =
  | { type: "CONFIRM"; paymentId: string }
  | { type: "START_PREPARING" }
  | { type: "READY_FOR_PICKUP" }
  | { type: "ASSIGN_DELIVERY"; deliveryPersonId: string }
  | { type: "DELIVERED" }
  | { type: "CANCEL"; reason: string };

// Transitions type
type Transitions = {
  [S in OrderState]: {
    [E in OrderEvent["type"]]?: OrderState;
  };
};

const orderTransitions: Transitions = {
  pending: {
    CONFIRM: "confirmed",
    CANCEL: "cancelled",
  },
  confirmed: {
    START_PREPARING: "preparing",
    CANCEL: "cancelled",
  },
  preparing: {
    READY_FOR_PICKUP: "ready",
    CANCEL: "cancelled",
  },
  ready: {
    ASSIGN_DELIVERY: "delivering",
    CANCEL: "cancelled",
  },
  delivering: {
    DELIVERED: "delivered",
  },
  delivered: {},
  cancelled: {},
};

type StateMachine<S extends string, E extends { type: string }> = {
  state: S;
  send(event: E): StateMachine<S, E> | Error;
};

function createOrderMachine(
  initialState: OrderState = "pending"
): StateMachine<OrderState, OrderEvent> {
  let currentState = initialState;

  return {
    get state() {
      return currentState;
    },
    send(event: OrderEvent) {
      const transitions = orderTransitions[currentState];
      const nextState = transitions[event.type];

      if (!nextState) {
        return new Error(
          `ไม่สามารถทำ "${event.type}" ใน state "${currentState}" ได้`
        );
      }

      currentState = nextState;
      console.log(`สถานะเปลี่ยนจาก "${currentState}" ← "${event.type}"`);

      return this;
    },
  };
}

// การใช้งาน
const order = createOrderMachine();
console.log("สถานะเริ่มต้น:", order.state); // pending

order.send({ type: "CONFIRM", paymentId: "pay-123" });
console.log("หลัง confirm:", order.state); // confirmed

order.send({ type: "START_PREPARING" });
console.log("กำลังเตรียม:", order.state); // preparing

const error = order.send({ type: "CONFIRM", paymentId: "pay-456" });
if (error instanceof Error) {
  console.error("ข้อผิดพลาด:", error.message);
}
```

### ตัวอย่างที่ 12: Type-Safe State Machine ด้วย Generic

```typescript
type StateNode<S extends string, E extends string> = {
  on?: Partial<Record<E, S>>;
  entry?: () => void;
  exit?: () => void;
};

type MachineConfig<S extends string, E extends string> = {
  initial: S;
  states: Record<S, StateNode<S, E>>;
};

class TypedStateMachine<S extends string, E extends string> {
  private currentState: S;
  private config: MachineConfig<S, E>;

  constructor(config: MachineConfig<S, E>) {
    this.config = config;
    this.currentState = config.initial;
    config.states[config.initial].entry?.();
  }

  get state(): S {
    return this.currentState;
  }

  send(event: E): boolean {
    const stateConfig = this.config.states[this.currentState];
    const nextState = stateConfig.on?.[event];

    if (!nextState) {
      return false;
    }

    stateConfig.exit?.();
    this.currentState = nextState;
    this.config.states[nextState].entry?.();
    return true;
  }

  matches(state: S): boolean {
    return this.currentState === state;
  }
}

// ตัวอย่าง: Authentication Machine
type AuthState = "unauthenticated" | "authenticating" | "authenticated" | "error";
type AuthEvent = "LOGIN" | "SUCCESS" | "FAILURE" | "LOGOUT" | "RETRY";

const authMachine = new TypedStateMachine<AuthState, AuthEvent>({
  initial: "unauthenticated",
  states: {
    unauthenticated: {
      on: { LOGIN: "authenticating" },
      entry: () => console.log("กรุณาเข้าสู่ระบบ"),
    },
    authenticating: {
      on: {
        SUCCESS: "authenticated",
        FAILURE: "error",
      },
      entry: () => console.log("กำลังตรวจสอบข้อมูล..."),
    },
    authenticated: {
      on: { LOGOUT: "unauthenticated" },
      entry: () => console.log("เข้าสู่ระบบสำเร็จ!"),
      exit: () => console.log("กำลังออกจากระบบ..."),
    },
    error: {
      on: {
        RETRY: "authenticating",
        LOGOUT: "unauthenticated",
      },
      entry: () => console.log("เกิดข้อผิดพลาด กรุณาลองใหม่"),
    },
  },
});

authMachine.send("LOGIN");
authMachine.send("SUCCESS");
console.log("สถานะปัจจุบัน:", authMachine.state);
authMachine.send("LOGOUT");
```

---

## 5. Proof-Carrying Types

### ตัวอย่างที่ 13: Non-Empty List

```typescript
// NonEmptyArray: รับประกันว่ามีอย่างน้อย 1 element
type NonEmptyArray<T> = [T, ...T[]];

function head<T>(arr: NonEmptyArray<T>): T {
  return arr[0]; // ✅ ปลอดภัยเพราะรู้ว่ามีอย่างน้อย 1 element
}

function last<T>(arr: NonEmptyArray<T>): T {
  return arr[arr.length - 1] as T;
}

function toNonEmpty<T>(arr: T[]): NonEmptyArray<T> | null {
  if (arr.length === 0) return null;
  return arr as NonEmptyArray<T>;
}

function reduce<T>(arr: NonEmptyArray<T>, fn: (acc: T, curr: T) => T): T {
  const [first, ...rest] = arr;
  return rest.reduce(fn, first);
}

// ตัวอย่างการใช้งาน
const numbers: NonEmptyArray<number> = [1, 2, 3, 4, 5];
const firstNum = head(numbers); // number (ไม่ใช่ number | undefined)
const lastNum = last(numbers); // number

const sum = reduce(numbers, (a, b) => a + b);
console.log(`ผลรวม: ${sum}`); // 15

const max = reduce(numbers, Math.max);
console.log(`ค่าสูงสุด: ${max}`); // 5
```

### ตัวอย่างที่ 14: Sorted Array Proof

```typescript
// Phantom type สำหรับ sorted array
declare const _sorted: unique symbol;
type Sorted<T> = T[] & { readonly [_sorted]: true };

function sort<T>(arr: T[], compareFn?: (a: T, b: T) => number): Sorted<T> {
  return [...arr].sort(compareFn) as Sorted<T>;
}

// Binary search ทำงานได้เฉพาะกับ sorted array
function binarySearch<T>(
  sortedArr: Sorted<T>,
  target: T,
  compareFn: (a: T, b: T) => number
): number {
  let left = 0;
  let right = sortedArr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    const comparison = compareFn(sortedArr[mid], target);

    if (comparison === 0) return mid;
    if (comparison < 0) left = mid + 1;
    else right = mid - 1;
  }

  return -1;
}

const sortedNumbers = sort([5, 3, 1, 4, 2]);
const index = binarySearch(sortedNumbers, 3, (a, b) => a - b);
console.log(`พบ 3 ที่ index: ${index}`); // 2

// binarySearch([1, 3, 2, 5], 3, (a, b) => a - b); // ❌ Error! ต้องใช้ Sorted array
```

### ตัวอย่างที่ 15: Bounded Range

```typescript
// Type-level range checking
type InRange<Min extends number, Max extends number> = number & {
  readonly _min: Min;
  readonly _max: Max;
};

function inRange<Min extends number, Max extends number>(
  value: number,
  min: Min,
  max: Max
): InRange<Min, Max> | null {
  if (value < min || value > max) return null;
  return value as InRange<Min, Max>;
}

// ใช้สำหรับ port numbers
type ValidPort = InRange<1, 65535>;

function createServer(port: ValidPort): void {
  console.log(`เริ่ม server ที่ port ${port}`);
}

const port = inRange(3000, 1, 65535);
if (port !== null) {
  createServer(port); // ✅ ปลอดภัย
}

// Array index type safety
type ArrayIndex<T extends readonly unknown[]> = Extract<
  keyof T,
  `${number}`
> extends `${infer N extends number}`
  ? N
  : never;
```

---

## 6. Protocol Encoding in Types

### ตัวอย่างที่ 16: Type-Safe Builder Pattern

```typescript
// HTTP Request Builder ที่ปลอดภัยด้วย types
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

type RequestBuilder<
  HasMethod extends boolean = false,
  HasUrl extends boolean = false,
  HasBody extends boolean = false
> = {
  method: HasMethod extends true
    ? (method: HttpMethod) => RequestBuilder<true, HasUrl, HasBody>
    : (method: HttpMethod) => RequestBuilder<true, HasUrl, HasBody>;
  url: HasUrl extends true
    ? (url: string) => RequestBuilder<HasMethod, true, HasBody>
    : (url: string) => RequestBuilder<HasMethod, true, HasBody>;
  body: (body: unknown) => RequestBuilder<HasMethod, HasUrl, true>;
  // build() ทำได้เฉพาะเมื่อมี method และ url แล้ว
  build: [HasMethod, HasUrl] extends [true, true]
    ? () => Request
    : never;
};

interface Request {
  method: HttpMethod;
  url: string;
  body?: unknown;
  headers?: Record<string, string>;
}

class HttpRequestBuilder {
  private _method?: HttpMethod;
  private _url?: string;
  private _body?: unknown;
  private _headers: Record<string, string> = {};

  setMethod(method: HttpMethod): this {
    this._method = method;
    return this;
  }

  setUrl(url: string): this {
    this._url = url;
    return this;
  }

  setBody(body: unknown): this {
    this._body = body;
    return this;
  }

  setHeader(key: string, value: string): this {
    this._headers[key] = value;
    return this;
  }

  build(): Request {
    if (!this._method) throw new Error("ต้องระบุ method");
    if (!this._url) throw new Error("ต้องระบุ URL");

    return {
      method: this._method,
      url: this._url,
      body: this._body,
      headers: this._headers,
    };
  }
}

// Fluent Builder ที่ Type Safe กว่า
type Step1 = { withMethod: (method: HttpMethod) => Step2 };
type Step2 = { toUrl: (url: string) => Step3 };
type Step3 = {
  withBody: (body: unknown) => Step3;
  withHeader: (key: string, value: string) => Step3;
  build: () => Request;
};

function createRequest(): Step1 {
  const req: Partial<Request> = {};

  return {
    withMethod: (method) => {
      req.method = method;
      return {
        toUrl: (url) => {
          req.url = url;
          return {
            withBody: (body) => {
              req.body = body;
              return this as any;
            },
            withHeader: (key, value) => {
              req.headers = { ...req.headers, [key]: value };
              return this as any;
            },
            build: () => req as Request,
          };
        },
      };
    },
  };
}

const request = createRequest()
  .withMethod("POST")
  .toUrl("/api/users")
  .withBody({ name: "สมชาย" })
  .withHeader("Content-Type", "application/json")
  .build();

console.log("Request:", request);
```

### ตัวอย่างที่ 17: Type-Safe SQL Query Builder

```typescript
type SQLOperator = "=" | "!=" | ">" | "<" | ">=" | "<=" | "LIKE" | "IN";

type WhereClause<T> = {
  field: keyof T;
  operator: SQLOperator;
  value: T[keyof T] | T[keyof T][];
};

type OrderDirection = "ASC" | "DESC";
type OrderClause<T> = {
  field: keyof T;
  direction: OrderDirection;
};

type QueryBuilder<T, HasTable extends boolean = false> = {
  from: (table: string) => QueryBuilder<T, true>;
  select: HasTable extends true
    ? (...fields: (keyof T)[]) => QueryBuilder<T, true>
    : never;
  where: HasTable extends true
    ? (clause: WhereClause<T>) => QueryBuilder<T, true>
    : never;
  orderBy: HasTable extends true
    ? (clause: OrderClause<T>) => QueryBuilder<T, true>
    : never;
  limit: HasTable extends true
    ? (n: number) => QueryBuilder<T, true>
    : never;
  build: HasTable extends true ? () => string : never;
};

// ตัวอย่างการใช้งาน
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

class TypedQueryBuilder<T> {
  private _table: string = "";
  private _fields: (keyof T)[] = [];
  private _where: WhereClause<T>[] = [];
  private _orderBy?: OrderClause<T>;
  private _limit?: number;

  from(table: string): this {
    this._table = table;
    return this;
  }

  select(...fields: (keyof T)[]): this {
    this._fields = fields;
    return this;
  }

  where(clause: WhereClause<T>): this {
    this._where.push(clause);
    return this;
  }

  orderBy(clause: OrderClause<T>): this {
    this._orderBy = clause;
    return this;
  }

  limit(n: number): this {
    this._limit = n;
    return this;
  }

  build(): string {
    const fields = this._fields.length > 0
      ? this._fields.join(", ")
      : "*";

    let query = `SELECT ${fields} FROM ${this._table}`;

    if (this._where.length > 0) {
      const conditions = this._where.map(
        (w) => `${String(w.field)} ${w.operator} '${w.value}'`
      );
      query += ` WHERE ${conditions.join(" AND ")}`;
    }

    if (this._orderBy) {
      query += ` ORDER BY ${String(this._orderBy.field)} ${this._orderBy.direction}`;
    }

    if (this._limit !== undefined) {
      query += ` LIMIT ${this._limit}`;
    }

    return query;
  }
}

const query = new TypedQueryBuilder<Product>()
  .from("products")
  .select("id", "name", "price")
  .where({ field: "category", operator: "=", value: "Electronics" })
  .where({ field: "inStock", operator: "=", value: true })
  .orderBy({ field: "price", direction: "ASC" })
  .limit(10)
  .build();

console.log(query);
// SELECT id, name, price FROM products
// WHERE category = 'Electronics' AND inStock = 'true'
// ORDER BY price ASC LIMIT 10
```

---

## 7. Parse Don't Validate

หลักการ "Parse Don't Validate" คือการแปลงข้อมูลดิบเป็น typed data แทนการ validate ข้อมูลดิบ

### ตัวอย่างที่ 18: Parser ที่ปลอดภัย

```typescript
type ParseResult<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function parseString(value: unknown): ParseResult<string> {
  if (typeof value === "string") {
    return { success: true, data: value };
  }
  return { success: false, error: `ค่า "${value}" ไม่ใช่ string` };
}

function parseNumber(value: unknown): ParseResult<number> {
  if (typeof value === "number" && !isNaN(value)) {
    return { success: true, data: value };
  }
  if (typeof value === "string") {
    const num = parseFloat(value);
    if (!isNaN(num)) {
      return { success: true, data: num };
    }
  }
  return { success: false, error: `ค่า "${value}" ไม่ใช่ตัวเลข` };
}

function parseBoolean(value: unknown): ParseResult<boolean> {
  if (typeof value === "boolean") {
    return { success: true, data: value };
  }
  if (value === "true") return { success: true, data: true };
  if (value === "false") return { success: true, data: false };
  return { success: false, error: `ค่า "${value}" ไม่ใช่ boolean` };
}

function parseArray<T>(
  value: unknown,
  itemParser: (item: unknown) => ParseResult<T>
): ParseResult<T[]> {
  if (!Array.isArray(value)) {
    return { success: false, error: "ค่าไม่ใช่ array" };
  }

  const results: T[] = [];
  for (let i = 0; i < value.length; i++) {
    const result = itemParser(value[i]);
    if (!result.success) {
      return { success: false, error: `index ${i}: ${result.error}` };
    }
    results.push(result.data);
  }

  return { success: true, data: results };
}

function parseObject<T extends Record<string, unknown>>(
  value: unknown,
  schema: { [K in keyof T]: (v: unknown) => ParseResult<T[K]> }
): ParseResult<T> {
  if (typeof value !== "object" || value === null) {
    return { success: false, error: "ค่าไม่ใช่ object" };
  }

  const obj = value as Record<string, unknown>;
  const result: Partial<T> = {};

  for (const [key, parser] of Object.entries(schema) as [
    keyof T,
    (v: unknown) => ParseResult<T[keyof T]>
  ][]) {
    const parseResult = parser(obj[key as string]);
    if (!parseResult.success) {
      return { success: false, error: `field "${String(key)}": ${parseResult.error}` };
    }
    result[key] = parseResult.data;
  }

  return { success: true, data: result as T };
}

// ตัวอย่าง: Parse User จาก API response
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  isActive: boolean;
}

const parseUser = (value: unknown): ParseResult<User> =>
  parseObject<User>(value, {
    id: parseNumber,
    name: parseString,
    email: parseString,
    age: parseNumber,
    isActive: parseBoolean,
  });

// ทดสอบ
const rawApiResponse = {
  id: 1,
  name: "สมชาย",
  email: "somchai@example.com",
  age: 30,
  isActive: true,
};

const parsed = parseUser(rawApiResponse);
if (parsed.success) {
  console.log("Parse สำเร็จ:", parsed.data);
} else {
  console.error("Parse ล้มเหลว:", parsed.error);
}

// ทดสอบข้อมูลผิด
const badData = {
  id: "not-a-number",
  name: "สมหญิง",
  email: "invalid",
  age: 25,
  isActive: "yes",
};

const parsedBad = parseUser(badData);
if (!parsedBad.success) {
  console.error("ข้อผิดพลาด:", parsedBad.error);
  // field "id": ค่า "not-a-number" ไม่ใช่ตัวเลข
}
```

---

## 8. Smart Constructors

### ตัวอย่างที่ 19: Smart Constructors Pattern

```typescript
// Smart constructors สำหรับ domain types

// Email
class Email {
  private constructor(readonly value: string) {}

  static create(value: string): Email | Error {
    const trimmed = value.trim().toLowerCase();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(trimmed)) {
      return new Error(`"${value}" ไม่ใช่ email ที่ถูกต้อง`);
    }
    return new Email(trimmed);
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}

// Age
class Age {
  private constructor(readonly value: number) {}

  static create(value: number): Age | Error {
    if (!Number.isInteger(value)) {
      return new Error("อายุต้องเป็นจำนวนเต็ม");
    }
    if (value < 0 || value > 150) {
      return new Error(`อายุ ${value} ไม่อยู่ในช่วงที่ถูกต้อง (0-150)`);
    }
    return new Age(value);
  }

  isAdult(): boolean {
    return this.value >= 18;
  }

  toString(): string {
    return `${this.value} ปี`;
  }
}

// Username
class Username {
  private constructor(readonly value: string) {}

  static create(value: string): Username | Error {
    if (value.length < 3) {
      return new Error("ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร");
    }
    if (value.length > 50) {
      return new Error("ชื่อผู้ใช้ต้องไม่เกิน 50 ตัวอักษร");
    }
    if (!/^[a-zA-Z0-9_]+$/.test(value)) {
      return new Error("ชื่อผู้ใช้ใช้ได้เฉพาะ a-z, A-Z, 0-9 และ _");
    }
    return new Username(value);
  }
}

// Result type สำหรับ composition
class Result<T> {
  private constructor(
    private readonly _value: T | null,
    private readonly _error: Error | null
  ) {}

  static ok<T>(value: T): Result<T> {
    return new Result(value, null);
  }

  static err<T>(error: Error): Result<T> {
    return new Result<T>(null, error);
  }

  static from<T>(value: T | Error): Result<T> {
    if (value instanceof Error) return Result.err(value);
    return Result.ok(value);
  }

  isOk(): this is Result<T> & { value: T } {
    return this._error === null;
  }

  map<U>(fn: (value: T) => U): Result<U> {
    if (this._error !== null) return Result.err(this._error);
    return Result.ok(fn(this._value!));
  }

  flatMap<U>(fn: (value: T) => Result<U>): Result<U> {
    if (this._error !== null) return Result.err(this._error);
    return fn(this._value!);
  }

  getOrElse(defaultValue: T): T {
    return this._error !== null ? defaultValue : this._value!;
  }

  fold<U>(onError: (error: Error) => U, onSuccess: (value: T) => U): U {
    if (this._error !== null) return onError(this._error);
    return onSuccess(this._value!);
  }
}

// ตัวอย่างการสร้าง User
interface ValidUser {
  username: Username;
  email: Email;
  age: Age;
}

function createUser(
  username: string,
  email: string,
  age: number
): Result<ValidUser> {
  const usernameResult = Result.from(Username.create(username));
  const emailResult = Result.from(Email.create(email));
  const ageResult = Result.from(Age.create(age));

  if (!usernameResult.isOk()) {
    return Result.err(new Error(`Username: ${usernameResult._error?.message}`));
  }
  if (!emailResult.isOk()) {
    return Result.err(new Error(`Email: ${emailResult._error?.message}`));
  }
  if (!ageResult.isOk()) {
    return Result.err(new Error(`Age: ${ageResult._error?.message}`));
  }

  return Result.ok({
    username: usernameResult._value!,
    email: emailResult._value!,
    age: ageResult._value!,
  });
}

const userResult = createUser("somchai_123", "somchai@example.com", 25);
userResult.fold(
  (err) => console.error("ไม่สามารถสร้าง user:", err.message),
  (user) => console.log(`สร้าง user สำเร็จ: ${user.username.value}`)
);
```

---

## 9. Invariant Enforcement Through Types

### ตัวอย่างที่ 20: Positive Balance Account

```typescript
// บัญชีที่รับประกันว่า balance ไม่ติดลบ
declare const _positiveBalance: unique symbol;
type PositiveBalance = number & { readonly [_positiveBalance]: true };

class BankAccount {
  private readonly _balance: PositiveBalance;
  private readonly _id: string;

  private constructor(id: string, balance: PositiveBalance) {
    this._id = id;
    this._balance = balance;
  }

  static create(id: string, initialBalance: number = 0): BankAccount | Error {
    if (initialBalance < 0) {
      return new Error("ยอดเริ่มต้นต้องไม่ติดลบ");
    }
    return new BankAccount(id, initialBalance as PositiveBalance);
  }

  get balance(): PositiveBalance {
    return this._balance;
  }

  get id(): string {
    return this._id;
  }

  deposit(amount: number): BankAccount | Error {
    if (amount <= 0) {
      return new Error("จำนวนเงินฝากต้องมากกว่า 0");
    }
    const newBalance = (this._balance + amount) as PositiveBalance;
    return new BankAccount(this._id, newBalance);
  }

  withdraw(amount: number): BankAccount | Error {
    if (amount <= 0) {
      return new Error("จำนวนเงินถอนต้องมากกว่า 0");
    }
    if (amount > this._balance) {
      return new Error(`ยอดเงินไม่เพียงพอ (มี ${this._balance}, ต้องการ ${amount})`);
    }
    const newBalance = (this._balance - amount) as PositiveBalance;
    return new BankAccount(this._id, newBalance);
  }

  transfer(amount: number, target: BankAccount): [BankAccount, BankAccount] | Error {
    const withdrawResult = this.withdraw(amount);
    if (withdrawResult instanceof Error) return withdrawResult;

    const depositResult = target.deposit(amount);
    if (depositResult instanceof Error) return depositResult;

    return [withdrawResult, depositResult];
  }
}

// ทดสอบ
const accountResult = BankAccount.create("ACC-001", 1000);
if (!(accountResult instanceof Error)) {
  const account = accountResult;
  console.log("ยอดเงินเริ่มต้น:", account.balance);

  const afterDeposit = account.deposit(500);
  if (!(afterDeposit instanceof Error)) {
    console.log("หลังฝาก 500:", afterDeposit.balance); // 1500
  }

  const afterWithdraw = account.withdraw(200);
  if (!(afterWithdraw instanceof Error)) {
    console.log("หลังถอน 200:", afterWithdraw.balance); // 800
  }

  const overWithdraw = account.withdraw(5000);
  if (overWithdraw instanceof Error) {
    console.error("ข้อผิดพลาด:", overWithdraw.message);
  }
}
```

### ตัวอย่างที่ 21: Immutable Sorted Collection

```typescript
// Collection ที่ guarantee ว่าข้อมูลเรียงลำดับเสมอ
class SortedSet<T> {
  private readonly _items: ReadonlyArray<T>;
  private readonly _compareFn: (a: T, b: T) => number;

  private constructor(items: T[], compareFn: (a: T, b: T) => number) {
    this._compareFn = compareFn;
    this._items = [...items].sort(compareFn);
  }

  static empty<T>(compareFn: (a: T, b: T) => number): SortedSet<T> {
    return new SortedSet<T>([], compareFn);
  }

  static from<T>(
    items: T[],
    compareFn: (a: T, b: T) => number
  ): SortedSet<T> {
    return new SortedSet(items, compareFn);
  }

  get size(): number {
    return this._items.length;
  }

  contains(item: T): boolean {
    return this._items.some((i) => this._compareFn(i, item) === 0);
  }

  add(item: T): SortedSet<T> {
    if (this.contains(item)) return this;
    return new SortedSet(
      [...this._items, item],
      this._compareFn
    );
  }

  remove(item: T): SortedSet<T> {
    return new SortedSet(
      this._items.filter((i) => this._compareFn(i, item) !== 0),
      this._compareFn
    );
  }

  toArray(): ReadonlyArray<T> {
    return this._items;
  }

  min(): T | undefined {
    return this._items[0];
  }

  max(): T | undefined {
    return this._items[this._items.length - 1];
  }

  // ค้นหาด้วย binary search (ทำได้เพราะรู้ว่าข้อมูลเรียงแล้ว)
  binarySearch(item: T): number {
    let left = 0;
    let right = this._items.length - 1;

    while (left <= right) {
      const mid = Math.floor((left + right) / 2);
      const comparison = this._compareFn(this._items[mid], item);

      if (comparison === 0) return mid;
      if (comparison < 0) left = mid + 1;
      else right = mid - 1;
    }

    return -1;
  }
}

// ตัวอย่างการใช้งาน
const numberSet = SortedSet.from([5, 3, 1, 4, 2], (a, b) => a - b);
console.log("Sorted set:", numberSet.toArray()); // [1, 2, 3, 4, 5]
console.log("Min:", numberSet.min()); // 1
console.log("Max:", numberSet.max()); // 5

const withSix = numberSet.add(6);
console.log("หลังเพิ่ม 6:", withSix.toArray()); // [1, 2, 3, 4, 5, 6]

const withoutThree = numberSet.remove(3);
console.log("หลังลบ 3:", withoutThree.toArray()); // [1, 2, 4, 5]
```

---

## 10. Type-Safe Finite State Automata

### ตัวอย่างที่ 22: Workflow Engine

```typescript
// Type-safe workflow engine
type WorkflowState = string;
type WorkflowAction = string;

type Transition<S extends WorkflowState, A extends WorkflowAction> = {
  from: S;
  action: A;
  to: S;
  condition?: () => boolean;
  onTransition?: (from: S, to: S) => void;
};

class WorkflowEngine<
  S extends WorkflowState,
  A extends WorkflowAction
> {
  private currentState: S;
  private transitions: Transition<S, A>[];
  private history: Array<{ from: S; to: S; action: A; at: Date }> = [];

  constructor(
    initialState: S,
    transitions: Transition<S, A>[]
  ) {
    this.currentState = initialState;
    this.transitions = transitions;
  }

  get state(): S {
    return this.currentState;
  }

  get stateHistory(): ReadonlyArray<{ from: S; to: S; action: A; at: Date }> {
    return this.history;
  }

  canPerform(action: A): boolean {
    const transition = this.findTransition(action);
    if (!transition) return false;
    if (transition.condition && !transition.condition()) return false;
    return true;
  }

  perform(action: A): boolean {
    if (!this.canPerform(action)) return false;

    const transition = this.findTransition(action)!;
    const from = this.currentState;
    const to = transition.to;

    this.currentState = to;
    this.history.push({ from, to, action, at: new Date() });
    transition.onTransition?.(from, to);

    return true;
  }

  private findTransition(action: A): Transition<S, A> | undefined {
    return this.transitions.find(
      (t) => t.from === this.currentState && t.action === action
    );
  }

  getAvailableActions(): A[] {
    return this.transitions
      .filter(
        (t) =>
          t.from === this.currentState &&
          (!t.condition || t.condition())
      )
      .map((t) => t.action);
  }
}

// ตัวอย่าง: Document Review Workflow
type DocumentState =
  | "draft"
  | "submitted"
  | "under_review"
  | "approved"
  | "rejected"
  | "published"
  | "archived";

type DocumentAction =
  | "SUBMIT"
  | "START_REVIEW"
  | "APPROVE"
  | "REJECT"
  | "REVISE"
  | "PUBLISH"
  | "ARCHIVE"
  | "RESUBMIT";

const documentWorkflow = new WorkflowEngine<DocumentState, DocumentAction>(
  "draft",
  [
    {
      from: "draft",
      action: "SUBMIT",
      to: "submitted",
      onTransition: (from, to) =>
        console.log(`เอกสารถูกส่งเพื่อรีวิว`),
    },
    {
      from: "submitted",
      action: "START_REVIEW",
      to: "under_review",
      onTransition: () => console.log("เริ่มรีวิวเอกสาร"),
    },
    {
      from: "under_review",
      action: "APPROVE",
      to: "approved",
      onTransition: () => console.log("เอกสารได้รับการอนุมัติ"),
    },
    {
      from: "under_review",
      action: "REJECT",
      to: "rejected",
      onTransition: () => console.log("เอกสารถูกปฏิเสธ"),
    },
    {
      from: "rejected",
      action: "REVISE",
      to: "draft",
      onTransition: () => console.log("กลับไปแก้ไข"),
    },
    {
      from: "approved",
      action: "PUBLISH",
      to: "published",
      onTransition: () => console.log("เผยแพร่เอกสาร"),
    },
    {
      from: "published",
      action: "ARCHIVE",
      to: "archived",
      onTransition: () => console.log("เก็บเอกสารในคลัง"),
    },
  ]
);

console.log("สถานะปัจจุบัน:", documentWorkflow.state);
console.log("Actions ที่ทำได้:", documentWorkflow.getAvailableActions());

documentWorkflow.perform("SUBMIT");
documentWorkflow.perform("START_REVIEW");
documentWorkflow.perform("APPROVE");
documentWorkflow.perform("PUBLISH");

console.log("ประวัติการเปลี่ยนสถานะ:");
documentWorkflow.stateHistory.forEach((h) => {
  console.log(`  ${h.from} → ${h.to} (${h.action})`);
});
```

---

## 11. Advanced Type-Level Programming

### ตัวอย่างที่ 23: Type-Safe Event System

```typescript
// Type-safe event emitter
type EventMap = Record<string, unknown>;

type EventListener<T> = (event: T) => void;

class TypedEventEmitter<Events extends EventMap> {
  private listeners: {
    [K in keyof Events]?: Set<EventListener<Events[K]>>;
  } = {};

  on<K extends keyof Events>(
    event: K,
    listener: EventListener<Events[K]>
  ): () => void {
    if (!this.listeners[event]) {
      this.listeners[event] = new Set();
    }
    this.listeners[event]!.add(listener);

    // Return unsubscribe function
    return () => this.off(event, listener);
  }

  off<K extends keyof Events>(
    event: K,
    listener: EventListener<Events[K]>
  ): void {
    this.listeners[event]?.delete(listener);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners[event]?.forEach((listener) => listener(data));
  }

  once<K extends keyof Events>(
    event: K,
    listener: EventListener<Events[K]>
  ): void {
    const unsubscribe = this.on(event, (data) => {
      listener(data);
      unsubscribe();
    });
  }
}

// ตัวอย่างการใช้งาน
interface AppEvents {
  userLogin: { userId: string; timestamp: Date };
  userLogout: { userId: string };
  orderCreated: { orderId: string; amount: number; userId: string };
  orderCompleted: { orderId: string; completedAt: Date };
  error: { message: string; code: number; stack?: string };
}

const appEvents = new TypedEventEmitter<AppEvents>();

// Subscribe
const unsubLogin = appEvents.on("userLogin", (event) => {
  console.log(`ผู้ใช้ ${event.userId} เข้าสู่ระบบเมื่อ ${event.timestamp}`);
});

appEvents.on("orderCreated", (event) => {
  console.log(
    `คำสั่งซื้อ ${event.orderId} มูลค่า ${event.amount} บาท จากผู้ใช้ ${event.userId}`
  );
});

appEvents.on("error", (event) => {
  console.error(`Error ${event.code}: ${event.message}`);
});

// Emit
appEvents.emit("userLogin", {
  userId: "user-1",
  timestamp: new Date(),
});

appEvents.emit("orderCreated", {
  orderId: "order-123",
  amount: 1500,
  userId: "user-1",
});

// Unsubscribe
unsubLogin();
```

### ตัวอย่างที่ 24: Type-Safe Configuration

```typescript
// Configuration ที่มีค่า default และ validation
type ConfigSchema = {
  [key: string]: {
    type: "string" | "number" | "boolean" | "object";
    required?: boolean;
    default?: unknown;
    validate?: (value: unknown) => boolean;
    description?: string;
  };
};

type ExtractConfigType<S extends ConfigSchema> = {
  [K in keyof S]: S[K]["type"] extends "string"
    ? string
    : S[K]["type"] extends "number"
    ? number
    : S[K]["type"] extends "boolean"
    ? boolean
    : S[K]["type"] extends "object"
    ? Record<string, unknown>
    : never;
};

function createConfig<S extends ConfigSchema>(
  schema: S,
  overrides: Partial<ExtractConfigType<S>> = {}
): ExtractConfigType<S> {
  const result: Record<string, unknown> = {};

  for (const [key, fieldSchema] of Object.entries(schema)) {
    const value = overrides[key as keyof S] ?? fieldSchema.default;

    if (value === undefined && fieldSchema.required) {
      throw new Error(`ต้องระบุ config "${key}"`);
    }

    if (value !== undefined && fieldSchema.validate && !fieldSchema.validate(value)) {
      throw new Error(`ค่า config "${key}" ไม่ถูกต้อง: ${value}`);
    }

    result[key] = value;
  }

  return result as ExtractConfigType<S>;
}

// ตัวอย่างการใช้งาน
const serverConfigSchema = {
  host: {
    type: "string" as const,
    required: true,
    default: "localhost",
    description: "Server hostname",
  },
  port: {
    type: "number" as const,
    required: true,
    default: 3000,
    validate: (v: unknown) => typeof v === "number" && v > 0 && v < 65536,
    description: "Server port (1-65535)",
  },
  debug: {
    type: "boolean" as const,
    default: false,
    description: "Enable debug mode",
  },
  maxConnections: {
    type: "number" as const,
    default: 100,
    validate: (v: unknown) => typeof v === "number" && v > 0,
    description: "Maximum concurrent connections",
  },
} satisfies ConfigSchema;

const config = createConfig(serverConfigSchema, {
  port: 8080,
  debug: true,
});

console.log("Config:", config);
// { host: "localhost", port: 8080, debug: true, maxConnections: 100 }
```

---

## 12. Advanced Examples: Real-World Type-Driven Patterns

### ตัวอย่างที่ 25: Type-Safe Repository Pattern

```typescript
// Repository Pattern ที่ปลอดภัยด้วย types
interface Entity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

type CreateInput<T extends Entity> = Omit<T, "id" | "createdAt" | "updatedAt">;
type UpdateInput<T extends Entity> = Partial<Omit<T, "id" | "createdAt" | "updatedAt">>;

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(input: CreateInput<T>): Promise<T>;
  update(id: string, input: UpdateInput<T>): Promise<T | null>;
  delete(id: string): Promise<boolean>;
  count(filter?: Partial<T>): Promise<number>;
}

// Memory implementation
class MemoryRepository<T extends Entity> implements Repository<T> {
  protected items: Map<string, T> = new Map();

  async findById(id: string): Promise<T | null> {
    return this.items.get(id) ?? null;
  }

  async findAll(filter?: Partial<T>): Promise<T[]> {
    const all = Array.from(this.items.values());
    if (!filter) return all;

    return all.filter((item) =>
      Object.entries(filter).every(
        ([key, value]) => item[key as keyof T] === value
      )
    );
  }

  async create(input: CreateInput<T>): Promise<T> {
    const now = new Date();
    const item = {
      ...input,
      id: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      createdAt: now,
      updatedAt: now,
    } as unknown as T;

    this.items.set(item.id, item);
    return item;
  }

  async update(id: string, input: UpdateInput<T>): Promise<T | null> {
    const existing = this.items.get(id);
    if (!existing) return null;

    const updated = {
      ...existing,
      ...input,
      updatedAt: new Date(),
    };

    this.items.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.items.delete(id);
  }

  async count(filter?: Partial<T>): Promise<number> {
    return (await this.findAll(filter)).length;
  }
}

// ตัวอย่าง domain entities
interface Product extends Entity {
  name: string;
  price: number;
  category: string;
  inStock: boolean;
  description: string;
}

interface Customer extends Entity {
  name: string;
  email: string;
  phone: string;
  tier: "bronze" | "silver" | "gold" | "platinum";
}

class ProductRepository extends MemoryRepository<Product> {
  async findByCategory(category: string): Promise<Product[]> {
    return this.findAll({ category } as Partial<Product>);
  }

  async findInStock(): Promise<Product[]> {
    return this.findAll({ inStock: true } as Partial<Product>);
  }

  async findByPriceRange(min: number, max: number): Promise<Product[]> {
    const all = await this.findAll();
    return all.filter((p) => p.price >= min && p.price <= max);
  }
}

// ใช้งาน
async function demonstrateRepository() {
  const repo = new ProductRepository();

  const laptop = await repo.create({
    name: "โน้ตบุ๊ค Pro 15",
    price: 45000,
    category: "Electronics",
    inStock: true,
    description: "โน้ตบุ๊คสำหรับงานหนัก",
  });

  const phone = await repo.create({
    name: "สมาร์ทโฟน X12",
    price: 25000,
    category: "Electronics",
    inStock: true,
    description: "สมาร์ทโฟนรุ่นล่าสุด",
  });

  console.log("สินค้าทั้งหมด:", await repo.findAll());
  console.log("สินค้าในสต็อก:", await repo.findInStock());
  console.log("ช่วงราคา 20000-30000:", await repo.findByPriceRange(20000, 30000));

  await repo.update(laptop.id, { price: 42000 });
  const updated = await repo.findById(laptop.id);
  console.log("ราคาหลังอัปเดต:", updated?.price);
}

demonstrateRepository();
```

---

## สรุป

Type-Driven Development เป็นแนวทางที่ทรงพลังสำหรับการพัฒนา TypeScript ที่:

1. **Making Illegal States Unrepresentable** - ออกแบบ type ให้ state ที่ invalid เป็นไปไม่ได้
2. **Phantom Types** - ใช้ type parameters ที่ไม่ปรากฏใน runtime เพื่อเพิ่มความปลอดภัย
3. **Branded Types** - สร้าง type-safe IDs และ primitive types
4. **State Machines** - encode business logic ใน type system
5. **Proof-Carrying Types** - type เป็น proof ของ property ของข้อมูล
6. **Parse Don't Validate** - แปลงข้อมูลดิบเป็น typed data
7. **Smart Constructors** - validate ตอนสร้าง, ใช้งานโดยไม่ต้อง validate ซ้ำ

การใช้ Type-Driven Development ช่วยให้:
- Bugs ถูกจับได้ตั้งแต่ compile time
- Code documentation อยู่ใน type definitions
- Refactoring ปลอดภัยขึ้น
- Business rules ถูก encode ใน type system
- Team communication ดีขึ้นผ่าน expressive types

---

*จบตอนที่ 76 - Type-Driven Development*
