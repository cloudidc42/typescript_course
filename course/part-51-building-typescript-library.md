# ส่วนที่ 51: การสร้าง TypeScript Libraries

## บทนำ

การสร้าง TypeScript Library ที่ดีต้องการความใส่ใจในหลายด้าน ตั้งแต่การตั้งค่าโปรเจค การสร้าง type definitions ที่ดี การ bundle และการ publish ไปจนถึงการเขียน documentation บทนี้จะสอนทุกอย่างที่จำเป็นสำหรับการสร้าง library คุณภาพสูง

---

## 1. Library Project Setup (การตั้งค่าโปรเจค Library)

### 1.1 โครงสร้าง Directory ที่ดี

```
my-typescript-library/
├── src/
│   ├── index.ts          # Main entry point
│   ├── types.ts          # Type definitions
│   ├── utils/
│   │   ├── index.ts
│   │   └── helpers.ts
│   └── core/
│       ├── index.ts
│       └── processor.ts
├── tests/
│   ├── unit/
│   │   └── processor.test.ts
│   └── integration/
│       └── index.test.ts
├── dist/                 # Build output (gitignored)
├── docs/                 # Generated documentation
├── examples/             # Usage examples
│   └── basic/
│       └── index.ts
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── publish.yml
├── package.json
├── tsconfig.json
├── tsconfig.build.json
├── tsup.config.ts
├── jest.config.ts
├── .eslintrc.js
├── .prettierrc
├── CHANGELOG.md
└── README.md
```

### 1.2 การเริ่มต้นโปรเจค

```bash
# สร้าง directory
mkdir my-typescript-library
cd my-typescript-library

# Initialize npm
npm init -y

# ติดตั้ง dependencies
npm install --save-dev typescript tsup @types/node
npm install --save-dev jest @jest/globals ts-jest @types/jest
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install --save-dev prettier
npm install --save-dev typedoc
npm install --save-dev changesets  # สำหรับ versioning

# สร้างไฟล์ config
npx tsc --init
```

```typescript
// src/index.ts - Main entry point ของ library
export { DataProcessor } from './core/processor';
export { formatDate, formatCurrency, formatBytes } from './utils/formatters';
export { createValidator } from './utils/validators';
export type {
  ProcessorOptions,
  DataTransformer,
  ValidationResult,
  FormatterOptions,
} from './types';
```

---

## 2. Package.json สำหรับ Libraries

### 2.1 การตั้งค่า package.json อย่างสมบูรณ์

```json
{
  "name": "@myorg/my-typescript-library",
  "version": "1.0.0",
  "description": "A comprehensive TypeScript utility library",
  "author": "Your Name <you@example.com>",
  "license": "MIT",
  "keywords": ["typescript", "utility", "library"],
  "homepage": "https://github.com/myorg/my-typescript-library#readme",
  "repository": {
    "type": "git",
    "url": "https://github.com/myorg/my-typescript-library.git"
  },
  "bugs": {
    "url": "https://github.com/myorg/my-typescript-library/issues"
  },
  
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.js"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    },
    "./utils": {
      "import": {
        "types": "./dist/utils/index.d.ts",
        "default": "./dist/utils/index.js"
      },
      "require": {
        "types": "./dist/utils/index.d.cts",
        "default": "./dist/utils/index.cjs"
      }
    }
  },
  
  "files": [
    "dist",
    "README.md",
    "CHANGELOG.md",
    "LICENSE"
  ],
  
  "scripts": {
    "build": "tsup",
    "build:watch": "tsup --watch",
    "typecheck": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src tests",
    "lint:fix": "eslint src tests --fix",
    "format": "prettier --write src tests",
    "docs": "typedoc",
    "prepublishOnly": "npm run build && npm run test && npm run typecheck",
    "changeset": "changeset",
    "version": "changeset version",
    "release": "changeset publish"
  },
  
  "peerDependencies": {
    "typescript": ">=4.9.0"
  },
  "peerDependenciesMeta": {
    "typescript": {
      "optional": true
    }
  },
  
  "devDependencies": {
    "@changesets/cli": "^2.26.0",
    "@jest/globals": "^29.0.0",
    "@types/jest": "^29.0.0",
    "@types/node": "^18.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "jest": "^29.0.0",
    "prettier": "^3.0.0",
    "ts-jest": "^29.0.0",
    "tsup": "^7.0.0",
    "typedoc": "^0.25.0",
    "typescript": "^5.0.0"
  },
  
  "engines": {
    "node": ">=16.0.0"
  },
  
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  },
  
  "sideEffects": false
}
```

---

## 3. tsconfig สำหรับ Libraries

### 3.1 tsconfig.json หลัก

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2020"],
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests", "examples", "**/*.test.ts", "**/*.spec.ts"]
}
```

### 3.2 tsconfig.build.json

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "removeComments": false,
    "noEmit": false
  },
  "exclude": [
    "node_modules",
    "dist",
    "tests",
    "examples",
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

### 3.3 tsconfig.test.json

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "moduleResolution": "node",
    "types": ["jest", "node"],
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noEmit": true
  },
  "include": ["src/**/*", "tests/**/*"]
}
```

---

## 4. Dual CJS/ESM Output

### 4.1 tsup Configuration

```typescript
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: {
    index: 'src/index.ts',
    'utils/index': 'src/utils/index.ts',
  },
  format: ['cjs', 'esm'],
  dts: true,
  sourcemap: true,
  clean: true,
  treeshake: true,
  splitting: false,
  minify: process.env.NODE_ENV === 'production',
  
  // ให้แน่ใจว่า external packages ไม่ถูก bundle
  external: ['react', 'react-dom'],
  
  // Banner สำหรับ license
  banner: {
    js: `/**
 * @license MIT
 * my-typescript-library v${process.env.npm_package_version}
 * Copyright (c) ${new Date().getFullYear()} Your Name
 */`,
  },
  
  // esbuild options
  esbuildOptions(options) {
    options.target = 'es2020';
  },
  
  // สร้าง .cjs files พร้อม package.json
  outExtension({ format }) {
    return {
      js: format === 'cjs' ? '.cjs' : '.js',
    };
  },
  
  // onSuccess hook
  onSuccess: async () => {
    console.log('Build completed successfully!');
  },
});
```

### 4.2 Rollup Configuration (ทางเลือก)

```typescript
// rollup.config.ts
import { RollupOptions } from 'rollup';
import typescript from '@rollup/plugin-typescript';
import { nodeResolve } from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import terser from '@rollup/plugin-terser';
import dts from 'rollup-plugin-dts';

const isProduction = process.env.NODE_ENV === 'production';

const config: RollupOptions[] = [
  // ESM build
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/index.js',
      format: 'esm',
      sourcemap: true,
    },
    plugins: [
      nodeResolve(),
      commonjs(),
      typescript({
        tsconfig: './tsconfig.build.json',
      }),
      isProduction && terser(),
    ].filter(Boolean),
    external: ['react', 'react-dom', /node_modules/],
  },
  
  // CJS build
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/index.cjs',
      format: 'cjs',
      sourcemap: true,
    },
    plugins: [
      nodeResolve(),
      commonjs(),
      typescript({
        tsconfig: './tsconfig.build.json',
      }),
      isProduction && terser(),
    ].filter(Boolean),
    external: ['react', 'react-dom', /node_modules/],
  },
  
  // Type declarations
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/index.d.ts',
      format: 'esm',
    },
    plugins: [dts()],
  },
];

export default config;
```

---

## 5. Declaration Files (.d.ts)

### 5.1 การเขียน Declaration Files ที่ดี

```typescript
// src/types.ts - Type definitions ที่ครอบคลุม

/** Options สำหรับ DataProcessor */
export interface ProcessorOptions {
  /** ขนาดสูงสุดของ batch ที่ประมวลผลพร้อมกัน @default 100 */
  batchSize?: number;
  
  /** เวลา timeout สำหรับแต่ละ operation (milliseconds) @default 5000 */
  timeout?: number;
  
  /** จำนวนครั้งที่ retry เมื่อเกิดข้อผิดพลาด @default 3 */
  retries?: number;
  
  /** Function ที่เรียกเมื่อเกิดข้อผิดพลาด */
  onError?: (error: Error, item: unknown) => void;
  
  /** เปิดใช้งาน logging หรือไม่ @default false */
  debug?: boolean;
}

/**
 * Transformer function สำหรับแปลงข้อมูล
 * @template T - ประเภทข้อมูลนำเข้า
 * @template R - ประเภทข้อมูลที่ได้หลังแปลง
 */
export type DataTransformer<T, R = T> = (
  item: T,
  index: number,
  array: T[]
) => R | Promise<R>;

/** ผลลัพธ์จากการ validate */
export interface ValidationResult<T = unknown> {
  /** ข้อมูลที่ผ่านการ validate และ transform แล้ว */
  data?: T;
  
  /** รายการ errors ที่พบ */
  errors: ValidationError[];
  
  /** ผ่านการ validate หรือไม่ */
  isValid: boolean;
}

/** ข้อมูล error จากการ validate */
export interface ValidationError {
  /** ชื่อ field ที่มีปัญหา */
  field: string;
  
  /** ข้อความแสดงข้อผิดพลาด */
  message: string;
  
  /** ค่าที่ได้รับ */
  received?: unknown;
  
  /** ค่าที่คาดหวัง */
  expected?: string;
}

/** Options สำหรับ formatter */
export interface FormatterOptions {
  /** locale สำหรับ formatting @default 'en-US' */
  locale?: string;
  
  /** timezone สำหรับ date formatting */
  timezone?: string;
}

// ✅ Utility types ที่เป็นประโยชน์
export type Awaitable<T> = T | Promise<T>;
export type MaybeNull<T> = T | null;
export type MaybeUndefined<T> = T | undefined;
export type Nullable<T> = T | null | undefined;

export type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

export type DeepRequired<T> = T extends object
  ? { [P in keyof T]-?: DeepRequired<T[P]> }
  : T;

export type DeepReadonly<T> = T extends object
  ? { readonly [P in keyof T]: DeepReadonly<T[P]> }
  : T;

export type Prettify<T> = {
  [K in keyof T]: T[K];
} & {};

export type PickOptional<T> = {
  [K in keyof T as undefined extends T[K] ? K : never]?: T[K];
};

export type RequireAtLeastOne<T, Keys extends keyof T = keyof T> = 
  Pick<T, Exclude<keyof T, Keys>> &
  { [K in Keys]-?: Required<Pick<T, K>> & Partial<Pick<T, Exclude<Keys, K>>> }[Keys];
```

### 5.2 Declaration Merging

```typescript
// src/augmentations.ts
// ✅ Declaration merging สำหรับ extending existing types

// Extend Error class
interface Error {
  code?: string;
  statusCode?: number;
  meta?: Record<string, unknown>;
}

// Global augmentation
declare global {
  interface Window {
    myLibrary: {
      version: string;
      config: Record<string, unknown>;
    };
  }
  
  interface Array<T> {
    /** Group array items by key */
    groupBy<K extends string | number>(
      key: (item: T) => K
    ): Record<K, T[]>;
  }
}

// Module augmentation
declare module 'express' {
  interface Request {
    user?: {
      id: string;
      email: string;
      role: string;
    };
    requestId?: string;
  }
}

export {};
```

---

## 6. Bundle กับ tsup และ Rollup

### 6.1 tsup Advanced Configuration

```typescript
// tsup.config.ts (advanced version)
import { defineConfig, Options } from 'tsup';
import { readFileSync } from 'fs';

const pkg = JSON.parse(readFileSync('./package.json', 'utf-8'));

const baseConfig: Options = {
  entry: ['src/index.ts'],
  dts: true,
  sourcemap: true,
  clean: true,
  treeshake: {
    preset: 'smallest',
  },
  
  // External dependencies ที่ไม่ต้อง bundle
  external: [
    ...Object.keys(pkg.peerDependencies ?? {}),
    ...Object.keys(pkg.dependencies ?? {}),
  ],
  
  // esbuild plugins
  esbuildPlugins: [],
  
  // Inject shims สำหรับ ESM/CJS compatibility
  shims: true,
};

export default defineConfig([
  // Browser build (ESM)
  {
    ...baseConfig,
    format: ['esm'],
    platform: 'browser',
    target: ['es2020', 'chrome80', 'safari14', 'firefox80'],
    outDir: 'dist',
    entry: { index: 'src/index.ts' },
  },
  
  // Node.js build (CJS + ESM)
  {
    ...baseConfig,
    format: ['cjs', 'esm'],
    platform: 'node',
    target: ['node16'],
    outDir: 'dist',
    entry: { 'node/index': 'src/node/index.ts' },
  },
  
  // Minified browser build
  {
    ...baseConfig,
    format: ['esm', 'iife'],
    platform: 'browser',
    minify: true,
    globalName: 'MyLibrary',
    outDir: 'dist',
    entry: { 'index.min': 'src/index.ts' },
    dts: false, // ไม่ต้อง gen types สำหรับ minified
  },
]);
```

---

## 7. Versioning Strategy (กลยุทธ์การ Version)

### 7.1 Semantic Versioning

```typescript
// scripts/version-check.ts
import semver from 'semver';
import { readFileSync } from 'fs';

interface ChangeType {
  breaking: boolean;
  features: string[];
  fixes: string[];
  deprecations: string[];
}

function determineNextVersion(
  currentVersion: string,
  changes: ChangeType
): string {
  if (changes.breaking) {
    return semver.inc(currentVersion, 'major') ?? currentVersion;
  }
  
  if (changes.features.length > 0) {
    return semver.inc(currentVersion, 'minor') ?? currentVersion;
  }
  
  if (changes.fixes.length > 0) {
    return semver.inc(currentVersion, 'patch') ?? currentVersion;
  }
  
  return currentVersion;
}

// ✅ Changesets configuration
// .changeset/config.json
const changesetsConfig = {
  "$schema": "https://unpkg.com/@changesets/config@2.3.1/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
};

// ✅ สร้าง changeset ใหม่
// npx changeset
// เลือก package
// เลือกประเภทการเปลี่ยนแปลง (major/minor/patch)
// เพิ่ม description

// ✅ Version bump
// npx changeset version
// commit เพื่อ bump versions

// ✅ Publish
// npx changeset publish
```

### 7.2 Pre-release Versions

```bash
# Alpha release
npm version prerelease --preid=alpha
# => 1.2.3-alpha.0

# Beta release
npm version prerelease --preid=beta
# => 1.2.3-beta.0

# Release candidate
npm version prerelease --preid=rc
# => 1.2.3-rc.0

# Publish prerelease
npm publish --tag beta

# ติดตั้ง prerelease
npm install @myorg/my-library@beta
npm install @myorg/my-library@1.2.3-beta.0
```

---

## 8. Changelogs (บันทึกการเปลี่ยนแปลง)

### 8.1 CHANGELOG.md Format

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- เพิ่ม feature X
- รองรับ TypeScript 5.x

### Changed
- ปรับปรุงประสิทธิภาพของ DataProcessor

### Deprecated
- `legacyMethod()` จะถูกลบในเวอร์ชัน 2.0.0 ใช้ `newMethod()` แทน

### Removed
- ลบ `oldAPI` ที่ deprecated ใน 0.x

### Fixed
- แก้ไข bug ใน validation ที่ทำให้ optional fields ถูก required

### Security
- อัปเดต dependencies เพื่อแก้ security vulnerabilities

## [1.2.0] - 2024-01-15

### Added
- เพิ่ม `formatBytes()` utility function
- รองรับ `ESM` และ `CJS` dual output

### Fixed
- แก้ไข TypeScript types ที่ไม่ถูกต้องสำหรับ generic parameters

## [1.1.0] - 2024-01-01

### Added
- เพิ่ม `createValidator()` function
- เพิ่ม JSDoc comments สำหรับทุก public API

## [1.0.0] - 2023-12-15

### Added
- Initial release
- `DataProcessor` class
- Basic utility functions

[Unreleased]: https://github.com/myorg/my-library/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/myorg/my-library/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/myorg/my-library/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/myorg/my-library/releases/tag/v1.0.0
```

---

## 9. Publishing to NPM (การ Publish ขึ้น NPM)

### 9.1 ขั้นตอนการ Publish

```bash
# 1. สร้าง npm account (ถ้ายังไม่มี)
# https://www.npmjs.com/signup

# 2. Login
npm login

# 3. ตรวจสอบว่า package.json ถูกต้อง
npm run build
npm run test
npm pack --dry-run  # ดูว่าจะ publish ไฟล์อะไรบ้าง

# 4. Publish
npm publish
# สำหรับ scoped package
npm publish --access public

# 5. Tag release ใน git
git tag v1.0.0
git push --tags
```

### 9.2 .npmignore

```
# .npmignore
src/
tests/
examples/
docs/
coverage/
.github/
node_modules/
*.test.ts
*.spec.ts
tsconfig*.json
tsup.config.ts
rollup.config.ts
jest.config.ts
.eslintrc.js
.prettierrc
*.log
.env
.env.*
```

### 9.3 Automated Publishing Script

```typescript
// scripts/publish.ts
import { execSync } from 'child_process';
import { readFileSync } from 'fs';
import path from 'path';

interface PublishOptions {
  tag?: 'latest' | 'beta' | 'alpha' | 'next';
  dryRun?: boolean;
  skipTests?: boolean;
}

async function publish(options: PublishOptions = {}): Promise<void> {
  const { tag = 'latest', dryRun = false, skipTests = false } = options;
  const pkg = JSON.parse(
    readFileSync(path.join(process.cwd(), 'package.json'), 'utf-8')
  );
  
  console.log(`\n🚀 Publishing ${pkg.name}@${pkg.version} (tag: ${tag})`);
  
  if (dryRun) {
    console.log('🔍 Dry run mode - ไม่มีการ publish จริง\n');
  }
  
  // ตรวจสอบว่า branch ถูกต้อง
  const currentBranch = execSync('git rev-parse --abbrev-ref HEAD')
    .toString()
    .trim();
  
  if (tag === 'latest' && currentBranch !== 'main') {
    throw new Error(`Publishing 'latest' ต้องอยู่บน main branch (current: ${currentBranch})`);
  }
  
  // ตรวจสอบว่าไม่มี uncommitted changes
  const status = execSync('git status --porcelain').toString().trim();
  if (status) {
    throw new Error('มี uncommitted changes กรุณา commit ก่อน publish');
  }
  
  // Run tests
  if (!skipTests) {
    console.log('🧪 Running tests...');
    execSync('npm test', { stdio: 'inherit' });
  }
  
  // Type check
  console.log('🔍 Running type check...');
  execSync('npm run typecheck', { stdio: 'inherit' });
  
  // Build
  console.log('🔨 Building...');
  execSync('npm run build', { stdio: 'inherit' });
  
  // Publish
  const publishCmd = dryRun
    ? `npm publish --tag ${tag} --dry-run`
    : `npm publish --tag ${tag}`;
  
  console.log(`\n📦 ${dryRun ? '[DRY RUN] ' : ''}Running: ${publishCmd}`);
  execSync(publishCmd, { stdio: 'inherit' });
  
  if (!dryRun) {
    // Tag ใน git
    execSync(`git tag v${pkg.version}`);
    execSync('git push --tags');
    
    console.log(`\n✅ Published ${pkg.name}@${pkg.version} successfully!`);
    console.log(`   npm: https://www.npmjs.com/package/${pkg.name}`);
  }
}

// รัน script
const args = process.argv.slice(2);
const isDryRun = args.includes('--dry-run');
const tag = args.find(a => a.startsWith('--tag='))?.split('=')[1] as
  | 'latest' | 'beta' | 'alpha' | 'next'
  | undefined;

publish({ tag, dryRun: isDryRun }).catch(error => {
  console.error('❌ Publish failed:', error.message);
  process.exit(1);
});
```

---

## 10. Documentation กับ TypeDoc

### 10.1 TypeDoc Configuration

```typescript
// typedoc.config.ts
import { TypeDocOptions } from 'typedoc';

const config: Partial<TypeDocOptions> = {
  entryPoints: ['src/index.ts'],
  out: 'docs',
  
  // Theme
  theme: 'default',
  
  // Exclusions
  excludePrivate: true,
  excludeProtected: false,
  excludeInternal: true,
  excludeExternals: true,
  
  // Navigation
  navigation: {
    includeCategories: true,
    includeGroups: true,
  },
  
  // Plugin options
  hideGenerator: true,
  
  // README
  readme: 'README.md',
  
  // Categorize by @category tag
  categorizeByGroup: true,
  
  // Plugin: typedoc-plugin-markdown สำหรับ output เป็น Markdown
  // plugin: ['typedoc-plugin-markdown'],
};

export default config;
```

### 10.2 JSDoc Comments ที่ดี

```typescript
/**
 * DataProcessor class สำหรับประมวลผลข้อมูลแบบ batch
 * 
 * @example
 * ```typescript
 * const processor = new DataProcessor<string, number>({
 *   batchSize: 50,
 *   timeout: 3000,
 * });
 * 
 * const results = await processor.process(
 *   ['a', 'bb', 'ccc'],
 *   item => item.length
 * );
 * // results: [1, 2, 3]
 * ```
 * 
 * @category Core
 * @public
 */
export class DataProcessor<T, R = T> {
  private readonly options: Required<ProcessorOptions>;
  
  /**
   * สร้าง DataProcessor ใหม่
   * 
   * @param options - การตั้งค่าสำหรับ processor
   * @throws {TypeError} ถ้า batchSize น้อยกว่าหรือเท่ากับ 0
   */
  constructor(options: ProcessorOptions = {}) {
    if ((options.batchSize ?? 1) <= 0) {
      throw new TypeError('batchSize ต้องมากกว่า 0');
    }
    
    this.options = {
      batchSize: 100,
      timeout: 5000,
      retries: 3,
      onError: (error) => console.error(error),
      debug: false,
      ...options,
    };
  }
  
  /**
   * ประมวลผลข้อมูลทั้งหมดด้วย transformer function
   * 
   * @param items - ข้อมูลที่ต้องการประมวลผล
   * @param transformer - Function สำหรับแปลงข้อมูลแต่ละรายการ
   * @returns Promise ที่ resolve เป็น array ของผลลัพธ์
   * 
   * @example
   * ```typescript
   * // ประมวลผล synchronously
   * const doubled = await processor.process([1, 2, 3], x => x * 2);
   * 
   * // ประมวลผล asynchronously
   * const fetched = await processor.process(
   *   userIds,
   *   async (id) => {
   *     const response = await fetch(`/api/users/${id}`);
   *     return response.json();
   *   }
   * );
   * ```
   * 
   * @throws {ProcessingError} ถ้าเกิดข้อผิดพลาดที่ไม่สามารถ recover ได้
   */
  async process(
    items: T[],
    transformer: DataTransformer<T, R>
  ): Promise<R[]> {
    const results: R[] = [];
    
    for (let i = 0; i < items.length; i += this.options.batchSize) {
      const batch = items.slice(i, i + this.options.batchSize);
      const batchResults = await this.processBatch(batch, transformer, i);
      results.push(...batchResults);
    }
    
    return results;
  }
  
  /**
   * ประมวลผล batch เดียว
   * @internal
   */
  private async processBatch(
    batch: T[],
    transformer: DataTransformer<T, R>,
    startIndex: number
  ): Promise<R[]> {
    return Promise.all(
      batch.map((item, idx) => 
        this.withTimeout(
          () => transformer(item, startIndex + idx, batch),
          this.options.timeout
        )
      )
    );
  }
  
  private async withTimeout<V>(
    fn: () => Awaitable<V>,
    timeout: number
  ): Promise<V> {
    return new Promise<V>((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`Operation timed out after ${timeout}ms`));
      }, timeout);
      
      Promise.resolve(fn())
        .then(result => {
          clearTimeout(timer);
          resolve(result);
        })
        .catch(error => {
          clearTimeout(timer);
          reject(error);
        });
    });
  }
}

type Awaitable<T> = T | Promise<T>;
```

---

## 11. Writing Tests สำหรับ Libraries

### 11.1 Jest Configuration

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  
  // Test patterns
  testMatch: [
    '<rootDir>/tests/**/*.test.ts',
    '<rootDir>/src/**/*.test.ts',
  ],
  
  // Coverage
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
    '!src/index.ts', // Entry point usually just re-exports
  ],
  
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 85,
      lines: 85,
      statements: 85,
    },
  },
  
  // TypeScript config
  transform: {
    '^.+\\.tsx?$': [
      'ts-jest',
      {
        tsconfig: 'tsconfig.test.json',
      },
    ],
  },
  
  // Setup files
  setupFilesAfterFramework: ['<rootDir>/tests/setup.ts'],
  
  // Aliases
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  
  // Performance
  maxWorkers: '50%',
  
  // Verbose output
  verbose: true,
};

export default config;
```

### 11.2 Comprehensive Tests

```typescript
// tests/unit/processor.test.ts
import { describe, it, expect, jest, beforeEach, afterEach } from '@jest/globals';
import { DataProcessor } from '../../src/core/processor';
import type { ProcessorOptions } from '../../src/types';

describe('DataProcessor', () => {
  let processor: DataProcessor<number, number>;
  
  beforeEach(() => {
    processor = new DataProcessor<number, number>({
      batchSize: 3,
      timeout: 1000,
    });
  });
  
  afterEach(() => {
    jest.clearAllMocks();
  });
  
  describe('constructor', () => {
    it('ควร throw TypeError เมื่อ batchSize น้อยกว่าหรือเท่ากับ 0', () => {
      expect(() => new DataProcessor({ batchSize: 0 }))
        .toThrow(TypeError);
      expect(() => new DataProcessor({ batchSize: -1 }))
        .toThrow(TypeError);
    });
    
    it('ควร create ด้วย default options', () => {
      const defaultProcessor = new DataProcessor();
      expect(defaultProcessor).toBeDefined();
    });
  });
  
  describe('process', () => {
    it('ควร transform items ทั้งหมด', async () => {
      const input = [1, 2, 3, 4, 5];
      const result = await processor.process(input, x => x * 2);
      expect(result).toEqual([2, 4, 6, 8, 10]);
    });
    
    it('ควรรองรับ async transformers', async () => {
      const input = [1, 2, 3];
      const result = await processor.process(
        input,
        async x => {
          await new Promise(resolve => setTimeout(resolve, 10));
          return x + 10;
        }
      );
      expect(result).toEqual([11, 12, 13]);
    });
    
    it('ควรประมวลผล empty array', async () => {
      const result = await processor.process([], x => x);
      expect(result).toEqual([]);
    });
    
    it('ควร pass index และ array ไปยัง transformer', async () => {
      const indices: number[] = [];
      const arrays: number[][] = [];
      
      await processor.process(
        [10, 20, 30],
        (item, index, array) => {
          indices.push(index);
          arrays.push(array);
          return item;
        }
      );
      
      expect(indices).toEqual([0, 1, 2]);
      expect(arrays[0]).toEqual([10, 20, 30]);
    });
    
    it('ควร throw เมื่อ timeout', async () => {
      const slowProcessor = new DataProcessor<number, number>({
        timeout: 50,
      });
      
      await expect(
        slowProcessor.process(
          [1],
          () => new Promise(resolve => setTimeout(() => resolve(1), 200))
        )
      ).rejects.toThrow('timed out');
    });
    
    it('ควรประมวลผลเป็น batch ที่ถูกต้อง', async () => {
      const batchSizes: number[] = [];
      
      await processor.process(
        [1, 2, 3, 4, 5, 6, 7],
        (item, index, array) => {
          // บันทึก batch size แต่ละครั้ง
          if (index % 3 === 0) {
            batchSizes.push(array.length);
          }
          return item;
        }
      );
      
      // ควรมี 3 batches: [1,2,3], [4,5,6], [7]
      expect(batchSizes).toHaveLength(3);
    });
  });
  
  describe('type safety', () => {
    it('ควรรักษา type ผ่าน generic', async () => {
      const stringProcessor = new DataProcessor<string, number>();
      const result = await stringProcessor.process(
        ['hello', 'world'],
        str => str.length
      );
      
      // TypeScript ควร infer ว่า result เป็น number[]
      const sum: number = result.reduce((a, b) => a + b, 0);
      expect(sum).toBe(10);
    });
  });
});

// tests/integration/index.test.ts
import { describe, it, expect } from '@jest/globals';
import * as Library from '../../src/index';

describe('Library exports', () => {
  it('ควร export DataProcessor', () => {
    expect(Library.DataProcessor).toBeDefined();
    expect(typeof Library.DataProcessor).toBe('function');
  });
  
  it('ควร export utility functions', () => {
    expect(Library.formatDate).toBeDefined();
    expect(Library.formatCurrency).toBeDefined();
  });
  
  it('ควร export types', () => {
    // Type exports ไม่มี runtime value แต่ import ได้
    type TestType = Library.ProcessorOptions;
    const options: TestType = { batchSize: 10 };
    expect(options.batchSize).toBe(10);
  });
});
```

---

## 12. Peer Dependencies (การจัดการ Peer Dependencies)

### 12.1 การกำหนด Peer Dependencies

```json
{
  "peerDependencies": {
    "react": ">=17.0.0",
    "react-dom": ">=17.0.0",
    "typescript": ">=4.9.0"
  },
  "peerDependenciesMeta": {
    "react": {
      "optional": true
    },
    "react-dom": {
      "optional": true
    },
    "typescript": {
      "optional": true
    }
  },
  "dependencies": {
    "lodash-es": "^4.17.21"
  },
  "devDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "typescript": "^5.0.0"
  }
}
```

```typescript
// src/react/index.ts - React integration (conditional)
// ตรวจสอบว่า React มีให้ใช้หรือเปล่า
let React: typeof import('react') | null = null;

try {
  React = require('react');
} catch {
  // React is optional
}

export function isReactAvailable(): boolean {
  return React !== null;
}

// ✅ Guard สำหรับ optional peer dependencies
export function createReactComponent<P extends object>(
  componentFn: (props: P) => unknown
): unknown {
  if (!React) {
    throw new Error(
      'React is required for this feature. ' +
      'Install it with: npm install react'
    );
  }
  return componentFn;
}
```

---

## 13. Tree-shaking Friendly Code

### 13.1 การเขียนโค้ดที่ Tree-shakeable

```typescript
// ✅ Named exports แทน default exports
export function formatDate(date: Date, locale = 'th-TH'): string {
  return new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(date);
}

export function formatCurrency(
  amount: number,
  currency = 'THB',
  locale = 'th-TH'
): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(amount);
}

export function formatBytes(bytes: number, decimals = 2): string {
  if (bytes === 0) return '0 Bytes';
  
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  
  return `${parseFloat((bytes / Math.pow(k, i)).toFixed(decimals))} ${sizes[i]}`;
}

// ❌ Class methods ไม่ tree-shakeable เพราะต้อง import ทั้ง class
class Formatters {
  formatDate(date: Date): string { return date.toISOString(); }
  formatCurrency(amount: number): string { return `$${amount}`; }
  formatBytes(bytes: number): string { return `${bytes} B`; }
}

// ✅ ใช้ functions แทน
// ผู้ใช้ import เฉพาะที่ต้องการ
import { formatDate } from 'my-library';
// formatCurrency ไม่ถูก bundle!

// ✅ Conditional feature loading
export const features = {
  // Getter ที่ load lazily
  get charts() {
    return import('./features/charts').then(m => m.default);
  },
  get maps() {
    return import('./features/maps').then(m => m.default);
  },
};
```

---

## 14. README กับ TypeScript Examples

### 14.1 README.md Template

````markdown
# my-typescript-library

[![npm version](https://badge.fury.io/js/@myorg%2Fmy-library.svg)](https://badge.fury.io/js/@myorg%2Fmy-library)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue)](https://www.typescriptlang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![CI](https://github.com/myorg/my-library/actions/workflows/ci.yml/badge.svg)](https://github.com/myorg/my-library/actions)

> Brief description ของ library ในไทย

## Installation

```bash
npm install @myorg/my-library
# หรือ
yarn add @myorg/my-library
# หรือ
pnpm add @myorg/my-library
```

## Quick Start

```typescript
import { DataProcessor, formatDate } from '@myorg/my-library';

// สร้าง processor
const processor = new DataProcessor<string, number>({
  batchSize: 50,
});

// ประมวลผลข้อมูล
const results = await processor.process(
  ['hello', 'world', 'typescript'],
  item => item.length
);

console.log(results); // [5, 5, 10]

// Format utilities
const formattedDate = formatDate(new Date(), 'th-TH');
console.log(formattedDate); // "15 มกราคม 2567"
```

## API Reference

### DataProcessor

```typescript
class DataProcessor<T, R = T> {
  constructor(options?: ProcessorOptions);
  process(items: T[], transformer: DataTransformer<T, R>): Promise<R[]>;
}
```

### Types

```typescript
interface ProcessorOptions {
  batchSize?: number;  // default: 100
  timeout?: number;    // default: 5000ms
  retries?: number;    // default: 3
}

type DataTransformer<T, R = T> = (
  item: T,
  index: number,
  array: T[]
) => R | Promise<R>;
```

## Examples

### Basic Processing

```typescript
import { DataProcessor } from '@myorg/my-library';

const processor = new DataProcessor<number, string>();

const strings = await processor.process(
  [1, 2, 3, 4, 5],
  num => num.toString()
);
// ['1', '2', '3', '4', '5']
```

### Async Processing

```typescript
import { DataProcessor } from '@myorg/my-library';

interface User { id: number; name: string }

const userProcessor = new DataProcessor<number, User>({
  batchSize: 10,
  timeout: 3000,
});

const users = await userProcessor.process(
  [1, 2, 3],
  async (id) => {
    const response = await fetch(`https://api.example.com/users/${id}`);
    return response.json();
  }
);
```

## Contributing

ดู [CONTRIBUTING.md](CONTRIBUTING.md) สำหรับรายละเอียดการมีส่วนร่วม

## License

MIT © [Your Name](https://github.com/yourname)
````

---

## 15. GitHub Actions สำหรับ Publishing

### 15.1 CI/CD Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Type check
        run: npm run typecheck
      
      - name: Lint
        run: npm run lint
      
      - name: Test
        run: npm run test:coverage
      
      - name: Upload coverage
        if: matrix.node-version == '18.x'
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
      
      - name: Build
        run: npm run build

  # .github/workflows/publish.yml
```

```yaml
# .github/workflows/publish.yml
name: Publish to NPM

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    name: Publish
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      id-token: write  # สำหรับ npm provenance
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Verify package version matches tag
        run: |
          PKG_VERSION=$(node -p "require('./package.json').version")
          TAG_VERSION=${{ steps.version.outputs.VERSION }}
          if [ "$PKG_VERSION" != "$TAG_VERSION" ]; then
            echo "Version mismatch: package.json ($PKG_VERSION) vs tag ($TAG_VERSION)"
            exit 1
          fi
      
      - name: Publish to NPM
        run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.VERSION }}
          generate_release_notes: true
          files: |
            dist/*.js
            dist/*.cjs
            dist/*.d.ts
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```yaml
# .github/workflows/release.yml (ด้วย changesets)
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Create Release Pull Request or Publish
        uses: changesets/action@v1
        with:
          publish: npm run release
          version: npm run version
          commit: "chore: version packages"
          title: "chore: version packages"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## สรุปโดยรวม

ในบทนี้เราได้เรียนรู้การสร้าง TypeScript Library ที่สมบูรณ์ครอบคลุม:

1. **Library Project Setup** - โครงสร้าง directory ที่ดี, การเริ่มต้นโปรเจค
2. **Package.json** - การตั้งค่า exports, files, peerDependencies
3. **tsconfig** - การตั้งค่าสำหรับ build, test
4. **Dual CJS/ESM Output** - tsup และ rollup configuration
5. **Declaration Files** - การเขียน .d.ts ที่ดี, declaration merging
6. **Bundle** - tsup advanced configuration
7. **Versioning** - Semantic Versioning, changesets
8. **Changelogs** - CHANGELOG.md format
9. **Publishing** - npm publish, automated scripts
10. **TypeDoc** - Documentation generation, JSDoc comments
11. **Testing** - Jest configuration, comprehensive tests
12. **Peer Dependencies** - การจัดการ optional dependencies
13. **Tree-shaking** - Named exports, lazy loading
14. **README** - Documentation template กับตัวอย่าง TypeScript
15. **GitHub Actions** - CI/CD workflows

การสร้าง library ที่ดีต้องการความใส่ใจในรายละเอียดทุกด้าน โดยเฉพาะ TypeScript types ที่ชัดเจน, documentation ที่ครอบคลุม และ testing ที่ครอบคลุม

---

*จบ Series TypeScript Course*
