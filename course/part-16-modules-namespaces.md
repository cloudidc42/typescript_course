# ตอนที่ 16: Modules และ Namespaces ใน TypeScript

## บทนำ

Modules และ Namespaces เป็นกลไกสำคัญในการจัดระเบียบโค้ด TypeScript ให้มีโครงสร้างที่ดี ช่วยแยกส่วนการทำงาน ลดการชนกันของชื่อตัวแปร และทำให้โค้ดนำกลับมาใช้ใหม่ได้ง่าย

---

## 1. ES Modules ใน TypeScript

TypeScript รองรับ ES Modules มาตรฐาน ซึ่งใช้ `import` และ `export` ในการแบ่งโค้ดออกเป็นไฟล์ย่อย

### 1.1 การ Export พื้นฐาน

```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export function multiply(a: number, b: number): number {
  return a * b;
}

export function divide(a: number, b: number): number {
  if (b === 0) {
    throw new Error("ไม่สามารถหารด้วยศูนย์ได้");
  }
  return a / b;
}

export const PI = 3.14159265358979;
export const E = 2.71828182845905;
```

### 1.2 การ Import พื้นฐาน

```typescript
// main.ts
import { add, subtract, multiply, divide, PI } from './math';

console.log(add(5, 3));        // 8
console.log(subtract(10, 4));  // 6
console.log(multiply(3, 7));   // 21
console.log(divide(15, 3));    // 5
console.log(PI);               // 3.14159265358979
```

### 1.3 Export Interface และ Type

```typescript
// types.ts
export interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  createdAt: Date;
}

export interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

export type Status = 'active' | 'inactive' | 'pending' | 'deleted';

export type ApiResponse<T> = {
  data: T;
  success: boolean;
  message: string;
  timestamp: Date;
};
```

### 1.4 Export Class

```typescript
// services/UserService.ts
import { User, Status, ApiResponse } from '../types';

export class UserService {
  private users: User[] = [];

  addUser(user: User): void {
    this.users.push(user);
  }

  getUserById(id: number): User | undefined {
    return this.users.find(u => u.id === id);
  }

  getAllUsers(): User[] {
    return [...this.users];
  }

  updateUser(id: number, updates: Partial<User>): User | null {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) return null;
    
    this.users[index] = { ...this.users[index], ...updates };
    return this.users[index];
  }

  deleteUser(id: number): boolean {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) return false;
    
    this.users.splice(index, 1);
    return true;
  }
}
```

---

## 2. Default vs Named Exports

### 2.1 Default Export

Default export ใช้เมื่อไฟล์มีสิ่งที่ต้องการ export หลักเพียงอย่างเดียว

```typescript
// Calculator.ts
export default class Calculator {
  private history: string[] = [];

  add(a: number, b: number): number {
    const result = a + b;
    this.history.push(`${a} + ${b} = ${result}`);
    return result;
  }

  subtract(a: number, b: number): number {
    const result = a - b;
    this.history.push(`${a} - ${b} = ${result}`);
    return result;
  }

  multiply(a: number, b: number): number {
    const result = a * b;
    this.history.push(`${a} × ${b} = ${result}`);
    return result;
  }

  divide(a: number, b: number): number {
    if (b === 0) throw new Error("หารด้วยศูนย์ไม่ได้");
    const result = a / b;
    this.history.push(`${a} ÷ ${b} = ${result}`);
    return result;
  }

  getHistory(): string[] {
    return [...this.history];
  }

  clearHistory(): void {
    this.history = [];
  }
}
```

```typescript
// การ import default export
import Calculator from './Calculator';
// หรือตั้งชื่ออะไรก็ได้
import Calc from './Calculator';
import MyCalculator from './Calculator';

const calc = new Calculator();
console.log(calc.add(5, 3)); // 8
```

### 2.2 Named Export vs Default Export

```typescript
// utils.ts
// Named exports
export function formatDate(date: Date): string {
  return date.toLocaleDateString('th-TH');
}

export function formatCurrency(amount: number, currency: string = 'THB'): string {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency
  }).format(amount);
}

export function capitalize(str: string): string {
  return str.charAt(0).toUpperCase() + str.slice(1).toLowerCase();
}

export function slugify(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
}

// Default export
export default {
  formatDate,
  formatCurrency,
  capitalize,
  slugify
};
```

```typescript
// การใช้งาน
import utils, { formatDate, formatCurrency } from './utils';

// ใช้ default export
console.log(utils.capitalize("hello world"));

// ใช้ named exports
console.log(formatDate(new Date()));
console.log(formatCurrency(1500));
```

### 2.3 ข้อควรระวัง Default Export

```typescript
// ไม่แนะนำ - ยากต่อการ refactor
export default function() {
  return "anonymous function";
}

// แนะนำ - มีชื่อที่ชัดเจน
export default function greet(name: string): string {
  return `สวัสดี ${name}!`;
}
```

---

## 3. Re-exports

Re-export ช่วยสร้าง public API ที่สะอาดสำหรับ module

### 3.1 Re-export พื้นฐาน

```typescript
// models/index.ts - barrel file
export { User } from './User';
export { Product } from './Product';
export { Order } from './Order';
export { Category } from './Category';
export { Review } from './Review';
```

```typescript
// การใช้งาน
import { User, Product, Order } from './models';
// แทนที่จะต้อง
import { User } from './models/User';
import { Product } from './models/Product';
import { Order } from './models/Order';
```

### 3.2 Re-export พร้อมเปลี่ยนชื่อ

```typescript
// api/index.ts
export { get as apiGet, post as apiPost, put as apiPut } from './http';
export { UserService as Users } from './services/UserService';
export { ProductService as Products } from './services/ProductService';
```

### 3.3 Re-export ทั้งหมด

```typescript
// helpers/index.ts
export * from './string-helpers';
export * from './number-helpers';
export * from './date-helpers';
export * from './array-helpers';
export * from './object-helpers';
```

### 3.4 Re-export Default

```typescript
// index.ts
export { default as Calculator } from './Calculator';
export { default as Logger } from './Logger';
export { default } from './App'; // re-export default as default
```

### 3.5 ตัวอย่างการสร้าง Library Structure

```typescript
// lib/validators/email.ts
export function isValidEmail(email: string): boolean {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

// lib/validators/phone.ts
export function isValidThaiPhone(phone: string): boolean {
  const regex = /^(0[689][0-9]{8})$/;
  return regex.test(phone);
}

// lib/validators/index.ts
export * from './email';
export * from './phone';
export * from './password';
export * from './url';

// lib/index.ts
export * from './validators';
export * from './formatters';
export * from './parsers';
```

---

## 4. Module Resolution Strategies

### 4.1 Node Module Resolution

TypeScript ค้นหา module ตามลำดับ:

```
1. ./module.ts
2. ./module.tsx
3. ./module.d.ts
4. ./module/package.json (main field)
5. ./module/index.ts
6. ./module/index.tsx
7. ./module/index.d.ts
```

### 4.2 tsconfig.json สำหรับ Module Resolution

```json
{
  "compilerOptions": {
    "moduleResolution": "node",
    "module": "commonjs",
    "target": "ES2020",
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"],
      "@components/*": ["./components/*"],
      "@services/*": ["./services/*"],
      "@utils/*": ["./utils/*"],
      "@types/*": ["./types/*"]
    }
  }
}
```

### 4.3 Module Resolution แบบ Classic

```typescript
// TypeScript classic resolution
// import { x } from "moduleA"
// ค้นหาใน:
// 1. ./moduleA.ts
// 2. ./moduleA.tsx
// 3. ./moduleA.d.ts
// 4. ../moduleA.ts
// 5. ...ขึ้นไปเรื่อยๆ
```

### 4.4 Modern Module Resolution (Node16/NodeNext)

```json
{
  "compilerOptions": {
    "moduleResolution": "node16",
    "module": "node16"
  }
}
```

```typescript
// ต้องระบุ extension
import { helper } from './helper.js'; // ใช้ .js แม้จะเป็นไฟล์ .ts
```

---

## 5. Namespace Imports

### 5.1 Import ทั้งหมดเป็น Namespace

```typescript
// math.ts
export function add(a: number, b: number): number { return a + b; }
export function subtract(a: number, b: number): number { return a - b; }
export function multiply(a: number, b: number): number { return a * b; }
export const PI = 3.14159;

// main.ts
import * as Math from './math';

console.log(Math.add(1, 2));      // 3
console.log(Math.PI);              // 3.14159
console.log(Math.multiply(4, 5)); // 20
```

### 5.2 Namespace import กับ Third-party Libraries

```typescript
import * as _ from 'lodash';
import * as moment from 'moment';
import * as path from 'path';
import * as fs from 'fs';

// ใช้งาน
const numbers = [1, 2, 3, 4, 5];
console.log(_.sum(numbers));      // 15
console.log(_.max(numbers));      // 5
console.log(_.chunk(numbers, 2)); // [[1, 2], [3, 4], [5]]

const date = moment().format('DD/MM/YYYY');
const filePath = path.join(__dirname, 'data', 'users.json');
```

---

## 6. TypeScript Namespaces (Internal Modules)

Namespaces ใน TypeScript เป็น namespace สำหรับจัดกลุ่มโค้ดภายใน

### 6.1 การสร้าง Namespace

```typescript
namespace Validation {
  export interface StringValidator {
    isAcceptable(s: string): boolean;
  }

  const lettersRegexp = /^[A-Za-a]+$/;
  const numberRegexp = /^[0-9]+$/;

  export class LettersOnlyValidator implements StringValidator {
    isAcceptable(s: string): boolean {
      return lettersRegexp.test(s);
    }
  }

  export class ZipCodeValidator implements StringValidator {
    isAcceptable(s: string): boolean {
      return s.length === 5 && numberRegexp.test(s);
    }
  }
}

// การใช้งาน
let validators: { [s: string]: Validation.StringValidator } = {};
validators["ZIP code"] = new Validation.ZipCodeValidator();
validators["Letters only"] = new Validation.LettersOnlyValidator();
```

### 6.2 Nested Namespaces

```typescript
namespace App {
  export namespace Models {
    export interface User {
      id: number;
      name: string;
      email: string;
    }

    export interface Product {
      id: number;
      name: string;
      price: number;
    }
  }

  export namespace Services {
    export class UserService {
      private users: Models.User[] = [];

      getAll(): Models.User[] {
        return this.users;
      }
    }

    export class ProductService {
      private products: Models.Product[] = [];

      getAll(): Models.Product[] {
        return this.products;
      }
    }
  }

  export namespace Utils {
    export function formatCurrency(amount: number): string {
      return `฿${amount.toLocaleString()}`;
    }

    export function generateId(): string {
      return Math.random().toString(36).substr(2, 9);
    }
  }
}

// การใช้งาน
const userService = new App.Services.UserService();
const price = App.Utils.formatCurrency(1500);
```

### 6.3 Namespace ข้ามไฟล์ด้วย Triple-Slash

```typescript
// validators.ts
namespace Validators {
  export interface Validator {
    validate(value: string): boolean;
  }
}

// email-validator.ts
/// <reference path="validators.ts" />
namespace Validators {
  export class EmailValidator implements Validator {
    validate(email: string): boolean {
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    }
  }
}
```

---

## 7. Dynamic Imports

### 7.1 Dynamic Import พื้นฐาน

```typescript
// การ import แบบ dynamic
async function loadModule() {
  const math = await import('./math');
  console.log(math.add(5, 3));
}

// หรือแบบ then
import('./math').then(math => {
  console.log(math.multiply(4, 6));
});
```

### 7.2 Dynamic Import พร้อม Type

```typescript
async function loadPlugin(pluginName: string) {
  const plugin = await import(`./plugins/${pluginName}`);
  return plugin.default;
}

// Code splitting ใน React (ตัวอย่าง)
async function loadComponent(name: string) {
  const { default: Component } = await import(`./components/${name}`);
  return Component;
}
```

### 7.3 Conditional Dynamic Import

```typescript
async function initializeApp(config: { useAdvancedFeatures: boolean }) {
  const core = await import('./core');
  
  if (config.useAdvancedFeatures) {
    const { AdvancedFeatures } = await import('./advanced');
    core.registerPlugin(new AdvancedFeatures());
  }
  
  return core;
}
```

### 7.4 Dynamic Import กับ Error Handling

```typescript
async function safeImport<T>(modulePath: string): Promise<T | null> {
  try {
    const module = await import(modulePath);
    return module.default || module;
  } catch (error) {
    console.error(`ไม่สามารถโหลด module ${modulePath}:`, error);
    return null;
  }
}

// การใช้งาน
const logger = await safeImport<Logger>('./logger');
if (logger) {
  logger.log('Initialized');
}
```

### 7.5 Lazy Loading Pattern

```typescript
class LazyLoader<T> {
  private loaded: T | null = null;
  private loading: Promise<T> | null = null;

  constructor(private readonly importFn: () => Promise<{ default: T }>) {}

  async get(): Promise<T> {
    if (this.loaded) return this.loaded;
    
    if (!this.loading) {
      this.loading = this.importFn().then(module => {
        this.loaded = module.default;
        return this.loaded;
      });
    }
    
    return this.loading;
  }
}

// การใช้งาน
const heavyModule = new LazyLoader(() => import('./heavy-computation'));

async function doWork() {
  const module = await heavyModule.get();
  return module.compute();
}
```

---

## 8. Type-Only Imports

TypeScript 3.8+ รองรับ type-only imports เพื่อแยกการ import types ออกจาก runtime code

### 8.1 Type-Only Import พื้นฐาน

```typescript
// ก่อน TypeScript 3.8
import { User, UserService } from './user';

// TypeScript 3.8+
import type { User } from './user';
import { UserService } from './user';

// หรือ mixed
import { UserService, type User } from './user';
```

### 8.2 ประโยชน์ของ Type-Only Import

```typescript
// types.ts
export interface Config {
  apiUrl: string;
  timeout: number;
  retryCount: number;
}

export type Environment = 'development' | 'staging' | 'production';

export interface Logger {
  log(message: string): void;
  error(message: string, error?: Error): void;
  warn(message: string): void;
}

// app.ts
import type { Config, Environment, Logger } from './types';
// TypeScript จะตรวจสอบว่า type เหล่านี้ถูกใช้แค่เป็น type เท่านั้น
// ไม่ใช่ runtime value
// ทำให้ bundle size เล็กลง

function createApp(config: Config, env: Environment, logger: Logger) {
  logger.log(`Starting app in ${env} mode`);
  return { config, env };
}
```

### 8.3 Type-Only Export

```typescript
// models.ts
export interface UserModel {
  id: number;
  name: string;
}

export type UserRole = 'admin' | 'user' | 'guest';

// index.ts - re-export เฉพาะ types
export type { UserModel, UserRole } from './models';
```

### 8.4 การใช้กับ Enum

```typescript
// ระวัง: const enum จะถูก inlined
export const enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT"
}

// ใช้ type import กับ regular enum ได้
export enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE"
}

// ไม่ได้ - จะ error
import type { Status } from './enums';
const s = Status.Active; // Error: Status ถูก import เป็น type เท่านั้น

// ถูก - import เป็น value
import { Status } from './enums';
const s = Status.Active; // OK
```

---

## 9. Declaration Merging กับ Modules

### 9.1 Module Augmentation

```typescript
// เพิ่ม method ให้กับ existing module
// original: express.ts (library)
// declare module 'express' { ... }

// augmentation.ts
import 'express';

declare module 'express' {
  interface Request {
    user?: {
      id: number;
      name: string;
      role: string;
    };
    requestId?: string;
  }

  interface Response {
    success(data: any): void;
    fail(message: string, statusCode?: number): void;
  }
}
```

### 9.2 การใช้ Module Augmentation

```typescript
// middleware.ts
import express, { Request, Response, NextFunction } from 'express';

const app = express();

// เพิ่ม middleware ที่เติม custom properties
app.use((req: Request, res: Response, next: NextFunction) => {
  req.requestId = Math.random().toString(36).substr(2, 9);
  next();
});

// ใช้ custom response methods
app.use((req: Request, res: Response, next: NextFunction) => {
  res.success = (data: any) => {
    res.json({ success: true, data });
  };
  res.fail = (message: string, statusCode: number = 400) => {
    res.status(statusCode).json({ success: false, message });
  };
  next();
});

app.get('/users', (req: Request, res: Response) => {
  // TypeScript รู้ว่า req.requestId มีอยู่
  console.log(req.requestId);
  res.success({ users: [] }); // TypeScript รู้ว่า res.success มีอยู่
});
```

### 9.3 Merging Interfaces ข้าม Module

```typescript
// plugin-system.ts
export interface PluginOptions {
  name: string;
}

// my-plugin.ts
import { PluginOptions } from './plugin-system';

declare module './plugin-system' {
  interface PluginOptions {
    version: string;
    author: string;
  }
}

// ตอนนี้ PluginOptions มี name, version, และ author
const options: PluginOptions = {
  name: 'my-plugin',
  version: '1.0.0',
  author: 'developer'
};
```

---

## 10. Ambient Modules และ @types

### 10.1 Ambient Module Declarations

```typescript
// declarations.d.ts
declare module '*.png' {
  const content: string;
  export default content;
}

declare module '*.svg' {
  const content: React.FunctionComponent<React.SVGAttributes<SVGElement>>;
  export default content;
}

declare module '*.json' {
  const value: any;
  export default value;
}

declare module '*.css' {
  const styles: { [className: string]: string };
  export default styles;
}
```

### 10.2 Ambient Module สำหรับ Global Libraries

```typescript
// global.d.ts
declare global {
  interface Window {
    __APP_CONFIG__: {
      apiUrl: string;
      version: string;
      environment: string;
    };
    gtag: (command: string, ...args: any[]) => void;
  }

  const __DEV__: boolean;
  const __VERSION__: string;
}

export {}; // ต้องมี export เพื่อทำให้เป็น module
```

### 10.3 @types Packages

```bash
# ติดตั้ง type definitions
npm install --save-dev @types/node
npm install --save-dev @types/express
npm install --save-dev @types/lodash
npm install --save-dev @types/jest
npm install --save-dev @types/react
npm install --save-dev @types/react-dom
```

### 10.4 การสร้าง Custom Type Declarations

```typescript
// vendor.d.ts - สำหรับ library ที่ไม่มี types
declare module 'untyped-library' {
  export function doSomething(value: string): number;
  export function processData(data: any[]): any;
  
  export interface Options {
    timeout: number;
    retry: number;
  }
  
  export default class Client {
    constructor(options: Options);
    connect(): Promise<void>;
    disconnect(): void;
    send(data: any): Promise<any>;
  }
}
```

### 10.5 Wildcard Module Declarations

```typescript
// *.module.css
declare module '*.module.css' {
  const classes: Readonly<Record<string, string>>;
  export default classes;
}

// *.graphql
declare module '*.graphql' {
  import { DocumentNode } from 'graphql';
  const schema: DocumentNode;
  export default schema;
}

// *.wasm
declare module '*.wasm' {
  const wasmModule: WebAssembly.Module;
  export default wasmModule;
}
```

---

## 11. Path Aliases กับ tsconfig

### 11.1 การตั้งค่า Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"],
      "@components/*": ["./components/*"],
      "@pages/*": ["./pages/*"],
      "@services/*": ["./services/*"],
      "@utils/*": ["./utils/*"],
      "@hooks/*": ["./hooks/*"],
      "@types/*": ["./types/*"],
      "@assets/*": ["./assets/*"],
      "@config": ["./config/index.ts"],
      "@constants": ["./constants/index.ts"]
    }
  }
}
```

### 11.2 การใช้งาน Path Aliases

```typescript
// แทนที่จะใช้
import { Button } from '../../../components/ui/Button';
import { useAuth } from '../../hooks/useAuth';
import { formatDate } from '../../../utils/date';

// ใช้แบบนี้แทน
import { Button } from '@components/ui/Button';
import { useAuth } from '@hooks/useAuth';
import { formatDate } from '@utils/date';
import { API_URL } from '@constants';
```

### 11.3 Jest Configuration สำหรับ Path Aliases

```javascript
// jest.config.js
module.exports = {
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@types/(.*)$': '<rootDir>/src/types/$1',
  }
};
```

### 11.4 Webpack Configuration

```javascript
// webpack.config.js
const path = require('path');

module.exports = {
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src/'),
      '@components': path.resolve(__dirname, 'src/components/'),
      '@services': path.resolve(__dirname, 'src/services/'),
      '@utils': path.resolve(__dirname, 'src/utils/'),
    },
    extensions: ['.ts', '.tsx', '.js', '.jsx']
  }
};
```

---

## 12. Module Augmentation Advanced

### 12.1 Augmenting Global Libraries

```typescript
// lodash-extensions.d.ts
import 'lodash';

declare module 'lodash' {
  interface LoDashStatic {
    // เพิ่ม custom methods
    formatThaiDate(date: Date): string;
    parseThaiDate(dateStr: string): Date;
  }
}
```

### 12.2 Augmenting Built-in Objects

```typescript
// array-extensions.ts
declare global {
  interface Array<T> {
    groupBy<K extends keyof T>(key: K): Record<string, T[]>;
    unique(): T[];
    flatten<U>(this: U[][]): U[];
    last(): T | undefined;
  }
}

Array.prototype.groupBy = function<T, K extends keyof T>(key: K): Record<string, T[]> {
  return this.reduce((groups: Record<string, T[]>, item: T) => {
    const groupKey = String(item[key]);
    if (!groups[groupKey]) {
      groups[groupKey] = [];
    }
    groups[groupKey].push(item);
    return groups;
  }, {});
};

Array.prototype.unique = function<T>(): T[] {
  return [...new Set(this)];
};

Array.prototype.last = function<T>(): T | undefined {
  return this[this.length - 1];
};

export {};
```

### 12.3 Extending Third-party Types

```typescript
// prisma-extensions.d.ts
import { PrismaClient } from '@prisma/client';

declare module '@prisma/client' {
  interface PrismaClient {
    $transaction<T>(fn: (prisma: PrismaClient) => Promise<T>): Promise<T>;
  }
}
```

---

## 13. Module Patterns ขั้นสูง

### 13.1 Factory Pattern กับ Modules

```typescript
// database.ts
export interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  username: string;
  password: string;
}

export interface Database {
  query<T>(sql: string, params?: any[]): Promise<T[]>;
  execute(sql: string, params?: any[]): Promise<void>;
  transaction<T>(fn: () => Promise<T>): Promise<T>;
  close(): Promise<void>;
}

export function createDatabase(config: DatabaseConfig): Database {
  // Factory function ที่สร้าง database connection
  return {
    async query<T>(sql: string, params?: any[]): Promise<T[]> {
      // implementation
      return [];
    },
    async execute(sql: string, params?: any[]): Promise<void> {
      // implementation
    },
    async transaction<T>(fn: () => Promise<T>): Promise<T> {
      return fn();
    },
    async close(): Promise<void> {
      // implementation
    }
  };
}
```

### 13.2 Plugin Module Pattern

```typescript
// plugin-system.ts
export type Plugin = {
  name: string;
  version: string;
  install(app: Application): void;
  uninstall?(app: Application): void;
};

export class Application {
  private plugins: Map<string, Plugin> = new Map();

  use(plugin: Plugin): this {
    if (this.plugins.has(plugin.name)) {
      console.warn(`Plugin ${plugin.name} already installed`);
      return this;
    }
    plugin.install(this);
    this.plugins.set(plugin.name, plugin);
    return this;
  }

  remove(pluginName: string): this {
    const plugin = this.plugins.get(pluginName);
    if (plugin?.uninstall) {
      plugin.uninstall(this);
    }
    this.plugins.delete(pluginName);
    return this;
  }

  getPlugin(name: string): Plugin | undefined {
    return this.plugins.get(name);
  }
}

// my-plugin.ts
import { Plugin, Application } from './plugin-system';

export const myPlugin: Plugin = {
  name: 'my-plugin',
  version: '1.0.0',
  install(app: Application): void {
    console.log('Plugin installed!');
  },
  uninstall(app: Application): void {
    console.log('Plugin uninstalled!');
  }
};
```

### 13.3 Singleton Module Pattern

```typescript
// config.ts
interface AppConfig {
  apiUrl: string;
  apiVersion: string;
  timeout: number;
  maxRetries: number;
  debugMode: boolean;
}

class ConfigManager {
  private static instance: ConfigManager;
  private config: AppConfig;

  private constructor() {
    this.config = {
      apiUrl: process.env.API_URL || 'https://api.example.com',
      apiVersion: process.env.API_VERSION || 'v1',
      timeout: parseInt(process.env.TIMEOUT || '5000'),
      maxRetries: parseInt(process.env.MAX_RETRIES || '3'),
      debugMode: process.env.NODE_ENV === 'development'
    };
  }

  static getInstance(): ConfigManager {
    if (!ConfigManager.instance) {
      ConfigManager.instance = new ConfigManager();
    }
    return ConfigManager.instance;
  }

  get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    return this.config[key];
  }

  set<K extends keyof AppConfig>(key: K, value: AppConfig[K]): void {
    this.config[key] = value;
  }

  getAll(): Readonly<AppConfig> {
    return { ...this.config };
  }
}

// Export singleton instance
export const config = ConfigManager.getInstance();
export type { AppConfig };
```

---

## 14. ตัวอย่างโปรเจกต์จริง: E-commerce Module Structure

```typescript
// src/modules/products/types.ts
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  stock: number;
  categoryId: string;
  images: string[];
  tags: string[];
  createdAt: Date;
  updatedAt: Date;
}

export interface ProductFilter {
  categoryId?: string;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  tags?: string[];
  search?: string;
}

export interface ProductSort {
  field: 'name' | 'price' | 'createdAt';
  direction: 'asc' | 'desc';
}

export interface PaginatedProducts {
  items: Product[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

```typescript
// src/modules/products/api.ts
import type { Product, ProductFilter, PaginatedProducts } from './types';

const BASE_URL = '/api/products';

export async function getProducts(
  filter?: ProductFilter,
  page: number = 1,
  limit: number = 20
): Promise<PaginatedProducts> {
  const params = new URLSearchParams({
    page: String(page),
    limit: String(limit),
    ...Object.fromEntries(
      Object.entries(filter || {}).filter(([_, v]) => v !== undefined)
    )
  });
  
  const response = await fetch(`${BASE_URL}?${params}`);
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  return response.json();
}

export async function getProductById(id: string): Promise<Product> {
  const response = await fetch(`${BASE_URL}/${id}`);
  if (!response.ok) {
    throw new Error(`Product not found: ${id}`);
  }
  return response.json();
}

export async function createProduct(data: Omit<Product, 'id' | 'createdAt' | 'updatedAt'>): Promise<Product> {
  const response = await fetch(BASE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  if (!response.ok) {
    throw new Error('Failed to create product');
  }
  return response.json();
}
```

```typescript
// src/modules/products/index.ts
export type { Product, ProductFilter, PaginatedProducts } from './types';
export { getProducts, getProductById, createProduct } from './api';
export { ProductService } from './service';
export { productReducer } from './reducer';
export * from './hooks';
```

---

## สรุป

Modules และ Namespaces เป็นเครื่องมือสำคัญในการจัดระเบียบโค้ด TypeScript:

1. **ES Modules** - มาตรฐานสมัยใหม่สำหรับการแบ่ง code
2. **Named vs Default Exports** - เลือกใช้ตามความเหมาะสม
3. **Re-exports** - สร้าง clean public API
4. **Module Resolution** - เข้าใจวิธีที่ TypeScript ค้นหา modules
5. **Dynamic Imports** - โหลด modules แบบ lazy loading
6. **Type-Only Imports** - แยก type imports ออกจาก runtime code
7. **Module Augmentation** - เพิ่มความสามารถให้ existing modules
8. **Path Aliases** - ทำให้ import paths สะอาดขึ้น

การใช้ modules อย่างถูกต้องจะทำให้โค้ดมีโครงสร้างที่ดี บำรุงรักษาง่าย และนำกลับมาใช้ใหม่ได้อย่างมีประสิทธิภาพ
