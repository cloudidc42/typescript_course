# Part 81: Advanced Generic Patterns ใน TypeScript

## บทนำ

Generic types เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ TypeScript ในบทนี้เราจะสำรวจ patterns ขั้นสูงที่นักพัฒนา TypeScript ระดับมืออาชีพใช้ในการสร้างระบบ type-safe ที่ซับซ้อน

---

## 1. Conditional Generic Types

Conditional types ช่วยให้เราสร้าง types ที่ตัดสินใจตาม condition ได้

```typescript
// รูปแบบพื้นฐาน
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false
type C = IsString<"hello">; // true

// Conditional type ที่ซับซ้อนขึ้น
type NonNullable<T> = T extends null | undefined ? never : T;

type D = NonNullable<string | null | undefined>; // string
type E = NonNullable<number | null>;              // number

// Extract และ Exclude patterns
type Extract<T, U> = T extends U ? T : never;
type Exclude<T, U> = T extends U ? never : T;

type F = Extract<string | number | boolean, string | number>; // string | number
type G = Exclude<string | number | boolean, string | number>; // boolean
```

### Conditional Types กับ Union Types

```typescript
// TypeScript กระจาย conditional types ผ่าน union types โดยอัตโนมัติ
type ToArray<T> = T extends any ? T[] : never;

type H = ToArray<string | number>; // string[] | number[]

// ถ้าไม่ต้องการให้กระจาย ให้ใช้ tuple
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type I = ToArrayNonDist<string | number>; // (string | number)[]

// ตัวอย่างการใช้งานจริง
type Flatten<T> = T extends Array<infer Item> ? Item : T;

type J = Flatten<number[]>;        // number
type K = Flatten<string[][]>;      // string[]
type L = Flatten<string>;          // string (ไม่ใช่ array จึงคืน T เอง)
```

---

## 2. Type Inference ใน Conditional Types

`infer` keyword ช่วยให้เราดึง type จาก conditional type ได้

```typescript
// infer พื้นฐาน
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function greet(name: string): string {
    return `Hello, ${name}!`;
}

type GreetReturn = ReturnType<typeof greet>; // string

// infer กับ Parameters
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

function add(a: number, b: number): number {
    return a + b;
}

type AddParams = Parameters<typeof add>; // [a: number, b: number]

// infer กับ Promise
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type M = Awaited<Promise<string>>;              // string
type N = Awaited<Promise<Promise<number>>>;     // number
type O = Awaited<string>;                       // string

// infer กับ Array
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Tail<T extends any[]> = T extends [any, ...infer T] ? T : never;

type HeadType = Head<[1, 2, 3]>; // 1
type TailType = Tail<[1, 2, 3]>; // [2, 3]

// infer กับ Constructor
type InstanceType<T> = T extends new (...args: any[]) => infer R ? R : never;

class Dog {
    bark() { return "Woof!"; }
}

type DogInstance = InstanceType<typeof Dog>; // Dog
```

### Advanced Inference Patterns

```typescript
// ดึง key-value pairs จาก type
type ValueOf<T> = T[keyof T];

interface Config {
    host: string;
    port: number;
    debug: boolean;
}

type ConfigValues = ValueOf<Config>; // string | number | boolean

// ดึง function properties
type FunctionProperties<T> = {
    [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];

interface Person {
    name: string;
    age: number;
    greet(): void;
    sayHello(): string;
}

type PersonMethods = FunctionProperties<Person>; // "greet" | "sayHello"

// Pattern matching กับ string literal types
type ExtractRouteParams<T extends string> = 
    T extends `${infer _Start}:${infer Param}/${infer Rest}`
        ? Param | ExtractRouteParams<Rest>
        : T extends `${infer _Start}:${infer Param}`
            ? Param
            : never;

type RouteParams = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

---

## 3. Higher-Kinded Types (HKT) Simulation

TypeScript ไม่รองรับ HKT โดยตรง แต่เราสามารถจำลองได้ด้วย type-level encoding

```typescript
// วิธีจำลอง HKT ใน TypeScript
// ขั้นตอนที่ 1: สร้าง URI registry
interface URItoKind<A> {
    // จะเพิ่ม entries ตรงนี้ภายหลัง
}

// ขั้นตอนที่ 2: สร้าง type alias สำหรับ HKT
type URIS = keyof URItoKind<any>;
type Kind<F extends URIS, A> = URItoKind<A>[F];

// ขั้นตอนที่ 3: สร้าง Functor interface
interface Functor<F extends URIS> {
    map<A, B>(fa: Kind<F, A>, f: (a: A) => B): Kind<F, B>;
}

// ขั้นตอนที่ 4: Implement สำหรับ Array
declare module "./hkt" {
    interface URItoKind<A> {
        "Array": A[];
    }
}

// ตัวอย่างที่สมบูรณ์กว่า
interface HKTUri {
    "Option": unknown;
    "Either": unknown;
    "List": unknown;
}

type HKT<F extends keyof HKTUri, A> = {
    "Option": Option<A>;
    "Either": Either<unknown, A>;
    "List": List<A>;
}[F];

// Simple Option type
type Option<A> = { tag: "None" } | { tag: "Some"; value: A };

function none<A>(): Option<A> {
    return { tag: "None" };
}

function some<A>(value: A): Option<A> {
    return { tag: "Some", value };
}

// Simple Either type
type Either<E, A> = { tag: "Left"; left: E } | { tag: "Right"; right: A };

// Simple List type
type List<A> = A[];

// Functor interface using HKT
interface FunctorHKT<F extends keyof HKTUri> {
    map<A, B>(fa: HKT<F, A>, f: (a: A) => B): HKT<F, B>;
}

// Implement Functor สำหรับ Option
const OptionFunctor: FunctorHKT<"Option"> = {
    map<A, B>(fa: Option<A>, f: (a: A) => B): Option<B> {
        if (fa.tag === "None") return none<B>();
        return some(f(fa.value));
    }
};

// ทดสอบ
const opt = some(42);
const result = OptionFunctor.map(opt, x => x * 2);
console.log(result); // { tag: "Some", value: 84 }
```

---

## 4. Type-Safe Event Emitter

```typescript
// Event map definition
type EventMap = Record<string, any>;

// Type-safe event emitter
class TypedEventEmitter<Events extends EventMap> {
    private listeners: Partial<{
        [K in keyof Events]: Array<(payload: Events[K]) => void>;
    }> = {};

    on<K extends keyof Events>(
        event: K,
        listener: (payload: Events[K]) => void
    ): this {
        if (!this.listeners[event]) {
            this.listeners[event] = [];
        }
        this.listeners[event]!.push(listener);
        return this;
    }

    off<K extends keyof Events>(
        event: K,
        listener: (payload: Events[K]) => void
    ): this {
        const eventListeners = this.listeners[event];
        if (eventListeners) {
            this.listeners[event] = eventListeners.filter(
                l => l !== listener
            ) as any;
        }
        return this;
    }

    emit<K extends keyof Events>(event: K, payload: Events[K]): void {
        const eventListeners = this.listeners[event];
        if (eventListeners) {
            eventListeners.forEach(listener => listener(payload));
        }
    }

    once<K extends keyof Events>(
        event: K,
        listener: (payload: Events[K]) => void
    ): this {
        const onceWrapper = (payload: Events[K]) => {
            listener(payload);
            this.off(event, onceWrapper);
        };
        return this.on(event, onceWrapper);
    }
}

// การใช้งาน
interface AppEvents {
    userLogin: { userId: string; timestamp: Date };
    userLogout: { userId: string };
    messageReceived: { from: string; content: string; timestamp: Date };
    error: { code: number; message: string };
}

const emitter = new TypedEventEmitter<AppEvents>();

// TypeScript จะตรวจสอบ type ของ payload อัตโนมัติ
emitter.on("userLogin", ({ userId, timestamp }) => {
    console.log(`User ${userId} logged in at ${timestamp}`);
});

emitter.on("messageReceived", ({ from, content }) => {
    console.log(`Message from ${from}: ${content}`);
});

emitter.emit("userLogin", {
    userId: "user123",
    timestamp: new Date()
});

// TypeScript Error: ใส่ payload ผิด type
// emitter.emit("userLogin", { userId: 123 }); // Error!
```

### Advanced Event Emitter กับ Middleware

```typescript
type Middleware<Events extends EventMap, K extends keyof Events> = (
    event: K,
    payload: Events[K],
    next: () => void
) => void;

class MiddlewareEventEmitter<Events extends EventMap> extends TypedEventEmitter<Events> {
    private middlewares: Array<Middleware<Events, keyof Events>> = [];

    use(middleware: Middleware<Events, keyof Events>): this {
        this.middlewares.push(middleware);
        return this;
    }

    emit<K extends keyof Events>(event: K, payload: Events[K]): void {
        const runMiddleware = (index: number) => {
            if (index >= this.middlewares.length) {
                super.emit(event, payload);
                return;
            }
            this.middlewares[index](event, payload, () => runMiddleware(index + 1));
        };
        runMiddleware(0);
    }
}

// ตัวอย่างการใช้งาน
const app = new MiddlewareEventEmitter<AppEvents>();

// เพิ่ม logging middleware
app.use((event, payload, next) => {
    console.log(`[LOG] Event: ${String(event)}`, payload);
    next();
});

// เพิ่ม authentication middleware
app.use((event, payload, next) => {
    if (event === "userLogin") {
        console.log("[AUTH] Checking authentication...");
    }
    next();
});

app.on("userLogin", ({ userId }) => {
    console.log(`Handling login for ${userId}`);
});

app.emit("userLogin", { userId: "abc", timestamp: new Date() });
```

---

## 5. Type-Safe Builder Pattern

```typescript
// Type-safe builder ด้วย generic types
type Prettify<T> = {
    [K in keyof T]: T[K];
} & {};

class Builder<T extends object> {
    private data: Partial<T> = {};

    set<K extends keyof T>(key: K, value: T[K]): Builder<T> {
        this.data[key] = value;
        return this;
    }

    build(): T {
        return this.data as T;
    }
}

// Advanced Builder ที่ track ว่า fields ใดถูก set แล้ว
type RequiredFields<T, R extends keyof T> = {
    [K in keyof T]: K extends R ? T[K] : T[K] | undefined;
};

class TypeSafeBuilder<
    T extends object,
    Set extends keyof T = never
> {
    private data: Partial<T> = {};

    set<K extends keyof T>(
        key: K,
        value: T[K]
    ): TypeSafeBuilder<T, Set | K> {
        this.data[key] = value;
        return this as any;
    }

    build<Required extends keyof T>(
        this: TypeSafeBuilder<T, Required>
    ): Required extends Set ? T : never {
        return this.data as any;
    }
}

// ตัวอย่างการใช้งาน
interface UserConfig {
    name: string;
    age: number;
    email: string;
    role: "admin" | "user";
}

const userBuilder = new TypeSafeBuilder<UserConfig>();

const user = userBuilder
    .set("name", "John")
    .set("age", 30)
    .set("email", "john@example.com")
    .set("role", "admin")
    .build();

console.log(user); // { name: "John", age: 30, email: "john@example.com", role: "admin" }

// Fluent Builder Pattern
class QueryBuilder<
    Table extends string,
    Selected extends keyof any = never,
    HasWhere extends boolean = false
> {
    private query: {
        table: string;
        select: string[];
        where: string[];
        limit?: number;
        offset?: number;
    } = { table: "", select: [], where: [] };

    constructor(table: Table) {
        this.query.table = table;
    }

    select<Fields extends string>(
        ...fields: Fields[]
    ): QueryBuilder<Table, Selected | Fields, HasWhere> {
        this.query.select.push(...fields);
        return this as any;
    }

    where(condition: string): QueryBuilder<Table, Selected, true> {
        this.query.where.push(condition);
        return this as any;
    }

    limit(n: number): this {
        this.query.limit = n;
        return this;
    }

    offset(n: number): this {
        this.query.offset = n;
        return this;
    }

    toSQL(): string {
        const selectClause = this.query.select.length > 0
            ? this.query.select.join(", ")
            : "*";
        let sql = `SELECT ${selectClause} FROM ${this.query.table}`;
        if (this.query.where.length > 0) {
            sql += ` WHERE ${this.query.where.join(" AND ")}`;
        }
        if (this.query.limit !== undefined) {
            sql += ` LIMIT ${this.query.limit}`;
        }
        if (this.query.offset !== undefined) {
            sql += ` OFFSET ${this.query.offset}`;
        }
        return sql;
    }
}

// การใช้งาน
const query = new QueryBuilder("users")
    .select("name", "email")
    .where("age > 18")
    .where("active = true")
    .limit(10)
    .offset(20)
    .toSQL();

console.log(query);
// SELECT name, email FROM users WHERE age > 18 AND active = true LIMIT 10 OFFSET 20
```

---

## 6. Type-Safe Command Pattern

```typescript
// Command interface
interface Command<Input, Output> {
    execute(input: Input): Output;
    undo?(output: Output): void;
}

// Command registry
type CommandRegistry = Record<string, Command<any, any>>;

// Type-safe command bus
class CommandBus<Registry extends CommandRegistry> {
    private commands: Partial<Registry> = {};

    register<K extends keyof Registry>(
        name: K,
        command: Registry[K]
    ): void {
        this.commands[name] = command;
    }

    execute<K extends keyof Registry>(
        name: K,
        input: Registry[K] extends Command<infer I, any> ? I : never
    ): Registry[K] extends Command<any, infer O> ? O : never {
        const command = this.commands[name];
        if (!command) {
            throw new Error(`Command ${String(name)} not registered`);
        }
        return command.execute(input);
    }
}

// ตัวอย่างการใช้งาน
interface CreateUserInput {
    name: string;
    email: string;
}

interface CreateUserOutput {
    id: string;
    name: string;
    email: string;
    createdAt: Date;
}

interface AppCommands {
    createUser: Command<CreateUserInput, CreateUserOutput>;
    deleteUser: Command<{ id: string }, void>;
    updateUser: Command<{ id: string; data: Partial<CreateUserInput> }, CreateUserOutput>;
}

const bus = new CommandBus<AppCommands>();

bus.register("createUser", {
    execute(input): CreateUserOutput {
        return {
            id: Math.random().toString(36),
            name: input.name,
            email: input.email,
            createdAt: new Date()
        };
    }
});

bus.register("deleteUser", {
    execute({ id }) {
        console.log(`Deleting user ${id}`);
    }
});

// Type-safe execution
const newUser = bus.execute("createUser", {
    name: "Alice",
    email: "alice@example.com"
});

console.log(newUser.id); // TypeScript รู้ว่า result เป็น CreateUserOutput
```

---

## 7. Builder ด้วย Method Chaining

```typescript
// Immutable Builder Pattern
type DeepReadonly<T> = {
    readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

class ImmutableBuilder<T extends object> {
    private constructor(private readonly value: Partial<T>) {}

    static create<T extends object>(): ImmutableBuilder<T> {
        return new ImmutableBuilder<T>({});
    }

    with<K extends keyof T>(key: K, value: T[K]): ImmutableBuilder<T> {
        return new ImmutableBuilder<T>({ ...this.value, [key]: value });
    }

    without<K extends keyof T>(key: K): ImmutableBuilder<T> {
        const { [key]: _, ...rest } = this.value;
        return new ImmutableBuilder<T>(rest as Partial<T>);
    }

    build(): T {
        return this.value as T;
    }

    toReadonly(): DeepReadonly<T> {
        return Object.freeze({ ...this.value }) as DeepReadonly<T>;
    }
}

// HTML Element Builder
interface HtmlAttributes {
    id?: string;
    className?: string;
    style?: string;
    onClick?: string;
    "data-testid"?: string;
}

class HtmlElementBuilder {
    private tag: string;
    private attrs: HtmlAttributes = {};
    private children: string[] = [];
    private text: string = "";

    constructor(tag: string) {
        this.tag = tag;
    }

    id(value: string): this {
        this.attrs.id = value;
        return this;
    }

    class(value: string): this {
        this.attrs.className = value;
        return this;
    }

    style(value: string): this {
        this.attrs.style = value;
        return this;
    }

    content(text: string): this {
        this.text = text;
        return this;
    }

    child(builder: HtmlElementBuilder): this {
        this.children.push(builder.build());
        return this;
    }

    testId(value: string): this {
        this.attrs["data-testid"] = value;
        return this;
    }

    build(): string {
        const attrStr = Object.entries(this.attrs)
            .filter(([, v]) => v !== undefined)
            .map(([k, v]) => `${k === "className" ? "class" : k}="${v}"`)
            .join(" ");

        const openTag = attrStr ? `<${this.tag} ${attrStr}>` : `<${this.tag}>`;
        const content = this.text || this.children.join("\n");
        return `${openTag}${content}</${this.tag}>`;
    }
}

// Factory functions
const div = () => new HtmlElementBuilder("div");
const span = () => new HtmlElementBuilder("span");
const p = () => new HtmlElementBuilder("p");
const button = () => new HtmlElementBuilder("button");

// การใช้งาน
const html = div()
    .id("container")
    .class("flex flex-col gap-4")
    .child(
        div()
            .class("header")
            .child(
                span().class("title").content("Hello World")
            )
    )
    .child(
        button()
            .class("btn btn-primary")
            .testId("submit-btn")
            .content("Click Me")
    )
    .build();

console.log(html);
```

---

## 8. Generic Middleware Pipeline

```typescript
// Middleware type definition
type Next = () => Promise<void>;
type MiddlewareFn<Context> = (ctx: Context, next: Next) => Promise<void>;

// Pipeline builder
class Pipeline<Context extends object> {
    private middlewares: MiddlewareFn<Context>[] = [];

    use(middleware: MiddlewareFn<Context>): this {
        this.middlewares.push(middleware);
        return this;
    }

    async execute(ctx: Context): Promise<void> {
        const dispatch = async (index: number): Promise<void> => {
            if (index >= this.middlewares.length) return;
            await this.middlewares[index](ctx, () => dispatch(index + 1));
        };
        await dispatch(0);
    }
}

// HTTP-like context
interface HttpContext {
    method: string;
    path: string;
    headers: Record<string, string>;
    body: unknown;
    response: {
        status: number;
        body: unknown;
        headers: Record<string, string>;
    };
    user?: {
        id: string;
        role: string;
    };
}

// Typed middleware helpers
const createMiddleware = <Context extends object>(
    fn: MiddlewareFn<Context>
): MiddlewareFn<Context> => fn;

// Middleware implementations
const loggingMiddleware = createMiddleware<HttpContext>(async (ctx, next) => {
    const start = Date.now();
    console.log(`→ ${ctx.method} ${ctx.path}`);
    await next();
    console.log(`← ${ctx.method} ${ctx.path} [${Date.now() - start}ms] ${ctx.response.status}`);
});

const authMiddleware = createMiddleware<HttpContext>(async (ctx, next) => {
    const token = ctx.headers["authorization"];
    if (token) {
        // จำลองการตรวจสอบ token
        ctx.user = { id: "user123", role: "user" };
    }
    await next();
});

const corsMiddleware = createMiddleware<HttpContext>(async (ctx, next) => {
    ctx.response.headers["Access-Control-Allow-Origin"] = "*";
    ctx.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE";
    await next();
});

const errorHandlerMiddleware = createMiddleware<HttpContext>(async (ctx, next) => {
    try {
        await next();
    } catch (error) {
        ctx.response.status = 500;
        ctx.response.body = { error: "Internal Server Error" };
    }
});

// สร้าง pipeline
const app = new Pipeline<HttpContext>()
    .use(errorHandlerMiddleware)
    .use(loggingMiddleware)
    .use(corsMiddleware)
    .use(authMiddleware);

// Typed Reducer Pattern
type Action<Type extends string, Payload = void> = Payload extends void
    ? { type: Type }
    : { type: Type; payload: Payload };

type ReducerMap<State, Actions extends Action<string, any>> = {
    [K in Actions["type"]]: (
        state: State,
        action: Extract<Actions, { type: K }>
    ) => State;
};

function createReducer<State, Actions extends Action<string, any>>(
    initialState: State,
    handlers: ReducerMap<State, Actions>
): (state: State | undefined, action: Actions) => State {
    return (state = initialState, action) => {
        const handler = handlers[action.type as Actions["type"]];
        if (handler) {
            return handler(state, action as any);
        }
        return state;
    };
}

// ตัวอย่าง usage
interface CounterState {
    count: number;
}

type CounterActions =
    | Action<"increment">
    | Action<"decrement">
    | Action<"reset">
    | Action<"set", number>;

const counterReducer = createReducer<CounterState, CounterActions>(
    { count: 0 },
    {
        increment: (state) => ({ count: state.count + 1 }),
        decrement: (state) => ({ count: state.count - 1 }),
        reset: () => ({ count: 0 }),
        set: (state, action) => ({ count: action.payload }),
    }
);

let state = counterReducer(undefined, { type: "increment" });
state = counterReducer(state, { type: "increment" });
state = counterReducer(state, { type: "set", payload: 10 });
console.log(state); // { count: 10 }
```

---

## 9. Type-Level Programming Tricks

```typescript
// Arithmetic ใน type system
type Length<T extends any[]> = T["length"];

type Tuple1 = [1];
type Tuple3 = [1, 2, 3];

type L1 = Length<Tuple1>; // 1
type L3 = Length<Tuple3>; // 3

// Build array ด้วย recursive types
type BuildTuple<L extends number, T extends any[] = []> =
    T["length"] extends L ? T : BuildTuple<L, [...T, unknown]>;

type Tup5 = BuildTuple<5>; // [unknown, unknown, unknown, unknown, unknown]

// Addition ใน type system
type Add<A extends number, B extends number> = 
    Length<[...BuildTuple<A>, ...BuildTuple<B>]>;

type Sum = Add<3, 4>; // 7

// Comparison
type GT<A extends number, B extends number> = 
    BuildTuple<A> extends [...BuildTuple<B>, ...infer _] ? true : false;

type GT_5_3 = GT<5, 3>; // true
type GT_3_5 = GT<3, 5>; // false

// Type-safe path accessor
type Path<T, P extends string> = 
    P extends keyof T 
        ? T[P]
        : P extends `${infer K}.${infer Rest}`
            ? K extends keyof T
                ? Path<T[K], Rest>
                : never
            : never;

interface DeepObject {
    a: {
        b: {
            c: string;
        };
        d: number;
    };
    e: boolean;
}

type ValueAtPath = Path<DeepObject, "a.b.c">; // string
type ValueAtD = Path<DeepObject, "a.d">;      // number
type ValueAtE = Path<DeepObject, "e">;         // boolean

// Type-safe getter function
function get<T, P extends string>(obj: T, path: P): Path<T, P> {
    const keys = path.split(".");
    let current: any = obj;
    for (const key of keys) {
        current = current?.[key];
    }
    return current;
}

const obj: DeepObject = {
    a: { b: { c: "hello" }, d: 42 },
    e: true
};

const val = get(obj, "a.b.c"); // TypeScript รู้ว่าเป็น string
console.log(val); // "hello"
```

### Template Literal Types

```typescript
// สร้าง CSS class names
type Breakpoint = "sm" | "md" | "lg" | "xl";
type CssProperty = "flex" | "grid" | "hidden" | "block";

type ResponsiveClass = `${Breakpoint}:${CssProperty}`;

const classes: ResponsiveClass[] = ["sm:flex", "md:grid", "lg:hidden"];

// Event name patterns
type EventNames<T extends string> = 
    | T 
    | `on${Capitalize<T>}`
    | `before${Capitalize<T>}`
    | `after${Capitalize<T>}`;

type ClickEvents = EventNames<"click">;
// "click" | "onClick" | "beforeClick" | "afterClick"

// API endpoint types
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
type ApiVersion = "v1" | "v2";
type Resource = "users" | "posts" | "comments";

type ApiEndpoint = `/${ApiVersion}/${Resource}`;
// "/v1/users" | "/v1/posts" | ... | "/v2/comments"

// สร้าง getter/setter types
type Getters<T> = {
    [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
    [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface User {
    name: string;
    age: number;
    email: string;
}

type UserGetters = Getters<User>;
// {
//     getName: () => string;
//     getAge: () => number;
//     getEmail: () => string;
// }

type UserSetters = Setters<User>;
// {
//     setName: (value: string) => void;
//     setAge: (value: number) => void;
//     setEmail: (value: string) => void;
// }
```

---

## 10. Performance Considerations สำหรับ Complex Types

```typescript
// ปัญหา: recursive types ที่ลึกเกินไปจะทำให้ TypeScript ช้า

// ❌ ไม่ดี: recursive type ที่ไม่มี depth limit
type DeepPartialBad<T> = {
    [K in keyof T]?: T[K] extends object ? DeepPartialBad<T[K]> : T[K];
};

// ✅ ดีกว่า: มี depth limit
type DeepPartial<T, Depth extends number = 5> = Depth extends 0
    ? T
    : {
        [K in keyof T]?: T[K] extends object
            ? DeepPartial<T[K], [-1, 0, 1, 2, 3, 4][Depth]>
            : T[K];
    };

// ✅ ใช้ Lazy evaluation เพื่อลดการคำนวณ
type LazyType<T> = () => T;

// ❌ ไม่ดี: คำนวณทุกอย่างทันที
type EagerUnion = string | number | boolean | null | undefined | symbol | bigint;

// ✅ ดีกว่า: แยก types ออกเป็นชิ้นๆ
type Primitive = string | number | boolean;
type Nullish = null | undefined;
type OtherPrimitives = symbol | bigint;

type AllPrimitives = Primitive | Nullish | OtherPrimitives;

// ✅ ใช้ interface แทน type aliases เมื่อเป็นไปได้ (interface merge ได้ดีกว่า)
interface Cache {
    [key: string]: unknown;
}

// ✅ Cached types ป้องกันการคำนวณซ้ำ
type ComputeOnce<T> = T extends infer U ? U : never;

// Simplify complex union types
type Simplify<T> = T extends infer U ? { [K in keyof U]: U[K] } : never;

interface A {
    x: number;
}
interface B {
    y: string;
}

type AB = Simplify<A & B>; // { x: number; y: string; }

// ✅ ใช้ mapped types แทน conditional types เมื่อทำได้
// ❌ ช้ากว่า
type SlowRequired<T> = {
    [K in keyof T]: NonNullable<T[K]>;
};

// ✅ เร็วกว่า (ใช้ built-in)
type FastRequired<T> = Required<T>;

// Best practices สำหรับ generic constraints
// ❌ ไม่ดี: constraint ที่กว้างเกินไป
function processAny<T>(value: T): T {
    return value;
}

// ✅ ดีกว่า: constraint ที่ชัดเจน
function processSerializable<T extends Record<string, unknown>>(value: T): T {
    return JSON.parse(JSON.stringify(value));
}
```

---

## 11. Practical Examples: Type-Safe Form Builder

```typescript
// Type definitions
type FieldType = "text" | "number" | "email" | "password" | "checkbox" | "select";

interface BaseField<T extends FieldType, V> {
    type: T;
    name: string;
    label: string;
    required?: boolean;
    defaultValue?: V;
    validate?: (value: V) => string | undefined;
}

interface TextField extends BaseField<"text" | "email" | "password", string> {
    minLength?: number;
    maxLength?: number;
    pattern?: RegExp;
}

interface NumberField extends BaseField<"number", number> {
    min?: number;
    max?: number;
    step?: number;
}

interface CheckboxField extends BaseField<"checkbox", boolean> {
    checkedValue?: string;
    uncheckedValue?: string;
}

interface SelectField<T extends string> extends BaseField<"select", T> {
    options: Array<{ value: T; label: string }>;
    multiple?: boolean;
}

type Field<T extends FieldType = FieldType> = 
    T extends "text" | "email" | "password" ? TextField :
    T extends "number" ? NumberField :
    T extends "checkbox" ? CheckboxField :
    T extends "select" ? SelectField<string> :
    never;

type FieldValue<F extends Field> = 
    F extends TextField ? string :
    F extends NumberField ? number :
    F extends CheckboxField ? boolean :
    F extends SelectField<infer T> ? T :
    never;

// Form schema ที่ type-safe
type FormSchema = Record<string, Field>;

type FormValues<Schema extends FormSchema> = {
    [K in keyof Schema]: FieldValue<Schema[K]>;
};

// Form builder
class FormBuilder<Schema extends FormSchema = {}> {
    private schema: Schema;

    constructor(schema: Schema = {} as Schema) {
        this.schema = schema;
    }

    addTextField<Name extends string>(
        name: Name,
        config: Omit<TextField, "type" | "name">
    ): FormBuilder<Schema & Record<Name, TextField>> {
        return new FormBuilder({
            ...this.schema,
            [name]: { type: "text", name, ...config }
        } as any);
    }

    addNumberField<Name extends string>(
        name: Name,
        config: Omit<NumberField, "type" | "name">
    ): FormBuilder<Schema & Record<Name, NumberField>> {
        return new FormBuilder({
            ...this.schema,
            [name]: { type: "number", name, ...config }
        } as any);
    }

    addSelectField<Name extends string, T extends string>(
        name: Name,
        config: Omit<SelectField<T>, "type" | "name">
    ): FormBuilder<Schema & Record<Name, SelectField<T>>> {
        return new FormBuilder({
            ...this.schema,
            [name]: { type: "select", name, ...config }
        } as any);
    }

    getSchema(): Schema {
        return this.schema;
    }

    createValidator(): (values: FormValues<Schema>) => Record<string, string> {
        return (values) => {
            const errors: Record<string, string> = {};
            for (const [key, field] of Object.entries(this.schema)) {
                const value = (values as any)[key];
                if (field.required && (value === undefined || value === null || value === "")) {
                    errors[key] = `${field.label} is required`;
                }
                if (field.validate) {
                    const error = field.validate(value);
                    if (error) errors[key] = error;
                }
            }
            return errors;
        };
    }
}

// การใช้งาน
const loginForm = new FormBuilder()
    .addTextField("email", {
        label: "Email",
        required: true,
        validate: (value) => {
            if (!value.includes("@")) return "Invalid email format";
        }
    })
    .addTextField("password", {
        label: "Password",
        required: true,
        validate: (value) => {
            if (value.length < 8) return "Password must be at least 8 characters";
        }
    });

type LoginFormValues = FormValues<ReturnType<typeof loginForm.getSchema>>;
// { email: string; password: string; }

const validate = loginForm.createValidator();
const errors = validate({ email: "invalid", password: "123" });
console.log(errors);
// { email: "Invalid email format", password: "Password must be at least 8 characters" }
```

---

## 12. Generic Repository Pattern

```typescript
// Generic CRUD repository
interface Entity {
    id: string;
    createdAt: Date;
    updatedAt: Date;
}

type CreateInput<T extends Entity> = Omit<T, "id" | "createdAt" | "updatedAt">;
type UpdateInput<T extends Entity> = Partial<CreateInput<T>>;

interface Repository<T extends Entity> {
    findById(id: string): Promise<T | null>;
    findAll(filter?: Partial<T>): Promise<T[]>;
    create(input: CreateInput<T>): Promise<T>;
    update(id: string, input: UpdateInput<T>): Promise<T | null>;
    delete(id: string): Promise<boolean>;
    count(filter?: Partial<T>): Promise<number>;
}

// In-memory implementation
class InMemoryRepository<T extends Entity> implements Repository<T> {
    protected items: Map<string, T> = new Map();

    async findById(id: string): Promise<T | null> {
        return this.items.get(id) ?? null;
    }

    async findAll(filter?: Partial<T>): Promise<T[]> {
        const items = Array.from(this.items.values());
        if (!filter) return items;
        return items.filter(item => 
            Object.entries(filter).every(([key, value]) => 
                item[key as keyof T] === value
            )
        );
    }

    async create(input: CreateInput<T>): Promise<T> {
        const item = {
            ...input,
            id: Math.random().toString(36).slice(2),
            createdAt: new Date(),
            updatedAt: new Date(),
        } as T;
        this.items.set(item.id, item);
        return item;
    }

    async update(id: string, input: UpdateInput<T>): Promise<T | null> {
        const existing = this.items.get(id);
        if (!existing) return null;
        const updated = {
            ...existing,
            ...input,
            id,
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

// ตัวอย่าง entity
interface UserEntity extends Entity {
    name: string;
    email: string;
    role: "admin" | "user";
}

interface PostEntity extends Entity {
    title: string;
    content: string;
    authorId: string;
    published: boolean;
}

// Repositories
class UserRepository extends InMemoryRepository<UserEntity> {
    async findByEmail(email: string): Promise<UserEntity | null> {
        const users = await this.findAll();
        return users.find(u => u.email === email) ?? null;
    }

    async findAdmins(): Promise<UserEntity[]> {
        return this.findAll({ role: "admin" });
    }
}

class PostRepository extends InMemoryRepository<PostEntity> {
    async findByAuthor(authorId: string): Promise<PostEntity[]> {
        return this.findAll({ authorId });
    }

    async findPublished(): Promise<PostEntity[]> {
        return this.findAll({ published: true });
    }
}

// Unit of Work pattern
class UnitOfWork {
    users: UserRepository = new UserRepository();
    posts: PostRepository = new PostRepository();

    async transaction<T>(work: (uow: UnitOfWork) => Promise<T>): Promise<T> {
        // จำลอง transaction (ในระบบจริงจะต้องทำ rollback ได้)
        try {
            return await work(this);
        } catch (error) {
            console.error("Transaction failed:", error);
            throw error;
        }
    }
}

// การใช้งาน
async function main() {
    const uow = new UnitOfWork();

    await uow.transaction(async (uow) => {
        const user = await uow.users.create({
            name: "Alice",
            email: "alice@example.com",
            role: "user"
        });

        const post = await uow.posts.create({
            title: "Hello TypeScript",
            content: "TypeScript is awesome!",
            authorId: user.id,
            published: true
        });

        console.log("Created user:", user.id);
        console.log("Created post:", post.id);
    });

    const admins = await uow.users.findAdmins();
    console.log("Admins:", admins.length);
}

main().catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Conditional Generic Types** - การสร้าง types ที่ตัดสินใจตาม condition
2. **Type Inference** - การใช้ `infer` เพื่อดึง types จาก conditional types
3. **HKT Simulation** - การจำลอง Higher-Kinded Types ใน TypeScript
4. **Type-Safe Event Emitter** - การสร้าง event system ที่ type-safe
5. **Builder Pattern** - การสร้าง fluent builders ที่ type-safe
6. **Command Pattern** - การสร้าง command bus ที่ type-safe
7. **Middleware Pipeline** - การสร้าง pipeline สำหรับ middleware
8. **Type-Level Programming** - การเขียนโปรแกรมใน type system
9. **Performance Considerations** - การเพิ่มประสิทธิภาพ type definitions
10. **Repository Pattern** - การสร้าง CRUD repositories ที่ type-safe

patterns เหล่านี้จะช่วยให้คุณสร้างระบบที่ robust, maintainable และ type-safe ได้อย่างมีประสิทธิภาพ
