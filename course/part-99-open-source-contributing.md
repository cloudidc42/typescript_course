# Part 99: การ Contribute ให้กับ Open Source TypeScript

## บทนำ

การมีส่วนร่วมกับ open source เป็นวิธีที่ดีที่สุดในการพัฒนาทักษะ TypeScript และสร้าง network ในวงการ บทนี้จะแนะนำวิธีค้นหาโปรเจกต์ที่เหมาะสม, อ่าน codebase ที่ไม่รู้จัก, เขียน TypeScript-friendly APIs, และทำ PR ที่ได้รับการยอมรับ

---

## 99.1 หาโปรเจกต์ TypeScript ที่เหมาะสม

### เกณฑ์การเลือกโปรเจกต์

```typescript
// เกณฑ์สำหรับประเมินโปรเจกต์
interface ProjectCriteria {
  hasGoodDocumentation: boolean;
  hasContributingGuide: boolean;
  hasTypeScriptStrict: boolean;
  hasTests: boolean;
  hasActiveMaintenence: boolean;
  hasGoodFirstIssues: boolean;
  codeQuality: 'excellent' | 'good' | 'fair';
}

// ตัวอย่างโปรเจกต์ที่ดีสำหรับ beginners
const beginnerFriendlyProjects = [
  {
    name: 'TypeScript itself',
    url: 'https://github.com/microsoft/TypeScript',
    difficulty: 'hard',
    description: 'TypeScript compiler and language service',
  },
  {
    name: 'type-challenges',
    url: 'https://github.com/type-challenges/type-challenges',
    difficulty: 'medium',
    description: 'TypeScript type system challenges',
  },
  {
    name: 'zod',
    url: 'https://github.com/colinhacks/zod',
    difficulty: 'medium',
    description: 'TypeScript-first schema validation',
  },
  {
    name: 'ts-pattern',
    url: 'https://github.com/gvergnaud/ts-pattern',
    difficulty: 'medium',
    description: 'Pattern matching for TypeScript',
  },
  {
    name: 'trpc',
    url: 'https://github.com/trpc/trpc',
    difficulty: 'hard',
    description: 'End-to-end typesafe APIs',
  },
];

// Helper สำหรับประเมินโปรเจกต์
function evaluateProject(
  starCount: number,
  lastCommitDays: number,
  issueCount: number,
  hasGoodFirstLabel: boolean
): string {
  const score = [
    starCount > 100 ? 1 : 0,
    lastCommitDays < 30 ? 1 : 0,
    issueCount > 5 ? 1 : 0,
    hasGoodFirstLabel ? 2 : 0,
  ].reduce((a, b) => a + b, 0);

  if (score >= 4) return 'Highly recommended';
  if (score >= 2) return 'Good choice';
  return 'Consider alternatives';
}
```

### GitHub Search สำหรับ TypeScript Issues

```bash
# ค้นหาบน GitHub
# label:"good first issue" language:TypeScript
# label:"help wanted" language:TypeScript
# label:"bug" language:TypeScript is:open

# ใช้ GitHub CLI
gh issue list \
  --repo microsoft/TypeScript \
  --label "good first issue" \
  --state open \
  --limit 20

# ค้นหา issues ที่ไม่มีคนทำ
gh search issues \
  --language TypeScript \
  --label "good first issue" \
  --no-assignee \
  --limit 50
```

---

## 99.2 การอ่าน TypeScript Codebases

```typescript
// เทคนิคการอ่าน codebase ที่ไม่รู้จัก

// 1. เริ่มจาก entry point
// ดูที่ package.json -> main หรือ exports field

// 2. อ่าน tsconfig.json เพื่อเข้าใจ project structure
interface TSConfigAnalysis {
  paths: Record<string, string[]>;  // Module aliases
  strict: boolean;                   // Type strictness level
  target: string;                    // JavaScript target version
  lib: string[];                     // Built-in type definitions
}

// 3. ทำ dependency graph
type DependencyMap = Map<string, Set<string>>;

function analyzeDependencies(sourceFiles: string[]): DependencyMap {
  const deps: DependencyMap = new Map();
  
  for (const file of sourceFiles) {
    // In real code, use ts-morph or similar to parse
    const imports = extractImports(file);
    deps.set(file, new Set(imports));
  }
  
  return deps;
}

function extractImports(_filePath: string): string[] {
  // Mock - จริงๆ ใช้ ts-morph หรือ babel parser
  return [];
}

// 4. ค้นหา patterns ที่ใช้ซ้ำๆ
type DesignPattern = 
  | 'singleton'
  | 'factory'
  | 'builder'
  | 'observer'
  | 'strategy'
  | 'decorator';

interface CodebaseAnalysis {
  patterns: DesignPattern[];
  conventions: {
    namingConvention: 'camelCase' | 'PascalCase' | 'snake_case';
    fileOrganization: 'by-feature' | 'by-type' | 'monolithic';
    testFramework: 'jest' | 'vitest' | 'mocha' | 'none';
    linter: 'eslint' | 'biome' | 'none';
  };
  typeUtilization: 'heavy' | 'moderate' | 'light';
}
```

---

## 99.3 การเข้าใจ Declaration Files

```typescript
// src/understanding-declarations/index.ts

/**
 * Declaration files (.d.ts) คือ TypeScript's way of describing
 * JavaScript code ที่ไม่มี type information
 */

// ตัวอย่าง: อ่าน declaration file
// @types/express/index.d.ts (simplified)

// Ambient declarations
declare module 'some-library' {
  // Export types
  export interface Config {
    timeout: number;
    retries: number;
    debug?: boolean;
  }

  export function initialize(config: Config): void;
  export function destroy(): Promise<void>;
  
  export class Client {
    constructor(config: Config);
    connect(): Promise<void>;
    disconnect(): void;
    send(data: unknown): Promise<void>;
  }

  // Default export
  export default Client;
}

// Augmenting existing modules (Declaration Merging)
declare module 'express' {
  interface Request {
    // เพิ่ม custom properties ให้ Request
    user?: {
      id: string;
      email: string;
      role: 'admin' | 'user';
    };
    correlationId?: string;
  }

  interface Response {
    // เพิ่ม custom methods
    success<T>(data: T, message?: string): void;
    error(message: string, statusCode?: number): void;
  }
}

// Global augmentation
declare global {
  interface Window {
    analytics?: {
      track(event: string, properties?: Record<string, unknown>): void;
      identify(userId: string, traits?: Record<string, unknown>): void;
    };
    __CONFIG__?: {
      apiUrl: string;
      version: string;
      environment: 'production' | 'staging' | 'development';
    };
  }

  // เพิ่ม global functions
  function $$(selector: string): Element[];
}
```

```typescript
// src/writing-declarations/create-declarations.ts
import * as fs from 'fs';
import * as path from 'path';

/**
 * เขียน Declaration Files สำหรับ JavaScript libraries
 */

// ตัวอย่าง: สร้าง declaration file สำหรับ legacy JS library
// legacy-lib.js (original JS)
// function formatDate(date, format) { ... }
// function parseDate(str) { ... }

// legacy-lib.d.ts (declaration file เราสร้าง)
declare module 'legacy-lib' {
  type DateFormat = 'YYYY-MM-DD' | 'DD/MM/YYYY' | 'MM-DD-YYYY';

  export function formatDate(date: Date, format: DateFormat): string;
  export function parseDate(dateString: string): Date | null;
  export function isValidDate(date: unknown): date is Date;
  
  export interface DateRange {
    start: Date;
    end: Date;
  }
  
  export function getDaysInRange(range: DateRange): Date[];
}

// Tools สำหรับช่วยสร้าง declarations
// npm install -g dts-gen
// dts-gen -m some-module

// หรือใช้ TypeScript compiler
// tsc --declaration --emitDeclarationOnly

// Template สำหรับ declaration file
const DECLARATION_TEMPLATE = `
// Type definitions for [package-name] [version]
// Project: [project-url]
// Definitions by: [your-name] <[your-github-url]>
// Definitions: https://github.com/DefinitelyTyped/DefinitelyTyped

export interface Options {
  // ...
}

export function create(options?: Options): Instance;

export interface Instance {
  // ...
  destroy(): void;
}

export default create;
`;

function createDeclarationFile(
  moduleName: string,
  outputPath: string
): void {
  const content = DECLARATION_TEMPLATE.replace('[package-name]', moduleName);
  fs.writeFileSync(outputPath, content, 'utf-8');
  console.log(`Created declaration file: ${outputPath}`);
}
```

---

## 99.4 TypeScript-Friendly APIs

```typescript
// src/api-design/typescript-friendly.ts

/**
 * หลักการออกแบบ API ที่ TypeScript-friendly
 */

// 1. Use branded types for safety
type UserId = string & { readonly __brand: 'UserId' };
type ProductId = string & { readonly __brand: 'ProductId' };

function createUserId(id: string): UserId {
  return id as UserId;
}

function createProductId(id: string): ProductId {
  return id as ProductId;
}

// ป้องกันการสลับ IDs โดยบังเอิญ
function getUser(_id: UserId): void {
  // Implementation
}

// 2. Builder pattern กับ TypeScript
class QueryBuilder<T extends Record<string, unknown>> {
  private conditions: string[] = [];
  private selectedFields: (keyof T)[] = [];
  private limitValue?: number;
  private offsetValue?: number;
  private orderFields: Array<{ field: keyof T; direction: 'ASC' | 'DESC' }> = [];

  select<K extends keyof T>(...fields: K[]): QueryBuilder<Pick<T, K>> {
    this.selectedFields = fields;
    return this as unknown as QueryBuilder<Pick<T, K>>;
  }

  where(condition: string): this {
    this.conditions.push(condition);
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

  orderBy(field: keyof T, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.orderFields.push({ field, direction });
    return this;
  }

  build(): string {
    const fields = this.selectedFields.length > 0
      ? this.selectedFields.join(', ')
      : '*';
    
    let query = `SELECT ${fields} FROM table`;
    
    if (this.conditions.length > 0) {
      query += ` WHERE ${this.conditions.join(' AND ')}`;
    }
    
    if (this.orderFields.length > 0) {
      const orders = this.orderFields.map(
        ({ field, direction }) => `${String(field)} ${direction}`
      );
      query += ` ORDER BY ${orders.join(', ')}`;
    }
    
    if (this.limitValue !== undefined) {
      query += ` LIMIT ${this.limitValue}`;
    }
    
    if (this.offsetValue !== undefined) {
      query += ` OFFSET ${this.offsetValue}`;
    }
    
    return query;
  }
}

// 3. Discriminated unions สำหรับ clear API
type ApiResponse<T> =
  | { status: 'success'; data: T; timestamp: number }
  | { status: 'error'; error: string; code: number; timestamp: number }
  | { status: 'loading' };

function handleResponse<T>(response: ApiResponse<T>): void {
  switch (response.status) {
    case 'success':
      console.log('Data:', response.data);
      break;
    case 'error':
      console.error(`Error ${response.code}: ${response.error}`);
      break;
    case 'loading':
      console.log('Loading...');
      break;
  }
}

// 4. Fluent interfaces
interface EmailBuilder {
  to(address: string): EmailBuilder;
  from(address: string): EmailBuilder;
  subject(subject: string): EmailBuilder;
  body(content: string): EmailBuilder;
  html(content: string): EmailBuilder;
  cc(address: string): EmailBuilder;
  bcc(address: string): EmailBuilder;
  attach(filename: string, content: Buffer): EmailBuilder;
  send(): Promise<{ messageId: string }>;
}

// 5. Conditional types สำหรับ flexible APIs
type DeepReadonly<T> = T extends (infer U)[]
  ? DeepReadonlyArray<U>
  : T extends Record<string, unknown>
  ? DeepReadonlyObject<T>
  : T;

type DeepReadonlyArray<T> = ReadonlyArray<DeepReadonly<T>>;

type DeepReadonlyObject<T> = {
  readonly [P in keyof T]: DeepReadonly<T[P]>;
};

// 6. Type-safe event system
type EventMap = {
  userCreated: { id: string; email: string };
  userDeleted: { id: string };
  orderPlaced: { orderId: string; userId: string; total: number };
  paymentReceived: { orderId: string; amount: number };
};

class TypedEventEmitter<T extends Record<string, unknown>> {
  private listeners = new Map<keyof T, Set<(data: unknown) => void>>();

  on<K extends keyof T>(event: K, listener: (data: T[K]) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener as (data: unknown) => void);
    
    // Return unsubscribe function
    return () => this.off(event, listener);
  }

  off<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
    this.listeners.get(event)?.delete(listener as (data: unknown) => void);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners.get(event)?.forEach((listener) => listener(data));
  }
}

const emitter = new TypedEventEmitter<EventMap>();

// Type-safe event listener
emitter.on('userCreated', (user) => {
  console.log(user.id, user.email); // type-safe!
});

emitter.emit('orderPlaced', {
  orderId: 'order-1',
  userId: 'user-1',
  total: 500,
});
```

---

## 99.5 การเขียน Tests สำหรับ Contributions

```typescript
// src/testing/contribution-tests.ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';

/**
 * หลักการเขียน tests ที่ดีสำหรับ contributions
 */

// 1. Test file structure
// ✅ Good: อธิบาย behavior ชัดเจน
describe('UserService', () => {
  describe('createUser', () => {
    it('should create a user with valid data', async () => {
      // Arrange
      const userData = { email: 'test@example.com', name: 'Test User' };
      
      // Act
      const user = await createUser(userData);
      
      // Assert
      expect(user).toMatchObject({
        email: 'test@example.com',
        name: 'Test User',
        id: expect.stringMatching(/^[0-9a-f-]{36}$/),
        createdAt: expect.any(Date),
      });
    });

    it('should throw ValidationError when email is invalid', async () => {
      await expect(
        createUser({ email: 'not-an-email', name: 'Test' })
      ).rejects.toThrow('ValidationError');
    });

    it('should throw ConflictError when email already exists', async () => {
      const email = 'existing@example.com';
      
      // Create first user
      await createUser({ email, name: 'First User' });
      
      // Try to create duplicate
      await expect(
        createUser({ email, name: 'Second User' })
      ).rejects.toThrow('ConflictError');
    });
  });
});

// Mock functions
async function createUser(data: { email: string; name: string }) {
  if (!data.email.includes('@')) {
    throw new Error('ValidationError: Invalid email');
  }
  return {
    id: '123e4567-e89b-12d3-a456-426614174000',
    email: data.email,
    name: data.name,
    createdAt: new Date(),
  };
}

// 2. Testing TypeScript-specific features
describe('TypeScript type guards', () => {
  function isString(value: unknown): value is string {
    return typeof value === 'string';
  }

  function isNonEmptyString(value: unknown): value is string {
    return isString(value) && value.length > 0;
  }

  it('should narrow type correctly', () => {
    const values: unknown[] = ['hello', 42, '', null, 'world'];
    const strings = values.filter(isString);
    
    expect(strings).toEqual(['hello', 42, '', null, 'world'].filter(isString));
    // TypeScript knows strings is string[]
  });

  it('should identify non-empty strings', () => {
    expect(isNonEmptyString('')).toBe(false);
    expect(isNonEmptyString('hello')).toBe(true);
    expect(isNonEmptyString(42)).toBe(false);
    expect(isNonEmptyString(null)).toBe(false);
  });
});

// 3. Testing async code patterns
describe('Async patterns', () => {
  it('should handle promise rejection', async () => {
    const failingFn = async (): Promise<never> => {
      throw new Error('Network error');
    };

    await expect(failingFn()).rejects.toThrow('Network error');
  });

  it('should timeout properly', async () => {
    const slowOperation = (ms: number) =>
      new Promise((resolve) => setTimeout(resolve, ms));

    await expect(
      Promise.race([
        slowOperation(5000),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), 100)
        ),
      ])
    ).rejects.toThrow('Timeout');
  });
});

// 4. Snapshot testing
describe('Component rendering', () => {
  it('should match snapshot', () => {
    const renderButton = (text: string, variant: 'primary' | 'secondary') => ({
      tag: 'button',
      class: `btn btn-${variant}`,
      text,
    });

    expect(renderButton('Click me', 'primary')).toMatchSnapshot();
    expect(renderButton('Cancel', 'secondary')).toMatchSnapshot();
  });
});
```

---

## 99.6 TSDoc สำหรับ Documentation

```typescript
// src/documentation/tsdoc-examples.ts

/**
 * TypeScript Date utility functions
 * @packageDocumentation
 */

/**
 * Options สำหรับการ format วันที่
 * @public
 */
export interface DateFormatOptions {
  /** Locale string เช่น 'th-TH', 'en-US' */
  locale?: string;

  /** Format สำหรับ output */
  format?: 'short' | 'medium' | 'long' | 'full';

  /** แสดงเวลาหรือไม่ */
  includeTime?: boolean;

  /** Timezone เช่น 'Asia/Bangkok' */
  timezone?: string;
}

/**
 * Format วันที่เป็น string ที่อ่านได้
 *
 * @param date - วันที่ที่ต้องการ format
 * @param options - ตัวเลือกสำหรับการ format
 * @returns วันที่ในรูปแบบ string ที่อ่านได้
 * @throws {TypeError} เมื่อ date parameter ไม่ใช่ Date object หรือ timestamp
 *
 * @example
 * ```typescript
 * const date = new Date('2024-01-15T10:30:00Z');
 *
 * // Basic usage
 * formatDate(date);
 * // => '15 มกราคม 2567'
 *
 * // With options
 * formatDate(date, { locale: 'en-US', format: 'full', includeTime: true });
 * // => 'Monday, January 15, 2024 at 5:30 PM'
 * ```
 *
 * @public
 */
export function formatDate(date: Date, options: DateFormatOptions = {}): string {
  const {
    locale = 'th-TH',
    format = 'medium',
    includeTime = false,
    timezone,
  } = options;

  if (!(date instanceof Date) || isNaN(date.getTime())) {
    throw new TypeError('Invalid date provided');
  }

  const formatOptions: Intl.DateTimeFormatOptions = {
    dateStyle: format,
    ...(includeTime && { timeStyle: 'short' }),
    ...(timezone && { timeZone: timezone }),
  };

  return new Intl.DateTimeFormat(locale, formatOptions).format(date);
}

/**
 * คำนวณจำนวนวันระหว่างวันที่สองวัน
 *
 * @param start - วันที่เริ่มต้น
 * @param end - วันที่สิ้นสุด (default: วันนี้)
 * @returns จำนวนวัน (บวก = end หลัง start, ลบ = end ก่อน start)
 *
 * @remarks
 * ฟังก์ชันนี้ ignore time component และคำนวณเฉพาะวัน
 * สามารถใช้กับ timezone ต่างๆ ได้
 *
 * @example
 * ```typescript
 * const start = new Date('2024-01-01');
 * const end = new Date('2024-01-15');
 * daysBetween(start, end); // => 14
 * daysBetween(end, start); // => -14
 * ```
 *
 * @public
 */
export function daysBetween(start: Date, end: Date = new Date()): number {
  const startDay = new Date(start);
  startDay.setHours(0, 0, 0, 0);

  const endDay = new Date(end);
  endDay.setHours(0, 0, 0, 0);

  const diffMs = endDay.getTime() - startDay.getTime();
  return Math.round(diffMs / (1000 * 60 * 60 * 24));
}

/**
 * @internal
 * Helper function - ไม่ควร export หรือใช้โดยตรง
 */
function _normalizeDate(date: Date | string | number): Date {
  if (date instanceof Date) return date;
  return new Date(date);
}

/**
 * @deprecated ใช้ {@link formatDate} แทน
 * @see formatDate
 */
export function oldFormatDate(date: Date): string {
  return date.toLocaleDateString();
}
```

---

## 99.7 Semantic Versioning

```typescript
// src/versioning/semver.ts

/**
 * Semantic Versioning (SemVer) management
 * MAJOR.MINOR.PATCH
 * - MAJOR: breaking changes
 * - MINOR: new features (backward compatible)
 * - PATCH: bug fixes (backward compatible)
 */

interface Version {
  major: number;
  minor: number;
  patch: number;
  prerelease?: string;
  buildMetadata?: string;
}

const SEMVER_REGEX =
  /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-((?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+([0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$/;

export function parseVersion(versionString: string): Version {
  const match = SEMVER_REGEX.exec(versionString);
  if (!match) {
    throw new Error(`Invalid semver: ${versionString}`);
  }

  return {
    major: parseInt(match[1] ?? '0', 10),
    minor: parseInt(match[2] ?? '0', 10),
    patch: parseInt(match[3] ?? '0', 10),
    prerelease: match[4],
    buildMetadata: match[5],
  };
}

export function formatVersion(version: Version): string {
  let str = `${version.major}.${version.minor}.${version.patch}`;
  if (version.prerelease) str += `-${version.prerelease}`;
  if (version.buildMetadata) str += `+${version.buildMetadata}`;
  return str;
}

export type BumpType = 'major' | 'minor' | 'patch';

export function bumpVersion(current: string, type: BumpType): string {
  const version = parseVersion(current);

  switch (type) {
    case 'major':
      return formatVersion({
        major: version.major + 1,
        minor: 0,
        patch: 0,
      });
    case 'minor':
      return formatVersion({
        major: version.major,
        minor: version.minor + 1,
        patch: 0,
      });
    case 'patch':
      return formatVersion({
        major: version.major,
        minor: version.minor,
        patch: version.patch + 1,
      });
  }
}

export function compareVersions(a: string, b: string): -1 | 0 | 1 {
  const va = parseVersion(a);
  const vb = parseVersion(b);

  if (va.major !== vb.major) return va.major > vb.major ? 1 : -1;
  if (va.minor !== vb.minor) return va.minor > vb.minor ? 1 : -1;
  if (va.patch !== vb.patch) return va.patch > vb.patch ? 1 : -1;
  return 0;
}

// ตัวอย่าง
const versions = ['1.0.0', '2.1.0', '1.5.3', '2.0.0-beta.1', '1.5.10'];
const sorted = [...versions].sort(compareVersions);
console.log(sorted);
// ['1.0.0', '1.5.3', '1.5.10', '2.0.0-beta.1', '2.1.0']
```

---

## 99.8 PR Best Practices

```typescript
// src/pr-practices/checklist.ts

/**
 * Checklist สำหรับการทำ PR ที่ดี
 */

interface PRChecklist {
  // Code quality
  hasTests: boolean;
  testsPass: boolean;
  noTypeErrors: boolean;
  lintPassed: boolean;
  formatPassed: boolean;
  
  // Documentation
  hasJSDoc: boolean;
  updatedReadme: boolean;
  updatedChangelog: boolean;
  
  // PR Description
  describesChanges: boolean;
  linkToIssue: boolean;
  hasScreenshots: boolean; // For UI changes
  
  // Breaking changes
  hasBreakingChanges: boolean;
  markedAsBraking: boolean;
  migrationGuide: boolean;
}

function validatePR(checklist: PRChecklist): string[] {
  const issues: string[] = [];

  if (!checklist.hasTests) {
    issues.push('❌ Missing tests for new functionality');
  }

  if (!checklist.testsPass) {
    issues.push('❌ Tests are failing');
  }

  if (!checklist.noTypeErrors) {
    issues.push('❌ TypeScript type errors found');
  }

  if (!checklist.lintPassed) {
    issues.push('❌ Lint errors found (run: npm run lint:fix)');
  }

  if (checklist.hasBreakingChanges && !checklist.markedAsBraking) {
    issues.push('❌ Breaking changes must be marked in PR title and changelog');
  }

  if (checklist.hasBreakingChanges && !checklist.migrationGuide) {
    issues.push('⚠️  Breaking changes should have migration guide');
  }

  if (!checklist.linkToIssue) {
    issues.push('ℹ️  Consider linking to related issue');
  }

  return issues;
}

// PR Template (สำหรับ .github/pull_request_template.md)
const PR_TEMPLATE = `
## Summary
Brief description of what this PR does

## Type of Change
- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change (requires major version bump)
- [ ] Documentation update
- [ ] Refactoring

## Related Issues
Closes #ISSUE_NUMBER

## Testing
- [ ] Added unit tests
- [ ] Added integration tests
- [ ] Existing tests still pass

## Checklist
- [ ] My code follows the project's code style
- [ ] I have performed a self-review
- [ ] I have commented complex code
- [ ] I have updated documentation
- [ ] My changes generate no warnings
- [ ] I have run the test suite
`;

// Conventional Commits format
type CommitType =
  | 'feat'    // New feature
  | 'fix'     // Bug fix
  | 'docs'    // Documentation
  | 'style'   // Formatting (no code change)
  | 'refactor' // Refactoring
  | 'test'    // Adding tests
  | 'chore'   // Maintenance
  | 'perf'    // Performance improvement
  | 'ci'      // CI configuration
  | 'build'   // Build system changes
  | 'revert'; // Revert a commit

interface ConventionalCommit {
  type: CommitType;
  scope?: string;
  description: string;
  body?: string;
  footer?: string;
  breaking?: boolean;
}

function formatCommit(commit: ConventionalCommit): string {
  let result = commit.type;

  if (commit.scope) {
    result += `(${commit.scope})`;
  }

  if (commit.breaking) {
    result += '!';
  }

  result += `: ${commit.description}`;

  if (commit.body) {
    result += `\n\n${commit.body}`;
  }

  if (commit.breaking) {
    result += '\n\nBREAKING CHANGE: describe the breaking change';
  }

  if (commit.footer) {
    result += `\n\n${commit.footer}`;
  }

  return result;
}

// ตัวอย่าง commits
const commits: ConventionalCommit[] = [
  {
    type: 'feat',
    scope: 'auth',
    description: 'add OAuth2 authentication support',
    body: 'Implement OAuth2 flow with PKCE for secure authentication.',
    footer: 'Closes #123',
  },
  {
    type: 'fix',
    scope: 'api',
    description: 'handle null response from user endpoint',
    footer: 'Fixes #456',
  },
  {
    type: 'refactor',
    scope: 'db',
    description: 'migrate from callbacks to async/await',
    breaking: true,
    body: 'All database methods now return Promises instead of using callbacks.',
  },
];

commits.forEach((commit) => {
  console.log(formatCommit(commit));
  console.log('---');
});
```

---

## 99.9 TypeScript-specific OSS Patterns

```typescript
// src/oss-patterns/plugin-system.ts

/**
 * Plugin system ที่ type-safe
 * Pattern นี้ใช้กันมากใน TypeScript OSS projects
 */

// Plugin interface
export interface Plugin<TConfig = Record<string, unknown>> {
  name: string;
  version: string;
  setup(config?: TConfig): void | Promise<void>;
  teardown?(): void | Promise<void>;
}

// Plugin registry
export class PluginRegistry {
  private plugins = new Map<string, Plugin>();
  private initialized = new Set<string>();

  register<T>(plugin: Plugin<T>, config?: T): this {
    if (this.plugins.has(plugin.name)) {
      throw new Error(`Plugin '${plugin.name}' already registered`);
    }
    
    // Wrap with config
    this.plugins.set(plugin.name, {
      ...plugin,
      setup: () => plugin.setup(config),
    });
    
    return this;
  }

  async initialize(): Promise<void> {
    for (const [name, plugin] of this.plugins) {
      if (!this.initialized.has(name)) {
        await plugin.setup();
        this.initialized.add(name);
        console.log(`Plugin '${name}' initialized`);
      }
    }
  }

  async teardown(): Promise<void> {
    for (const [name, plugin] of [...this.plugins].reverse()) {
      if (this.initialized.has(name) && plugin.teardown) {
        await plugin.teardown();
        this.initialized.delete(name);
      }
    }
  }

  getPlugin<T extends Plugin>(name: string): T | undefined {
    return this.plugins.get(name) as T | undefined;
  }

  isRegistered(name: string): boolean {
    return this.plugins.has(name);
  }
}

// ตัวอย่าง plugins
const loggerPlugin: Plugin<{ level: 'info' | 'debug' | 'error' }> = {
  name: 'logger',
  version: '1.0.0',
  setup(config) {
    const level = config?.level ?? 'info';
    console.log(`Logger plugin setup with level: ${level}`);
  },
  teardown() {
    console.log('Logger plugin teardown');
  },
};

const cachePlugin: Plugin<{ ttl: number; maxSize: number }> = {
  name: 'cache',
  version: '1.0.0',
  setup(config) {
    const ttl = config?.ttl ?? 3600;
    const maxSize = config?.maxSize ?? 1000;
    console.log(`Cache plugin setup: TTL=${ttl}s, MaxSize=${maxSize}`);
  },
};

// ใช้งาน
const registry = new PluginRegistry();
registry
  .register(loggerPlugin, { level: 'debug' })
  .register(cachePlugin, { ttl: 300, maxSize: 500 });

// middleware pattern
type MiddlewareFn<T> = (context: T, next: () => Promise<void>) => Promise<void>;

class MiddlewareChain<T> {
  private middlewares: MiddlewareFn<T>[] = [];

  use(middleware: MiddlewareFn<T>): this {
    this.middlewares.push(middleware);
    return this;
  }

  async execute(context: T): Promise<void> {
    const runner = async (index: number): Promise<void> => {
      if (index >= this.middlewares.length) return;
      
      const middleware = this.middlewares[index]!;
      await middleware(context, () => runner(index + 1));
    };
    
    await runner(0);
  }
}

// ตัวอย่าง
interface RequestContext {
  path: string;
  method: string;
  headers: Record<string, string>;
  body?: unknown;
  response?: unknown;
  startTime: number;
}

const chain = new MiddlewareChain<RequestContext>();

chain
  .use(async (ctx, next) => {
    console.log(`${ctx.method} ${ctx.path}`);
    ctx.startTime = Date.now();
    await next();
    const duration = Date.now() - ctx.startTime;
    console.log(`Completed in ${duration}ms`);
  })
  .use(async (ctx, next) => {
    // Auth check
    if (!ctx.headers['authorization']) {
      ctx.response = { error: 'Unauthorized', status: 401 };
      return;
    }
    await next();
  })
  .use(async (ctx, _next) => {
    // Handle request
    ctx.response = { data: 'Hello!', status: 200 };
  });
```

---

## 99.10 Changelog Management

```typescript
// src/changelog/changelog-manager.ts
import * as fs from 'fs';

type ChangeType = 'added' | 'changed' | 'deprecated' | 'removed' | 'fixed' | 'security';

interface ChangeEntry {
  type: ChangeType;
  description: string;
  pr?: number;
  issue?: number;
}

interface VersionEntry {
  version: string;
  date: string;
  changes: ChangeEntry[];
  breaking?: boolean;
}

export class ChangelogManager {
  private entries: VersionEntry[] = [];

  addVersion(entry: VersionEntry): this {
    this.entries.unshift(entry); // Newest first
    return this;
  }

  generateMarkdown(): string {
    const header = `# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

`;

    const versionSections = this.entries.map((entry) => {
      const title = entry.breaking
        ? `## [${entry.version}] - ${entry.date} **BREAKING**`
        : `## [${entry.version}] - ${entry.date}`;

      const changesByType = this.groupByType(entry.changes);
      
      const sections = Object.entries(changesByType).map(([type, changes]) => {
        const typeTitle = type.charAt(0).toUpperCase() + type.slice(1);
        const items = changes.map((c) => {
          let item = `- ${c.description}`;
          if (c.pr) item += ` ([#${c.pr}](https://github.com/org/repo/pull/${c.pr}))`;
          if (c.issue) item += ` ([#${c.issue}](https://github.com/org/repo/issues/${c.issue}))`;
          return item;
        });
        
        return `### ${typeTitle}\n${items.join('\n')}`;
      });

      return `${title}\n\n${sections.join('\n\n')}`;
    });

    return header + versionSections.join('\n\n---\n\n');
  }

  private groupByType(
    changes: ChangeEntry[]
  ): Partial<Record<ChangeType, ChangeEntry[]>> {
    const order: ChangeType[] = [
      'added', 'changed', 'deprecated', 'removed', 'fixed', 'security',
    ];
    
    const grouped: Partial<Record<ChangeType, ChangeEntry[]>> = {};
    
    for (const change of changes) {
      if (!grouped[change.type]) {
        grouped[change.type] = [];
      }
      grouped[change.type]!.push(change);
    }

    // Sort by order
    const sorted: Partial<Record<ChangeType, ChangeEntry[]>> = {};
    for (const type of order) {
      if (grouped[type]) {
        sorted[type] = grouped[type];
      }
    }
    
    return sorted;
  }

  saveToFile(filePath: string): void {
    const content = this.generateMarkdown();
    fs.writeFileSync(filePath, content, 'utf-8');
    console.log(`Changelog saved to ${filePath}`);
  }
}

// ตัวอย่างการใช้งาน
const changelog = new ChangelogManager();

changelog
  .addVersion({
    version: '2.0.0',
    date: '2024-01-15',
    breaking: true,
    changes: [
      { type: 'added', description: 'New async API with Promise-based interface', pr: 123 },
      { type: 'removed', description: 'Removed deprecated callback API', issue: 89 },
      { type: 'changed', description: 'Minimum Node.js version is now 18' },
    ],
  })
  .addVersion({
    version: '1.5.2',
    date: '2024-01-01',
    changes: [
      { type: 'fixed', description: 'Fix race condition in connection pool', pr: 110 },
      { type: 'security', description: 'Update dependencies to patch CVE-2023-XXXX' },
    ],
  });

console.log(changelog.generateMarkdown());
```

---

## บทสรุป Part 99

ในบทนี้เราได้เรียนรู้:

1. **หาโปรเจกต์** - เกณฑ์และวิธีค้นหา TypeScript projects ที่เหมาะสม
2. **อ่าน Codebase** - เทคนิคการเข้าใจ codebase ที่ไม่รู้จัก
3. **Declaration Files** - การเขียนและ augment type definitions
4. **TypeScript-Friendly APIs** - Branded types, builder pattern, event emitter
5. **เขียน Tests** - Unit tests, snapshot tests สำหรับ contributions
6. **TSDoc** - Documentation ที่มีคุณภาพ
7. **SemVer** - Semantic versioning management
8. **PR Best Practices** - Conventional commits, checklists
9. **OSS Patterns** - Plugin systems, middleware chains
10. **Changelog** - การจัดการ changelog อย่างมืออาชีพ

การ contribute ให้กับ open source ไม่ได้ยากอย่างที่คิด เริ่มจาก issues เล็กๆ ก่อน เช่น documentation fixes หรือ test improvements แล้วค่อยๆ ขยายไปสู่ features ใหญ่ขึ้น

---

## แบบฝึกหัด

1. หา TypeScript project ที่สนใจและ fork มัน
2. อ่าน CONTRIBUTING.md และ setup development environment
3. ทำ bug fix เล็กๆ และส่ง PR แรกของคุณ
4. เขียน declaration file สำหรับ JavaScript library ที่ไม่มี types
5. เพิ่ม tests ให้กับ untested code ใน OSS project ที่คุณชอบ
