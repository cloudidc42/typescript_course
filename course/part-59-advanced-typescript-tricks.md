# Part 59: Advanced TypeScript Tricks (เทคนิค TypeScript ขั้นสูง)

## บทนำ

ในบทนี้เราจะสำรวจ TypeScript type system อย่างลึกซึ้ง เรียนรู้การเขียน types ที่ซับซ้อน และใช้ประโยชน์จากระบบ types อย่างเต็มที่ด้วยกว่า 60 ตัวอย่าง

---

## ส่วนที่ 1: Type Gymnastics (การยิมนาสติก Type)

### ตัวอย่างที่ 1: DeepReadonly

```typescript
// DeepReadonly ทำให้ทุก property ใน object เป็น readonly แบบ recursive
type DeepReadonly<T> =
  T extends (infer U)[]
    ? ReadonlyArray<DeepReadonly<U>>
    : T extends Map<infer K, infer V>
    ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
    : T extends Set<infer U>
    ? ReadonlySet<DeepReadonly<U>>
    : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;

// ตัวอย่างการใช้งาน
interface AppConfig {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
  features: string[];
  settings: Map<string, unknown>;
}

type ImmutableConfig = DeepReadonly<AppConfig>;

const config: ImmutableConfig = {
  database: {
    host: "localhost",
    port: 5432,
    credentials: {
      username: "admin",
      password: "secret"
    }
  },
  features: ["auth", "payments"],
  settings: new Map()
};

// config.database.host = "new"; // Error!
// config.database.credentials.password = "hack"; // Error!
// config.features.push("new"); // Error!
```

---

### ตัวอย่างที่ 2: DeepPartial

```typescript
// DeepPartial ทำให้ทุก property เป็น optional แบบ recursive
type DeepPartial<T> =
  T extends Array<infer U>
    ? Array<DeepPartial<U>>
    : T extends Map<infer K, infer V>
    ? Map<K, DeepPartial<V>>
    : T extends object
    ? { [K in keyof T]?: DeepPartial<T[K]> }
    : T;

// ตัวอย่างการใช้งาน - good for update operations
interface UserProfile {
  name: string;
  address: {
    street: string;
    city: string;
    country: string;
  };
  preferences: {
    theme: "light" | "dark";
    language: string;
    notifications: {
      email: boolean;
      push: boolean;
    };
  };
}

type PartialUserUpdate = DeepPartial<UserProfile>;

function updateUser(id: string, update: PartialUserUpdate): void {
  // สามารถ update เฉพาะส่วนที่ต้องการได้
}

updateUser("123", {
  preferences: {
    theme: "dark"
    // ไม่ต้องระบุ language หรือ notifications
  }
});
```

---

### ตัวอย่างที่ 3: DeepRequired

```typescript
// DeepRequired ทำให้ทุก property เป็น required แบบ recursive
type DeepRequired<T> =
  T extends Array<infer U>
    ? Array<DeepRequired<U>>
    : T extends object
    ? { [K in keyof T]-?: DeepRequired<T[K]> }
    : T;

// ตัวอย่างการใช้งาน
interface Config {
  api?: {
    baseUrl?: string;
    timeout?: number;
    retry?: {
      count?: number;
      delay?: number;
    };
  };
}

type RequiredConfig = DeepRequired<Config>;

const config: RequiredConfig = {
  api: {
    baseUrl: "https://api.example.com",
    timeout: 5000,
    retry: {
      count: 3,
      delay: 1000
    }
  }
};
```

---

### ตัวอย่างที่ 4: DeepMutable (ลบ readonly แบบ recursive)

```typescript
type DeepMutable<T> =
  T extends ReadonlyArray<infer U>
    ? Array<DeepMutable<U>>
    : T extends ReadonlyMap<infer K, infer V>
    ? Map<DeepMutable<K>, DeepMutable<V>>
    : T extends ReadonlySet<infer U>
    ? Set<DeepMutable<U>>
    : T extends object
    ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
    : T;

// Test
type Frozen = Readonly<{ x: Readonly<{ y: number }> }>;
type Mutable = DeepMutable<Frozen>;
// { x: { y: number } } - ลบ readonly ออกทั้งหมด
```

---

### ตัวอย่างที่ 5: Recursive Conditional Types

```typescript
// ลบ types ใน nested arrays
type FlattenArray<T> =
  T extends Array<infer U>
    ? FlattenArray<U>
    : T;

type Nested = number[][][][];
type Flat = FlattenArray<Nested>; // number

// Unwrap Promise
type Awaited<T> =
  T extends Promise<infer U>
    ? Awaited<U>
    : T;

type Result = Awaited<Promise<Promise<string>>>;
// string

// Deep pick
type DeepPick<T, K extends string> =
  K extends `${infer Head}.${infer Tail}`
    ? Head extends keyof T
      ? { [P in Head]: DeepPick<T[P], Tail> }
      : never
    : K extends keyof T
    ? Pick<T, K>
    : never;

interface Config {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
  server: {
    port: number;
  };
}

type HostOnly = DeepPick<Config, "database.host">;
// { database: { host: string } }

type CredentialsOnly = DeepPick<Config, "database.credentials.username">;
// { database: { credentials: { username: string } } }
```

---

## ส่วนที่ 2: String Manipulation Types

### ตัวอย่างที่ 6: String Operations

```typescript
// Built-in String Manipulation Types
type Upper = Uppercase<"hello">; // "HELLO"
type Lower = Lowercase<"HELLO">; // "hello"
type Cap = Capitalize<"hello">; // "Hello"
type Uncap = Uncapitalize<"Hello">; // "hello"

// Split string type
type Split<
  S extends string,
  Delimiter extends string
> = S extends `${infer Head}${Delimiter}${infer Tail}`
  ? [Head, ...Split<Tail, Delimiter>]
  : [S];

type Parts = Split<"a.b.c.d", ".">;
// ["a", "b", "c", "d"]

// Join string type
type Join<T extends string[], Delimiter extends string> =
  T extends []
    ? ""
    : T extends [infer F extends string]
    ? F
    : T extends [infer F extends string, ...infer R extends string[]]
    ? `${F}${Delimiter}${Join<R, Delimiter>}`
    : never;

type Joined = Join<["a", "b", "c"], ".">;
// "a.b.c"

// CamelCase converter
type CamelCase<S extends string> =
  S extends `${infer Head}_${infer Tail}`
    ? `${Head}${Capitalize<CamelCase<Tail>>}`
    : S;

type CamelResult = CamelCase<"my_variable_name">; // "myVariableName"

// SnakeCase converter
type SnakeCase<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Tail extends `${Uppercase<Tail>}`
      ? `${Head}_${Lowercase<Tail>}`
      : `${Head}${SnakeCase<Tail>}`
    : S;

// KebabCase
type KebabCase<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Head extends Uppercase<Head>
      ? `${Head extends Uppercase<string> ? "-" : ""}${Lowercase<Head>}${KebabCase<Tail>}`
      : `${Head}${KebabCase<Tail>}`
    : S;
```

---

### ตัวอย่างที่ 7: Template Literal Advanced

```typescript
// Path parameters extraction
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractRouteParams<Rest>]: string }
    : T extends `${string}:${infer Param}`
    ? { [K in Param]: string }
    : {};

type UserRoute = ExtractRouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }

// Type-safe URL builder
function buildUrl<T extends string>(
  template: T,
  params: ExtractRouteParams<T>
): string {
  let url = template as string;
  for (const [key, value] of Object.entries(params as Record<string, string>)) {
    url = url.replace(`:${key}`, value);
  }
  return url;
}

const url = buildUrl("/users/:userId/posts/:postId", {
  userId: "123",
  postId: "456"
});
// "/users/123/posts/456"

// CSS Class builder types
type ClassValue = string | number | boolean | null | undefined;
type ClassArray = ClassValue[];
type ClassDictionary = Record<string, unknown>;
type ClassInput = ClassValue | ClassArray | ClassDictionary;

// Variant props pattern
type Variant<T extends Record<string, Record<string, string>>> = {
  [K in keyof T]?: keyof T[K];
};

const buttonVariants = {
  intent: {
    primary: "bg-blue-500",
    secondary: "bg-gray-500",
    danger: "bg-red-500"
  },
  size: {
    sm: "text-sm",
    md: "text-base",
    lg: "text-lg"
  }
} as const;

type ButtonProps = Variant<typeof buttonVariants>;
// { intent?: "primary" | "secondary" | "danger"; size?: "sm" | "md" | "lg"; }
```

---

## ส่วนที่ 3: Number Manipulation Types

### ตัวอย่างที่ 8: Type-level Arithmetic

```typescript
// สร้าง array ขนาด N
type BuildTuple<N extends number, T extends unknown[] = []> =
  T["length"] extends N ? T : BuildTuple<N, [...T, unknown]>;

type Tuple5 = BuildTuple<5>; // [unknown, unknown, unknown, unknown, unknown]

// Addition
type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"];

type Sum = Add<3, 4>; // 7

// Subtraction
type Subtract<A extends number, B extends number> =
  BuildTuple<A> extends [...BuildTuple<B>, ...infer Rest]
    ? Rest["length"]
    : never;

type Diff = Subtract<10, 3>; // 7

// Comparison
type LessThan<A extends number, B extends number> =
  BuildTuple<A> extends [...BuildTuple<B>, ...infer _]
    ? false
    : BuildTuple<B> extends [...BuildTuple<A>, ...infer _]
    ? true
    : false;

type IsLess = LessThan<3, 5>; // true
type IsLess2 = LessThan<5, 3>; // false

// Range type
type Range<
  Start extends number,
  End extends number,
  Acc extends number[] = []
> = Start extends End
  ? Acc
  : Range<Add<Start, 1> extends number ? Add<Start, 1> : never, End, [...Acc, Start]>;

type OneToFive = Range<1, 6>; // [1, 2, 3, 4, 5]
```

---

## ส่วนที่ 4: Branded Types (Type Tagging)

### ตัวอย่างที่ 9: Advanced Branded Types

```typescript
// ตัวอย่างการสร้าง brand
declare const __brand: unique symbol;

type Brand<Base, BrandTag extends string> = Base & {
  readonly [__brand]: BrandTag;
};

// Domain-specific types
type Email = Brand<string, "Email">;
type Username = Brand<string, "Username">;
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type Money = Brand<number, "Money">;
type Percentage = Brand<number, "Percentage">;
type Timestamp = Brand<number, "Timestamp">;
type Milliseconds = Brand<number, "Milliseconds">;

// Validators/constructors
function createEmail(value: string): Email {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
    throw new Error(`Invalid email: ${value}`);
  }
  return value as Email;
}

function createMoney(amount: number): Money {
  if (amount < 0) throw new Error("Money cannot be negative");
  return Math.round(amount * 100) / 100 as Money;
}

function createPercentage(value: number): Percentage {
  if (value < 0 || value > 100) throw new Error("Percentage must be 0-100");
  return value as Percentage;
}

// ใช้งาน
function sendEmail(to: Email, subject: string): void {
  console.log(`Sending to ${to}: ${subject}`);
}

const email = createEmail("user@example.com");
sendEmail(email, "Welcome!");
// sendEmail("user@example.com", "Welcome!"); // Error! string ไม่ใช่ Email

// Type-safe money operations
function addMoney(a: Money, b: Money): Money {
  return (a + b) as Money;
}

function applyDiscount(price: Money, discount: Percentage): Money {
  return createMoney(price * (1 - discount / 100));
}

const price = createMoney(100);
const discount = createPercentage(20);
const discounted = applyDiscount(price, discount);

// Nominal ID types
function getUser(id: UserId): void {}
function getOrder(id: OrderId): void {}

const userId = "user-123" as UserId;
const orderId = "order-456" as OrderId;

getUser(userId);
// getUser(orderId); // Error! OrderId ไม่ใช่ UserId
```

---

### ตัวอย่างที่ 10: Opaque Types with Runtime Validation

```typescript
// Opaque type factory
function createOpaqueType<T, Brand extends string>(
  validate: (value: unknown) => value is T,
  errorMessage: string
) {
  type Opaque = T & { readonly __brand: Brand };
  
  return {
    create(value: T): Opaque {
      if (!validate(value)) {
        throw new Error(errorMessage);
      }
      return value as Opaque;
    },
    is(value: unknown): value is Opaque {
      return validate(value);
    },
    unsafe(value: T): Opaque {
      return value as Opaque;
    }
  };
}

// Create specific types
const EmailType = createOpaqueType<string, "Email">(
  (v): v is string => typeof v === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
  "Invalid email format"
);

const PositiveIntType = createOpaqueType<number, "PositiveInt">(
  (v): v is number => typeof v === "number" && Number.isInteger(v) && v > 0,
  "Must be a positive integer"
);

// การใช้งาน
type Email = ReturnType<typeof EmailType.create>;
type PositiveInt = ReturnType<typeof PositiveIntType.create>;

function createUser(email: Email, age: PositiveInt) {
  return { email, age };
}

const user = createUser(
  EmailType.create("john@example.com"),
  PositiveIntType.create(25)
);

// สามารถ validate ได้
if (EmailType.is("not-email")) {
  // ไม่เข้า block นี้
}
```

---

## ส่วนที่ 5: Phantom Types

### ตัวอย่างที่ 11: State Machine with Phantom Types

```typescript
// Type-safe State Machine
declare const _phantom: unique symbol;
type PhantomType<T, S> = T & { readonly [_phantom]: S };

// States
type Pending = { readonly type: "pending" };
type Active = { readonly type: "active" };
type Completed = { readonly type: "completed" };
type Cancelled = { readonly type: "cancelled" };

// Entity with state
type Order<State> = PhantomType<{
  id: string;
  items: string[];
  total: number;
}, State>;

// State-specific constructors
function createOrder(items: string[], total: number): Order<Pending> {
  return { id: Math.random().toString(), items, total } as Order<Pending>;
}

// State transitions - type enforce valid transitions
function activateOrder(order: Order<Pending>): Order<Active> {
  return order as unknown as Order<Active>;
}

function completeOrder(order: Order<Active>): Order<Completed> {
  return order as unknown as Order<Completed>;
}

function cancelOrder(order: Order<Pending> | Order<Active>): Order<Cancelled> {
  return order as unknown as Order<Cancelled>;
}

// State-specific operations
function processPayment(order: Order<Active>): void {
  console.log(`Processing payment for order ${(order as any).id}`);
}

// ใช้งาน
const pending = createOrder(["item1", "item2"], 100);
const active = activateOrder(pending);
processPayment(active);
// processPayment(pending); // Error! Pending ไม่ใช่ Active

const completed = completeOrder(active);
// completeOrder(pending); // Error! Pending ไม่ใช่ Active
```

---

### ตัวอย่างที่ 12: Validated Types (Refined Types)

```typescript
// ตัวอย่าง: Non-empty array
type NonEmptyArray<T> = [T, ...T[]];

function first<T>(arr: NonEmptyArray<T>): T {
  return arr[0];
}

const empty: never[] = [];
// first(empty); // Error! empty ไม่ใช่ NonEmptyArray

const nonEmpty: NonEmptyArray<number> = [1, 2, 3];
const f = first(nonEmpty); // type: number ไม่ใช่ number | undefined

// Exact object type
type Exact<T, Shape = T> =
  T extends Shape
    ? Exclude<keyof T, keyof Shape> extends never
      ? T
      : never
    : never;

function exactMatch<T>(obj: Exact<T, { name: string; age: number }>): void {
  console.log(obj);
}

exactMatch({ name: "John", age: 30 }); // OK
// exactMatch({ name: "John", age: 30, extra: "field" }); // Error!

// Type-safe tuple of length N
type TupleOf<T, N extends number, Acc extends T[] = []> =
  Acc["length"] extends N ? Acc : TupleOf<T, N, [...Acc, T]>;

type Pair<T> = TupleOf<T, 2>;
type Triple<T> = TupleOf<T, 3>;

const pair: Pair<string> = ["hello", "world"]; // OK
// const pair2: Pair<string> = ["a", "b", "c"]; // Error! length 3 ไม่ใช่ 2
```

---

## ส่วนที่ 6: Type-safe Patterns

### ตัวอย่างที่ 13: Type-safe Builder Pattern

```typescript
// Step-by-step builder ที่ enforce ลำดับ
type BuilderState = {
  hasName: boolean;
  hasAge: boolean;
  hasEmail: boolean;
};

class PersonBuilder<State extends BuilderState = {
  hasName: false;
  hasAge: false;
  hasEmail: false;
}> {
  private data: Partial<{ name: string; age: number; email: string }> = {};
  
  setName(name: string): PersonBuilder<State & { hasName: true }> {
    this.data.name = name;
    return this as any;
  }
  
  setAge(age: number): PersonBuilder<State & { hasAge: true }> {
    this.data.age = age;
    return this as any;
  }
  
  setEmail(email: string): PersonBuilder<State & { hasEmail: true }> {
    this.data.email = email;
    return this as any;
  }
  
  // ต้องกรอกทุก required field ก่อน build
  build(
    this: PersonBuilder<{
      hasName: true;
      hasAge: true;
      hasEmail: true;
    }>
  ): { name: string; age: number; email: string } {
    return this.data as { name: string; age: number; email: string };
  }
}

// ใช้งาน
const person = new PersonBuilder()
  .setName("John")
  .setAge(30)
  .setEmail("john@example.com")
  .build(); // OK

// new PersonBuilder().setName("John").build(); // Error! hasAge และ hasEmail เป็น false
```

---

### ตัวอย่างที่ 14: Type-safe Router

```typescript
// Type-safe Router
type RouteConfig = {
  path: string;
  params?: Record<string, string>;
  query?: Record<string, string | string[]>;
};

type Routes = {
  home: { path: "/"; params: {}; query: {} };
  userProfile: { path: "/users/:userId"; params: { userId: string }; query: {} };
  userPosts: {
    path: "/users/:userId/posts";
    params: { userId: string };
    query: { page?: string; limit?: string };
  };
  postDetail: {
    path: "/posts/:postId";
    params: { postId: string };
    query: { ref?: string };
  };
};

type RouteNames = keyof Routes;

type RouteParams<T extends RouteNames> = Routes[T]["params"];
type RouteQuery<T extends RouteNames> = Routes[T]["query"];

// Type-safe navigate function
function navigate<T extends RouteNames>(
  route: T,
  options: keyof RouteParams<T> extends never
    ? { query?: RouteQuery<T> }
    : { params: RouteParams<T>; query?: RouteQuery<T> }
): string {
  const config = routes[route];
  let path = config.path;
  
  if ("params" in options && options.params) {
    for (const [key, value] of Object.entries(options.params as Record<string, string>)) {
      path = path.replace(`:${key}`, value);
    }
  }
  
  const query = options.query;
  if (query && Object.keys(query).length > 0) {
    const params = new URLSearchParams();
    for (const [key, value] of Object.entries(query)) {
      if (value !== undefined) {
        params.append(key, String(value));
      }
    }
    path += `?${params.toString()}`;
  }
  
  return path;
}

const routes: { [K in RouteNames]: { path: Routes[K]["path"] } } = {
  home: { path: "/" },
  userProfile: { path: "/users/:userId" },
  userPosts: { path: "/users/:userId/posts" },
  postDetail: { path: "/posts/:postId" }
};

// ใช้งาน
const homeUrl = navigate("home", {});
const profileUrl = navigate("userProfile", { params: { userId: "123" } });
const postsUrl = navigate("userPosts", {
  params: { userId: "123" },
  query: { page: "2", limit: "10" }
});

// navigate("userProfile", {}); // Error! params เป็น required
```

---

### ตัวอย่างที่ 15: Type-safe Event System

```typescript
// Type-safe Event Emitter ขั้นสูง
type EventPayloadMap = {
  "user.created": { userId: string; email: string; createdAt: Date };
  "user.updated": { userId: string; changes: Partial<{ name: string; email: string }> };
  "user.deleted": { userId: string; reason?: string };
  "order.placed": { orderId: string; userId: string; total: number };
  "order.shipped": { orderId: string; trackingNumber: string };
  "payment.received": { orderId: string; amount: number; method: string };
  "payment.failed": { orderId: string; reason: string; retryCount: number };
};

type EventName = keyof EventPayloadMap;
type EventPayload<T extends EventName> = EventPayloadMap[T];

type EventHandler<T extends EventName> = (
  payload: EventPayload<T>
) => void | Promise<void>;

type WildcardHandler = <T extends EventName>(
  event: T,
  payload: EventPayload<T>
) => void | Promise<void>;

class EventBus {
  private handlers: {
    [K in EventName]?: Set<EventHandler<K>>;
  } = {} as any;
  
  private wildcardHandlers: Set<WildcardHandler> = new Set();
  
  on<T extends EventName>(event: T, handler: EventHandler<T>): () => void {
    if (!this.handlers[event]) {
      this.handlers[event] = new Set() as any;
    }
    (this.handlers[event] as Set<EventHandler<T>>).add(handler);
    
    // Return unsubscribe function
    return () => this.off(event, handler);
  }
  
  off<T extends EventName>(event: T, handler: EventHandler<T>): void {
    (this.handlers[event] as Set<EventHandler<T>> | undefined)?.delete(handler);
  }
  
  once<T extends EventName>(event: T, handler: EventHandler<T>): () => void {
    const wrappedHandler: EventHandler<T> = async (payload) => {
      await handler(payload);
      this.off(event, wrappedHandler);
    };
    return this.on(event, wrappedHandler);
  }
  
  onAny(handler: WildcardHandler): () => void {
    this.wildcardHandlers.add(handler);
    return () => this.wildcardHandlers.delete(handler);
  }
  
  async emit<T extends EventName>(
    event: T,
    payload: EventPayload<T>
  ): Promise<void> {
    const handlers = this.handlers[event] as Set<EventHandler<T>> | undefined;
    
    const promises: Promise<void>[] = [];
    
    handlers?.forEach(handler => {
      promises.push(Promise.resolve(handler(payload)));
    });
    
    this.wildcardHandlers.forEach(handler => {
      promises.push(Promise.resolve(handler(event, payload)));
    });
    
    await Promise.all(promises);
  }
  
  emitSync<T extends EventName>(event: T, payload: EventPayload<T>): void {
    const handlers = this.handlers[event] as Set<EventHandler<T>> | undefined;
    handlers?.forEach(handler => handler(payload));
    this.wildcardHandlers.forEach(handler => handler(event, payload));
  }
}

// ใช้งาน
const bus = new EventBus();

const unsubscribe = bus.on("user.created", ({ userId, email }) => {
  console.log(`New user: ${userId} (${email})`);
});

bus.on("order.placed", async ({ orderId, total }) => {
  console.log(`Order ${orderId} placed: ฿${total}`);
});

bus.onAny((event, payload) => {
  console.log(`Event: ${event}`, payload);
});

await bus.emit("user.created", {
  userId: "123",
  email: "john@example.com",
  createdAt: new Date()
});

unsubscribe(); // ยกเลิก subscription
```

---

### ตัวอย่างที่ 16: Type-safe Dependency Injection

```typescript
// Simple Type-safe DI Container
type Constructor<T> = new (...args: any[]) => T;
type FactoryFunction<T> = (...args: any[]) => T;
type Token<T> = symbol & { _type: T };

function createToken<T>(description: string): Token<T> {
  return Symbol(description) as Token<T>;
}

class Container {
  private bindings = new Map<symbol, { factory: () => unknown; singleton: boolean; instance?: unknown }>();
  
  bind<T>(token: Token<T>): {
    to(constructor: Constructor<T>): void;
    toFactory(factory: () => T): void;
    toValue(value: T): void;
  } {
    return {
      to: (constructor: Constructor<T>) => {
        this.bindings.set(token as symbol, {
          factory: () => new constructor(),
          singleton: true
        });
      },
      toFactory: (factory: () => T) => {
        this.bindings.set(token as symbol, {
          factory,
          singleton: false
        });
      },
      toValue: (value: T) => {
        this.bindings.set(token as symbol, {
          factory: () => value,
          singleton: true,
          instance: value
        });
      }
    };
  }
  
  get<T>(token: Token<T>): T {
    const binding = this.bindings.get(token as symbol);
    if (!binding) {
      throw new Error(`No binding for token: ${String(token)}`);
    }
    
    if (binding.singleton) {
      if (!binding.instance) {
        binding.instance = binding.factory();
      }
      return binding.instance as T;
    }
    
    return binding.factory() as T;
  }
}

// ตัวอย่างการใช้งาน
interface Logger {
  log(message: string): void;
}

interface Database {
  query(sql: string): Promise<unknown[]>;
}

const LOGGER = createToken<Logger>("Logger");
const DATABASE = createToken<Database>("Database");

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}

class PostgresDatabase implements Database {
  async query(sql: string): Promise<unknown[]> {
    return []; // implement
  }
}

const container = new Container();
container.bind(LOGGER).to(ConsoleLogger);
container.bind(DATABASE).to(PostgresDatabase);

const logger = container.get(LOGGER); // type: Logger
const db = container.get(DATABASE);   // type: Database
```

---

### ตัวอย่างที่ 17: Type-safe Fetch Wrapper ขั้นสูง

```typescript
// Advanced Type-safe HTTP Client
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

interface Endpoint<
  TPath extends string,
  TMethod extends HttpMethod,
  TBody,
  TResponse,
  TQuery extends Record<string, string> = {}
> {
  path: TPath;
  method: TMethod;
  body?: TBody;
  query?: TQuery;
  response: TResponse;
}

type ApiSchema = {
  getUsers: Endpoint<"/users", "GET", never, { users: User[]; total: number }, { page?: string; limit?: string }>;
  createUser: Endpoint<"/users", "POST", { name: string; email: string }, User>;
  getUserById: Endpoint<"/users/:id", "GET", never, User>;
  updateUser: Endpoint<"/users/:id", "PUT", Partial<User>, User>;
  deleteUser: Endpoint<"/users/:id", "DELETE", never, { success: boolean }>;
};

type EndpointName = keyof ApiSchema;

type EndpointConfig<T extends EndpointName> = ApiSchema[T];

type RequestConfig<T extends EndpointName> = {
  params?: Record<string, string>;
  query?: EndpointConfig<T>["query"];
  body?: EndpointConfig<T>["body"];
};

class ApiClient {
  private baseUrl: string;
  
  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }
  
  async request<T extends EndpointName>(
    endpoint: T,
    config?: RequestConfig<T>
  ): Promise<EndpointConfig<T>["response"]> {
    const endpointConfig = this.getEndpointConfig(endpoint);
    
    let url = this.baseUrl + endpointConfig.path;
    
    // Replace path params
    if (config?.params) {
      for (const [key, value] of Object.entries(config.params)) {
        url = url.replace(`:${key}`, value);
      }
    }
    
    // Add query params
    if (config?.query) {
      const searchParams = new URLSearchParams();
      for (const [key, value] of Object.entries(config.query)) {
        if (value !== undefined) {
          searchParams.append(key, value);
        }
      }
      const qs = searchParams.toString();
      if (qs) url += `?${qs}`;
    }
    
    const options: RequestInit = {
      method: endpointConfig.method,
      headers: { "Content-Type": "application/json" }
    };
    
    if (config?.body) {
      options.body = JSON.stringify(config.body);
    }
    
    const response = await fetch(url, options);
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    return response.json();
  }
  
  private getEndpointConfig(endpoint: EndpointName): { path: string; method: string } {
    const configs: Record<EndpointName, { path: string; method: string }> = {
      getUsers: { path: "/users", method: "GET" },
      createUser: { path: "/users", method: "POST" },
      getUserById: { path: "/users/:id", method: "GET" },
      updateUser: { path: "/users/:id", method: "PUT" },
      deleteUser: { path: "/users/:id", method: "DELETE" }
    };
    return configs[endpoint];
  }
}

interface User {
  id: string;
  name: string;
  email: string;
}

// ใช้งาน
const api = new ApiClient("https://api.example.com");

const users = await api.request("getUsers", {
  query: { page: "1", limit: "10" }
});
// users.users เป็น User[], users.total เป็น number

const user = await api.request("getUserById", {
  params: { id: "123" }
});
// user เป็น User

const created = await api.request("createUser", {
  body: { name: "John", email: "john@example.com" }
});
// created เป็น User
```

---

### ตัวอย่างที่ 18: Variadic Generics

```typescript
// Variadic Tuple Types
type Head<T extends unknown[]> = T extends [infer H, ...infer _] ? H : never;
type Tail<T extends unknown[]> = T extends [infer _, ...infer T] ? T : never;
type Last<T extends unknown[]> = T extends [...infer _, infer L] ? L : never;
type Init<T extends unknown[]> = T extends [...infer I, infer _] ? I : never;

type H = Head<[1, 2, 3]>; // 1
type T = Tail<[1, 2, 3]>; // [2, 3]
type L = Last<[1, 2, 3]>; // 3
type I = Init<[1, 2, 3]>; // [1, 2]

// Function composition with type safety
type Fn<A, B> = (a: A) => B;

function compose<A, B, C>(f: Fn<B, C>, g: Fn<A, B>): Fn<A, C>;
function compose<A, B, C, D>(f: Fn<C, D>, g: Fn<B, C>, h: Fn<A, B>): Fn<A, D>;
function compose<A, B, C, D, E>(
  f: Fn<D, E>,
  g: Fn<C, D>,
  h: Fn<B, C>,
  i: Fn<A, B>
): Fn<A, E>;
function compose(...fns: Fn<any, any>[]): Fn<any, any> {
  return (x: any) => fns.reduceRight((v, f) => f(v), x);
}

const double = (x: number) => x * 2;
const addOne = (x: number) => x + 1;
const toString = (x: number) => x.toString();

const process = compose(toString, addOne, double);
const result = process(5); // "11" (type: string)

// Zip types
type Zip<T extends unknown[], U extends unknown[]> =
  T extends [infer TH, ...infer TT]
    ? U extends [infer UH, ...infer UT]
      ? [[TH, UH], ...Zip<TT, UT>]
      : []
    : [];

type Zipped = Zip<[1, 2, 3], ["a", "b", "c"]>;
// [[1, "a"], [2, "b"], [3, "c"]]
```

---

## ส่วนที่ 7: Advanced Patterns

### ตัวอย่างที่ 19: Type-safe Redux-like State Management

```typescript
// Type-safe Action System
type Action<T extends string, P = void> =
  P extends void ? { type: T } : { type: T; payload: P };

type ActionCreator<T extends string, P = void> =
  P extends void
    ? () => Action<T>
    : (payload: P) => Action<T, P>;

function createAction<T extends string>(type: T): ActionCreator<T, void>;
function createAction<T extends string, P>(type: T): ActionCreator<T, P>;
function createAction<T extends string, P = void>(type: T): ActionCreator<T, P> {
  return ((payload?: P) => ({ type, payload })) as ActionCreator<T, P>;
}

// Define actions
const increment = createAction<"INCREMENT">("INCREMENT");
const decrement = createAction<"DECREMENT">("DECREMENT");
const setCount = createAction<"SET_COUNT", number>("SET_COUNT");
const reset = createAction<"RESET">("RESET");

// Type-safe reducer
type CounterState = { count: number };
type CounterAction =
  | ReturnType<typeof increment>
  | ReturnType<typeof decrement>
  | ReturnType<typeof setCount>
  | ReturnType<typeof reset>;

function counterReducer(
  state: CounterState = { count: 0 },
  action: CounterAction
): CounterState {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    case "DECREMENT":
      return { count: state.count - 1 };
    case "SET_COUNT":
      return { count: action.payload }; // TypeScript รู้ว่า payload เป็น number
    case "RESET":
      return { count: 0 };
    default:
      return state;
  }
}

// Type-safe Store
class Store<S, A> {
  private state: S;
  private reducer: (state: S, action: A) => S;
  private listeners: Set<() => void> = new Set();
  
  constructor(reducer: (state: S, action: A) => S, initialState: S) {
    this.reducer = reducer;
    this.state = initialState;
  }
  
  getState(): S {
    return this.state;
  }
  
  dispatch(action: A): void {
    this.state = this.reducer(this.state, action);
    this.listeners.forEach(listener => listener());
  }
  
  subscribe(listener: () => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
}

const store = new Store(counterReducer, { count: 0 });
store.dispatch(increment());
store.dispatch(setCount(10));
console.log(store.getState()); // { count: 10 }
```

---

### ตัวอย่างที่ 20: Type-level JSON Schema Validation

```typescript
// JSON Schema → TypeScript Type
type JSONSchemaToType<T> =
  T extends { type: "string" }
    ? string
    : T extends { type: "number" }
    ? number
    : T extends { type: "boolean" }
    ? boolean
    : T extends { type: "null" }
    ? null
    : T extends { type: "array"; items: infer I }
    ? JSONSchemaToType<I>[]
    : T extends { type: "object"; properties: infer P }
    ? T extends { required: readonly (infer R extends string)[] }
      ? {
          [K in R & keyof P]-?: JSONSchemaToType<P[K extends keyof P ? K : never]>;
        } & {
          [K in Exclude<keyof P, R>]?: JSONSchemaToType<P[K]>;
        }
      : { [K in keyof P]?: JSONSchemaToType<P[K]> }
    : unknown;

// ตัวอย่าง schema
const userSchema = {
  type: "object" as const,
  properties: {
    id: { type: "string" as const },
    name: { type: "string" as const },
    age: { type: "number" as const },
    isActive: { type: "boolean" as const }
  },
  required: ["id", "name"] as const
};

type UserFromSchema = JSONSchemaToType<typeof userSchema>;
// {
//   id: string;
//   name: string;
//   age?: number;
//   isActive?: boolean;
// }
```

---

### ตัวอย่างที่ 21: Higher Kinded Types Simulation

```typescript
// TypeScript ไม่มี HKT โดยตรง แต่สามารถ simulate ได้

// Type class simulation
interface Functor<F> {
  map<A, B>(fa: { __type: F; value: A }, f: (a: A) => B): { __type: F; value: B };
}

// Maybe monad
type Maybe<T> = Just<T> | Nothing;
type Just<T> = { readonly _tag: "Just"; readonly value: T };
type Nothing = { readonly _tag: "Nothing" };

function just<T>(value: T): Just<T> {
  return { _tag: "Just", value };
}
const nothing: Nothing = { _tag: "Nothing" };

function mapMaybe<A, B>(fa: Maybe<A>, f: (a: A) => B): Maybe<B> {
  if (fa._tag === "Just") return just(f(fa.value));
  return nothing;
}

function flatMapMaybe<A, B>(fa: Maybe<A>, f: (a: A) => Maybe<B>): Maybe<B> {
  if (fa._tag === "Just") return f(fa.value);
  return nothing;
}

function getOrElse<T>(fa: Maybe<T>, defaultValue: T): T {
  return fa._tag === "Just" ? fa.value : defaultValue;
}

// Pipeline with Maybe
const result = flatMapMaybe(
  flatMapMaybe(
    just({ name: "John", address: { city: "Bangkok" } }),
    obj => obj.address ? just(obj.address) : nothing
  ),
  addr => addr.city ? just(addr.city) : nothing
);

const city = getOrElse(result, "Unknown");
console.log(city); // "Bangkok"
```

---

### ตัวอย่างที่ 22: Type-safe Observer Pattern

```typescript
// Type-safe Observer
interface Observer<T> {
  onNext(value: T): void;
  onError?(error: Error): void;
  onComplete?(): void;
}

class Observable<T> {
  private subscribers: Set<Observer<T>> = new Set();
  
  subscribe(observer: Observer<T>): () => void {
    this.subscribers.add(observer);
    return () => this.subscribers.delete(observer);
  }
  
  protected next(value: T): void {
    this.subscribers.forEach(obs => {
      try {
        obs.onNext(value);
      } catch (error) {
        obs.onError?.(error as Error);
      }
    });
  }
  
  protected error(err: Error): void {
    this.subscribers.forEach(obs => obs.onError?.(err));
  }
  
  protected complete(): void {
    this.subscribers.forEach(obs => obs.onComplete?.());
  }
  
  map<U>(transform: (value: T) => U): Observable<U> {
    const mapped = new MappedObservable<T, U>(this, transform);
    return mapped;
  }
  
  filter(predicate: (value: T) => boolean): Observable<T> {
    const filtered = new FilteredObservable<T>(this, predicate);
    return filtered;
  }
}

class MappedObservable<T, U> extends Observable<U> {
  constructor(source: Observable<T>, transform: (value: T) => U) {
    super();
    source.subscribe({
      onNext: (value) => this.next(transform(value)),
      onError: (err) => this.error(err),
      onComplete: () => this.complete()
    });
  }
}

class FilteredObservable<T> extends Observable<T> {
  constructor(source: Observable<T>, predicate: (value: T) => boolean) {
    super();
    source.subscribe({
      onNext: (value) => {
        if (predicate(value)) this.next(value);
      },
      onError: (err) => this.error(err),
      onComplete: () => this.complete()
    });
  }
}

class Subject<T> extends Observable<T> {
  emit(value: T): void { this.next(value); }
  fail(err: Error): void { this.error(err); }
  done(): void { this.complete(); }
}

// ใช้งาน
const clicks = new Subject<{ x: number; y: number }>();

const unsubscribe = clicks
  .filter(({ x }) => x > 100)
  .map(({ x, y }) => `Clicked at ${x}, ${y}`)
  .subscribe({
    onNext: (msg) => console.log(msg),
    onError: (err) => console.error(err)
  });

clicks.emit({ x: 50, y: 100 });  // กรองออก
clicks.emit({ x: 150, y: 200 }); // "Clicked at 150, 200"
```

---

### ตัวอย่างที่ 23: Exhaustive Pattern Matching

```typescript
// Exhaustive Switch
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number }
  | { kind: "ellipse"; rx: number; ry: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
    case "ellipse":
      return Math.PI * shape.rx * shape.ry;
    default:
      return assertNever(shape); // TypeScript Error ถ้า case ไม่ครบ
  }
}

// Pattern matching helper
type Pattern<T extends { kind: string }> = {
  [K in T["kind"]]: (
    value: Extract<T, { kind: K }>
  ) => unknown;
};

function match<T extends { kind: string }>(value: T) {
  return <R>(patterns: {
    [K in T["kind"]]: (value: Extract<T, { kind: K }>) => R;
  } & { _?: () => R }): R => {
    const handler = (patterns as any)[value.kind];
    if (handler) return handler(value);
    if (patterns._) return patterns._();
    throw new Error(`No pattern for: ${(value as any).kind}`);
  };
}

// ใช้งาน
const calculateArea = (shape: Shape): number =>
  match(shape)({
    circle: ({ radius }) => Math.PI * radius ** 2,
    rectangle: ({ width, height }) => width * height,
    triangle: ({ base, height }) => (base * height) / 2,
    ellipse: ({ rx, ry }) => Math.PI * rx * ry
  });
```

---

### ตัวอย่างที่ 24-30: Advanced Generic Utilities

```typescript
// Flatten nested type
type Flatten<T> =
  T extends Array<infer U> ? Flatten<U> :
  T extends object ? { [K in keyof T]: Flatten<T[K]> } :
  T;

// Type-safe Object.entries
type Entries<T> = {
  [K in keyof T]: [K, T[K]];
}[keyof T][];

function typedEntries<T extends object>(obj: T): Entries<T> {
  return Object.entries(obj) as Entries<T>;
}

const obj = { name: "John", age: 30, active: true };
const entries = typedEntries(obj);
// [["name", string] | ["age", number] | ["active", boolean]][]

// Type-safe Object.fromEntries
type FromEntries<T extends [string, unknown][]> = {
  [K in T[number] as K[0]]: K[1];
};

// Merge objects deeply
type DeepMerge<T, U> = {
  [K in keyof T | keyof U]:
    K extends keyof T & keyof U
      ? T[K] extends object
        ? U[K] extends object
          ? DeepMerge<T[K], U[K]>
          : U[K]
        : U[K]
      : K extends keyof T
      ? T[K]
      : K extends keyof U
      ? U[K]
      : never;
};

type Config1 = { a: { x: number; y: number }; b: string };
type Config2 = { a: { y: number; z: number }; c: boolean };
type MergedConfig = DeepMerge<Config1, Config2>;
// { a: { x: number; y: number; z: number }; b: string; c: boolean }

// Overwrite properties
type Overwrite<T, U extends Partial<T>> = Omit<T, keyof U> & U;

interface User {
  id: string;
  name: string;
  email: string;
  role: "user" | "admin";
}

type AdminUser = Overwrite<User, { role: "admin" }>;
// id: string; name: string; email: string; role: "admin"

// Require certain fields
type RequireFields<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

type UserWithRequiredEmail = RequireFields<Partial<User>, "email">;
// All optional except email which is required

// Create union from object values
type Values<T extends object> = T[keyof T];

const STATUS = {
  PENDING: "pending",
  ACTIVE: "active",
  INACTIVE: "inactive"
} as const;

type Status = Values<typeof STATUS>;
// "pending" | "active" | "inactive"

// Infer parameters
type FirstParam<T extends (...args: any) => any> =
  Parameters<T>[0];

type AllParams<T extends (...args: any) => any> =
  Parameters<T>;

function greet(name: string, age: number): string {
  return `Hello ${name}, age ${age}`;
}

type FirstArg = FirstParam<typeof greet>; // string
type AllArgs = AllParams<typeof greet>; // [string, number]
```

---

## ส่วนที่ 8: Performance Patterns

### ตัวอย่างที่ 31: Lazy Evaluation Types

```typescript
// ใช้ interface แทน type alias สำหรับ recursive types (lazy evaluation)
interface JSONObject {
  [key: string]: JSONValue;
}

interface JSONArray extends Array<JSONValue> {}

type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONObject
  | JSONArray;

// เร็วกว่า:
// type JSONValue = string | number | boolean | null
//   | { [key: string]: JSONValue }  // ช้า - eager evaluation
//   | JSONValue[];                   // ช้า

// Type caching with interface
interface TypeCache {
  _1: never;
}

// Use type aliases for simple unions (faster)
type Primitive = string | number | boolean | null | undefined;
type NonNullable<T> = T extends null | undefined ? never : T;

// Avoid deep recursion
type Depth = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

type DeepReadonlyWithDepth<T, D extends number = 10> = {
  readonly [K in keyof T]: [D] extends [0]
    ? T[K]
    : T[K] extends object
    ? DeepReadonlyWithDepth<T[K], Depth[D]>
    : T[K];
};
```

---

### ตัวอย่างที่ 32: Conditional Type Distribution

```typescript
// Distributive vs Non-distributive Conditional Types

// Distributive (T extends condition)
type IsString<T> = T extends string ? "yes" : "no";

type R1 = IsString<string>;         // "yes"
type R2 = IsString<number>;         // "no"
type R3 = IsString<string | number>; // "yes" | "no" (distributive!)

// Non-distributive (wrap in tuple)
type IsStringND<T> = [T] extends [string] ? "yes" : "no";

type R4 = IsStringND<string | number>; // "no" (not distributive)

// ใช้ประโยชน์จาก distributive
type ExtractByValue<T extends Record<string, unknown>, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

interface UserRecord {
  id: string;
  name: string;
  age: number;
  isActive: boolean;
  score: number;
}

type StringFields = ExtractByValue<UserRecord, string>;
// "id" | "name"

type NumberFields = ExtractByValue<UserRecord, number>;
// "age" | "score"

// Filter by type
type FilterByType<T extends Record<string, unknown>, V> = Pick<
  T,
  ExtractByValue<T, V>
>;

type OnlyStrings = FilterByType<UserRecord, string>;
// { id: string; name: string }
```

---

### ตัวอย่างที่ 33-40: Real-world Advanced Types

```typescript
// Type-safe CSS-in-TS
type CSSUnit = "px" | "em" | "rem" | "vh" | "vw" | "%";
type CSSValue = `${number}${CSSUnit}` | "auto" | "inherit" | "initial";

type CSSProperties = {
  width?: CSSValue;
  height?: CSSValue;
  margin?: CSSValue | `${CSSValue} ${CSSValue}` | `${CSSValue} ${CSSValue} ${CSSValue} ${CSSValue}`;
  padding?: CSSValue | `${CSSValue} ${CSSValue}`;
  color?: string;
  backgroundColor?: string;
  fontSize?: CSSValue;
  fontWeight?: "normal" | "bold" | 100 | 200 | 300 | 400 | 500 | 600 | 700 | 800 | 900;
};

// Type-safe environment variables
type EnvType = "string" | "number" | "boolean" | "json";

interface EnvSchema {
  [key: string]: { type: EnvType; required?: boolean; default?: unknown };
}

type InferEnvType<T extends EnvType> =
  T extends "string" ? string :
  T extends "number" ? number :
  T extends "boolean" ? boolean :
  T extends "json" ? unknown :
  never;

type ParsedEnv<Schema extends EnvSchema> = {
  [K in keyof Schema]: Schema[K]["required"] extends true
    ? InferEnvType<Schema[K]["type"]>
    : InferEnvType<Schema[K]["type"]> | undefined;
};

function parseEnv<S extends EnvSchema>(
  schema: S,
  env: Record<string, string | undefined>
): ParsedEnv<S> {
  const result: Record<string, unknown> = {};
  
  for (const [key, config] of Object.entries(schema)) {
    const raw = env[key] ?? (config.default !== undefined ? String(config.default) : undefined);
    
    if (raw === undefined) {
      if (config.required) {
        throw new Error(`Missing required environment variable: ${key}`);
      }
      continue;
    }
    
    switch (config.type) {
      case "string": result[key] = raw; break;
      case "number": result[key] = Number(raw); break;
      case "boolean": result[key] = raw === "true"; break;
      case "json": result[key] = JSON.parse(raw); break;
    }
  }
  
  return result as ParsedEnv<S>;
}

const envSchema = {
  PORT: { type: "number" as const, required: true },
  NODE_ENV: { type: "string" as const, default: "development" },
  DEBUG: { type: "boolean" as const },
  CONFIG: { type: "json" as const }
} satisfies EnvSchema;

const env = parseEnv(envSchema, process.env as Record<string, string>);
// env.PORT เป็น number (required)
// env.NODE_ENV เป็น string | undefined
// env.DEBUG เป็น boolean | undefined

// Type-safe feature flags
interface FeatureFlags {
  darkMode: boolean;
  betaFeature: boolean;
  newCheckout: boolean;
}

class FeatureFlagService<T extends Record<string, boolean>> {
  constructor(private flags: T) {}
  
  isEnabled<K extends keyof T>(flag: K): boolean {
    return this.flags[flag];
  }
  
  getAll(): T {
    return { ...this.flags };
  }
  
  update(flag: keyof T, value: boolean): FeatureFlagService<T> {
    return new FeatureFlagService({ ...this.flags, [flag]: value } as T);
  }
}

const featureFlags = new FeatureFlagService<FeatureFlags>({
  darkMode: true,
  betaFeature: false,
  newCheckout: false
});

const isDarkMode = featureFlags.isEnabled("darkMode"); // boolean
// featureFlags.isEnabled("unknownFlag"); // Error!
```

---

### ตัวอย่างที่ 41-50: Type-safe Database Query Builder

```typescript
// Type-safe Query Builder
interface TableSchema {
  users: {
    id: string;
    name: string;
    email: string;
    age: number;
    createdAt: Date;
  };
  posts: {
    id: string;
    title: string;
    content: string;
    authorId: string;
    publishedAt: Date | null;
  };
  comments: {
    id: string;
    content: string;
    postId: string;
    authorId: string;
  };
}

type TableName = keyof TableSchema;
type TableColumns<T extends TableName> = keyof TableSchema[T];

type WhereOperator = "=" | "!=" | "<" | "<=" | ">" | ">=" | "LIKE" | "IN";

type WhereClause<T extends TableName> = {
  column: TableColumns<T>;
  operator: WhereOperator;
  value: TableSchema[T][TableColumns<T>];
};

type OrderDirection = "ASC" | "DESC";

type OrderClause<T extends TableName> = {
  column: TableColumns<T>;
  direction: OrderDirection;
};

class TypedQueryBuilder<
  T extends TableName,
  Selected extends TableColumns<T> = TableColumns<T>
> {
  private selectColumns: Selected[] = [] as Selected[];
  private whereClasses: WhereClause<T>[] = [];
  private orderClauses: OrderClause<T>[] = [];
  private limitValue?: number;
  private offsetValue?: number;
  
  constructor(private table: T) {}
  
  select<K extends TableColumns<T>>(
    ...columns: K[]
  ): TypedQueryBuilder<T, K> {
    const builder = new TypedQueryBuilder<T, K>(this.table);
    builder.selectColumns = columns;
    builder.whereClasses = this.whereClasses as any;
    return builder;
  }
  
  where(
    column: TableColumns<T>,
    operator: WhereOperator,
    value: TableSchema[T][typeof column]
  ): this {
    this.whereClasses.push({ column, operator, value: value as any });
    return this;
  }
  
  orderBy(column: TableColumns<T>, direction: OrderDirection = "ASC"): this {
    this.orderClauses.push({ column, direction });
    return this;
  }
  
  limit(n: number): this {
    this.limitValue = n;
    return this;
  }
  
  offset(n: number): this {
    this.offsetValue = n;
    return this;
  }
  
  build(): {
    sql: string;
    params: unknown[];
    type: Pick<TableSchema[T], Selected>;
  } {
    const columns = this.selectColumns.length > 0
      ? this.selectColumns.join(", ")
      : "*";
    
    let sql = `SELECT ${columns} FROM ${this.table}`;
    const params: unknown[] = [];
    
    if (this.whereClasses.length > 0) {
      const conditions = this.whereClasses.map((w, i) => {
        params.push(w.value);
        return `${String(w.column)} ${w.operator} $${i + 1}`;
      });
      sql += ` WHERE ${conditions.join(" AND ")}`;
    }
    
    if (this.orderClauses.length > 0) {
      const orders = this.orderClauses.map(
        o => `${String(o.column)} ${o.direction}`
      );
      sql += ` ORDER BY ${orders.join(", ")}`;
    }
    
    if (this.limitValue) sql += ` LIMIT ${this.limitValue}`;
    if (this.offsetValue) sql += ` OFFSET ${this.offsetValue}`;
    
    return { sql, params, type: {} as Pick<TableSchema[T], Selected> };
  }
}

// Helper function
function from<T extends TableName>(table: T): TypedQueryBuilder<T> {
  return new TypedQueryBuilder(table);
}

// ใช้งาน
const query = from("users")
  .select("id", "name", "email")
  .where("age", ">=", 18)
  .where("name", "LIKE", "%John%")
  .orderBy("createdAt", "DESC")
  .limit(10)
  .offset(0)
  .build();

console.log(query.sql);
// SELECT id, name, email FROM users WHERE age >= $1 AND name LIKE $2 ORDER BY createdAt DESC LIMIT 10 OFFSET 0
// query.type เป็น { id: string; name: string; email: string }
```

---

## สรุป

ใน Advanced TypeScript Tricks เราได้เรียนรู้:

1. **Deep Types**: DeepReadonly, DeepPartial, DeepRequired, DeepMutable
2. **String Manipulation**: Split, Join, CamelCase, Template Literals
3. **Number Types**: Arithmetic, Comparisons, Ranges
4. **Branded Types**: Type safety ในระดับ domain
5. **Phantom Types**: State machines ที่ type-safe
6. **Builder Patterns**: Type-safe step builders
7. **Event Systems**: Type-safe event emitters
8. **DI Container**: Type-safe dependency injection
9. **Variadic Generics**: Complex tuple operations
10. **Query Builders**: Type-safe database queries

การฝึกฝน Advanced TypeScript:
- เริ่มจาก utility types ที่มีอยู่แล้ว
- ทดลองใน TypeScript Playground
- อ่าน type definitions ของ libraries ชื่อดัง
- ทำ Type Challenges บน GitHub

---

*จบ Part 59 - Advanced TypeScript Tricks*
