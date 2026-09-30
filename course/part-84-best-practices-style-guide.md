# Part 84: TypeScript Best Practices & Style Guide

## บทนำ

การเขียน TypeScript ที่ดีไม่ใช่แค่การทำให้ compile ผ่าน แต่ต้องเป็นโค้ดที่อ่านง่าย maintain ง่าย และ type-safe อย่างแท้จริง บทนี้รวบรวม best practices และ style guide ที่นักพัฒนา TypeScript มืออาชีพใช้

---

## 1. Naming Conventions

### Variables และ Constants

```typescript
// ✅ ดี: camelCase สำหรับ variables
const userName = "Alice";
let itemCount = 0;
let isLoggedIn = false;

// ✅ ดี: SCREAMING_SNAKE_CASE สำหรับ constants ที่ไม่เปลี่ยนแปลง
const MAX_RETRY_ATTEMPTS = 3;
const API_BASE_URL = "https://api.example.com";
const DEFAULT_TIMEOUT_MS = 5000;

// ❌ ไม่ดี: ใช้ underscore prefix สำหรับ private (ควรใช้ private keyword)
const _privateVar = "bad";

// ✅ ดี: ใช้ private keyword
class Service {
    private baseUrl: string = "https://api.example.com";
}

// ✅ ดี: boolean variables ควรเริ่มด้วย is/has/can/should/will
const isActive = true;
const hasPermission = false;
const canEdit = true;
const shouldRefetch = false;
const willExpire = true;
```

### Functions

```typescript
// ✅ ดี: camelCase, verb-noun pattern
function getUserById(id: string): User | null { return null; }
function calculateTotalPrice(items: Item[]): number { return 0; }
function validateEmail(email: string): boolean { return true; }
function sendNotification(userId: string, message: string): Promise<void> { return Promise.resolve(); }

// ✅ ดี: event handlers ขึ้นต้นด้วย handle หรือ on
function handleButtonClick(event: MouseEvent): void {}
function onUserLogin(userId: string): void {}

// ✅ ดี: boolean functions ขึ้นต้นด้วย is/has/can/check
function isValidEmail(email: string): boolean { return true; }
function hasPermission(user: User, action: string): boolean { return true; }
function canUserEdit(user: User, resource: Resource): boolean { return true; }

// ❌ ไม่ดี: ชื่อที่ไม่บ่งบอกความหมาย
function process(x: any): any { return x; }
function doStuff(data: unknown): void {}
function fn1(): void {}
```

### Types และ Interfaces

```typescript
// ✅ ดี: PascalCase สำหรับ types/interfaces
interface UserProfile {
    id: string;
    name: string;
}

type ButtonVariant = "primary" | "secondary" | "danger";
type UserId = string;

// ✅ ดี: Generic type parameters ใช้ตัวอักษรหรือชื่อที่มีความหมาย
interface Repository<TEntity extends { id: string }> {
    findById(id: string): TEntity | null;
}

// T, K, V สำหรับ generic อย่างง่าย
function identity<T>(value: T): T { return value; }
function mapObject<K extends string, V>(obj: Record<K, V>): Record<K, V> { return obj; }

// ❌ ไม่ดี: prefix I หรือ T (ล้าสมัย)
interface IUserService {}  // ❌
type TConfig = {};         // ❌

// ✅ ดี: suffix ที่บ่งบอก role
interface UserService {}   // ✅
type ConfigOptions = {};   // ✅

// ✅ ดี: Type aliases สำหรับ unions ที่ซับซ้อน
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
type EventHandler<T = Event> = (event: T) => void;
type Nullable<T> = T | null;
type Maybe<T> = T | null | undefined;
```

### Classes

```typescript
// ✅ ดี: PascalCase สำหรับ class names, camelCase สำหรับ methods
class UserAuthService {
    private readonly userRepository: UserRepository;
    
    constructor(userRepository: UserRepository) {
        this.userRepository = userRepository;
    }
    
    async authenticate(credentials: Credentials): Promise<AuthToken> {
        throw new Error("Not implemented");
    }
    
    private hashPassword(password: string): string {
        return password; // simplified
    }
}

// ✅ ดี: static constants ใช้ UPPER_CASE
class Config {
    static readonly DEFAULT_TIMEOUT = 5000;
    static readonly MAX_CONNECTIONS = 10;
}
```

---

## 2. File Organization

```
project/
├── src/
│   ├── types/          # Type definitions, interfaces
│   │   ├── index.ts
│   │   ├── user.types.ts
│   │   └── api.types.ts
│   ├── utils/          # Utility functions
│   │   ├── index.ts
│   │   ├── date.utils.ts
│   │   └── string.utils.ts
│   ├── services/       # Business logic
│   │   ├── user.service.ts
│   │   └── auth.service.ts
│   ├── repositories/   # Data access
│   │   └── user.repository.ts
│   ├── controllers/    # Request handlers
│   │   └── user.controller.ts
│   ├── middleware/     # Express/Koa middleware
│   │   └── auth.middleware.ts
│   ├── config/         # Configuration
│   │   └── database.config.ts
│   └── index.ts        # Entry point
├── tests/
│   ├── unit/
│   └── integration/
└── tsconfig.json
```

### File Naming

```typescript
// ✅ ดี: kebab-case สำหรับ filenames
// user-profile.component.ts
// auth.service.ts
// date-formatter.utils.ts
// user.types.ts

// ✅ ดี: index files สำหรับ re-export
// src/types/index.ts
export * from "./user.types";
export * from "./api.types";
export * from "./common.types";

// ✅ ดี: feature-based organization สำหรับ large projects
// src/features/auth/
// src/features/user/
// src/features/products/

// ✅ ดี: barrel exports
// src/features/auth/index.ts
export { AuthService } from "./auth.service";
export { AuthController } from "./auth.controller";
export type { LoginInput, AuthToken } from "./auth.types";
```

---

## 3. Import Ordering

```typescript
// ✅ ดี: จัดเรียง imports ตามลำดับ
// 1. Node built-in modules
import * as fs from "fs";
import * as path from "path";
import { EventEmitter } from "events";

// 2. Third-party packages
import express from "express";
import { z } from "zod";
import axios from "axios";

// 3. Internal absolute imports
import { UserService } from "@/services/user.service";
import { logger } from "@/utils/logger";

// 4. Relative imports
import { validateUser } from "./validators";
import type { User, UserInput } from "./types";

// 5. Side effects (no exports used)
import "./polyfills";
import "./styles.css";
```

---

## 4. Type vs Interface Decision Guide

```typescript
// ใช้ Interface เมื่อ:
// 1. กำหนด structure ของ object หรือ class
interface UserRepository {
    findById(id: string): Promise<User | null>;
    save(user: User): Promise<User>;
    delete(id: string): Promise<void>;
}

// 2. ต้องการ declaration merging
interface Window {
    myCustomProperty: string;
}

// 3. กำหนด contract สำหรับ class implementation
interface Serializable {
    serialize(): string;
    deserialize(data: string): void;
}

// ใช้ Type Alias เมื่อ:
// 1. Union types
type Status = "active" | "inactive" | "pending";
type StringOrNumber = string | number;

// 2. Intersection types
type AdminUser = User & { adminLevel: number };

// 3. Mapped types
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// 4. Conditional types
type IsArray<T> = T extends any[] ? true : false;

// 5. Template literal types
type EventName = `on${Capitalize<string>}`;

// 6. Tuple types
type Pair<T> = [T, T];
type Triple<T> = [T, T, T];

// ✅ Guideline: ใช้ interface สำหรับ public API, type สำหรับ complex types

// ❌ อย่าผสมโดยไม่มีเหตุผล
type SimpleObject = {  // ควรใช้ interface
    name: string;
    age: number;
};

interface ComplexType = string | number; // ❌ syntax error, ควรใช้ type
```

---

## 5. Any vs Unknown

```typescript
// ❌ ไม่ดี: ใช้ any ทำลาย type safety
function processData(data: any): any {
    return data.toString(); // ไม่มี type checking!
}

// ✅ ดี: ใช้ unknown แทน any
function processData(data: unknown): string {
    if (typeof data === "string") return data;
    if (typeof data === "number") return data.toString();
    if (data === null || data === undefined) return "";
    return String(data);
}

// ✅ ดี: ใช้ type narrowing กับ unknown
function handleError(error: unknown): string {
    if (error instanceof Error) {
        return error.message;
    }
    if (typeof error === "string") {
        return error;
    }
    return "An unknown error occurred";
}

// ✅ ดี: JSON parsing ด้วย unknown
function parseJson(json: string): unknown {
    return JSON.parse(json);
}

// กรณีที่ any อาจเหมาะสม:
// 1. Migration: ใช้ระหว่าง migrate JS -> TS
// 2. Third-party libraries ที่ไม่มี types
// 3. Mock objects ใน tests
// 4. Very dynamic code ที่ type inference ไม่ทำงาน

// ✅ ดี: type assertion กับ unknown
interface ApiResponse {
    data: User[];
    total: number;
}

async function fetchUsers(): Promise<ApiResponse> {
    const response = await fetch("/api/users");
    const data: unknown = await response.json();
    
    // Validate structure ก่อน cast
    if (!isApiResponse(data)) {
        throw new Error("Invalid API response");
    }
    
    return data;
}

function isApiResponse(data: unknown): data is ApiResponse {
    return (
        typeof data === "object" &&
        data !== null &&
        "data" in data &&
        "total" in data &&
        Array.isArray((data as any).data)
    );
}
```

---

## 6. การหลีกเลี่ยงข้อผิดพลาดที่พบบ่อย

```typescript
// ❌ ข้อผิดพลาด 1: Type assertion ที่ไม่ปลอดภัย
const element = document.getElementById("root") as HTMLElement;
element.innerHTML = "Hello"; // อาจ crash ถ้า element เป็น null

// ✅ ดี: ตรวจสอบก่อน
const element = document.getElementById("root");
if (element) {
    element.innerHTML = "Hello";
}

// หรือใช้ non-null assertion (!) เมื่อมั่นใจ 100%
const definitelyElement = document.getElementById("root")!;

// ❌ ข้อผิดพลาด 2: Object mutation
function addItem(arr: readonly string[], item: string): readonly string[] {
    // arr.push(item); // ❌ Error: readonly
    return [...arr, item]; // ✅
}

// ❌ ข้อผิดพลาด 3: Implicit any ใน callbacks
[1, 2, 3].map(x => x * 2);  // x เป็น number ✅
const data = JSON.parse("[]");
data.map(item => item.name); // item เป็น any ❌

// ✅ ดี: Type annotation ใน callbacks
const items = JSON.parse("[]") as Array<{ name: string }>;
items.map(item => item.name); // ✅

// ❌ ข้อผิดพลาด 4: Optional chaining ที่ขาดหาย
function getUserCity(user: User | null): string {
    return user.address.city; // ❌ อาจ crash
}

// ✅ ดี: Optional chaining
function getUserCity(user: User | null): string | undefined {
    return user?.address?.city;
}

// ❌ ข้อผิดพลาด 5: Forgotten await
async function fetchData(): Promise<void> {
    const data = fetch("/api/data"); // ❌ Promise ไม่ถูก await
    console.log(data); // [object Promise]
}

// ✅ ดี:
async function fetchData(): Promise<void> {
    const data = await fetch("/api/data"); // ✅
    const json = await data.json();
    console.log(json);
}

// ❌ ข้อผิดพลาด 6: Object spread ที่ทำให้ type เปลี่ยน
interface Config {
    host: string;
    port: number;
}

function mergeConfig(base: Config, override: Partial<Config>): Config {
    return { ...base, ...override }; // ✅ Type ยังคงเป็น Config
}

// ❌ ข้อผิดพลาด 7: Circular imports
// a.ts: import { B } from "./b"
// b.ts: import { A } from "./a"  <- Circular!

// ✅ ดี: แยก shared types ออกมา
// types.ts: export interface SharedType {}
// a.ts: import { SharedType } from "./types"
// b.ts: import { SharedType } from "./types"
```

---

## 7. Code Review Checklist

```typescript
// Checklist สำหรับ Code Review TypeScript

// ✅ Type Safety
// - ไม่มี any ที่ไม่จำเป็น
// - มี return types สำหรับ public functions
// - ใช้ strict mode
// - มีการจัดการ null/undefined

// ✅ Code Quality
// - Functions ทำงานเดียว (Single Responsibility)
// - ไม่มี dead code
// - ใช้ const สำหรับ values ที่ไม่เปลี่ยน
// - Extract constants ออกจาก magic numbers/strings

// ✅ Error Handling
// - ทุก async operation มี try/catch
// - Error messages มีความหมาย
// - ใช้ custom error types

// ✅ Performance
// - ไม่มี N+1 queries
// - ใช้ memoization เมื่อเหมาะสม
// - Lazy loading สำหรับ heavy modules

// ตัวอย่าง: Before (Code Review ติ)
async function badGetUser(id: string): Promise<any> {
    try {
        const user = await db.query(`SELECT * FROM users WHERE id = ${id}`); // SQL injection!
        return user;
    } catch(e) {
        console.log(e);
        return null;
    }
}

// ตัวอย่าง: After (Fixed)
class UserNotFoundError extends Error {
    constructor(id: string) {
        super(`User with id ${id} not found`);
        this.name = "UserNotFoundError";
    }
}

async function goodGetUser(id: string): Promise<User> {
    if (!id || typeof id !== "string") {
        throw new TypeError("User ID must be a non-empty string");
    }
    
    const user = await db.query<User>(
        "SELECT * FROM users WHERE id = $1",
        [id]  // Parameterized query
    );
    
    if (!user) {
        throw new UserNotFoundError(id);
    }
    
    return user;
}
```

---

## 8. Documentation Patterns

```typescript
/**
 * JSDoc สำหรับ TypeScript
 * ใช้ JSDoc comments สำหรับ public APIs
 */

/**
 * บริการจัดการ User authentication
 * @example
 * ```typescript
 * const auth = new AuthService(userRepo, tokenService);
 * const token = await auth.login({ email: "user@example.com", password: "secret" });
 * ```
 */
class AuthService {
    /**
     * ตรวจสอบ credentials และสร้าง authentication token
     * @param credentials - Email และ password ของ user
     * @returns JWT token ที่ใช้สำหรับ authentication
     * @throws {InvalidCredentialsError} เมื่อ email หรือ password ไม่ถูกต้อง
     * @throws {UserDisabledError} เมื่อ account ถูก disable
     */
    async login(credentials: LoginCredentials): Promise<AuthToken> {
        throw new Error("Not implemented");
    }
    
    /**
     * ยกเลิก session ของ user
     * @param token - Token ที่ต้องการ revoke
     * @returns void
     * @remarks ไม่ throw error ถ้า token ไม่มีอยู่หรือ expire แล้ว
     */
    async logout(token: string): Promise<void> {
        throw new Error("Not implemented");
    }
}

// Type documentation
/**
 * Configuration สำหรับ database connection
 * @see https://docs.example.com/database-config
 */
interface DatabaseConfig {
    /** Hostname หรือ IP address ของ database server */
    host: string;
    /** Port number (default: 5432 สำหรับ PostgreSQL) */
    port?: number;
    /** Database name */
    database: string;
    /** @deprecated ใช้ connectionString แทน */
    user?: string;
    /** @deprecated ใช้ connectionString แทน */
    password?: string;
    /** Connection string รูปแบบ: postgres://user:pass@host:port/db */
    connectionString?: string;
    /**
     * Maximum number of connections ใน pool
     * @default 10
     * @minimum 1
     * @maximum 100
     */
    maxConnections?: number;
}
```

---

## 9. Generic Naming Conventions

```typescript
// ✅ ดี: ใช้ชื่อที่มีความหมายสำหรับ generic parameters

// T สำหรับ general type
function first<T>(arr: T[]): T | undefined {
    return arr[0];
}

// K, V สำหรับ key-value pairs
function mapToRecord<K extends string, V>(
    keys: K[],
    valueFor: (key: K) => V
): Record<K, V> {
    return Object.fromEntries(keys.map(k => [k, valueFor(k)])) as Record<K, V>;
}

// TEntity สำหรับ entities
interface Repository<TEntity extends { id: string }> {
    findById(id: string): Promise<TEntity | null>;
}

// TInput, TOutput สำหรับ transformations
interface Transformer<TInput, TOutput> {
    transform(input: TInput): TOutput;
}

// TState, TAction สำหรับ state management
interface Reducer<TState, TAction> {
    (state: TState, action: TAction): TState;
}

// TError สำหรับ error handling
type Result<TValue, TError = Error> = 
    | { success: true; value: TValue }
    | { success: false; error: TError };

// ✅ ดี: ตัวอย่าง generic ที่มีความหมาย
function fetchAndTransform<
    TRaw,           // Raw data จาก API
    TTransformed    // Transformed data
>(
    url: string,
    transformer: (raw: TRaw) => TTransformed
): Promise<TTransformed> {
    return fetch(url)
        .then(r => r.json() as Promise<TRaw>)
        .then(transformer);
}
```

---

## 10. Error Handling Best Practices

```typescript
// ✅ ดี: สร้าง custom error hierarchy
class AppError extends Error {
    constructor(
        message: string,
        public readonly code: string,
        public readonly statusCode: number = 500
    ) {
        super(message);
        this.name = new.target.name;
        Object.setPrototypeOf(this, new.target.prototype);
    }
}

class ValidationError extends AppError {
    constructor(
        public readonly field: string,
        message: string
    ) {
        super(message, "VALIDATION_ERROR", 400);
    }
}

class NotFoundError extends AppError {
    constructor(resource: string, id: string) {
        super(`${resource} with id '${id}' not found`, "NOT_FOUND", 404);
    }
}

class UnauthorizedError extends AppError {
    constructor(message = "Unauthorized") {
        super(message, "UNAUTHORIZED", 401);
    }
}

// ✅ ดี: Result type pattern
type Success<T> = { ok: true; value: T };
type Failure<E = Error> = { ok: false; error: E };
type Result<T, E = Error> = Success<T> | Failure<E>;

function succeed<T>(value: T): Success<T> {
    return { ok: true, value };
}

function fail<E = Error>(error: E): Failure<E> {
    return { ok: false, error };
}

// ใช้ Result type
async function divideNumbers(a: number, b: number): Promise<Result<number, string>> {
    if (b === 0) {
        return fail("Cannot divide by zero");
    }
    return succeed(a / b);
}

// Caller ต้อง handle ทั้งสองกรณี
async function compute(): Promise<void> {
    const result = await divideNumbers(10, 2);
    if (!result.ok) {
        console.error(`Error: ${result.error}`);
        return;
    }
    console.log(`Result: ${result.value}`);
}

// ✅ ดี: Try/catch ที่ type-safe
async function safeJsonParse<T>(json: string): Promise<Result<T, SyntaxError>> {
    try {
        const value = JSON.parse(json) as T;
        return succeed(value);
    } catch (error) {
        if (error instanceof SyntaxError) {
            return fail(error);
        }
        return fail(new SyntaxError(`Unexpected error: ${String(error)}`));
    }
}
```

---

## 11. Testing Best Practices

```typescript
// ✅ ดี: Type-safe mocks
import { jest } from "@jest/globals";

interface UserService {
    getUser(id: string): Promise<User>;
    createUser(input: CreateUserInput): Promise<User>;
}

// สร้าง typed mock
function createMockUserService(): jest.Mocked<UserService> {
    return {
        getUser: jest.fn(),
        createUser: jest.fn(),
    };
}

// Test ที่มีความหมาย
describe("UserController", () => {
    let userService: jest.Mocked<UserService>;
    let controller: UserController;
    
    beforeEach(() => {
        userService = createMockUserService();
        controller = new UserController(userService);
    });
    
    describe("getUser", () => {
        it("returns user when found", async () => {
            // Arrange
            const mockUser: User = {
                id: "123",
                name: "Alice",
                email: "alice@example.com"
            };
            userService.getUser.mockResolvedValue(mockUser);
            
            // Act
            const result = await controller.getUser("123");
            
            // Assert
            expect(result).toEqual(mockUser);
            expect(userService.getUser).toHaveBeenCalledWith("123");
        });
        
        it("throws NotFoundError when user does not exist", async () => {
            // Arrange
            userService.getUser.mockRejectedValue(new NotFoundError("User", "999"));
            
            // Act & Assert
            await expect(controller.getUser("999"))
                .rejects
                .toThrow(NotFoundError);
        });
    });
});

// ✅ ดี: Test helpers ที่ type-safe
function createMockUser(overrides: Partial<User> = {}): User {
    return {
        id: "test-id",
        name: "Test User",
        email: "test@example.com",
        createdAt: new Date("2024-01-01"),
        ...overrides,
    };
}

// ✅ ดี: Custom matchers
expect.extend({
    toBeValidEmail(received: string) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        const pass = emailRegex.test(received);
        return {
            message: () =>
                pass
                    ? `expected ${received} not to be a valid email`
                    : `expected ${received} to be a valid email`,
            pass,
        };
    },
});

// TypeScript declarations สำหรับ custom matchers
declare global {
    namespace jest {
        interface Matchers<R> {
            toBeValidEmail(): R;
        }
    }
}

// ทดสอบ
it("validates email format", () => {
    expect("user@example.com").toBeValidEmail();
    expect("invalid-email").not.toBeValidEmail();
});
```

---

## 12. Performance Best Practices

```typescript
// ✅ ดี: Lazy imports สำหรับ heavy modules
async function processImage(imagePath: string): Promise<void> {
    // Import เฉพาะเมื่อจำเป็น
    const { createCanvas } = await import("canvas");
    const canvas = createCanvas(800, 600);
    // ...
}

// ✅ ดี: Memoization
function memoize<TArgs extends any[], TReturn>(
    fn: (...args: TArgs) => TReturn
): (...args: TArgs) => TReturn {
    const cache = new Map<string, TReturn>();
    
    return (...args: TArgs): TReturn => {
        const key = JSON.stringify(args);
        if (cache.has(key)) {
            return cache.get(key)!;
        }
        const result = fn(...args);
        cache.set(key, result);
        return result;
    };
}

// ✅ ดี: Debounce และ Throttle ที่ type-safe
function debounce<T extends (...args: any[]) => any>(
    fn: T,
    delay: number
): (...args: Parameters<T>) => void {
    let timeoutId: ReturnType<typeof setTimeout>;
    
    return (...args: Parameters<T>) => {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => fn(...args), delay);
    };
}

function throttle<T extends (...args: any[]) => any>(
    fn: T,
    limit: number
): (...args: Parameters<T>) => void {
    let lastRun = 0;
    
    return (...args: Parameters<T>) => {
        const now = Date.now();
        if (now - lastRun >= limit) {
            lastRun = now;
            fn(...args);
        }
    };
}

// ✅ ดี: Object pooling สำหรับ frequently created objects
class ObjectPool<T> {
    private pool: T[] = [];
    private readonly factory: () => T;
    private readonly reset: (obj: T) => void;
    
    constructor(factory: () => T, reset: (obj: T) => void, initialSize = 10) {
        this.factory = factory;
        this.reset = reset;
        
        // Pre-populate pool
        for (let i = 0; i < initialSize; i++) {
            this.pool.push(factory());
        }
    }
    
    acquire(): T {
        return this.pool.pop() ?? this.factory();
    }
    
    release(obj: T): void {
        this.reset(obj);
        this.pool.push(obj);
    }
}

// ✅ ดี: Avoid ไม่จำเป็นต้อง allocate objects
// ❌ Bad: สร้าง array ใหม่ทุกครั้ง
function getActiveUsers(users: User[]): User[] {
    return users.filter(u => u.isActive); // สร้าง array ใหม่
}

// ✅ Better: ใช้ generator
function* activeUsers(users: User[]): Generator<User> {
    for (const user of users) {
        if (user.isActive) yield user;
    }
}
```

---

## 13. Team Collaboration Patterns

```typescript
// ✅ ดี: Shared type definitions
// types/shared.ts
export interface PaginationOptions {
    page: number;
    pageSize: number;
    sortBy?: string;
    sortOrder?: "asc" | "desc";
}

export interface PaginatedResponse<T> {
    data: T[];
    total: number;
    page: number;
    pageSize: number;
    hasNext: boolean;
    hasPrev: boolean;
}

// ✅ ดี: Consistent error responses
export interface ApiError {
    code: string;
    message: string;
    details?: Record<string, string[]>;
    timestamp: string;
    requestId: string;
}

// ✅ ดี: Feature flags ที่ type-safe
type FeatureFlag = 
    | "NEW_DASHBOARD"
    | "BETA_CHECKOUT"
    | "DARK_MODE"
    | "AI_SUGGESTIONS";

interface FeatureFlags {
    isEnabled(flag: FeatureFlag): boolean;
    getVariant(flag: FeatureFlag): string | null;
}

// ✅ ดี: Environment configuration
interface Environment {
    NODE_ENV: "development" | "staging" | "production";
    PORT: string;
    DATABASE_URL: string;
    JWT_SECRET: string;
    API_KEY?: string;
}

function getEnv<K extends keyof Environment>(
    key: K
): K extends "API_KEY" ? string | undefined : string {
    const value = process.env[key];
    if (value === undefined && key !== "API_KEY") {
        throw new Error(`Environment variable ${key} is required but not set`);
    }
    return value as any;
}
```

---

## 14. Before/After Examples

### Example 1: Function Signatures

```typescript
// ❌ Before
function process(data: any, options?: any): any {
    if (options && options.format) {
        return data.toString();
    }
    return data;
}

// ✅ After
interface ProcessOptions {
    format?: "string" | "json" | "number";
    encoding?: "utf-8" | "base64";
}

function processData<T>(
    data: T,
    options?: ProcessOptions
): options extends { format: "string" } ? string : T {
    if (options?.format === "string") {
        return String(data) as any;
    }
    return data as any;
}
```

### Example 2: Error Handling

```typescript
// ❌ Before
async function getUser(id: string) {
    try {
        const user = await fetch(`/users/${id}`);
        return user.json();
    } catch(e) {
        return null;
    }
}

// ✅ After
interface User {
    id: string;
    name: string;
    email: string;
}

class ApiError extends Error {
    constructor(
        public readonly status: number,
        message: string
    ) {
        super(message);
        this.name = "ApiError";
    }
}

async function getUser(id: string): Promise<User> {
    const response = await fetch(`/users/${encodeURIComponent(id)}`);
    
    if (!response.ok) {
        throw new ApiError(
            response.status,
            `Failed to fetch user: ${response.statusText}`
        );
    }
    
    const data: unknown = await response.json();
    
    if (!isUser(data)) {
        throw new TypeError("Invalid user data received from API");
    }
    
    return data;
}

function isUser(data: unknown): data is User {
    return (
        typeof data === "object" &&
        data !== null &&
        "id" in data &&
        "name" in data &&
        "email" in data &&
        typeof (data as any).id === "string" &&
        typeof (data as any).name === "string" &&
        typeof (data as any).email === "string"
    );
}
```

### Example 3: Class Design

```typescript
// ❌ Before
class UserManager {
    users: any[] = [];
    
    addUser(user: any) {
        this.users.push(user);
    }
    
    getUser(id: string) {
        return this.users.find(u => u.id === id);
    }
    
    updateUser(id: string, data: any) {
        const idx = this.users.findIndex(u => u.id === id);
        if (idx !== -1) {
            this.users[idx] = { ...this.users[idx], ...data };
        }
    }
}

// ✅ After
interface User {
    readonly id: string;
    name: string;
    email: string;
    createdAt: Date;
}

type UpdateUserInput = Partial<Pick<User, "name" | "email">>;

class UserRepository {
    private readonly users: Map<string, User> = new Map();
    
    add(user: User): void {
        if (this.users.has(user.id)) {
            throw new Error(`User with id ${user.id} already exists`);
        }
        this.users.set(user.id, { ...user }); // Store copy
    }
    
    findById(id: string): User | undefined {
        const user = this.users.get(id);
        return user ? { ...user } : undefined; // Return copy
    }
    
    update(id: string, input: UpdateUserInput): User {
        const existing = this.users.get(id);
        if (!existing) {
            throw new NotFoundError("User", id);
        }
        
        const updated: User = { ...existing, ...input };
        this.users.set(id, updated);
        return { ...updated }; // Return copy
    }
    
    delete(id: string): void {
        if (!this.users.delete(id)) {
            throw new NotFoundError("User", id);
        }
    }
    
    findAll(): User[] {
        return Array.from(this.users.values()).map(u => ({ ...u }));
    }
}
```

---

## สรุป TypeScript Best Practices

1. **Naming**: ใช้ camelCase สำหรับ variables/functions, PascalCase สำหรับ types/classes
2. **Organization**: จัดโครงสร้าง files ตาม features หรือ layers
3. **Types**: ใช้ interface สำหรับ objects, type สำหรับ complex types
4. **Safety**: หลีกเลี่ยง any, ใช้ unknown แทน
5. **Null Handling**: ใช้ optional chaining และ nullish coalescing
6. **Errors**: สร้าง custom error classes, ใช้ Result pattern
7. **Testing**: สร้าง typed mocks, เขียน descriptive test names
8. **Performance**: ใช้ lazy loading, memoization, debounce/throttle
9. **Documentation**: เขียน JSDoc สำหรับ public APIs
10. **Collaboration**: ใช้ shared types, consistent patterns ทั้ง team
