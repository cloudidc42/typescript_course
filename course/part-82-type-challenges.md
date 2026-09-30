# Part 82: TypeScript Type Challenges

## บทนำ

Type Challenges คือปัญหาท้าทายที่ให้เราเขียน type-level programs ใน TypeScript บทนี้รวบรวม challenges ตั้งแต่ระดับ Easy ไปจนถึง Extreme พร้อมคำอธิบายและ solution ที่ครบถ้วน

---

## ระดับ Easy

### Challenge 1: Implement Partial

**โจทย์**: สร้าง `MyPartial<T>` ที่ทำให้ทุก property เป็น optional

```typescript
// โจทย์
type MyPartial<T> = /* ตอบตรงนี้ */

// ทดสอบ
interface User {
    name: string;
    age: number;
    email: string;
}

type PartialUser = MyPartial<User>;
// ควรได้: { name?: string; age?: number; email?: string; }
```

**Solution**:
```typescript
type MyPartial<T> = {
    [K in keyof T]?: T[K];
};

// อธิบาย:
// - keyof T ดึง union ของ keys ทั้งหมดจาก T
// - K in keyof T วน iterate ผ่านแต่ละ key
// - ? ทำให้ property เป็น optional
// - T[K] คือ type ของ property K

// ตัวอย่าง
type PartialUser = MyPartial<User>;
// { name?: string | undefined; age?: number | undefined; email?: string | undefined; }

const partialUser: PartialUser = { name: "Alice" }; // ✅ valid
```

---

### Challenge 2: Implement Required

**โจทย์**: สร้าง `MyRequired<T>` ที่ทำให้ทุก property เป็น required (ลบ optional ออก)

```typescript
type MyRequired<T> = /* ตอบตรงนี้ */

type PartialConfig = {
    host?: string;
    port?: number;
    debug?: boolean;
};

type RequiredConfig = MyRequired<PartialConfig>;
// ควรได้: { host: string; port: number; debug: boolean; }
```

**Solution**:
```typescript
type MyRequired<T> = {
    [K in keyof T]-?: T[K];
};

// อธิบาย:
// -? ลบ ? modifier ออกจาก property
// ทำให้ property ที่เคยเป็น optional กลายเป็น required

const config: RequiredConfig = {
    host: "localhost",
    port: 3000,
    debug: true
}; // ✅ valid

// const badConfig: RequiredConfig = { host: "localhost" }; // ❌ Error
```

---

### Challenge 3: Implement Readonly

**โจทย์**: สร้าง `MyReadonly<T>` ที่ทำให้ทุก property เป็น readonly

```typescript
type MyReadonly<T> = /* ตอบตรงนี้ */

interface Point {
    x: number;
    y: number;
}

type ReadonlyPoint = MyReadonly<Point>;
// ควรได้: { readonly x: number; readonly y: number; }
```

**Solution**:
```typescript
type MyReadonly<T> = {
    readonly [K in keyof T]: T[K];
};

const point: ReadonlyPoint = { x: 1, y: 2 };
// point.x = 3; // ❌ Error: Cannot assign to 'x' because it is read-only
```

---

### Challenge 4: Tuple to Object

**โจทย์**: สร้าง `TupleToObject<T>` ที่แปลง tuple เป็น object

```typescript
type TupleToObject<T extends readonly (string | number | symbol)[]> = /* ตอบตรงนี้ */

const tuple = ["tesla", "model 3", "model X"] as const;
type Result = TupleToObject<typeof tuple>;
// ควรได้: { tesla: "tesla"; "model 3": "model 3"; "model X": "model X"; }
```

**Solution**:
```typescript
type TupleToObject<T extends readonly (string | number | symbol)[]> = {
    [K in T[number]]: K;
};

// อธิบาย:
// T[number] ดึง union ของ values ทั้งหมดจาก tuple
// K in T[number] ใช้แต่ละ value เป็น key
// : K ทำให้ value เป็น type เดียวกับ key

const fruits = ["apple", "banana", "cherry"] as const;
type FruitObject = TupleToObject<typeof fruits>;
// { apple: "apple"; banana: "banana"; cherry: "cherry"; }
```

---

### Challenge 5: First of Array

**โจทย์**: สร้าง `First<T>` ที่ดึง element แรกจาก array

```typescript
type First<T extends any[]> = /* ตอบตรงนี้ */

type A = First<[3, 2, 1]>;    // ควรได้: 3
type B = First<[() => 123, { a: string }]>; // ควรได้: () => 123
type C = First<[]>;            // ควรได้: never
```

**Solution**:
```typescript
type First<T extends any[]> = T extends [infer F, ...any[]] ? F : never;

// อธิบาย:
// T extends [infer F, ...any[]] ตรวจสอบว่า T มี element อย่างน้อย 1 ตัว
// infer F ดึง type ของ element แรก
// ถ้า T ว่างเปล่า คืน never

// วิธีอื่น:
type First2<T extends any[]> = T["length"] extends 0 ? never : T[0];
```

---

### Challenge 6: Length of Tuple

**โจทย์**: สร้าง `Length<T>` ที่คืน length ของ tuple

```typescript
type Length<T extends readonly any[]> = /* ตอบตรงนี้ */

type A = Length<[1, 2, 3]>;    // ควรได้: 3
type B = Length<["a", "b"]>;   // ควรได้: 2
type C = Length<[]>;           // ควรได้: 0
```

**Solution**:
```typescript
type Length<T extends readonly any[]> = T["length"];

// TypeScript รู้ length ของ tuple โดยอัตโนมัติ
// T["length"] ดึง literal type ของ length

type L1 = Length<[1, 2, 3]>;  // 3
type L2 = Length<[]>;          // 0
```

---

### Challenge 7: Implement Exclude

**โจทย์**: สร้าง `MyExclude<T, U>` ที่ลบ types ใน U ออกจาก T

```typescript
type MyExclude<T, U> = /* ตอบตรงนี้ */

type A = MyExclude<"a" | "b" | "c", "a">;       // "b" | "c"
type B = MyExclude<string | number | boolean, boolean>; // string | number
```

**Solution**:
```typescript
type MyExclude<T, U> = T extends U ? never : T;

// อธิบาย:
// TypeScript กระจาย conditional types ผ่าน union types
// ถ้า T extends U คืน never (ลบออก)
// ถ้าไม่ คืน T (เก็บไว้)
// never ใน union จะถูกลบออกโดยอัตโนมัติ

type C = MyExclude<"a" | "b" | "c", "a" | "b">;
// = ("a" extends "a" | "b" ? never : "a") |
//   ("b" extends "a" | "b" ? never : "b") |
//   ("c" extends "a" | "b" ? never : "c")
// = never | never | "c"
// = "c"
```

---

### Challenge 8: Awaited

**โจทย์**: สร้าง `MyAwaited<T>` ที่ unwrap type จาก Promise

```typescript
type MyAwaited<T> = /* ตอบตรงนี้ */

type X = Promise<string>;
type Y = Promise<{ field: number }>;
type Z = Promise<Promise<string | number>>;

type A = MyAwaited<X>; // string
type B = MyAwaited<Y>; // { field: number }
type C = MyAwaited<Z>; // string | number
```

**Solution**:
```typescript
type MyAwaited<T> = T extends PromiseLike<infer U> 
    ? MyAwaited<U>  // recursive สำหรับ nested Promises
    : T;

// อธิบาย:
// ตรวจสอบว่า T เป็น PromiseLike หรือไม่
// ถ้าใช่ ดึง inner type และ recurse
// ถ้าไม่ คืน T เอง
// รองรับ Promise ซ้อนหลายชั้น

type D = MyAwaited<Promise<Promise<Promise<number>>>>;
// = MyAwaited<Promise<Promise<number>>>
// = MyAwaited<Promise<number>>
// = MyAwaited<number>
// = number
```

---

### Challenge 9: If

**โจทย์**: สร้าง `If<C, T, F>` ที่คืน T ถ้า C เป็น true และ F ถ้า C เป็น false

```typescript
type If<C extends boolean, T, F> = /* ตอบตรงนี้ */

type A = If<true, "a", "b">;  // "a"
type B = If<false, "a", "b">; // "b"
```

**Solution**:
```typescript
type If<C extends boolean, T, F> = C extends true ? T : F;

// ตัวอย่างการใช้งาน
type IsAdmin = true;
type AdminRoute = If<IsAdmin, "/admin/dashboard", "/user/dashboard">;
// "/admin/dashboard"

type IsLoggedIn = false;
type DefaultPage = If<IsLoggedIn, "/home", "/login">;
// "/login"
```

---

## ระดับ Medium

### Challenge 10: MyPick

**โจทย์**: สร้าง `MyPick<T, K>` ที่เลือก subset ของ properties จาก T

```typescript
type MyPick<T, K extends keyof T> = /* ตอบตรงนี้ */

interface Todo {
    title: string;
    description: string;
    completed: boolean;
}

type TodoPreview = MyPick<Todo, "title" | "completed">;
// { title: string; completed: boolean; }
```

**Solution**:
```typescript
type MyPick<T, K extends keyof T> = {
    [P in K]: T[P];
};

// อธิบาย:
// K extends keyof T ทำให้แน่ใจว่า K เป็น key ที่มีใน T
// P in K วน iterate ผ่าน keys ใน K เท่านั้น
// T[P] ดึง type ของแต่ละ property

// ตัวอย่างการใช้งานกับ API response
interface ApiResponse {
    data: unknown;
    status: number;
    statusText: string;
    headers: Record<string, string>;
    config: unknown;
}

type MinimalResponse = MyPick<ApiResponse, "data" | "status">;
// { data: unknown; status: number; }
```

---

### Challenge 11: MyOmit

**โจทย์**: สร้าง `MyOmit<T, K>` ที่ลบ properties บางอย่างออกจาก T

```typescript
type MyOmit<T, K extends keyof any> = /* ตอบตรงนี้ */

interface Todo {
    title: string;
    description: string;
    completed: boolean;
    createdAt: number;
}

type TodoPreview = MyOmit<Todo, "description">;
// { title: string; completed: boolean; createdAt: number; }
```

**Solution**:
```typescript
// วิธีที่ 1: ใช้ Exclude กับ mapped types
type MyOmit<T, K extends keyof any> = {
    [P in Exclude<keyof T, K>]: T[P];
};

// วิธีที่ 2: ใช้ as clause
type MyOmit2<T, K extends keyof any> = {
    [P in keyof T as P extends K ? never : P]: T[P];
};

// ตัวอย่าง
interface User {
    id: string;
    name: string;
    password: string;
    email: string;
    role: string;
}

type SafeUser = MyOmit<User, "password">;
// { id: string; name: string; email: string; role: string; }
```

---

### Challenge 12: ReadonlyDeep

**โจทย์**: สร้าง `DeepReadonly<T>` ที่ทำให้ทุก property (รวมถึง nested) เป็น readonly

```typescript
type DeepReadonly<T> = /* ตอบตรงนี้ */

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
        ssl: boolean;
    };
}

type ReadonlyConfig = DeepReadonly<Config>;
// ทุก property รวมถึง nested จะเป็น readonly
```

**Solution**:
```typescript
type DeepReadonly<T> = T extends (...args: any[]) => any
    ? T  // ถ้าเป็น function ไม่ต้อง wrap
    : T extends object
        ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
        : T;  // primitive types ไม่ต้อง wrap

// ทดสอบ
const config: ReadonlyConfig = {
    database: {
        host: "localhost",
        port: 5432,
        credentials: {
            username: "admin",
            password: "secret"
        }
    },
    server: {
        port: 3000,
        ssl: true
    }
};

// config.database.host = "remote"; // ❌ Error
// config.database.credentials.password = "new"; // ❌ Error
```

---

### Challenge 13: TupleToUnion

**โจทย์**: สร้าง `TupleToUnion<T>` ที่แปลง tuple เป็น union type

```typescript
type TupleToUnion<T extends any[]> = /* ตอบตรงนี้ */

type Arr = ["1", "2", "3"];
type Test = TupleToUnion<Arr>; // "1" | "2" | "3"
```

**Solution**:
```typescript
type TupleToUnion<T extends any[]> = T[number];

// อธิบาย:
// T[number] ดึง union ของ element types ทั้งหมด
// เพราะ number เป็น index ของ array

// วิธีอื่น: recursive
type TupleToUnion2<T extends any[]> = 
    T extends [infer F, ...infer R] 
        ? F | TupleToUnion2<R> 
        : never;

// ตัวอย่าง
type Colors = ["red", "green", "blue"];
type Color = TupleToUnion<Colors>; // "red" | "green" | "blue"
```

---

### Challenge 14: Chainable Options

**โจทย์**: สร้าง `Chainable<O>` ที่รองรับ method chaining ในการ set options

```typescript
type Chainable<O = {}> = {
    option<K extends string, V>(
        key: K,
        value: V
    ): Chainable<O & Record<K, V>>;
    get(): O;
};

declare const config: Chainable;

const result = config
    .option("foo", 123)
    .option("name", "type-challenges")
    .option("bar", { value: "Hello World" })
    .get();

type Expected = {
    foo: number;
    name: string;
    bar: { value: string };
};
```

**Solution**:
```typescript
type Chainable<O = {}> = {
    option<K extends string, V>(
        key: K extends keyof O ? never : K,  // ป้องกันการ set key ซ้ำ
        value: V
    ): Chainable<Omit<O, K> & Record<K, V>>;
    get(): O;
};

// Implementation
function createChainable<O extends object>(obj: O): Chainable<O> {
    return {
        option<K extends string, V>(key: K, value: V) {
            return createChainable({ ...obj, [key]: value } as any);
        },
        get() {
            return obj as O;
        }
    };
}

const config = createChainable({})
    .option("theme", "dark")
    .option("language", "th")
    .option("fontSize", 14)
    .get();

console.log(config.theme);    // "dark"
console.log(config.language); // "th"
console.log(config.fontSize); // 14
```

---

### Challenge 15: Last of Array

**โจทย์**: สร้าง `Last<T>` ที่ดึง element สุดท้ายจาก array

```typescript
type Last<T extends any[]> = /* ตอบตรงนี้ */

type A = Last<[3, 2, 1]>;   // 1
type B = Last<[() => 123, { a: string }]>; // { a: string }
```

**Solution**:
```typescript
type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;

// อธิบาย:
// [...any[], infer L] match กับ array ที่มี element อย่างน้อย 1 ตัว
// infer L ดึง type ของ element สุดท้าย

// วิธีอื่น:
type Last2<T extends any[]> = [never, ...T][T["length"]];
// ใช้ shift technique: เพิ่ม never ไว้ข้างหน้า แล้วเข้าถึงด้วย T.length
```

---

### Challenge 16: Pop

**โจทย์**: สร้าง `Pop<T>` ที่ลบ element สุดท้ายออกจาก tuple

```typescript
type Pop<T extends any[]> = /* ตอบตรงนี้ */

type A = Pop<[3, 2, 1]>;         // [3, 2]
type B = Pop<[() => 123, { a: string }]>; // [() => 123]
```

**Solution**:
```typescript
type Pop<T extends any[]> = T extends [...infer R, any] ? R : never;

// อธิบาย:
// [...infer R, any] match กับ array ที่มี element อย่างน้อย 1 ตัว
// infer R ดึง elements ทั้งหมดยกเว้นตัวสุดท้าย

// ทดสอบ
type C = Pop<[]>; // never (array ว่างเปล่า)
```

---

### Challenge 17: Push

**โจทย์**: สร้าง `Push<T, V>` ที่เพิ่ม element ต่อท้าย tuple

```typescript
type Push<T extends any[], V> = /* ตอบตรงนี้ */

type A = Push<[1, 2], "3">; // [1, 2, "3"]
```

**Solution**:
```typescript
type Push<T extends any[], V> = [...T, V];

// Shift (เพิ่มข้างหน้า)
type Unshift<T extends any[], V> = [V, ...T];

// ทดสอบ
type B = Push<[number, string], boolean>; // [number, string, boolean]
type C = Unshift<[number, string], boolean>; // [boolean, number, string]
```

---

### Challenge 18: Parameters

**โจทย์**: สร้าง `MyParameters<T>` ที่ดึง parameter types จาก function

```typescript
type MyParameters<T extends (...args: any[]) => any> = /* ตอบตรงนี้ */

function foo(arg1: string, arg2: number): void {}
function bar(arg1: boolean, arg2: { a: "A" }): void {}

type A = MyParameters<typeof foo>; // [arg1: string, arg2: number]
type B = MyParameters<typeof bar>; // [arg1: boolean, arg2: { a: "A" }]
```

**Solution**:
```typescript
type MyParameters<T extends (...args: any[]) => any> = 
    T extends (...args: infer P) => any ? P : never;

// ตัวอย่างการใช้งาน
type EventHandler = (event: MouseEvent, data: { x: number; y: number }) => void;
type HandlerParams = MyParameters<EventHandler>;
// [event: MouseEvent, data: { x: number; y: number }]

// ใช้ spread เพื่อเรียก function ด้วย parameters ที่ถูกต้อง
function wrapHandler<T extends (...args: any[]) => any>(
    handler: T,
    ...args: MyParameters<T>
): ReturnType<T> {
    return handler(...args);
}
```

---

### Challenge 19: ReturnType

**โจทย์**: สร้าง `MyReturnType<T>` ที่ดึง return type จาก function

```typescript
type MyReturnType<T extends (...args: any[]) => any> = /* ตอบตรงนี้ */

const fn = (v: boolean) => (v ? 1 : 2);
type A = MyReturnType<typeof fn>; // 1 | 2
```

**Solution**:
```typescript
type MyReturnType<T extends (...args: any[]) => any> = 
    T extends (...args: any[]) => infer R ? R : never;

// ตัวอย่าง
async function fetchUser(id: string): Promise<{ id: string; name: string }> {
    return { id, name: "Alice" };
}

type FetchResult = MyReturnType<typeof fetchUser>;
// Promise<{ id: string; name: string }>

// ร่วมกับ Awaited
type UnwrappedFetchResult = Awaited<FetchResult>;
// { id: string; name: string }
```

---

### Challenge 20: Omit

**โจทย์**: สร้าง `Omit<T, K>` โดยไม่ใช้ built-in Omit

```typescript
type MyOmit<T, K extends keyof T> = /* ตอบตรงนี้ */
```

**Solution แบบ Advanced**:
```typescript
// วิธีที่ 1: ใช้ Pick + Exclude
type MyOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

// วิธีที่ 2: ใช้ mapped types กับ as
type MyOmit2<T, K extends keyof T> = {
    [P in keyof T as P extends K ? never : P]: T[P];
};

// วิธีที่ 3: ใช้ infer
type MyOmit3<T, K extends keyof any> = {
    [P in keyof T as Exclude<P, K>]: T[P];
};

// ตัวอย่าง: สร้าง type สำหรับ update operations
type UpdateInput<T> = MyOmit<T, "id" | "createdAt" | "updatedAt">;

interface Article {
    id: string;
    title: string;
    content: string;
    createdAt: Date;
    updatedAt: Date;
}

type UpdateArticle = UpdateInput<Article>;
// { title: string; content: string; }
```

---

## ระดับ Medium-Hard

### Challenge 21: Type Lookup

**โจทย์**: สร้าง `LookUp<U, T>` ที่ค้นหา type จาก union ด้วย type property

```typescript
interface Cat {
    type: "cat";
    breeds: string[];
}

interface Dog {
    type: "dog";
    color: string;
}

type Animal = Cat | Dog;

type LookUp<U, T> = /* ตอบตรงนี้ */

type A = LookUp<Animal, "cat">; // Cat
type B = LookUp<Animal, "dog">; // Dog
```

**Solution**:
```typescript
type LookUp<U, T extends string> = U extends { type: T } ? U : never;

// ตัวอย่างกับ action types
type LoginAction = { type: "LOGIN"; payload: { username: string } };
type LogoutAction = { type: "LOGOUT" };
type UpdateAction = { type: "UPDATE"; payload: { data: unknown } };

type Action = LoginAction | LogoutAction | UpdateAction;

type GetAction<T extends string> = LookUp<Action, T>;
type Login = GetAction<"LOGIN">; // LoginAction
type Logout = GetAction<"LOGOUT">; // LogoutAction
```

---

### Challenge 22: Trim Left

**โจทย์**: สร้าง `TrimLeft<S>` ที่ลบ whitespace ออกจากซ้ายของ string

```typescript
type TrimLeft<S extends string> = /* ตอบตรงนี้ */

type A = TrimLeft<"  Hello World  ">; // "Hello World  "
type B = TrimLeft<"   \n\n\n    ">; // ""
```

**Solution**:
```typescript
type Whitespace = " " | "\n" | "\t";

type TrimLeft<S extends string> = 
    S extends `${Whitespace}${infer Rest}` 
        ? TrimLeft<Rest>  // recursive ลบ whitespace ทีละตัว
        : S;

// Trim Right
type TrimRight<S extends string> = 
    S extends `${infer Rest}${Whitespace}` 
        ? TrimRight<Rest> 
        : S;

// Trim ทั้งสองข้าง
type Trim<S extends string> = TrimLeft<TrimRight<S>>;

type C = Trim<"  Hello World  ">; // "Hello World"
```

---

### Challenge 23: Capitalize

**โจทย์**: สร้าง `MyCapitalize<S>` ที่ทำให้ตัวอักษรแรกเป็นตัวพิมพ์ใหญ่

```typescript
type MyCapitalize<S extends string> = /* ตอบตรงนี้ */

type A = MyCapitalize<"hello world">; // "Hello world"
```

**Solution**:
```typescript
type MyCapitalize<S extends string> = 
    S extends `${infer F}${infer R}` 
        ? `${Uppercase<F>}${R}` 
        : S;

// หรือใช้ built-in Capitalize
type MyCapitalize2<S extends string> = Capitalize<S>;

// เพิ่ม utility types
type PascalCase<S extends string> = 
    S extends `${infer Word}_${infer Rest}` 
        ? `${Capitalize<Word>}${PascalCase<Rest>}` 
        : Capitalize<S>;

type A = PascalCase<"hello_world_foo">;
// "HelloWorldFoo"
```

---

### Challenge 24: Replace

**โจทย์**: สร้าง `Replace<S, From, To>` ที่แทนที่ string

```typescript
type Replace<S extends string, From extends string, To extends string> = /* ตอบตรงนี้ */

type A = Replace<"types are fun!", "fun", "awesome">; // "types are awesome!"
```

**Solution**:
```typescript
type Replace<S extends string, From extends string, To extends string> = 
    From extends "" 
        ? S 
        : S extends `${infer L}${From}${infer R}` 
            ? `${L}${To}${R}` 
            : S;

// ReplaceAll
type ReplaceAll<S extends string, From extends string, To extends string> = 
    From extends "" 
        ? S 
        : S extends `${infer L}${From}${infer R}` 
            ? ReplaceAll<`${L}${To}${R}`, From, To> 
            : S;

type B = ReplaceAll<"t y p e s", " ", "">; // "types"
```

---

### Challenge 25: Append Argument

**โจทย์**: สร้าง `AppendArgument<Fn, A>` ที่เพิ่ม argument ใหม่ต่อท้าย function

```typescript
type AppendArgument<Fn extends (...args: any[]) => any, A> = /* ตอบตรงนี้ */

type Fn = (a: number, b: string) => number;
type Result = AppendArgument<Fn, boolean>;
// (a: number, b: string, x: boolean) => number
```

**Solution**:
```typescript
type AppendArgument<Fn extends (...args: any[]) => any, A> = 
    Fn extends (...args: infer Args) => infer R 
        ? (...args: [...Args, A]) => R 
        : never;

// PrependArgument
type PrependArgument<Fn extends (...args: any[]) => any, A> = 
    Fn extends (...args: infer Args) => infer R 
        ? (...args: [A, ...Args]) => R 
        : never;

// ตัวอย่าง
type Logger = (message: string) => void;
type TimestampedLogger = PrependArgument<Logger, Date>;
// (arg: Date, message: string) => void
```

---

### Challenge 26: Permutation

**โจทย์**: สร้าง `Permutation<T>` ที่สร้าง permutations ทั้งหมดของ union type

```typescript
type Permutation<T, K = T> = /* ตอบตรงนี้ */

type A = Permutation<"A" | "B" | "C">;
// ["A", "B", "C"] | ["A", "C", "B"] | ["B", "A", "C"] | ...
```

**Solution**:
```typescript
type Permutation<T, K = T> = [T] extends [never] 
    ? [] 
    : K extends K 
        ? [K, ...Permutation<Exclude<T, K>>] 
        : never;

// อธิบาย:
// [T] extends [never] ตรวจสอบว่า T ว่างเปล่า (หยุด recursion)
// K extends K ใช้ distributive conditional types
// [K, ...Permutation<Exclude<T, K>>] สร้าง permutation โดย fix K แล้ว recurse กับที่เหลือ
```

---

### Challenge 27: Length of String

**โจทย์**: สร้าง `LengthOfString<S>` ที่คืน length ของ string ใน type level

```typescript
type LengthOfString<S extends string> = /* ตอบตรงนี้ */

type A = LengthOfString<"kumiko">; // 6
type B = LengthOfString<"reina">; // 5
```

**Solution**:
```typescript
type LengthOfString<S extends string, T extends string[] = []> = 
    S extends `${string}${infer Rest}` 
        ? LengthOfString<Rest, [string, ...T]> 
        : T["length"];

// อธิบาย:
// ใช้ accumulator array T เพื่อนับตัวอักษร
// แต่ละ recursive step เพิ่ม element ใน T
// เมื่อ string หมด คืน T.length
```

---

### Challenge 28: Flatten

**โจทย์**: สร้าง `Flatten<T>` ที่ flatten nested arrays

```typescript
type Flatten<T extends any[]> = /* ตอบตรงนี้ */

type A = Flatten<[1, 2, [3, 4], [[[5]]]]>; // [1, 2, 3, 4, 5]
```

**Solution**:
```typescript
type Flatten<T extends any[]> = T extends [infer F, ...infer R] 
    ? F extends any[] 
        ? [...Flatten<F>, ...Flatten<R>] 
        : [F, ...Flatten<R>] 
    : [];

// ตัวอย่าง
type B = Flatten<[[1, 2], [3, [4, 5]], 6]>; // [1, 2, 3, 4, 5, 6]
```

---

## ระดับ Hard

### Challenge 29: Simple Vue

**โจทย์**: สร้าง `SimpleVue` ที่ type-safe สำหรับ Vue options API

```typescript
declare function SimpleVue<D, C, M>(options: {
    data: () => D;
    computed: C & ThisType<D>;
    methods: M & ThisType<D & C & M>;
}): unknown;

const instance = SimpleVue({
    data() {
        return { firstName: "John", lastName: "Doe", age: 10 };
    },
    computed: {
        fullName() {
            return `${this.firstName} ${this.lastName}`;
        },
    },
    methods: {
        hi() {
            alert(this.fullName); // สามารถเข้าถึง computed ได้
        },
    },
});
```

**Solution**:
```typescript
declare function SimpleVue<D, C, M>(options: {
    data: () => D;
    computed: C & ThisType<D>;
    methods: M & ThisType<D & {
        [K in keyof C]: C[K] extends () => infer R ? R : never;
    } & M>;
}): unknown;
```

---

### Challenge 30: Currying

**โจทย์**: สร้าง `Currying<T>` สำหรับ curry function

```typescript
type Currying<T> = /* ตอบตรงนี้ */

declare function Currying<T extends (...args: any[]) => any>(
    fn: T
): Currying<T>;

const add = Currying((a: number, b: number, c: number) => a + b + c);
const addResult = add(1)(2)(3); // number
```

**Solution**:
```typescript
type Currying<T> = T extends ((...args: infer Args) => infer R)
    ? Args extends [infer First, ...infer Rest]
        ? Rest extends []
            ? (arg: First) => R
            : (arg: First) => Currying<(...args: Rest) => R>
        : () => R
    : T;

// Implementation
function curry<T extends (...args: any[]) => any>(fn: T): Currying<T> {
    const arity = fn.length;
    return function curried(...args: any[]): any {
        if (args.length >= arity) {
            return fn(...args);
        }
        return (...moreArgs: any[]) => curried(...args, ...moreArgs);
    } as any;
}

const multiply = curry((a: number, b: number, c: number) => a * b * c);
console.log(multiply(2)(3)(4)); // 24
```

---

### Challenge 31: Union to Intersection

**โจทย์**: สร้าง `UnionToIntersection<U>` ที่แปลง union เป็น intersection

```typescript
type UnionToIntersection<U> = /* ตอบตรงนี้ */

type A = UnionToIntersection<"a" | "b">; // "a" & "b" (= never)
type B = UnionToIntersection<{ name: string } | { age: number }>;
// { name: string } & { age: number }
```

**Solution**:
```typescript
type UnionToIntersection<U> = 
    (U extends any ? (x: U) => void : never) extends (x: infer I) => void 
        ? I 
        : never;

// อธิบาย:
// ขั้นที่ 1: แปลง union U เป็น union ของ functions ที่รับ U
//   (U extends any ? (x: U) => void : never)
//   = ((x: A) => void) | ((x: B) => void)
//
// ขั้นที่ 2: infer I จาก function ที่รับ I
//   เนื่องจาก function contravariant ใน parameter position
//   ((x: A) => void) | ((x: B) => void) extends (x: I) => void
//   ทำให้ I = A & B

// ตัวอย่าง
type Merged = UnionToIntersection<
    { id: string } | { name: string } | { age: number }
>;
// { id: string } & { name: string } & { age: number }
// = { id: string; name: string; age: number }
```

---

### Challenge 32: Get Required Keys

**โจทย์**: สร้าง `RequiredKeys<T>` ที่ดึง keys ที่เป็น required

```typescript
type RequiredKeys<T> = /* ตอบตรงนี้ */

interface User {
    name: string;
    age?: number;
    email: string;
    phone?: string;
}

type Required = RequiredKeys<User>; // "name" | "email"
```

**Solution**:
```typescript
type RequiredKeys<T> = {
    [K in keyof T]-?: {} extends Pick<T, K> ? never : K;
}[keyof T];

// อธิบาย:
// Pick<T, K> สร้าง type ที่มีแค่ property K
// {} extends Pick<T, K> ตรวจสอบว่า property เป็น optional
// ถ้า optional -> never (ไม่เอา)
// ถ้า required -> K (เอา)

// OptionalKeys
type OptionalKeys<T> = {
    [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];

type Optional = OptionalKeys<User>; // "age" | "phone"
```

---

### Challenge 33: Zip

**โจทย์**: สร้าง `Zip<T, U>` ที่รวม tuples สองอันเข้าด้วยกัน

```typescript
type Zip<T extends any[], U extends any[]> = /* ตอบตรงนี้ */

type A = Zip<[1, 2], [true, false]>; // [[1, true], [2, false]]
type B = Zip<[1, 2, 3], ["a"]>;      // [[1, "a"]]
```

**Solution**:
```typescript
type Zip<T extends any[], U extends any[]> = 
    T extends [infer THead, ...infer TTail]
        ? U extends [infer UHead, ...infer UTail]
            ? [[THead, UHead], ...Zip<TTail, UTail>]
            : []
        : [];

// Unzip
type Unzip<T extends [any, any][]> = {
    [K in 0 | 1]: {
        [I in keyof T]: T[I][K];
    };
};
```

---

### Challenge 34: IsTuple

**โจทย์**: สร้าง `IsTuple<T>` ที่ตรวจสอบว่า T เป็น tuple หรือ array

```typescript
type IsTuple<T> = /* ตอบตรงนี้ */

type A = IsTuple<[number]>;       // true
type B = IsTuple<readonly [1]>;   // true
type C = IsTuple<number[]>;       // false
```

**Solution**:
```typescript
type IsTuple<T> = [T] extends [never] 
    ? false 
    : T extends readonly any[] 
        ? number extends T["length"] 
            ? false 
            : true 
        : false;

// อธิบาย:
// number extends T["length"] ตรวจสอบว่า length เป็น number (array) หรือ literal (tuple)
// tuple มี length เป็น literal เช่น 0, 1, 2, ...
// array มี length เป็น number

type D = IsTuple<[1, 2, 3]>;    // true  (length = 3 ซึ่งเป็น literal)
type E = IsTuple<string[]>;      // false (length = number)
```

---

## ระดับ Extreme

### Challenge 35: Type Arithmetic

**โจทย์**: สร้าง arithmetic operations ใน type level

```typescript
type Add<A extends number, B extends number> = /* ตอบตรงนี้ */
type Sub<A extends number, B extends number> = /* ตอบตรงนี้ */
type Mul<A extends number, B extends number> = /* ตอบตรงนี้ */

type Sum = Add<3, 7>;  // 10
type Diff = Sub<10, 4>; // 6
type Prod = Mul<3, 4>;  // 12
```

**Solution**:
```typescript
// Helper: สร้าง tuple ขนาด N
type BuildTuple<N extends number, T extends unknown[] = []> = 
    T["length"] extends N ? T : BuildTuple<N, [...T, unknown]>;

// Add
type Add<A extends number, B extends number> = 
    [...BuildTuple<A>, ...BuildTuple<B>]["length"];

// Sub
type Sub<A extends number, B extends number> = 
    BuildTuple<A> extends [...BuildTuple<B>, ...infer R] 
        ? R["length"] 
        : never;

// Mul (A * B = A + A + ... (B times))
type Mul<A extends number, B extends number, Acc extends unknown[] = []> = 
    B extends 0 
        ? Acc["length"] 
        : Mul<A, Sub<B, 1> & number, [...Acc, ...BuildTuple<A>]>;

// ทดสอบ
type Sum = Add<5, 7>;    // 12
type Diff = Sub<10, 3>;  // 7
type Prod = Mul<4, 5>;   // 20

// Division
type Div<A extends number, B extends number, Acc extends unknown[] = []> = 
    A extends 0 
        ? Acc["length"] 
        : Sub<A, B> extends number 
            ? Div<Sub<A, B>, B, [...Acc, unknown]> 
            : Acc["length"];

type Quotient = Div<12, 4>; // 3
```

---

### Challenge 36: Serialize

**โจทย์**: สร้าง `Serialize<T>` ที่แปลง object เป็น query string type

```typescript
type Serialize<T> = /* ตอบตรงนี้ */

type Config = {
    debug: true;
    env: "production";
    port: 3000;
};

type Serialized = Serialize<Config>;
// "debug=true&env=production&port=3000"
```

**Solution**:
```typescript
type Serialize<T extends object> = {
    [K in keyof T]: `${K & string}=${T[K] & (string | number | boolean)}`;
}[keyof T] extends infer U 
    ? U extends string 
        ? U 
        : never 
    : never;

// Complex version with joining
type JoinWith<T extends string, Sep extends string, Acc extends string = ""> = 
    T extends `${infer F}|${infer R}` 
        ? JoinWith<R, Sep, `${Acc}${F}${Sep}`> 
        : `${Acc}${T}`;
```

---

### Challenge 37: Deep Merge

**โจทย์**: สร้าง `DeepMerge<T, U>` ที่ merge objects แบบ deep

```typescript
type DeepMerge<T, U> = /* ตอบตรงนี้ */

type A = {
    a: { x: number; y: number };
    b: string;
};

type B = {
    a: { z: number };
    c: boolean;
};

type Merged = DeepMerge<A, B>;
// {
//   a: { x: number; y: number; z: number };
//   b: string;
//   c: boolean;
// }
```

**Solution**:
```typescript
type DeepMerge<T extends object, U extends object> = {
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

// ทดสอบ
type Config1 = {
    server: { host: string; port: number };
    debug: boolean;
};

type Config2 = {
    server: { ssl: boolean };
    db: { host: string };
};

type MergedConfig = DeepMerge<Config1, Config2>;
// {
//   server: { host: string; port: number; ssl: boolean };
//   debug: boolean;
//   db: { host: string };
// }
```

---

### Challenge 38: Pattern Matching

**โจทย์**: สร้าง exhaustive pattern matching สำหรับ discriminated unions

```typescript
type Match<T, Patterns> = /* ตอบตรงนี้ */

type Shape = 
    | { kind: "circle"; radius: number }
    | { kind: "rectangle"; width: number; height: number }
    | { kind: "triangle"; base: number; height: number };

// ต้องการ match ที่ exhaustive
const area = matchShape(shape, {
    circle: (s) => Math.PI * s.radius ** 2,
    rectangle: (s) => s.width * s.height,
    triangle: (s) => 0.5 * s.base * s.height,
});
```

**Solution**:
```typescript
type DiscriminatedUnionKey<T extends { kind: string }> = T["kind"];

type PatternHandlers<T extends { kind: string }> = {
    [K in T["kind"]]: (shape: Extract<T, { kind: K }>) => unknown;
};

function match<T extends { kind: string }>(
    value: T,
    handlers: PatternHandlers<T>
): ReturnType<PatternHandlers<T>[T["kind"]]> {
    const handler = handlers[value.kind as T["kind"]];
    return (handler as any)(value);
}

// ตัวอย่างที่สมบูรณ์
type Action = 
    | { type: "ADD"; payload: { item: string } }
    | { type: "REMOVE"; payload: { id: number } }
    | { type: "CLEAR" };

function handleAction(action: Action): string {
    return match(
        // ต้องการ discriminant key เป็น "kind" ดังนั้น adapt Action
        { ...action, kind: action.type } as any,
        {
            ADD: (a: any) => `Added ${a.payload.item}`,
            REMOVE: (a: any) => `Removed ${a.payload.id}`,
            CLEAR: () => "Cleared all",
        }
    );
}
```

---

### Challenge 39: Recursive Object Path Types

**โจทย์**: สร้าง `ObjectPaths<T>` ที่ดึง dot-notation paths ทั้งหมดจาก object

```typescript
type ObjectPaths<T, Prefix extends string = ""> = /* ตอบตรงนี้ */

interface Data {
    user: {
        name: string;
        address: {
            street: string;
            city: string;
        };
    };
    settings: {
        theme: string;
    };
}

type Paths = ObjectPaths<Data>;
// "user" | "user.name" | "user.address" | "user.address.street" | 
// "user.address.city" | "settings" | "settings.theme"
```

**Solution**:
```typescript
type ObjectPaths<T, Prefix extends string = ""> = {
    [K in keyof T & string]: T[K] extends object
        ? | `${Prefix extends "" ? K : `${Prefix}.${K}`}`
          | ObjectPaths<T[K], Prefix extends "" ? K : `${Prefix}.${K}`>
        : `${Prefix extends "" ? K : `${Prefix}.${K}`}`;
}[keyof T & string];

// ตัวอย่างการใช้งาน: type-safe path accessor
function getByPath<T, P extends ObjectPaths<T>>(
    obj: T,
    path: P
): any {
    return path.split(".").reduce((acc: any, key) => acc?.[key], obj);
}

const data: Data = {
    user: {
        name: "Alice",
        address: { street: "123 Main St", city: "Bangkok" }
    },
    settings: { theme: "dark" }
};

const city = getByPath(data, "user.address.city"); // "Bangkok"
```

---

### Challenge 40: Type-Level JSON Parser (Simplified)

**โจทย์**: สร้าง type ที่ parse JSON string ใน type level (เวอร์ชันง่าย)

```typescript
type ParseBoolean<S extends string> = 
    S extends "true" ? true : S extends "false" ? false : never;

type ParseNumber<S extends string> = 
    S extends `${infer N extends number}` ? N : never;

type ParseNull<S extends string> = 
    S extends "null" ? null : never;

type ParsePrimitive<S extends string> = 
    ParseBoolean<S> extends never 
        ? ParseNull<S> extends never 
            ? ParseNumber<S> extends never 
                ? S  // treat as string
                : ParseNumber<S>
            : ParseNull<S>
        : ParseBoolean<S>;

type A = ParsePrimitive<"true">;   // true
type B = ParsePrimitive<"42">;     // 42
type C = ParsePrimitive<"null">;   // null
type D = ParsePrimitive<"hello">;  // "hello"
```

---

### Challenge 41: Strict Omit

**โจทย์**: สร้าง `StrictOmit<T, K>` ที่ error ถ้า K ไม่ใช่ key ของ T

```typescript
type StrictOmit<T, K extends keyof T> = Omit<T, K>;

// ต่างจาก Omit ตรงที่ K ต้อง extends keyof T
// Omit<{ a: string }, "b"> = { a: string } (ไม่ error!)
// StrictOmit<{ a: string }, "b"> // Error: "b" is not a key

interface User {
    id: string;
    name: string;
    password: string;
}

type SafeUser = StrictOmit<User, "password">; // OK
// type BadUser = StrictOmit<User, "nonExistent">; // Error!
```

---

### Challenge 42: Mutable

**โจทย์**: สร้าง `Mutable<T>` ที่ลบ readonly ออกจากทุก property

```typescript
type Mutable<T> = /* ตอบตรงนี้ */

type ReadonlyUser = {
    readonly id: string;
    readonly name: string;
    readonly age: number;
};

type MutableUser = Mutable<ReadonlyUser>;
// { id: string; name: string; age: number; }
```

**Solution**:
```typescript
type Mutable<T> = {
    -readonly [K in keyof T]: T[K];
};

// DeepMutable
type DeepMutable<T> = {
    -readonly [K in keyof T]: T[K] extends object ? DeepMutable<T[K]> : T[K];
};

type ReadonlyNested = {
    readonly name: string;
    readonly address: {
        readonly street: string;
        readonly city: string;
    };
};

type MutableNested = DeepMutable<ReadonlyNested>;
// { name: string; address: { street: string; city: string; } }
```

---

## Bonus Challenges

### Challenge 43: Type-Safe CSS

```typescript
// Type-safe CSS properties
type CSSUnit = "px" | "em" | "rem" | "%";
type CSSValue = `${number}${CSSUnit}`;
type CSSColor = `#${string}` | `rgb(${number}, ${number}, ${number})`;

interface CSSProperties {
    width?: CSSValue;
    height?: CSSValue;
    margin?: CSSValue;
    padding?: CSSValue;
    color?: CSSColor;
    backgroundColor?: CSSColor;
    fontSize?: CSSValue;
}

const styles: CSSProperties = {
    width: "100px",
    height: "50%",
    color: "#ff0000",
    backgroundColor: "rgb(0, 0, 255)",
};

// สร้าง function ที่รับ CSS properties และสร้าง inline style string
function cssToString(styles: CSSProperties): string {
    return Object.entries(styles)
        .filter(([, v]) => v !== undefined)
        .map(([k, v]) => `${k.replace(/([A-Z])/g, "-$1").toLowerCase()}: ${v}`)
        .join("; ");
}

const styleStr = cssToString(styles);
console.log(styleStr);
// "width: 100px; height: 50%; color: #ff0000; background-color: rgb(0, 0, 255)"
```

---

### Challenge 44: Recursive String Split

```typescript
// Split string ด้วย delimiter
type Split<S extends string, D extends string> = 
    S extends `${infer F}${D}${infer R}` 
        ? [F, ...Split<R, D>] 
        : [S];

type A = Split<"hello,world,foo", ",">;
// ["hello", "world", "foo"]

type B = Split<"a.b.c.d", ".">;
// ["a", "b", "c", "d"]

// Join array of strings
type Join<T extends string[], D extends string> = 
    T extends [] 
        ? "" 
        : T extends [infer F extends string] 
            ? F 
            : T extends [infer F extends string, ...infer R extends string[]] 
                ? `${F}${D}${Join<R, D>}` 
                : never;

type C = Join<["hello", "world", "foo"], " ">;
// "hello world foo"
```

---

### Challenge 45: Type-Safe SQL Query Builder

```typescript
// Table schema definition
interface Tables {
    users: {
        id: number;
        name: string;
        email: string;
        age: number;
    };
    posts: {
        id: number;
        title: string;
        authorId: number;
        published: boolean;
    };
}

type TableName = keyof Tables;
type ColumnOf<T extends TableName> = keyof Tables[T] & string;

// Type-safe WHERE condition
type WhereCondition<T extends TableName> = {
    [K in ColumnOf<T>]?: Tables[T][K];
};

// Type-safe SELECT result
type SelectResult<T extends TableName, Cols extends ColumnOf<T>> = {
    [K in Cols]: Tables[T][K];
};

// Query builder
class TypedQuery<T extends TableName, Selected extends ColumnOf<T> = ColumnOf<T>> {
    private tableName: T;
    private selectedCols: string[] = [];
    private conditions: string[] = [];

    constructor(table: T) {
        this.tableName = table;
    }

    select<C extends ColumnOf<T>>(...columns: C[]): TypedQuery<T, C> {
        this.selectedCols = columns;
        return this as any;
    }

    where(condition: WhereCondition<T>): this {
        Object.entries(condition).forEach(([k, v]) => {
            this.conditions.push(`${k} = ${JSON.stringify(v)}`);
        });
        return this;
    }

    toSQL(): string {
        const select = this.selectedCols.length > 0 
            ? this.selectedCols.join(", ") 
            : "*";
        let sql = `SELECT ${select} FROM ${this.tableName}`;
        if (this.conditions.length > 0) {
            sql += ` WHERE ${this.conditions.join(" AND ")}`;
        }
        return sql;
    }
}

function from<T extends TableName>(table: T): TypedQuery<T> {
    return new TypedQuery<T>(table);
}

// การใช้งาน
const q1 = from("users")
    .select("name", "email")
    .where({ age: 25 })
    .toSQL();

console.log(q1);
// SELECT name, email FROM users WHERE age = 25

const q2 = from("posts")
    .select("title", "published")
    .where({ published: true })
    .toSQL();

console.log(q2);
// SELECT title, published FROM posts WHERE published = true
```

---

## สรุป

ในบทนี้เราได้แก้ Type Challenges ที่ครอบคลุม:

**Easy Level**:
- Partial, Required, Readonly (3 challenges)
- TupleToObject, First, Length (3 challenges)
- Exclude, Awaited, If (3 challenges)

**Medium Level**:
- Pick, Omit, DeepReadonly (3 challenges)
- TupleToUnion, Chainable, Last (3 challenges)
- Pop, Push, Parameters, ReturnType (4 challenges)
- Trim, Capitalize, Replace (3 challenges)
- Permutation, Length of String (2 challenges)

**Hard Level**:
- Vue SimpleVue, Currying (2 challenges)
- UnionToIntersection, RequiredKeys (2 challenges)
- Zip, IsTuple (2 challenges)
- Type Arithmetic (1 challenge)
- DeepMerge (1 challenge)

**Extreme Level**:
- Pattern Matching, Object Paths (2 challenges)
- Recursive Split/Join (1 challenge)
- Type-Safe SQL Builder (1 challenge)

ทักษะเหล่านี้จะช่วยให้คุณเขียน TypeScript ที่ type-safe และ expressive ได้อย่างมืออาชีพ
