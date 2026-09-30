# ตอนที่ 78: Meta-programming in TypeScript (การเขียนโปรแกรมเชิง Meta)

## บทนำ

Meta-programming คือการเขียนโปรแกรมที่จัดการโปรแกรมอื่น (หรือตัวเอง) เป็น data TypeScript มีฟีเจอร์หลายอย่างสำหรับ meta-programming ได้แก่ Decorators, Proxies, Symbols, Property Descriptors และอื่นๆ

---

## 1. Decorators Deep Dive

Decorators ใน TypeScript เป็น special declarations ที่สามารถ attach ไปยัง class declarations, methods, accessors, properties หรือ parameters

### ตัวอย่างที่ 1: Class Decorator พื้นฐาน

```typescript
// เปิดใช้งาน experimentalDecorators ใน tsconfig.json
// "experimentalDecorators": true

// Class Decorator แบบง่าย
function Singleton<T extends { new (...args: unknown[]): {} }>(
  constructor: T
): T {
  let instance: InstanceType<T> | null = null;

  return class extends constructor {
    constructor(...args: unknown[]) {
      if (instance) return instance as any;
      super(...args);
      instance = this as unknown as InstanceType<T>;
    }
  };
}

@Singleton
class DatabaseConnection {
  private readonly id: string;

  constructor() {
    this.id = Math.random().toString(36).substr(2, 9);
    console.log(`สร้าง connection ใหม่: ${this.id}`);
  }

  query(sql: string): string {
    return `ผลลัพธ์จาก ${this.id}: ${sql}`;
  }
}

const db1 = new DatabaseConnection(); // สร้าง connection ใหม่
const db2 = new DatabaseConnection(); // ใช้ instance เดิม!
console.log(db1 === db2); // true
```

### ตัวอย่างที่ 2: Decorator Factory

```typescript
// Decorator Factory: ส่ง parameters ให้ decorator ได้
function Log(prefix: string = "") {
  return function <T extends { new (...args: unknown[]): {} }>(
    constructor: T
  ): T {
    return class extends constructor {
      constructor(...args: unknown[]) {
        console.log(`${prefix}[${constructor.name}] กำลังสร้าง instance...`);
        super(...args);
        console.log(`${prefix}[${constructor.name}] สร้าง instance สำเร็จ`);
      }
    };
  };
}

function Timestamp<T extends { new (...args: unknown[]): {} }>(
  constructor: T
): T {
  return class extends constructor {
    readonly createdAt = new Date();
    readonly updatedAt = new Date();
  };
}

@Log("🔵 ")
@Timestamp
class UserService {
  private users: Map<string, unknown> = new Map();

  addUser(id: string, user: unknown): void {
    this.users.set(id, user);
  }
}

const service = new UserService();
console.log((service as any).createdAt); // มี timestamp แล้ว
```

### ตัวอย่างที่ 3: Method Decorators

```typescript
// Method Decorator สำหรับ logging
function LogMethod(
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: unknown[]) {
    console.log(
      `[LOG] เรียก ${String(propertyKey)} ด้วย args: ${JSON.stringify(args)}`
    );

    const startTime = Date.now();
    const result = originalMethod.apply(this, args);

    if (result instanceof Promise) {
      return result.then((res: unknown) => {
        const duration = Date.now() - startTime;
        console.log(
          `[LOG] ${String(propertyKey)} เสร็จสิ้น (${duration}ms): ${JSON.stringify(res)}`
        );
        return res;
      });
    }

    const duration = Date.now() - startTime;
    console.log(
      `[LOG] ${String(propertyKey)} เสร็จสิ้น (${duration}ms): ${JSON.stringify(result)}`
    );
    return result;
  };

  return descriptor;
}

// Memoize decorator
function Memoize(
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const originalMethod = descriptor.value;
  const cache = new Map<string, unknown>();

  descriptor.value = function (...args: unknown[]) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      console.log(`[CACHE HIT] ${String(propertyKey)}(${key})`);
      return cache.get(key);
    }

    const result = originalMethod.apply(this, args);
    cache.set(key, result);
    return result;
  };

  return descriptor;
}

// Throttle decorator
function Throttle(milliseconds: number) {
  return function (
    target: object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ): PropertyDescriptor {
    const originalMethod = descriptor.value;
    let lastCall = 0;

    descriptor.value = function (...args: unknown[]) {
      const now = Date.now();
      if (now - lastCall >= milliseconds) {
        lastCall = now;
        return originalMethod.apply(this, args);
      }
      console.log(`[THROTTLE] ${String(propertyKey)} ถูก throttle`);
    };

    return descriptor;
  };
}

class CalculatorService {
  @LogMethod
  @Memoize
  fibonacci(n: number): number {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }

  @Throttle(1000)
  handleClick(): void {
    console.log("คลิก!");
  }
}

const calc = new CalculatorService();
console.log(calc.fibonacci(10)); // คำนวณ
console.log(calc.fibonacci(10)); // ใช้ cache
```

### ตัวอย่างที่ 4: Property Decorators

```typescript
// Property decorator สำหรับ validation
function Validate(validator: (value: unknown) => boolean, message: string) {
  return function (target: object, propertyKey: string | symbol): void {
    let value: unknown;

    const getter = function () {
      return value;
    };

    const setter = function (newValue: unknown) {
      if (!validator(newValue)) {
        throw new Error(`Validation failed for ${String(propertyKey)}: ${message}`);
      }
      value = newValue;
    };

    Object.defineProperty(target, propertyKey, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true,
    });
  };
}

// ReadOnly decorator
function ReadOnly(
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  return {
    ...descriptor,
    writable: false,
  };
}

// Observable decorator
function Observable(target: object, propertyKey: string | symbol): void {
  const listeners = new Map<string | symbol, Set<(value: unknown) => void>>();

  let value: unknown;

  Object.defineProperty(target, propertyKey, {
    get: () => value,
    set: (newValue: unknown) => {
      const oldValue = value;
      value = newValue;

      const propertyListeners = listeners.get(propertyKey);
      propertyListeners?.forEach((listener) => listener(newValue));

      console.log(
        `[OBSERVABLE] ${String(propertyKey)}: ${JSON.stringify(oldValue)} → ${JSON.stringify(newValue)}`
      );
    },
    enumerable: true,
    configurable: true,
  });
}

class UserProfile {
  @Validate(
    (v) => typeof v === "string" && v.length >= 2,
    "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"
  )
  name: string = "";

  @Validate(
    (v) => typeof v === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v as string),
    "รูปแบบ email ไม่ถูกต้อง"
  )
  email: string = "";

  @Validate(
    (v) => typeof v === "number" && v >= 0 && v <= 150,
    "อายุต้องอยู่ระหว่าง 0-150"
  )
  age: number = 0;

  @Observable
  status: string = "offline";
}

const profile = new UserProfile();
profile.name = "สมชาย"; // ✅ OK
profile.email = "somchai@example.com"; // ✅ OK
profile.age = 25; // ✅ OK
profile.status = "online"; // แสดง log การเปลี่ยนแปลง

try {
  profile.name = "ก"; // ❌ Error: ชื่อต้องมีอย่างน้อย 2 ตัวอักษร
} catch (e) {
  console.error("Validation error:", (e as Error).message);
}
```

### ตัวอย่างที่ 5: Parameter Decorators

```typescript
import "reflect-metadata";

const PARAM_METADATA_KEY = Symbol("paramMetadata");

interface ParamMetadata {
  index: number;
  key: string;
  source: "body" | "query" | "params" | "headers";
  required: boolean;
}

function Body(key?: string) {
  return function (
    target: object,
    methodName: string | symbol,
    paramIndex: number
  ): void {
    const existing: ParamMetadata[] =
      Reflect.getMetadata(PARAM_METADATA_KEY, target, methodName) ?? [];

    existing.push({
      index: paramIndex,
      key: key ?? "",
      source: "body",
      required: true,
    });

    Reflect.defineMetadata(PARAM_METADATA_KEY, existing, target, methodName);
  };
}

function Query(key: string, required = false) {
  return function (
    target: object,
    methodName: string | symbol,
    paramIndex: number
  ): void {
    const existing: ParamMetadata[] =
      Reflect.getMetadata(PARAM_METADATA_KEY, target, methodName) ?? [];

    existing.push({ index: paramIndex, key, source: "query", required });

    Reflect.defineMetadata(PARAM_METADATA_KEY, existing, target, methodName);
  };
}

function Param(key: string) {
  return function (
    target: object,
    methodName: string | symbol,
    paramIndex: number
  ): void {
    const existing: ParamMetadata[] =
      Reflect.getMetadata(PARAM_METADATA_KEY, target, methodName) ?? [];

    existing.push({ index: paramIndex, key, source: "params", required: true });

    Reflect.defineMetadata(PARAM_METADATA_KEY, existing, target, methodName);
  };
}

// Controller decorator
function Controller(basePath: string) {
  return function (target: new (...args: unknown[]) => {}) {
    Reflect.defineMetadata("basePath", basePath, target);
  };
}

function Get(path: string = "") {
  return function (
    target: object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) {
    Reflect.defineMetadata("httpMethod", "GET", target, propertyKey);
    Reflect.defineMetadata("path", path, target, propertyKey);
  };
}

function Post(path: string = "") {
  return function (
    target: object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) {
    Reflect.defineMetadata("httpMethod", "POST", target, propertyKey);
    Reflect.defineMetadata("path", path, target, propertyKey);
  };
}

@Controller("/users")
class UserController {
  @Get("/:id")
  async getUser(
    @Param("id") id: string
  ) {
    return { id, name: "สมชาย" };
  }

  @Post("/")
  async createUser(
    @Body() userData: { name: string; email: string },
    @Query("notify") notify: string
  ) {
    return { id: "new-id", ...userData };
  }
}
```

---

## 2. Reflect.metadata

### ตัวอย่างที่ 6: Dependency Injection Container

```typescript
import "reflect-metadata";

const INJECT_METADATA_KEY = Symbol("inject");
const SINGLETON_METADATA_KEY = Symbol("singleton");

// Injection token
class InjectionToken<T> {
  constructor(readonly description: string) {}
}

function Injectable() {
  return function (target: new (...args: unknown[]) => {}) {
    // mark class as injectable
    Reflect.defineMetadata("injectable", true, target);
  };
}

function Inject(token: InjectionToken<unknown> | (new (...args: unknown[]) => unknown)) {
  return function (
    target: object,
    _propertyKey: string | symbol | undefined,
    paramIndex: number
  ) {
    const tokens: unknown[] =
      Reflect.getMetadata(INJECT_METADATA_KEY, target) ?? [];
    tokens[paramIndex] = token;
    Reflect.defineMetadata(INJECT_METADATA_KEY, tokens, target);
  };
}

// DI Container
class Container {
  private bindings = new Map<
    unknown,
    { factory: () => unknown; singleton: boolean; instance?: unknown }
  >();

  bind<T>(
    token: new (...args: unknown[]) => T | InjectionToken<T>,
    factory: () => T,
    options: { singleton?: boolean } = {}
  ): void {
    this.bindings.set(token, {
      factory,
      singleton: options.singleton ?? false,
    });
  }

  resolve<T>(token: new (...args: unknown[]) => T): T {
    const binding = this.bindings.get(token);

    if (!binding) {
      // Auto-resolve ถ้าไม่มี binding
      return this.autoResolve(token);
    }

    if (binding.singleton) {
      if (!binding.instance) {
        binding.instance = binding.factory();
      }
      return binding.instance as T;
    }

    return binding.factory() as T;
  }

  private autoResolve<T>(
    constructor: new (...args: unknown[]) => T
  ): T {
    const paramTypes = Reflect.getMetadata(
      "design:paramtypes",
      constructor
    ) as (new (...args: unknown[]) => unknown)[] ?? [];

    const injectTokens = Reflect.getMetadata(
      INJECT_METADATA_KEY,
      constructor
    ) as unknown[] ?? [];

    const args = paramTypes.map((paramType, index) => {
      const token = injectTokens[index] ?? paramType;
      if (!token) return undefined;
      return this.resolve(token as new (...args: unknown[]) => unknown);
    });

    return new constructor(...args);
  }
}

// ตัวอย่างการใช้งาน
@Injectable()
class LoggerService {
  log(message: string): void {
    console.log(`[LOG] ${new Date().toISOString()}: ${message}`);
  }
}

@Injectable()
class DatabaseService {
  private connection: string = "connected";

  query(sql: string): unknown[] {
    return [{ id: 1, name: "ข้อมูล" }];
  }
}

@Injectable()
class UserService {
  constructor(
    private logger: LoggerService,
    private db: DatabaseService
  ) {}

  getUser(id: number): unknown {
    this.logger.log(`ดึงข้อมูล user ${id}`);
    const results = this.db.query(`SELECT * FROM users WHERE id = ${id}`);
    return results[0];
  }
}

const container = new Container();
container.bind(LoggerService, () => new LoggerService(), { singleton: true });
container.bind(DatabaseService, () => new DatabaseService(), { singleton: true });
container.bind(
  UserService,
  () =>
    new UserService(
      container.resolve(LoggerService),
      container.resolve(DatabaseService)
    )
);

const userService = container.resolve(UserService);
const user = userService.getUser(1);
console.log("User:", user);
```

---

## 3. Proxy กับ TypeScript

### ตัวอย่างที่ 7: Type-Safe Proxy

```typescript
// Proxy สำหรับ validation อัตโนมัติ
function createValidatedProxy<T extends object>(
  target: T,
  validators: Partial<{ [K in keyof T]: (value: T[K]) => string | null }>
): T {
  return new Proxy(target, {
    set(obj, prop, value) {
      const validator = validators[prop as keyof T];
      if (validator) {
        const error = validator(value);
        if (error) {
          throw new Error(`Validation failed for "${String(prop)}": ${error}`);
        }
      }
      (obj as Record<string | symbol, unknown>)[prop] = value;
      return true;
    },
  });
}

interface UserData {
  name: string;
  email: string;
  age: number;
}

const userData = createValidatedProxy<UserData>(
  { name: "", email: "", age: 0 },
  {
    name: (v) => (v.length < 2 ? "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร" : null),
    email: (v) =>
      !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) ? "Email ไม่ถูกต้อง" : null,
    age: (v) => (v < 0 || v > 150 ? "อายุไม่ถูกต้อง" : null),
  }
);

userData.name = "สมชาย"; // ✅
userData.email = "somchai@example.com"; // ✅
userData.age = 25; // ✅

try {
  userData.name = "ก"; // ❌ Error
} catch (e) {
  console.error((e as Error).message);
}
```

### ตัวอย่างที่ 8: Lazy Loading Proxy

```typescript
// Proxy สำหรับ lazy loading
function createLazyProxy<T extends object>(
  loader: () => T | Promise<T>
): T {
  let instance: T | null = null;
  let loadPromise: Promise<T> | null = null;

  return new Proxy({} as T, {
    get(_, prop) {
      if (!instance) {
        const result = loader();
        if (result instanceof Promise) {
          if (!loadPromise) {
            loadPromise = result.then((loaded) => {
              instance = loaded;
              return loaded;
            });
          }
          // Return a promise proxy
          return (...args: unknown[]) => loadPromise!.then((inst) => {
            const method = (inst as Record<string | symbol, unknown>)[prop];
            if (typeof method === "function") {
              return (method as Function).apply(inst, args);
            }
            return method;
          });
        }
        instance = result;
      }

      const value = (instance as Record<string | symbol, unknown>)[prop];
      if (typeof value === "function") {
        return value.bind(instance);
      }
      return value;
    },
  });
}

// Observable Proxy
function createObservable<T extends object>(
  target: T,
  onChange: (prop: string | symbol, newValue: unknown, oldValue: unknown) => void
): T {
  return new Proxy(target, {
    set(obj, prop, value) {
      const oldValue = (obj as Record<string | symbol, unknown>)[prop];
      (obj as Record<string | symbol, unknown>)[prop] = value;
      onChange(prop, value, oldValue);
      return true;
    },

    deleteProperty(obj, prop) {
      const oldValue = (obj as Record<string | symbol, unknown>)[prop];
      delete (obj as Record<string | symbol, unknown>)[prop];
      onChange(prop, undefined, oldValue);
      return true;
    },
  });
}

// ตัวอย่างการใช้ Observable
interface AppState {
  count: number;
  user: { name: string } | null;
  items: string[];
}

const state = createObservable<AppState>(
  { count: 0, user: null, items: [] },
  (prop, newValue, oldValue) => {
    console.log(
      `State เปลี่ยน: ${String(prop)} = ${JSON.stringify(oldValue)} → ${JSON.stringify(newValue)}`
    );
  }
);

state.count = 1; // log
state.user = { name: "สมชาย" }; // log
state.count = 2; // log
```

### ตัวอย่างที่ 9: Revocable Proxy สำหรับ Access Control

```typescript
// Proxy ที่ revoke ได้ สำหรับ access control
interface SensitiveData {
  ssn: string;
  bankAccount: string;
  salary: number;
}

function createSecureProxy<T extends object>(
  target: T,
  allowedFields: (keyof T)[],
  auditLog: (action: string, field: string, userId: string) => void
) {
  const { proxy, revoke } = Proxy.revocable(target, {
    get(obj, prop) {
      if (!allowedFields.includes(prop as keyof T)) {
        console.warn(`⚠️ ความพยายามเข้าถึง field ที่ไม่ได้รับอนุญาต: ${String(prop)}`);
        return undefined;
      }

      auditLog("READ", String(prop), "current-user");
      return (obj as Record<string | symbol, unknown>)[prop];
    },

    set(obj, prop, value) {
      if (!allowedFields.includes(prop as keyof T)) {
        throw new Error(`ไม่ได้รับอนุญาตให้แก้ไข field: ${String(prop)}`);
      }

      auditLog("WRITE", String(prop), "current-user");
      (obj as Record<string | symbol, unknown>)[prop] = value;
      return true;
    },
  });

  return { proxy, revoke };
}

const sensitiveData: SensitiveData = {
  ssn: "XXX-XX-1234",
  bankAccount: "1234567890",
  salary: 50000,
};

const auditEntries: string[] = [];

const { proxy: secureData, revoke } = createSecureProxy(
  sensitiveData,
  ["salary"], // อนุญาตเฉพาะ salary
  (action, field, userId) => {
    const entry = `${new Date().toISOString()} | ${action} | ${field} | ${userId}`;
    auditEntries.push(entry);
    console.log(`[AUDIT] ${entry}`);
  }
);

console.log(secureData.salary); // ✅ 50000
console.log(secureData.ssn); // ❌ undefined (ไม่มีสิทธิ์)

// เพิกถอนสิทธิ์การเข้าถึง
revoke();

try {
  console.log(secureData.salary); // ❌ TypeError
} catch (e) {
  console.error("Proxy ถูก revoke แล้ว");
}
```

---

## 4. Symbols และ Well-Known Symbols

### ตัวอย่างที่ 10: Symbol.iterator

```typescript
// Custom iterable ด้วย Symbol.iterator
class Range {
  constructor(
    private start: number,
    private end: number,
    private step: number = 1
  ) {}

  [Symbol.iterator](): Iterator<number> {
    let current = this.start;
    const end = this.end;
    const step = this.step;

    return {
      next(): IteratorResult<number> {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined as any, done: true };
      },
    };
  }

  // เพิ่ม async iterator
  async *[Symbol.asyncIterator](): AsyncGenerator<number> {
    for (let i = this.start; i <= this.end; i += this.step) {
      await new Promise((resolve) => setTimeout(resolve, 10));
      yield i;
    }
  }
}

const range = new Range(1, 10, 2);
console.log("Range:", [...range]); // [1, 3, 5, 7, 9]

for (const num of new Range(0, 20, 5)) {
  console.log(num); // 0, 5, 10, 15, 20
}

// Custom Tree iterable
class TreeNode<T> {
  children: TreeNode<T>[] = [];

  constructor(public value: T) {}

  addChild(child: TreeNode<T>): this {
    this.children.push(child);
    return this;
  }

  // Depth-first traversal
  *[Symbol.iterator](): Iterator<T> {
    yield this.value;
    for (const child of this.children) {
      yield* child;
    }
  }

  // Breadth-first traversal
  *bfs(): Generator<T> {
    const queue: TreeNode<T>[] = [this];

    while (queue.length > 0) {
      const node = queue.shift()!;
      yield node.value;
      queue.push(...node.children);
    }
  }
}

const tree = new TreeNode("root");
const child1 = new TreeNode("child1");
const child2 = new TreeNode("child2");
child1.addChild(new TreeNode("grandchild1")).addChild(new TreeNode("grandchild2"));
child2.addChild(new TreeNode("grandchild3"));
tree.addChild(child1).addChild(child2);

console.log("DFS:", [...tree]); // root, child1, grandchild1, grandchild2, child2, grandchild3
console.log("BFS:", [...tree.bfs()]); // root, child1, child2, grandchild1, grandchild2, grandchild3
```

### ตัวอย่างที่ 11: Symbol.toPrimitive

```typescript
// Custom primitive conversion
class Money {
  constructor(
    readonly amount: number,
    readonly currency: string
  ) {}

  [Symbol.toPrimitive](hint: "number" | "string" | "default"): string | number {
    switch (hint) {
      case "number":
        return this.amount;
      case "string":
        return `${this.amount} ${this.currency}`;
      case "default":
        return this.amount;
    }
  }

  valueOf(): number {
    return this.amount;
  }

  toString(): string {
    return `${this.amount} ${this.currency}`;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error(`ไม่สามารถบวก ${this.currency} กับ ${other.currency}`);
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}

const price1 = new Money(100, "THB");
const price2 = new Money(200, "THB");

console.log(`ราคา: ${price1}`); // "100 THB"
console.log(price1 + 50); // 150 (number hint)
console.log(price1 > 50); // true (comparison)
console.log(+price1); // 100 (unary +)

// Complex number
class Complex {
  constructor(readonly real: number, readonly imaginary: number) {}

  [Symbol.toPrimitive](hint: string): string | number {
    if (hint === "string") {
      const sign = this.imaginary >= 0 ? "+" : "";
      return `${this.real}${sign}${this.imaginary}i`;
    }
    return Math.sqrt(this.real ** 2 + this.imaginary ** 2); // magnitude
  }

  add(other: Complex): Complex {
    return new Complex(
      this.real + other.real,
      this.imaginary + other.imaginary
    );
  }

  multiply(other: Complex): Complex {
    return new Complex(
      this.real * other.real - this.imaginary * other.imaginary,
      this.real * other.imaginary + this.imaginary * other.real
    );
  }
}

const c1 = new Complex(3, 4);
const c2 = new Complex(1, -2);

console.log(`${c1}`); // "3+4i"
console.log(+c1); // 5 (magnitude)
console.log(c1.add(c2)); // Complex { real: 4, imaginary: 2 }
```

### ตัวอย่างที่ 12: Symbol.hasInstance และ Symbol.species

```typescript
// Symbol.hasInstance: custom instanceof behavior
class EvenNumber {
  static [Symbol.hasInstance](instance: unknown): boolean {
    return typeof instance === "number" && instance % 2 === 0;
  }
}

console.log(4 instanceof EvenNumber); // true
console.log(3 instanceof EvenNumber); // false
console.log("4" instanceof EvenNumber); // false

// Custom collection ที่ใช้ Symbol.species
class TypedArray<T> extends Array<T> {
  static get [Symbol.species]() {
    return Array; // return Array แทน TypedArray เมื่อใช้ map/filter
  }

  where(predicate: (value: T) => boolean): TypedArray<T> {
    return this.filter(predicate) as TypedArray<T>;
  }

  select<U>(transform: (value: T) => U): U[] {
    return this.map(transform);
  }
}

const arr = new TypedArray<number>();
arr.push(1, 2, 3, 4, 5);

const filtered = arr.where((n) => n > 2);
const mapped = arr.select((n) => n * 2);

console.log("Filtered:", filtered); // [3, 4, 5]
console.log("Mapped:", mapped); // [2, 4, 6, 8, 10]
console.log(filtered instanceof TypedArray); // true
console.log(mapped instanceof Array); // true
```

---

## 5. Property Descriptors

### ตัวอย่างที่ 13: Object.defineProperty ขั้นสูง

```typescript
// สร้าง computed property ด้วย defineProperty
function computed<T extends object, K extends string>(
  obj: T,
  key: K,
  getter: () => unknown,
  setter?: (value: unknown) => void
): T & Record<K, unknown> {
  Object.defineProperty(obj, key, {
    get: getter,
    set: setter,
    enumerable: true,
    configurable: true,
  });
  return obj as T & Record<K, unknown>;
}

// Lazy property
function lazy<T extends object, K extends keyof T>(
  obj: T,
  key: K,
  factory: () => T[K]
): void {
  let computed = false;
  let value: T[K];

  Object.defineProperty(obj, key, {
    get() {
      if (!computed) {
        value = factory();
        computed = true;
        // Replace the descriptor with a simple value
        Object.defineProperty(obj, key, {
          value,
          writable: true,
          enumerable: true,
          configurable: true,
        });
      }
      return value;
    },
    enumerable: true,
    configurable: true,
  });
}

class Circle {
  constructor(readonly radius: number) {}
}

const circle = new Circle(5);

// เพิ่ม computed properties
lazy(circle as any, "area" as any, () => Math.PI * circle.radius ** 2);
lazy(circle as any, "circumference" as any, () => 2 * Math.PI * circle.radius);

console.log((circle as any).area); // 78.53...
console.log((circle as any).circumference); // 31.41...

// Non-enumerable property
function addHiddenProperty<T extends object>(
  obj: T,
  key: string,
  value: unknown
): T {
  Object.defineProperty(obj, key, {
    value,
    writable: true,
    enumerable: false, // ไม่แสดงใน for...in หรือ Object.keys
    configurable: true,
  });
  return obj;
}

const user = { name: "สมชาย", age: 25 };
addHiddenProperty(user, "_id", "hidden-id-123");
addHiddenProperty(user, "_createdAt", new Date());

console.log(Object.keys(user)); // ["name", "age"] ไม่มี _id
console.log(JSON.stringify(user)); // {"name":"สมชาย","age":25}
console.log((user as any)._id); // "hidden-id-123"
```

### ตัวอย่างที่ 14: Property Descriptor Utilities

```typescript
// Utility สำหรับ Property Descriptors
class PropertyDescriptorUtils {
  // Freeze individual property
  static freeze<T extends object, K extends keyof T>(obj: T, key: K): void {
    const descriptor = Object.getOwnPropertyDescriptor(obj, key);
    if (descriptor) {
      Object.defineProperty(obj, key, {
        ...descriptor,
        writable: false,
        configurable: false,
      });
    }
  }

  // Make property hidden
  static hide<T extends object, K extends keyof T>(obj: T, key: K): void {
    const descriptor = Object.getOwnPropertyDescriptor(obj, key);
    if (descriptor) {
      Object.defineProperty(obj, key, {
        ...descriptor,
        enumerable: false,
      });
    }
  }

  // Make property non-deletable
  static seal<T extends object, K extends keyof T>(obj: T, key: K): void {
    const descriptor = Object.getOwnPropertyDescriptor(obj, key);
    if (descriptor) {
      Object.defineProperty(obj, key, {
        ...descriptor,
        configurable: false,
      });
    }
  }

  // Get all descriptors including inherited
  static getAllDescriptors(
    obj: object
  ): Record<string, PropertyDescriptor> {
    const descriptors: Record<string, PropertyDescriptor> = {};
    let current: object | null = obj;

    while (current !== null && current !== Object.prototype) {
      Object.getOwnPropertyNames(current).forEach((name) => {
        if (!(name in descriptors)) {
          const descriptor = Object.getOwnPropertyDescriptor(current!, name);
          if (descriptor) {
            descriptors[name] = descriptor;
          }
        }
      });

      current = Object.getPrototypeOf(current);
    }

    return descriptors;
  }
}

// ตัวอย่างการใช้งาน
const config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
  retries: 3,
};

// Freeze สำคัญ config values
PropertyDescriptorUtils.freeze(config, "apiUrl");

try {
  config.apiUrl = "https://evil.com"; // ❌ Error in strict mode
} catch {
  console.log("ไม่สามารถแก้ไข apiUrl ได้");
}
```

---

## 6. Mixins Pattern

### ตัวอย่างที่ 15: Mixin Pattern ขั้นสูง

```typescript
// Base mixin type
type Constructor<T = {}> = new (...args: any[]) => T;
type GConstructor<T = {}> = new (...args: any[]) => T;

// Timestamp mixin
function WithTimestamps<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    readonly createdAt: Date = new Date();
    updatedAt: Date = new Date();

    touch(): void {
      this.updatedAt = new Date();
    }
  };
}

// Serializable mixin
function WithSerialization<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    toJSON(): string {
      return JSON.stringify(this, null, 2);
    }

    static fromJSON<T>(this: new (...args: any[]) => T, json: string): T {
      const data = JSON.parse(json);
      return Object.assign(new this(), data);
    }
  };
}

// Validation mixin
function WithValidation<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    private _errors: string[] = [];

    get errors(): string[] {
      return this._errors;
    }

    get isValid(): boolean {
      return this._errors.length === 0;
    }

    addError(message: string): void {
      this._errors.push(message);
    }

    clearErrors(): void {
      this._errors = [];
    }

    validate(): boolean {
      this.clearErrors();
      return this.isValid;
    }
  };
}

// Observable mixin
function WithObservable<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    private _observers: Map<string, Set<(value: unknown) => void>> = new Map();

    on(event: string, observer: (value: unknown) => void): () => void {
      if (!this._observers.has(event)) {
        this._observers.set(event, new Set());
      }
      this._observers.get(event)!.add(observer);

      return () => this._observers.get(event)?.delete(observer);
    }

    emit(event: string, value: unknown): void {
      this._observers.get(event)?.forEach((obs) => obs(value));
    }
  };
}

// Base class
class BaseEntity {
  constructor(public id: string, public name: string) {}
}

// Combined mixin
const TimestampedEntity = WithTimestamps(BaseEntity);
const SerializableTimestampedEntity = WithSerialization(TimestampedEntity);
const FullFeaturedEntity = WithObservable(
  WithValidation(SerializableTimestampedEntity)
);

class User extends FullFeaturedEntity {
  constructor(
    id: string,
    name: string,
    public email: string,
    public age: number
  ) {
    super(id, name);
  }

  override validate(): boolean {
    super.validate();
    if (!this.email.includes("@")) {
      this.addError("Email ไม่ถูกต้อง");
    }
    if (this.age < 0 || this.age > 150) {
      this.addError("อายุไม่ถูกต้อง");
    }
    return this.isValid;
  }
}

const user = new User("1", "สมชาย", "somchai@example.com", 25);

// Timestamps
console.log("สร้างเมื่อ:", user.createdAt);
user.touch();
console.log("อัปเดตเมื่อ:", user.updatedAt);

// Observable
const unsub = user.on("emailChanged", (newEmail) => {
  console.log("Email เปลี่ยนเป็น:", newEmail);
});

user.email = "new@example.com";
user.emit("emailChanged", user.email);
unsub();

// Validation
console.log("ถูกต้อง:", user.validate()); // true
user.email = "invalid-email";
console.log("ถูกต้อง:", user.validate()); // false
console.log("ข้อผิดพลาด:", user.errors);

// Serialization
const json = user.toJSON();
console.log("JSON:", json);
```

---

## 7. Aspect-Oriented Programming

### ตัวอย่างที่ 16: AOP ด้วย Decorators

```typescript
// Aspect-Oriented Programming (AOP) patterns

// Cross-cutting concern: Caching
function Cache(ttl: number = 60000) {
  return function (
    target: object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ): PropertyDescriptor {
    const originalMethod = descriptor.value;
    const cache = new Map<string, { value: unknown; expiry: number }>();

    descriptor.value = async function (...args: unknown[]) {
      const key = JSON.stringify(args);
      const cached = cache.get(key);

      if (cached && cached.expiry > Date.now()) {
        console.log(`[CACHE] Hit: ${String(propertyKey)}(${key})`);
        return cached.value;
      }

      const result = await originalMethod.apply(this, args);
      cache.set(key, { value: result, expiry: Date.now() + ttl });
      console.log(`[CACHE] Miss: ${String(propertyKey)}(${key}) - cached for ${ttl}ms`);
      return result;
    };

    return descriptor;
  };
}

// Cross-cutting concern: Retry
function Retry(
  maxAttempts: number = 3,
  delay: number = 1000,
  backoff: number = 2
) {
  return function (
    target: object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ): PropertyDescriptor {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: unknown[]) {
      let lastError: Error | undefined;
      let currentDelay = delay;

      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error as Error;
          if (attempt < maxAttempts) {
            console.log(
              `[RETRY] Attempt ${attempt}/${maxAttempts} failed: ${lastError.message}`
            );
            await new Promise((resolve) => setTimeout(resolve, currentDelay));
            currentDelay *= backoff;
          }
        }
      }

      throw new Error(
        `หลังจากพยายาม ${maxAttempts} ครั้งแล้วยังล้มเหลว: ${lastError?.message}`
      );
    };

    return descriptor;
  };
}

// Cross-cutting concern: Circuit Breaker
function CircuitBreaker(
  threshold: number = 5,
  timeout: number = 30000
) {
  return function (
    target: object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ): PropertyDescriptor {
    const originalMethod = descriptor.value;
    let failureCount = 0;
    let lastFailureTime: number | null = null;
    let state: "CLOSED" | "OPEN" | "HALF_OPEN" = "CLOSED";

    descriptor.value = async function (...args: unknown[]) {
      if (state === "OPEN") {
        if (
          lastFailureTime !== null &&
          Date.now() - lastFailureTime > timeout
        ) {
          state = "HALF_OPEN";
          console.log(`[CIRCUIT BREAKER] ${String(propertyKey)}: HALF_OPEN - ทดสอบ`);
        } else {
          throw new Error(
            `Circuit is OPEN - ${String(propertyKey)} ไม่สามารถเรียกได้`
          );
        }
      }

      try {
        const result = await originalMethod.apply(this, args);
        if (state === "HALF_OPEN") {
          state = "CLOSED";
          failureCount = 0;
          console.log(`[CIRCUIT BREAKER] ${String(propertyKey)}: CLOSED - กลับมาทำงานได้`);
        }
        return result;
      } catch (error) {
        failureCount++;
        lastFailureTime = Date.now();

        if (failureCount >= threshold) {
          state = "OPEN";
          console.log(
            `[CIRCUIT BREAKER] ${String(propertyKey)}: OPEN - ล้มเหลว ${failureCount} ครั้ง`
          );
        }

        throw error;
      }
    };

    return descriptor;
  };
}

// ตัวอย่างการใช้งาน
class ExternalAPIService {
  @Cache(5000)
  @Retry(3, 500)
  @CircuitBreaker(5, 30000)
  async fetchUserData(userId: string): Promise<{ id: string; name: string }> {
    // จำลอง API call
    if (Math.random() < 0.3) {
      throw new Error("Network error");
    }
    return { id: userId, name: "สมชาย" };
  }

  @Cache(60000)
  async getProductList(): Promise<string[]> {
    return ["สินค้า 1", "สินค้า 2", "สินค้า 3"];
  }
}

async function demonstrateAOP() {
  const service = new ExternalAPIService();

  try {
    const user = await service.fetchUserData("user-1");
    console.log("User:", user);

    // Second call - from cache
    const user2 = await service.fetchUserData("user-1");
    console.log("User (cached):", user2);
  } catch (error) {
    console.error("Error:", (error as Error).message);
  }
}

demonstrateAOP();
```

---

## 8. Abstract Mixin Classes

### ตัวอย่างที่ 17: Abstract Mixin สำหรับ Design Patterns

```typescript
// Abstract mixin สำหรับ Repository pattern
abstract class RepositoryMixin<T extends { id: string }> {
  protected abstract storage: Map<string, T>;

  findById(id: string): T | undefined {
    return this.storage.get(id);
  }

  findAll(): T[] {
    return Array.from(this.storage.values());
  }

  save(entity: T): T {
    this.storage.set(entity.id, entity);
    return entity;
  }

  delete(id: string): boolean {
    return this.storage.delete(id);
  }

  exists(id: string): boolean {
    return this.storage.has(id);
  }

  count(): number {
    return this.storage.size;
  }
}

// Abstract mixin สำหรับ Event Emitter
abstract class EventEmitterMixin {
  protected events: Map<string, Set<Function>> = new Map();

  on(event: string, handler: Function): this {
    if (!this.events.has(event)) {
      this.events.set(event, new Set());
    }
    this.events.get(event)!.add(handler);
    return this;
  }

  off(event: string, handler: Function): this {
    this.events.get(event)?.delete(handler);
    return this;
  }

  protected emit(event: string, ...args: unknown[]): void {
    this.events.get(event)?.forEach((handler) => handler(...args));
  }
}

// Concrete implementation
interface Product {
  id: string;
  name: string;
  price: number;
}

class ProductRepository extends RepositoryMixin<Product> {
  protected storage = new Map<string, Product>();

  findByName(name: string): Product | undefined {
    return this.findAll().find((p) => p.name.includes(name));
  }

  findByPriceRange(min: number, max: number): Product[] {
    return this.findAll().filter((p) => p.price >= min && p.price <= max);
  }
}

// การใช้งาน
const repo = new ProductRepository();

repo.save({ id: "1", name: "โน้ตบุ๊ค", price: 45000 });
repo.save({ id: "2", name: "สมาร์ทโฟน", price: 25000 });
repo.save({ id: "3", name: "แท็บเล็ต", price: 15000 });

console.log("สินค้าทั้งหมด:", repo.findAll());
console.log("หาตามชื่อ:", repo.findByName("บุ๊ค"));
console.log("ช่วงราคา:", repo.findByPriceRange(10000, 30000));
```

---

## สรุป

Meta-programming ใน TypeScript มีเครื่องมือหลายอย่าง:

1. **Decorators** - เพิ่มพฤติกรรมให้ classes, methods, properties และ parameters
2. **Reflect.metadata** - เก็บ metadata สำหรับ DI และ AOP
3. **Proxy** - intercept operations บน objects
4. **Symbols** - สร้าง unique identifiers และ customize built-in behaviors
5. **Property Descriptors** - ควบคุม property attributes
6. **Mixins** - reuse behaviors ผ่าน composition
7. **AOP** - แยก cross-cutting concerns ออกจาก business logic

Meta-programming ช่วยให้:
- โค้ดแยกออกได้ดีขึ้น (Separation of Concerns)
- สามารถเพิ่มพฤติกรรมโดยไม่แก้ original code
- สร้าง frameworks และ libraries ที่ยืดหยุ่น
- ลดโค้ดซ้ำผ่าน aspect-oriented patterns

---

*จบตอนที่ 78 - Meta-programming in TypeScript*
