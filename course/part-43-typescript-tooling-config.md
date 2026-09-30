# ตอนที่ 43: TypeScript Tooling และ Configuration

## บทนำ

การตั้งค่าและเครื่องมือที่ถูกต้องสำหรับ TypeScript ช่วยเพิ่มประสิทธิภาพการพัฒนา ลดข้อผิดพลาด และทำให้ codebase มีมาตรฐาน บทนี้ครอบคลุมทุกอย่างตั้งแต่ tsconfig.json จนถึง CI/CD

---

## 43.1 tsconfig.json Complete Reference

### ตัวอย่าง tsconfig.json สมบูรณ์

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    // === Target และ Module ===
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022", "DOM"],

    // === Output ===
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationDir": "./dist/types",
    "declarationMap": true,
    "sourceMap": true,
    "inlineSources": false,
    "removeComments": false,

    // === Strict Mode (แนะนำให้เปิดทั้งหมด) ===
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,
    "alwaysStrict": true,

    // === Additional Checks ===
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,

    // === Module ===
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "allowImportingTsExtensions": false,
    "resolveJsonModule": true,
    "isolatedModules": true,

    // === Path Mappings ===
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@shared/*": ["./src/shared/*"],
      "@utils/*": ["./src/utils/*"],
      "@types/*": ["./src/types/*"]
    },

    // === Experimental ===
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,

    // === Build Performance ===
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo",
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

### Compiler Options อธิบาย

```typescript
// ตัวอย่างผลของ compiler options ต่างๆ

// noImplicitAny: true
// ❌ Error - parameter 'data' implicitly has type 'any'
function process(data) { return data; }

// ✅ Good
function process(data: unknown) { return data; }

// strictNullChecks: true
// ❌ Error - Object is possibly null
const user: User | null = getUser();
console.log(user.name); // Error!

// ✅ Good
if (user !== null) {
  console.log(user.name);
}

// noUncheckedIndexedAccess: true
const arr = [1, 2, 3];
// ❌ arr[10] เป็น number | undefined
const value = arr[10]; // type: number | undefined
if (value !== undefined) {
  console.log(value + 1); // ✅
}

// exactOptionalPropertyTypes: true
interface Config {
  timeout?: number;
}

const config: Config = {};
// ❌ Error - cannot assign undefined to optional property
config.timeout = undefined; // Error with exactOptionalPropertyTypes!

// noImplicitOverride: true
class Base {
  greet() { return 'Hello'; }
}
class Child extends Base {
  // ❌ Error - Method overrides base class but missing 'override'
  greet() { return 'Hi'; }

  // ✅ Good
  override greet() { return 'Hi'; }
}
```

---

## 43.2 Strict Mode Settings

```json
{
  "compilerOptions": {
    // เปิด strict ทั้งหมดด้วย flag เดียว
    "strict": true,

    // หรือเปิดทีละ option
    "noImplicitAny": true,          // ไม่อนุญาต implicit any
    "strictNullChecks": true,        // null/undefined ต้องตรวจสอบ
    "strictFunctionTypes": true,     // Function type checking ที่เข้มงวด
    "strictBindCallApply": true,     // bind/call/apply type checking
    "strictPropertyInitialization": true, // Class properties ต้อง initialize
    "noImplicitThis": true,          // this ต้องมี type ชัดเจน
    "alwaysStrict": true             // เพิ่ม "use strict" ใน output
  }
}
```

### ตัวอย่าง Strict Function Types

```typescript
// strictFunctionTypes: true

// Parameter types contravariant
type EventHandler = (event: MouseEvent) => void;
type UIEventHandler = (event: UIEvent) => void;

const handler: UIEventHandler = (e) => console.log(e.target);

// ❌ MouseEvent extends UIEvent, แต่ handler ไม่รับ MouseEvent
const mouseHandler: EventHandler = handler; // Error!

// ✅ UIEvent เป็น supertype ของ MouseEvent
const mouseHandler2: EventHandler = (e: MouseEvent) => {};
```

---

## 43.3 Module Resolution

### NodeNext Resolution

```json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  }
}
```

```typescript
// ใช้ NodeNext ต้องระบุ extension
import { helper } from './utils/helper.js'; // ต้องมี .js
import type { Config } from './types/config.js';

// Named imports จาก package.json exports
import { createServer } from 'node:http';
```

### Bundler Resolution (สำหรับ Vite, Webpack)

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

```typescript
// ไม่ต้องมี extension
import { helper } from './utils/helper'; // ✅
import styles from './styles.css'; // ✅ (bundler handles this)
```

---

## 43.4 Path Mappings

### การตั้งค่า Path Aliases

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"],
      "@utils/*": ["src/utils/*"],
      "@api/*": ["src/api/*"],
      "@types/*": ["src/types/*"],
      "@constants": ["src/constants/index"]
    }
  }
}
```

```typescript
// ใช้งาน Path Aliases
import { Button } from '@components/Button';
import { useAuth } from '@hooks/useAuth';
import { formatDate } from '@utils/date';
import type { ApiResponse } from '@types/api';
```

### Webpack กับ Path Aliases

```javascript
// webpack.config.js
const path = require('path');
const { pathsToModuleNameMapper } = require('ts-jest');
const { compilerOptions } = require('./tsconfig.json');

module.exports = {
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
      '@hooks': path.resolve(__dirname, 'src/hooks'),
    },
    extensions: ['.ts', '.tsx', '.js', '.jsx'],
  },
};
```

### Vite กับ Path Aliases

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import path from 'path';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

---

## 43.5 Project References

### Monorepo Setup กับ Project References

```
packages/
├── shared/
│   ├── tsconfig.json
│   └── src/
├── server/
│   ├── tsconfig.json
│   └── src/
└── client/
    ├── tsconfig.json
    └── src/
tsconfig.json (root)
```

```json
// tsconfig.json (root)
{
  "files": [],
  "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/server" },
    { "path": "./packages/client" }
  ]
}
```

```json
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

```json
// packages/server/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../shared" }
  ],
  "include": ["src/**/*"]
}
```

### Build กับ Project References

```bash
# Build ทั้งหมดตาม dependency order
tsc --build

# Watch mode
tsc --build --watch

# Clean build
tsc --build --clean

# Force rebuild
tsc --build --force
```

---

## 43.6 ESLint กับ TypeScript

### การติดตั้ง

```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D eslint-config-prettier eslint-plugin-prettier
npm install -D eslint-plugin-import eslint-plugin-unused-imports
```

### eslint.config.ts (Flat Config)

```typescript
// eslint.config.ts
import type { Linter } from 'eslint';
import tseslint from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import prettier from 'eslint-config-prettier';

const config: Linter.FlatConfig[] = [
  {
    files: ['**/*.ts', '**/*.tsx'],
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        project: './tsconfig.json',
        ecmaVersion: 'latest',
        sourceType: 'module',
      },
    },
    plugins: {
      '@typescript-eslint': tseslint,
    },
    rules: {
      // TypeScript-specific rules
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      '@typescript-eslint/explicit-function-return-type': 'off',
      '@typescript-eslint/explicit-module-boundary-types': 'off',
      '@typescript-eslint/no-non-null-assertion': 'warn',
      '@typescript-eslint/prefer-nullish-coalescing': 'error',
      '@typescript-eslint/prefer-optional-chain': 'error',
      '@typescript-eslint/no-unnecessary-type-assertion': 'error',
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/await-thenable': 'error',
      '@typescript-eslint/require-await': 'error',
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'separate-type-imports',
      }],
      '@typescript-eslint/consistent-type-exports': 'error',
      '@typescript-eslint/no-import-type-side-effects': 'error',

      // General rules
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
      'no-var': 'error',
      'eqeqeq': ['error', 'always'],
      'curly': ['error', 'all'],
    },
  },
  prettier,
];

export default config;
```

---

## 43.7 ESLint Rules อธิบาย

```typescript
// @typescript-eslint/no-explicit-any
// ❌ Avoid
function process(data: any) { }

// ✅ Better
function process(data: unknown) {
  if (typeof data === 'string') {
    return data.toUpperCase();
  }
}

// @typescript-eslint/prefer-nullish-coalescing
// ❌ Avoid (falsy check ครอบเกิน)
const value = input || 'default'; // 0 หรือ '' จะได้ 'default'

// ✅ Better (null/undefined เท่านั้น)
const value = input ?? 'default';

// @typescript-eslint/prefer-optional-chain
// ❌ Avoid
const city = user && user.address && user.address.city;

// ✅ Better
const city = user?.address?.city;

// @typescript-eslint/no-floating-promises
// ❌ Avoid
async function main() {
  fetchData(); // Promise ถูกทิ้งโดยไม่รอ
}

// ✅ Better
async function main() {
  await fetchData();
  // หรือ
  fetchData().catch(console.error);
}

// @typescript-eslint/consistent-type-imports
// ❌ Avoid
import { User } from './types';
import { ApiResponse } from './api';

// ✅ Better
import type { User } from './types';
import type { ApiResponse } from './api';
```

---

## 43.8 Custom ESLint Rules

```typescript
// eslint-rules/no-direct-db-access.ts
import { Rule } from 'eslint';

// Custom rule: ห้าม import database ตรงๆ ใน controllers
const noDatabaseImport: Rule.RuleModule = {
  meta: {
    type: 'problem',
    docs: {
      description: 'Controllers ไม่ควร import database โดยตรง',
      recommended: true,
    },
    messages: {
      noDirectDbAccess:
        'ใช้ Repository/Service pattern แทนการ import database ตรงๆ',
    },
  },

  create(context) {
    const filename = context.getFilename();
    const isController = filename.includes('.controller.');

    return {
      ImportDeclaration(node) {
        if (!isController) return;

        const source = node.source.value as string;
        const forbiddenPatterns = [
          /typeorm/,
          /mongoose/,
          /prisma/,
          /knex/,
        ];

        if (forbiddenPatterns.some(pattern => pattern.test(source))) {
          context.report({
            node,
            messageId: 'noDirectDbAccess',
          });
        }
      },
    };
  },
};

export default noDatabaseImport;
```

```typescript
// eslint-rules/require-error-handling.ts
const requireErrorHandling: Rule.RuleModule = {
  meta: {
    type: 'suggestion',
    docs: {
      description: 'Async functions ควรมี error handling',
    },
    messages: {
      missingTryCatch: 'Async function ควรใช้ try-catch หรือ .catch()',
    },
  },

  create(context) {
    return {
      'FunctionDeclaration[async=true]': function (node: any) {
        const hasAwait = node.body.body.some(
          (stmt: any) =>
            stmt.type === 'ExpressionStatement' &&
            stmt.expression.type === 'AwaitExpression'
        );

        const hasTryCatch = node.body.body.some(
          (stmt: any) => stmt.type === 'TryStatement'
        );

        if (hasAwait && !hasTryCatch) {
          context.report({ node, messageId: 'missingTryCatch' });
        }
      },
    };
  },
};
```

---

## 43.9 Prettier Integration

### .prettierrc.json

```json
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "endOfLine": "lf",
  "overrides": [
    {
      "files": "*.json",
      "options": {
        "printWidth": 80
      }
    },
    {
      "files": "*.md",
      "options": {
        "proseWrap": "always",
        "printWidth": 80
      }
    }
  ]
}
```

### .prettierignore

```
# Build outputs
dist/
build/
out/

# Dependencies
node_modules/

# Generated files
*.generated.ts
*.d.ts

# Config files
coverage/
.next/
.nuxt/
```

---

## 43.10 Husky และ lint-staged

### การตั้งค่า Husky

```bash
npm install -D husky lint-staged
npx husky init
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged
```

```bash
# .husky/commit-msg
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx --no -- commitlint --edit "$1"
```

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run type-check && npm run test:ci
```

### lint-staged Configuration

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "tsc-files --noEmit"
    ],
    "*.{js,jsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"
    ]
  }
}
```

### Commitlint Configuration

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // Feature ใหม่
        'fix',      // Bug fix
        'docs',     // เอกสาร
        'style',    // Format, whitespace
        'refactor', // Code refactoring
        'perf',     // Performance
        'test',     // Tests
        'build',    // Build system
        'ci',       // CI configuration
        'chore',    // Other changes
        'revert',   // Revert commit
      ],
    ],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 200],
  },
};
```

---

## 43.11 GitHub Actions สำหรับ TypeScript

### CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20.x'
  CACHE_KEY: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check Prettier
        run: npm run format:check

  type-check:
    name: Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: TypeScript type check
        run: npm run type-check

  test:
    name: Test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, type-check, test]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7
```

### Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Generate changelog
        id: changelog
        uses: conventional-changelog-action@v3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          body: ${{ steps.changelog.outputs.clean_changelog }}
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Publish to NPM
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 43.12 Bundle Analysis

### Webpack Bundle Analyzer

```bash
npm install -D webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    process.env.ANALYZE === 'true' && new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json',
    }),
  ].filter(Boolean),
};
```

```bash
# รัน analysis
ANALYZE=true npm run build
```

### Rollup-plugin-visualizer

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      filename: 'stats.html',
      open: true,
      gzipSize: true,
      brotliSize: true,
      template: 'treemap', // 'sunburst' | 'treemap' | 'network'
    }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash', 'date-fns'],
        },
      },
    },
  },
});
```

---

## 43.13 Performance Profiling

### TypeScript Compiler Performance

```bash
# ดู compilation time
tsc --diagnostics

# Verbose output
tsc --extendedDiagnostics

# ดู files ที่ compile
tsc --listFiles

# ดู trace
tsc --generateTrace ./trace-output
```

```typescript
// tsconfig.json - ปรับ performance
{
  "compilerOptions": {
    // เปิด incremental build
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",

    // ข้าม type checking สำหรับ .d.ts ใน node_modules
    "skipLibCheck": true,

    // ไม่ emit เมื่อมี errors
    "noEmitOnError": true,

    // ลด work สำหรับ editor
    "disableSourceOfProjectReferenceRedirect": true
  }
}
```

### Runtime Performance Profiling

```typescript
// src/profiling/performance.ts

export class PerformanceProfiler {
  private marks = new Map<string, number>();
  private measures: Array<{
    name: string;
    duration: number;
    timestamp: Date;
  }> = [];

  mark(name: string): void {
    this.marks.set(name, performance.now());
  }

  measure(name: string, startMark: string, endMark?: string): number {
    const start = this.marks.get(startMark);
    if (!start) throw new Error(`Mark '${startMark}' not found`);

    const end = endMark ? this.marks.get(endMark) ?? performance.now() : performance.now();
    const duration = end - start;

    this.measures.push({
      name,
      duration,
      timestamp: new Date(),
    });

    return duration;
  }

  async profile<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const start = performance.now();
    try {
      const result = await fn();
      const duration = performance.now() - start;

      this.measures.push({ name, duration, timestamp: new Date() });
      console.log(`[Profile] ${name}: ${duration.toFixed(2)}ms`);

      return result;
    } catch (error) {
      const duration = performance.now() - start;
      console.error(`[Profile] ${name} failed after ${duration.toFixed(2)}ms`);
      throw error;
    }
  }

  getReport(): Record<string, { avg: number; min: number; max: number; count: number }> {
    const groups = new Map<string, number[]>();

    this.measures.forEach(m => {
      if (!groups.has(m.name)) groups.set(m.name, []);
      groups.get(m.name)!.push(m.duration);
    });

    const report: Record<string, any> = {};

    groups.forEach((durations, name) => {
      report[name] = {
        avg: durations.reduce((a, b) => a + b, 0) / durations.length,
        min: Math.min(...durations),
        max: Math.max(...durations),
        count: durations.length,
      };
    });

    return report;
  }

  clear(): void {
    this.marks.clear();
    this.measures = [];
  }
}

export const profiler = new PerformanceProfiler();
```

---

## 43.14 Declaration File Generation

### การสร้าง .d.ts Files

```json
// tsconfig.lib.json (สำหรับ library)
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "emitDeclarationOnly": true,
    "outDir": "./dist/types"
  },
  "exclude": [
    "**/*.test.ts",
    "**/*.spec.ts",
    "**/__tests__/**"
  ]
}
```

### Manual Declaration Files

```typescript
// src/types/declarations.d.ts

// Module declarations
declare module '*.svg' {
  const content: string;
  export default content;
}

declare module '*.png' {
  const content: string;
  export default content;
}

declare module '*.css' {
  const styles: { [className: string]: string };
  export default styles;
}

declare module '*.json' {
  const value: any;
  export default value;
}

// Global declarations
declare global {
  interface Window {
    __APP_CONFIG__: {
      apiUrl: string;
      version: string;
      environment: 'development' | 'production' | 'test';
    };
    analytics: {
      track(event: string, properties?: Record<string, any>): void;
      identify(userId: string, traits?: Record<string, any>): void;
    };
  }

  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: 'development' | 'production' | 'test';
      PORT?: string;
      DATABASE_URL: string;
      JWT_SECRET: string;
      REDIS_URL?: string;
    }
  }
}

// Ambient module declarations
declare module 'some-untyped-package' {
  export function doSomething(input: string): void;
  export interface Config {
    option1: boolean;
    option2?: string;
  }
  export default class SomeClass {
    constructor(config: Config);
    method(): Promise<void>;
  }
}
```

---

## 43.15 Complete Package.json Scripts

```json
{
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "build:watch": "tsc -p tsconfig.build.json --watch",
    "type-check": "tsc --noEmit",
    "type-check:watch": "tsc --noEmit --watch",
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,json,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,json,md}\"",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --reporters=default --reporters=jest-junit",
    "clean": "rimraf dist .tsbuildinfo coverage",
    "prepare": "husky",
    "postinstall": "husky"
  }
}
```

---

## 43.16 Jest Configuration กับ TypeScript

```javascript
// jest.config.js
/** @type {import('jest').Config} */
const config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: [
    '**/__tests__/**/*.ts',
    '**/*.spec.ts',
    '**/*.test.ts',
  ],
  transform: {
    '^.+\\.tsx?$': ['ts-jest', {
      tsconfig: 'tsconfig.test.json',
    }],
  },
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@types/(.*)$': '<rootDir>/src/types/$1',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.test.ts',
    '!src/**/index.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  setupFilesAfterFramework: ['<rootDir>/src/test/setup.ts'],
};

module.exports = config;
```

---

## 43.17 tsconfig สำหรับ Environments ต่างๆ

### Node.js Backend

```json
// tsconfig.json (Node.js)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "sourceMap": true,
    "declaration": true
  }
}
```

### React Frontend

```json
// tsconfig.json (React)
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### Deno

```json
{
  "compilerOptions": {
    "lib": ["deno.window"],
    "strict": true
  },
  "importMap": "./import_map.json"
}
```

---

## 43.18 เครื่องมือเพิ่มเติม

### ts-node

```json
// tsconfig.json
{
  "ts-node": {
    "esm": true,
    "experimentalSpecifierResolution": "node",
    "files": true,
    "transpileOnly": false,
    "swc": true
  }
}
```

```bash
# รัน TypeScript โดยตรง
ts-node src/index.ts

# Watch mode
ts-node-dev --respawn --transpile-only src/index.ts
```

### tsx (เร็วกว่า ts-node)

```bash
npm install -D tsx

# รัน TypeScript
tsx src/index.ts

# Watch mode
tsx watch src/index.ts
```

### tsc-alias (Path resolution หลัง build)

```bash
npm install -D tsc-alias
```

```json
{
  "scripts": {
    "build": "tsc && tsc-alias"
  }
}
```

---

## 43.19 VS Code Settings

```json
// .vscode/settings.json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "never",
    "source.addMissingImports": "explicit"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "typescript.preferences.importModuleSpecifier": "shortest",
  "typescript.suggest.autoImports": true,
  "typescript.updateImportsOnFileMove.enabled": "always",
  "typescript.inlayHints.parameterNames.enabled": "literals",
  "typescript.inlayHints.returnTypes.enabled": true,
  "eslint.validate": [
    "javascript",
    "typescript",
    "typescriptreact"
  ]
}
```

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next",
    "streetsidesoftware.code-spell-checker",
    "bradlc.vscode-tailwindcss",
    "eamodio.gitlens",
    "usernamehw.errorlens",
    "WallabyJs.quokka-vscode",
    "yoavbls.pretty-ts-errors"
  ]
}
```

---

## 43.20 EditorConfig

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[*.json]
indent_size = 2

[Makefile]
indent_style = tab
```

---

## สรุป

TypeScript Tooling และ Configuration ที่ดีช่วยให้:

1. **tsconfig.json** - ตั้งค่า compiler options อย่างเหมาะสมกับ project
2. **Strict Mode** - เปิด strict options ทั้งหมดเพื่อความปลอดภัย
3. **ESLint** - Linting rules ที่เหมาะกับ TypeScript
4. **Prettier** - Code formatting อัตโนมัติ
5. **Husky + lint-staged** - Pre-commit hooks
6. **GitHub Actions** - CI/CD pipeline
7. **Bundle Analysis** - ติดตาม bundle size
8. **Performance Profiling** - ตรวจสอบ performance
9. **Declaration Files** - สร้าง .d.ts สำหรับ libraries
10. **Project References** - Monorepo support

การลงทุนเวลาในการตั้งค่าเครื่องมือเหล่านี้จะช่วยประหยัดเวลาในระยะยาวและทำให้ทีมทำงานได้อย่างมีประสิทธิภาพ
