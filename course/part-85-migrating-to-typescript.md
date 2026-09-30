# Part 85: Migrating JavaScript to TypeScript

## บทนำ

การ migrate JavaScript codebase ไปสู่ TypeScript เป็นกระบวนการที่ต้องวางแผนอย่างรอบคอบ บทนี้จะแนะนำกลยุทธ์และเทคนิคต่างๆ สำหรับการ migrate อย่างมีประสิทธิภาพ

---

## 1. Migration Strategy

### ประเมิน Codebase ก่อนเริ่ม

```bash
# นับจำนวน files
find src -name "*.js" | wc -l

# วิเคราะห์ dependencies
npm ls --depth=0

# ตรวจสอบ test coverage
npm test -- --coverage
```

### กลยุทธ์การ Migrate

```
กลยุทธ์ที่ 1: Big Bang Migration
- แปลง files ทั้งหมดพร้อมกัน
- เหมาะกับ projects ขนาดเล็ก
- เสี่ยงสูง

กลยุทธ์ที่ 2: Strangler Fig Pattern
- เพิ่ม TypeScript support โดยไม่แตะ JS files
- ค่อยๆ แปลงทีละ module
- เหมาะกับ projects ขนาดใหญ่
- เสี่ยงต่ำ

กลยุทธ์ที่ 3: Type Gradual Adoption
- เริ่มด้วย allowJs + checkJs
- เพิ่ม JSDoc types
- ค่อยๆ เปลี่ยน .js เป็น .ts
```

### การเตรียม tsconfig.json เริ่มต้น

```json
// tsconfig.json สำหรับ migration (lenient mode)
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "CommonJS",
        "lib": ["ES2020"],
        "allowJs": true,
        "checkJs": false,
        "outDir": "./dist",
        "rootDir": "./src",
        "strict": false,
        "noImplicitAny": false,
        "strictNullChecks": false,
        "skipLibCheck": true,
        "esModuleInterop": true,
        "allowSyntheticDefaultImports": true,
        "moduleResolution": "node",
        "resolveJsonModule": true,
        "sourceMap": true,
        "declaration": true
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "dist"]
}
```

---

## 2. allowJs Configuration

`allowJs` ให้ TypeScript compile JavaScript files ได้

```json
// tsconfig.json
{
    "compilerOptions": {
        "allowJs": true,
        "outDir": "./dist"
    }
}
```

```javascript
// src/utils.js - JavaScript file ที่ยังไม่ได้ migrate
function formatDate(date) {
    return new Intl.DateTimeFormat("th-TH").format(date);
}

function calculateAge(birthDate) {
    const today = new Date();
    const birth = new Date(birthDate);
    return today.getFullYear() - birth.getFullYear();
}

module.exports = { formatDate, calculateAge };
```

```typescript
// src/app.ts - TypeScript file ที่ import จาก JS
import { formatDate, calculateAge } from "./utils"; // ✅ Works with allowJs

const today = formatDate(new Date()); // type: any (ยังไม่มี types)
const age = calculateAge("1990-01-01"); // type: any
```

### Incremental allowJs Setup

```json
// ขั้นที่ 1: เปิดแค่ allowJs
{
    "compilerOptions": {
        "allowJs": true,
        "checkJs": false
    }
}
```

```json
// ขั้นที่ 2: เปิด checkJs สำหรับบาง files
{
    "compilerOptions": {
        "allowJs": true,
        "checkJs": false  // default false
    }
}
// ใส่ // @ts-check ใน files ที่ต้องการ check
```

```json
// ขั้นที่ 3: เปิด checkJs ทั้งหมด
{
    "compilerOptions": {
        "allowJs": true,
        "checkJs": true
    }
}
// ใส่ // @ts-nocheck ใน files ที่ยังไม่พร้อม
```

---

## 3. checkJs Mode

`checkJs` ให้ TypeScript type-check JavaScript files

```javascript
// src/api.js
// @ts-check  <- เปิด type checking สำหรับ file นี้

/**
 * @param {string} url
 * @param {Object} options
 * @param {string} [options.method='GET']
 * @param {Record<string, string>} [options.headers]
 * @returns {Promise<unknown>}
 */
async function fetchJson(url, options = {}) {
    const { method = "GET", headers = {} } = options;
    
    const response = await fetch(url, {
        method,
        headers: {
            "Content-Type": "application/json",
            ...headers,
        },
    });
    
    if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    return response.json();
}

// TypeScript จะ check นี้
const result = fetchJson(123); // ❌ Error: Argument of type 'number' is not assignable to parameter of type 'string'
```

### JSDoc Types ใน JavaScript

```javascript
// @ts-check

// Object types
/** @type {{ name: string; age: number }} */
const user = { name: "Alice", age: 30 };

// Array types
/** @type {string[]} */
const names = ["Alice", "Bob", "Charlie"];

// Function types
/** @type {(x: number, y: number) => number} */
const add = (x, y) => x + y;

// Generic types
/** @type {Array<{ id: number; label: string }>} */
const items = [
    { id: 1, label: "Item 1" },
    { id: 2, label: "Item 2" },
];

// Union types
/** @type {string | null} */
let selectedId = null;

// Importing types จาก TypeScript files
/** @type {import('./types').User} */
let currentUser = null;

// Defining interfaces with @typedef
/**
 * @typedef {Object} Config
 * @property {string} host
 * @property {number} port
 * @property {boolean} [debug=false]
 */

/** @type {Config} */
const config = {
    host: "localhost",
    port: 3000,
};
```

---

## 4. Gradual Migration Approach

### Phase 1: Setup

```bash
# ติดตั้ง TypeScript
npm install --save-dev typescript @types/node

# สร้าง tsconfig.json
npx tsc --init

# แก้ไข scripts ใน package.json
```

```json
// package.json
{
    "scripts": {
        "build": "tsc",
        "build:watch": "tsc --watch",
        "start": "node dist/index.js",
        "dev": "ts-node src/index.ts"
    },
    "devDependencies": {
        "typescript": "^5.0.0",
        "@types/node": "^20.0.0",
        "ts-node": "^10.0.0"
    }
}
```

### Phase 2: เพิ่ม tsconfig ที่ lenient

```json
{
    "compilerOptions": {
        "allowJs": true,
        "checkJs": false,
        "noImplicitAny": false,
        "strictNullChecks": false,
        "strict": false,
        "skipLibCheck": true
    }
}
```

### Phase 3: แปลง files ทีละ module

```javascript
// src/utils/date.js - ก่อนแปลง
function formatDate(date) {
    return new Intl.DateTimeFormat("th-TH").format(date);
}

function addDays(date, days) {
    const result = new Date(date);
    result.setDate(result.getDate() + days);
    return result;
}

module.exports = { formatDate, addDays };
```

```typescript
// src/utils/date.ts - หลังแปลง
export function formatDate(date: Date): string {
    return new Intl.DateTimeFormat("th-TH").format(date);
}

export function addDays(date: Date, days: number): Date {
    const result = new Date(date);
    result.setDate(result.getDate() + days);
    return result;
}
```

### Phase 4: เพิ่ม strict mode ทีละขั้น

```json
// ขั้นที่ 1: เพิ่ม noImplicitAny
{
    "compilerOptions": {
        "noImplicitAny": true
    }
}
```

```json
// ขั้นที่ 2: เพิ่ม strictNullChecks
{
    "compilerOptions": {
        "noImplicitAny": true,
        "strictNullChecks": true
    }
}
```

```json
// ขั้นที่ 3: เปิด strict mode ทั้งหมด
{
    "compilerOptions": {
        "strict": true
    }
}
```

---

## 5. Adding Types Incrementally

### Pattern 1: เริ่มจาก Types ที่ง่าย

```typescript
// ก่อน: ทุกอย่างเป็น any
function processUser(user: any): any {
    return {
        displayName: user.firstName + " " + user.lastName,
        email: user.email,
    };
}

// ขั้นที่ 1: เพิ่ม return type
function processUser(user: any): { displayName: string; email: string } {
    return {
        displayName: user.firstName + " " + user.lastName,
        email: user.email,
    };
}

// ขั้นที่ 2: เพิ่ม parameter type
interface RawUser {
    firstName: string;
    lastName: string;
    email: string;
}

function processUser(user: RawUser): { displayName: string; email: string } {
    return {
        displayName: `${user.firstName} ${user.lastName}`,
        email: user.email,
    };
}

// ขั้นที่ 3: Extract return type
interface ProcessedUser {
    displayName: string;
    email: string;
}

function processUser(user: RawUser): ProcessedUser {
    return {
        displayName: `${user.firstName} ${user.lastName}`,
        email: user.email,
    };
}
```

### Pattern 2: ใช้ Type Widening ระหว่าง Migration

```typescript
// ระหว่าง migration ใช้ type ที่กว้างกว่าได้
type AnyObject = Record<string, unknown>;
type AnyFunction = (...args: any[]) => any;

// แล้วค่อยๆ narrow ลง
function processConfig(config: AnyObject): AnyObject {
    // implementation
    return config;
}

// เปลี่ยนเป็น type ที่ specific
interface AppConfig {
    port: number;
    host: string;
    debug: boolean;
}

function processConfig(config: AppConfig): AppConfig {
    return { ...config, host: config.host.toLowerCase() };
}
```

### Pattern 3: การจัดการ Dynamic Properties

```typescript
// JavaScript: dynamic property access
function getValue(obj: any, key: string): any {
    return obj[key];
}

// TypeScript: ใช้ index signature
interface Dictionary<T = unknown> {
    [key: string]: T;
}

function getValue<T>(obj: Dictionary<T>, key: string): T | undefined {
    return obj[key];
}

// หรือใช้ Record type
function getValue<K extends string, V>(
    obj: Record<K, V>,
    key: K
): V {
    return obj[key];
}
```

---

## 6. Third-party Module Types

### ติดตั้ง @types packages

```bash
# ติดตั้ง type definitions
npm install --save-dev @types/express @types/lodash @types/jest

# ตรวจสอบว่ามี @types สำหรับ package ที่ใช้
npm info @types/express
```

```typescript
// ใช้งาน Express กับ types
import express, { Request, Response, NextFunction } from "express";

const app = express();

app.get("/users/:id", async (req: Request, res: Response): Promise<void> => {
    const { id } = req.params;
    // req.params.id เป็น string ✅
    res.json({ id });
});
```

### สร้าง Declaration Files เองสำหรับ packages ที่ไม่มี @types

```typescript
// types/my-legacy-module.d.ts
declare module "my-legacy-module" {
    export interface LegacyConfig {
        host: string;
        port: number;
        timeout?: number;
    }
    
    export function connect(config: LegacyConfig): Promise<void>;
    export function disconnect(): void;
    export function query(sql: string, params?: unknown[]): Promise<unknown[]>;
    
    const legacyModule: {
        connect: typeof connect;
        disconnect: typeof disconnect;
        query: typeof query;
    };
    
    export default legacyModule;
}
```

### Augmenting Existing Types

```typescript
// types/express.d.ts - เพิ่ม custom properties ให้ Express Request
import { User } from "./user";

declare global {
    namespace Express {
        interface Request {
            user?: User;
            correlationId: string;
            startTime: number;
        }
    }
}

// types/environment.d.ts - Type-safe environment variables
declare global {
    namespace NodeJS {
        interface ProcessEnv {
            NODE_ENV: "development" | "staging" | "production";
            PORT?: string;
            DATABASE_URL: string;
            JWT_SECRET: string;
        }
    }
}
```

---

## 7. @ts-ignore vs @ts-expect-error

```typescript
// @ts-ignore: ข้ามการ check error บรรทัดถัดไป (ไม่แนะนำ)
// ❌ ไม่ดี: ใช้ไม่ควบคุม
// @ts-ignore
const result = someUntypedFunction(); // error ถูกซ่อน ไม่ว่าจะมีหรือไม่

// @ts-expect-error: คาดหวังว่ามี error
// ✅ ดีกว่า: บังคับให้มี error จริงๆ
// @ts-expect-error - this function doesn't exist yet
const result2 = futureFunction(); // ✅ ถ้า futureFunction มีอยู่จริง TypeScript จะ error

// @ts-expect-error ในทาง pratice
interface Config {
    port: number;
}

// ✅ ใช้สำหรับ test ที่ตั้งใจทดสอบ type errors
it("rejects invalid config", () => {
    // @ts-expect-error - intentionally passing wrong type
    expect(() => validateConfig({ port: "3000" })).toThrow();
});

// เมื่อไรควรใช้ @ts-ignore:
// - Third-party library มี bug ใน types
// - Temporary workaround ระหว่าง migration
// - Code ที่ generate โดย tools

// ✅ ดีกว่า @ts-ignore: สร้าง type assertion function
function assertType<T>(value: unknown): asserts value is T {
    // Runtime validation
}
```

---

## 8. Declaration Files สำหรับ Legacy Code

```typescript
// ตัวอย่าง: สร้าง .d.ts สำหรับ legacy JavaScript library

// legacy/calculator.js
/*
var Calculator = (function() {
    function Calculator(initialValue) {
        this.value = initialValue || 0;
    }
    
    Calculator.prototype.add = function(n) {
        this.value += n;
        return this;
    };
    
    Calculator.prototype.multiply = function(n) {
        this.value *= n;
        return this;
    };
    
    Calculator.prototype.result = function() {
        return this.value;
    };
    
    return Calculator;
})();
*/

// legacy/calculator.d.ts - Declaration file
declare class Calculator {
    value: number;
    
    constructor(initialValue?: number);
    add(n: number): this;
    multiply(n: number): this;
    result(): number;
}

declare const calculator: typeof Calculator;
export = calculator;

// การใช้งาน
import Calculator = require("./legacy/calculator");

const calc = new Calculator(10)
    .add(5)
    .multiply(2)
    .result(); // 30
```

### Module Declaration Patterns

```typescript
// สำหรับ CommonJS modules
declare module "legacy-utils" {
    function formatCurrency(amount: number, currency?: string): string;
    function formatDate(date: Date | string, format?: string): string;
    
    export { formatCurrency, formatDate };
}

// สำหรับ UMD modules (รองรับทั้ง AMD, CommonJS, Global)
declare module "legacy-chart" {
    interface ChartOptions {
        width: number;
        height: number;
        data: number[];
        color?: string;
    }
    
    class Chart {
        constructor(canvas: HTMLCanvasElement, options: ChartOptions);
        render(): void;
        update(data: number[]): void;
        destroy(): void;
    }
    
    export = Chart;
}

// Global variable declarations
declare var MyGlobalLib: {
    version: string;
    init(config: { apiKey: string }): void;
    destroy(): void;
};
```

---

## 9. Migrating React JavaScript to TypeScript

### ขั้นตอนการ Migrate React App

```bash
# ติดตั้ง type definitions สำหรับ React
npm install --save-dev @types/react @types/react-dom

# Rename .jsx เป็น .tsx, .js เป็น .ts
# (หรือทำทีละ file)
```

### React Component Migration

```javascript
// ก่อน: JavaScript React Component
// UserCard.jsx
import React, { useState, useEffect } from "react";
import PropTypes from "prop-types";

function UserCard({ userId, onSelect }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        fetchUser(userId).then(data => {
            setUser(data);
            setLoading(false);
        });
    }, [userId]);
    
    if (loading) return <div>Loading...</div>;
    if (!user) return <div>User not found</div>;
    
    return (
        <div className="user-card" onClick={() => onSelect(user)}>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
}

UserCard.propTypes = {
    userId: PropTypes.string.isRequired,
    onSelect: PropTypes.func.isRequired,
};

export default UserCard;
```

```typescript
// หลัง: TypeScript React Component
// UserCard.tsx
import React, { useState, useEffect } from "react";

interface User {
    id: string;
    name: string;
    email: string;
    avatar?: string;
}

interface UserCardProps {
    userId: string;
    onSelect: (user: User) => void;
    className?: string;
}

async function fetchUser(id: string): Promise<User | null> {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) return null;
    return response.json() as Promise<User>;
}

const UserCard: React.FC<UserCardProps> = ({ userId, onSelect, className }) => {
    const [user, setUser] = useState<User | null>(null);
    const [loading, setLoading] = useState<boolean>(true);
    const [error, setError] = useState<string | null>(null);
    
    useEffect(() => {
        let cancelled = false;
        
        setLoading(true);
        fetchUser(userId).then(data => {
            if (!cancelled) {
                setUser(data);
                if (!data) setError("User not found");
                setLoading(false);
            }
        }).catch(err => {
            if (!cancelled) {
                setError(err.message);
                setLoading(false);
            }
        });
        
        return () => { cancelled = true; };
    }, [userId]);
    
    if (loading) return <div className="loading">Loading...</div>;
    if (error || !user) return <div className="error">{error ?? "User not found"}</div>;
    
    return (
        <div
            className={`user-card ${className ?? ""}`}
            onClick={() => onSelect(user)}
        >
            {user.avatar && <img src={user.avatar} alt={user.name} />}
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
};

export default UserCard;
export type { User, UserCardProps };
```

### React Hooks Migration

```javascript
// ก่อน: Custom hook ใน JavaScript
function useLocalStorage(key, initialValue) {
    const [storedValue, setStoredValue] = useState(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            return initialValue;
        }
    });
    
    const setValue = (value) => {
        try {
            const valueToStore = value instanceof Function ? value(storedValue) : value;
            setStoredValue(valueToStore);
            window.localStorage.setItem(key, JSON.stringify(valueToStore));
        } catch (error) {
            console.error(error);
        }
    };
    
    return [storedValue, setValue];
}
```

```typescript
// หลัง: Custom hook ใน TypeScript
import { useState, Dispatch, SetStateAction } from "react";

type UseLocalStorageReturn<T> = [T, Dispatch<SetStateAction<T>>];

function useLocalStorage<T>(
    key: string,
    initialValue: T
): UseLocalStorageReturn<T> {
    const [storedValue, setStoredValue] = useState<T>(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? (JSON.parse(item) as T) : initialValue;
        } catch {
            return initialValue;
        }
    });
    
    const setValue: Dispatch<SetStateAction<T>> = (value) => {
        try {
            const valueToStore = value instanceof Function 
                ? value(storedValue) 
                : value;
            setStoredValue(valueToStore);
            window.localStorage.setItem(key, JSON.stringify(valueToStore));
        } catch (error) {
            console.error("Failed to save to localStorage:", error);
        }
    };
    
    return [storedValue, setValue];
}

// Type-safe usage
const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
const [count, setCount] = useLocalStorage<number>("count", 0);
```

---

## 10. Migrating Node.js to TypeScript

### Express App Migration

```javascript
// ก่อน: Express app ใน JavaScript
// app.js
const express = require("express");
const app = express();

app.use(express.json());

app.get("/users", async (req, res) => {
    try {
        const users = await db.query("SELECT * FROM users");
        res.json(users);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

app.post("/users", async (req, res) => {
    const { name, email } = req.body;
    try {
        const user = await db.query(
            "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
            [name, email]
        );
        res.status(201).json(user);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

```typescript
// หลัง: Express app ใน TypeScript
// app.ts
import express, { Application, Request, Response, NextFunction } from "express";

interface User {
    id: number;
    name: string;
    email: string;
    createdAt: Date;
}

interface CreateUserBody {
    name: string;
    email: string;
}

interface ApiErrorResponse {
    error: string;
    code?: string;
}

class AppError extends Error {
    constructor(
        message: string,
        public readonly statusCode: number = 500,
        public readonly code?: string
    ) {
        super(message);
        this.name = "AppError";
    }
}

function createApp(): Application {
    const app = express();
    app.use(express.json());
    
    // GET /users
    app.get("/users", async (
        _req: Request,
        res: Response<User[]>,
        next: NextFunction
    ) => {
        try {
            const users = await db.query<User>("SELECT * FROM users");
            res.json(users);
        } catch (error) {
            next(error);
        }
    });
    
    // POST /users
    app.post("/users", async (
        req: Request<{}, User, CreateUserBody>,
        res: Response<User>,
        next: NextFunction
    ) => {
        const { name, email } = req.body;
        
        if (!name || !email) {
            return next(new AppError("Name and email are required", 400, "VALIDATION_ERROR"));
        }
        
        try {
            const [user] = await db.query<User>(
                "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
                [name, email]
            );
            res.status(201).json(user);
        } catch (error) {
            next(error);
        }
    });
    
    // Error handler
    app.use((
        err: Error,
        _req: Request,
        res: Response<ApiErrorResponse>,
        _next: NextFunction
    ) => {
        if (err instanceof AppError) {
            return res.status(err.statusCode).json({
                error: err.message,
                code: err.code,
            });
        }
        
        console.error("Unhandled error:", err);
        res.status(500).json({ error: "Internal Server Error" });
    });
    
    return app;
}

const app = createApp();
app.listen(3000, () => console.log("Server running on port 3000"));

export { createApp };
```

### Environment Configuration

```typescript
// config/environment.ts
import { z } from "zod";

const envSchema = z.object({
    NODE_ENV: z.enum(["development", "staging", "production"]).default("development"),
    PORT: z.string().transform(Number).default("3000"),
    DATABASE_URL: z.string().url(),
    JWT_SECRET: z.string().min(32),
    REDIS_URL: z.string().url().optional(),
    LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

type Environment = z.infer<typeof envSchema>;

let _env: Environment;

export function getEnv(): Environment {
    if (!_env) {
        const result = envSchema.safeParse(process.env);
        if (!result.success) {
            console.error("Invalid environment variables:");
            result.error.errors.forEach(err => {
                console.error(`  ${err.path.join(".")}: ${err.message}`);
            });
            process.exit(1);
        }
        _env = result.data;
    }
    return _env;
}

// การใช้งาน
const env = getEnv();
console.log(env.PORT);       // number (ไม่ใช่ string)
console.log(env.NODE_ENV);   // "development" | "staging" | "production"
```

---

## 11. Common Migration Pitfalls

### Pitfall 1: Object Destructuring กับ Optional Properties

```typescript
// ❌ ปัญหา: destructuring ที่อาจเป็น undefined
function processRequest(req: Express.Request): void {
    const { user } = req; // user อาจเป็น undefined
    console.log(user.name); // ❌ Object is possibly 'undefined'
}

// ✅ แก้ไข: ตรวจสอบก่อน
function processRequest(req: Express.Request): void {
    if (!req.user) {
        throw new Error("Authentication required");
    }
    const { user } = req;
    console.log(user.name); // ✅
}
```

### Pitfall 2: Array Methods ที่ Return undefined

```typescript
// ❌ ปัญหา
const users: User[] = [];
const firstUser = users.find(u => u.id === "1");
console.log(firstUser.name); // ❌ Object is possibly 'undefined'

// ✅ แก้ไข
const firstUser = users.find(u => u.id === "1");
if (firstUser) {
    console.log(firstUser.name); // ✅
}

// หรือ
const firstUser = users.find(u => u.id === "1");
console.log(firstUser?.name); // ✅ Optional chaining
```

### Pitfall 3: Dynamic Object Keys

```typescript
// ❌ ปัญหา
const config = {};
config["key"] = "value"; // ❌ Element implicitly has an 'any' type

// ✅ แก้ไข
const config: Record<string, string> = {};
config["key"] = "value"; // ✅

// หรือ
const config: { [key: string]: string } = {};
config["key"] = "value"; // ✅
```

### Pitfall 4: Event Listeners

```typescript
// ❌ ปัญหา
document.addEventListener("click", function(event) {
    console.log(event.clientX); // อาจไม่มี type info ถ้า target ไม่ถูกต้อง
});

// ✅ แก้ไข
document.addEventListener("click", (event: MouseEvent) => {
    console.log(event.clientX); // ✅
});

// สำหรับ custom events
interface CustomEventDetail {
    userId: string;
    action: string;
}

document.addEventListener("userAction", (event: Event) => {
    const customEvent = event as CustomEvent<CustomEventDetail>;
    console.log(customEvent.detail.userId); // ✅
});
```

### Pitfall 5: Promise ที่ไม่ได้ await

```typescript
// ❌ ปัญหา
class UserService {
    users: User[] = [];
    
    async init(): Promise<void> {
        this.users = this.loadUsers(); // ❌ ไม่ได้ await
    }
    
    async loadUsers(): Promise<User[]> {
        return fetch("/api/users").then(r => r.json());
    }
}

// ✅ แก้ไข
class UserService {
    private users: User[] = [];
    
    async init(): Promise<void> {
        this.users = await this.loadUsers(); // ✅
    }
    
    private async loadUsers(): Promise<User[]> {
        const response = await fetch("/api/users");
        return response.json() as Promise<User[]>;
    }
}
```

---

## 12. Automation Tools

### ts-migrate

```bash
# ติดตั้ง ts-migrate
npm install --global @ts-migrate/ts-migrate

# migrate project
ts-migrate migrate path/to/project

# migrate ทีละ file
ts-migrate migrate-all path/to/file.js
```

### Codemod scripts

```typescript
// scripts/js-to-ts.ts
// Script สำหรับ rename .js เป็น .ts
import * as fs from "fs";
import * as path from "path";
import * as glob from "glob";

function renameJsToTs(srcDir: string): void {
    const jsFiles = glob.sync(`${srcDir}/**/*.js`, {
        ignore: ["**/node_modules/**", "**/dist/**"],
    });
    
    let renamed = 0;
    let skipped = 0;
    
    for (const jsFile of jsFiles) {
        const tsFile = jsFile.replace(/\.js$/, ".ts");
        
        // ตรวจสอบว่ามี .ts file อยู่แล้วหรือไม่
        if (fs.existsSync(tsFile)) {
            console.log(`Skipped (already exists): ${tsFile}`);
            skipped++;
            continue;
        }
        
        fs.renameSync(jsFile, tsFile);
        console.log(`Renamed: ${jsFile} -> ${tsFile}`);
        renamed++;
    }
    
    console.log(`\nDone! Renamed: ${renamed}, Skipped: ${skipped}`);
}

// เรียกใช้
renameJsToTs("./src");
```

### ESLint TypeScript Config

```json
// .eslintrc.json
{
    "extends": [
        "eslint:recommended",
        "@typescript-eslint/recommended",
        "@typescript-eslint/recommended-requiring-type-checking"
    ],
    "parser": "@typescript-eslint/parser",
    "parserOptions": {
        "project": "./tsconfig.json"
    },
    "plugins": ["@typescript-eslint"],
    "rules": {
        "@typescript-eslint/no-explicit-any": "warn",
        "@typescript-eslint/no-unsafe-assignment": "warn",
        "@typescript-eslint/no-unsafe-call": "warn",
        "@typescript-eslint/no-unsafe-member-access": "warn",
        "@typescript-eslint/explicit-function-return-type": "warn",
        "@typescript-eslint/no-unused-vars": "error",
        "@typescript-eslint/prefer-nullish-coalescing": "warn",
        "@typescript-eslint/prefer-optional-chain": "warn"
    }
}
```

---

## 13. Migration Examples: Before & After

### Example 1: Event Emitter Pattern

```javascript
// ก่อน: JavaScript
const EventEmitter = require("events");

class AppEmitter extends EventEmitter {
    emitLogin(userId) {
        this.emit("login", { userId, timestamp: new Date() });
    }
    
    onLogin(handler) {
        this.on("login", handler);
    }
}

const emitter = new AppEmitter();
emitter.onLogin(({ userId }) => {
    console.log(`User ${userId} logged in`);
});
```

```typescript
// หลัง: TypeScript
import { EventEmitter } from "events";

interface AppEvents {
    login: [{ userId: string; timestamp: Date }];
    logout: [{ userId: string }];
    error: [{ message: string; code: number }];
}

class TypedAppEmitter extends EventEmitter {
    emit<K extends keyof AppEvents>(event: K, ...args: AppEvents[K]): boolean {
        return super.emit(event as string, ...args);
    }
    
    on<K extends keyof AppEvents>(
        event: K,
        listener: (...args: AppEvents[K]) => void
    ): this {
        return super.on(event as string, listener as any);
    }
    
    emitLogin(userId: string): void {
        this.emit("login", { userId, timestamp: new Date() });
    }
    
    onLogin(handler: (payload: { userId: string; timestamp: Date }) => void): void {
        this.on("login", handler);
    }
}

const emitter = new TypedAppEmitter();
emitter.onLogin(({ userId, timestamp }) => {
    // TypeScript รู้ว่า userId เป็น string
    console.log(`User ${userId} logged in at ${timestamp.toISOString()}`);
});
```

### Example 2: Middleware Pattern

```javascript
// ก่อน: Express Middleware ใน JavaScript
function authMiddleware(req, res, next) {
    const token = req.headers.authorization;
    if (!token) {
        return res.status(401).json({ error: "No token provided" });
    }
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: "Invalid token" });
    }
}
```

```typescript
// หลัง: Express Middleware ใน TypeScript
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

interface JwtPayload {
    userId: string;
    email: string;
    role: "admin" | "user";
    iat: number;
    exp: number;
}

// Extend Express Request type
declare global {
    namespace Express {
        interface Request {
            user?: JwtPayload;
        }
    }
}

function authMiddleware(
    req: Request,
    res: Response,
    next: NextFunction
): void {
    const authHeader = req.headers.authorization;
    
    if (!authHeader?.startsWith("Bearer ")) {
        res.status(401).json({ error: "No token provided" });
        return;
    }
    
    const token = authHeader.slice(7);
    
    try {
        const decoded = jwt.verify(
            token,
            process.env.JWT_SECRET as string
        ) as JwtPayload;
        
        req.user = decoded;
        next();
    } catch (error) {
        if (error instanceof jwt.TokenExpiredError) {
            res.status(401).json({ error: "Token expired" });
        } else if (error instanceof jwt.JsonWebTokenError) {
            res.status(401).json({ error: "Invalid token" });
        } else {
            res.status(500).json({ error: "Authentication error" });
        }
    }
}

export { authMiddleware };
```

---

## 14. Testing Migration

```typescript
// ก่อน: Jest tests ใน JavaScript
const { UserService } = require("../services/user");
const { db } = require("../db");

jest.mock("../db");

describe("UserService", () => {
    let service;
    
    beforeEach(() => {
        service = new UserService(db);
        jest.clearAllMocks();
    });
    
    test("creates a user", async () => {
        db.query.mockResolvedValue([{ id: "1", name: "Alice" }]);
        
        const user = await service.createUser({ name: "Alice", email: "a@b.com" });
        
        expect(user.id).toBe("1");
        expect(db.query).toHaveBeenCalledWith(
            expect.stringContaining("INSERT"),
            expect.arrayContaining(["Alice"])
        );
    });
});
```

```typescript
// หลัง: Jest tests ใน TypeScript
import { UserService } from "../services/user";
import { Database } from "../db";

interface User {
    id: string;
    name: string;
    email: string;
}

// Type-safe mock
const mockDb: jest.Mocked<Pick<Database, "query">> = {
    query: jest.fn(),
};

describe("UserService", () => {
    let service: UserService;
    
    beforeEach(() => {
        service = new UserService(mockDb as unknown as Database);
        jest.clearAllMocks();
    });
    
    test("creates a user successfully", async () => {
        const expectedUser: User = {
            id: "1",
            name: "Alice",
            email: "alice@example.com"
        };
        
        mockDb.query.mockResolvedValue([expectedUser]);
        
        const user = await service.createUser({
            name: "Alice",
            email: "alice@example.com"
        });
        
        expect(user.id).toBe("1");
        expect(user.name).toBe("Alice");
        expect(mockDb.query).toHaveBeenCalledWith(
            expect.stringContaining("INSERT"),
            expect.arrayContaining(["Alice", "alice@example.com"])
        );
    });
    
    test("throws error when email already exists", async () => {
        mockDb.query.mockRejectedValue(
            new Error("duplicate key value violates unique constraint")
        );
        
        await expect(service.createUser({
            name: "Bob",
            email: "alice@example.com" // duplicate
        })).rejects.toThrow("Email already in use");
    });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การ migrate JavaScript ไปสู่ TypeScript:

1. **Migration Strategy** - วางแผน migrate อย่างมีระบบ
2. **allowJs** - เปิดใช้ TypeScript สำหรับ JavaScript files
3. **checkJs** - เปิด type checking ใน JavaScript
4. **Gradual Migration** - ค่อยๆ migrate ทีละส่วน
5. **Adding Types** - เพิ่ม types อย่างค่อยเป็นค่อยไป
6. **Third-party Types** - จัดการ @types packages
7. **@ts-ignore vs @ts-expect-error** - ใช้อย่างถูกต้อง
8. **Declaration Files** - สร้าง .d.ts สำหรับ legacy code
9. **React Migration** - แปลง React components
10. **Node.js Migration** - แปลง Express apps
11. **Common Pitfalls** - หลีกเลี่ยงข้อผิดพลาดที่พบบ่อย
12. **Automation** - ใช้ tools ช่วย migrate
13. **Before/After** - ตัวอย่าง migration จริง

การ migrate ที่ดีต้องอาศัยความอดทนและวางแผนอย่างรอบคอบ เริ่มจากส่วนที่ง่ายและค่อยๆ เพิ่ม strictness ขึ้นไปเรื่อยๆ
