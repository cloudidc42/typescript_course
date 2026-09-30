# ตอนที่ 45: Monorepo กับ Turborepo

## บทนำ

Monorepo คือการเก็บโค้ดของหลาย project ไว้ใน repository เดียว Turborepo เป็น build system ที่รวดเร็วและชาญฉลาดสำหรับ JavaScript/TypeScript monorepos โดยใช้การ cache และ parallel execution เพื่อเพิ่มความเร็ว

---

## 45.1 Monorepo Benefits

### ทำไมต้องใช้ Monorepo

**ข้อดีของ Monorepo:**
- **Code Sharing** - แชร์โค้ดระหว่าง packages ได้ง่าย
- **Atomic Changes** - เปลี่ยนแปลงหลาย packages พร้อมกันใน commit เดียว
- **Unified Tooling** - ใช้ tools เดียวกันทั้ง repo
- **Consistent Versions** - จัดการ dependency versions ง่ายขึ้น
- **Easier Refactoring** - Refactor ข้าม packages ได้ง่าย

### โครงสร้าง Monorepo ทั่วไป

```
my-monorepo/
├── apps/
│   ├── web/           # Next.js web application
│   ├── mobile/        # React Native app
│   └── admin/         # Admin dashboard
├── packages/
│   ├── ui/            # Shared UI components
│   ├── utils/         # Shared utilities
│   ├── config/        # Shared configs
│   └── types/         # Shared TypeScript types
├── turbo.json         # Turborepo config
├── package.json       # Root package.json
└── pnpm-workspace.yaml # pnpm workspace config
```

---

## 45.2 Turborepo Setup

### การสร้าง Monorepo ด้วย Turborepo

```bash
# สร้าง monorepo ใหม่ด้วย create-turbo
npx create-turbo@latest my-monorepo

# หรือ add turborepo เข้าไปใน project ที่มีอยู่
npm install turbo --save-dev
```

### โครงสร้างไฟล์เริ่มต้น

```bash
# สร้างโครงสร้างด้วยมือ
mkdir my-monorepo
cd my-monorepo

mkdir -p apps/web apps/docs packages/ui packages/utils packages/config

# สร้าง root package.json
cat > package.json << 'EOF'
{
  "name": "my-monorepo",
  "private": true,
  "version": "0.0.0",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "format": "prettier --write \"**/*.{ts,tsx,md}\"",
    "clean": "turbo run clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^1.13.0",
    "prettier": "^3.0.0",
    "typescript": "^5.3.0"
  },
  "packageManager": "pnpm@8.15.0"
}
EOF
```

---

## 45.3 Workspace Configuration

### pnpm-workspace.yaml

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

### root package.json ที่สมบูรณ์

```json
{
  "name": "my-monorepo",
  "private": true,
  "version": "0.0.0",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev --parallel",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "test:coverage": "turbo run test:coverage",
    "type-check": "turbo run type-check",
    "format": "prettier --write \"**/*.{ts,tsx,md,json}\"",
    "format:check": "prettier --check \"**/*.{ts,tsx,md,json}\"",
    "clean": "turbo run clean"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "prettier": "^3.0.0",
    "turbo": "^1.13.0",
    "typescript": "^5.3.0"
  },
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0"
  },
  "packageManager": "pnpm@8.15.0"
}
```

---

## 45.4 Shared TypeScript Configs

### packages/config/package.json

```json
{
  "name": "@my-monorepo/config",
  "version": "0.0.0",
  "private": true,
  "exports": {
    "./typescript/base": "./typescript/base.json",
    "./typescript/nextjs": "./typescript/nextjs.json",
    "./typescript/react-library": "./typescript/react-library.json"
  },
  "files": [
    "typescript/"
  ]
}
```

### packages/config/typescript/base.json

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "Default",
  "compilerOptions": {
    "composite": false,
    "declaration": true,
    "declarationMap": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "inlineSources": false,
    "isolatedModules": true,
    "moduleResolution": "bundler",
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "preserveWatchOutput": true,
    "skipLibCheck": true,
    "strict": true,
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext"
  },
  "exclude": [
    "node_modules"
  ]
}
```

### packages/config/typescript/nextjs.json

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "Next.js",
  "extends": "./base.json",
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ]
  },
  "exclude": ["node_modules"]
}
```

### packages/config/typescript/react-library.json

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "React Library",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "target": "ES2022",
    "jsx": "react-jsx"
  }
}
```

---

## 45.5 Shared ESLint Configs

### packages/config/eslint/index.js

```javascript
// packages/config/eslint/index.js
/** @type {import("eslint").Linter.Config} */
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  plugins: ["@typescript-eslint"],
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: "latest",
    sourceType: "module"
  },
  env: {
    node: true
  },
  rules: {
    "@typescript-eslint/no-unused-vars": ["error", { 
      "argsIgnorePattern": "^_",
      "varsIgnorePattern": "^_" 
    }],
    "@typescript-eslint/no-explicit-any": "warn",
    "@typescript-eslint/explicit-function-return-type": "off",
    "@typescript-eslint/explicit-module-boundary-types": "off"
  }
};
```

### packages/config/eslint/nextjs.js

```javascript
// packages/config/eslint/nextjs.js
/** @type {import("eslint").Linter.Config} */
module.exports = {
  extends: [
    "./index.js",
    "next",
    "next/core-web-vitals"
  ],
  rules: {
    "@next/next/no-html-link-for-pages": "off",
    "react/display-name": "off"
  }
};
```

### packages/config/eslint/react-internal.js

```javascript
// packages/config/eslint/react-internal.js
/** @type {import("eslint").Linter.Config} */
module.exports = {
  extends: [
    "./index.js",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended"
  ],
  settings: {
    react: {
      version: "detect"
    }
  },
  rules: {
    "react/react-in-jsx-scope": "off",
    "react/prop-types": "off"
  }
};
```

---

## 45.6 Shared UI Component Library

### packages/ui/package.json

```json
{
  "name": "@my-monorepo/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  },
  "scripts": {
    "lint": "eslint src/ --ext .ts,.tsx",
    "type-check": "tsc --noEmit",
    "build": "tsup src/index.ts --format esm,cjs --dts --external react",
    "dev": "tsup src/index.ts --format esm,cjs --dts --external react --watch",
    "clean": "rm -rf dist"
  },
  "devDependencies": {
    "@my-monorepo/config": "workspace:*",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "eslint": "^8.48.0",
    "react": "^18.2.0",
    "tsup": "^7.2.0",
    "typescript": "^5.3.0"
  },
  "peerDependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  }
}
```

### packages/ui/tsconfig.json

```json
{
  "extends": "@my-monorepo/config/typescript/react-library",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### packages/ui/src/components/Button.tsx

```tsx
// packages/ui/src/components/Button.tsx
import React from "react";

export interface ButtonProps {
  variant?: "primary" | "secondary" | "danger" | "ghost";
  size?: "sm" | "md" | "lg";
  disabled?: boolean;
  loading?: boolean;
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  children: React.ReactNode;
  type?: "button" | "submit" | "reset";
  className?: string;
  "aria-label"?: string;
}

const variantClasses: Record<NonNullable<ButtonProps["variant"]>, string> = {
  primary: "bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500",
  secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300 focus:ring-gray-500",
  danger: "bg-red-600 text-white hover:bg-red-700 focus:ring-red-500",
  ghost: "bg-transparent text-gray-700 hover:bg-gray-100 focus:ring-gray-500"
};

const sizeClasses: Record<NonNullable<ButtonProps["size"]>, string> = {
  sm: "px-3 py-1.5 text-sm",
  md: "px-4 py-2 text-base",
  lg: "px-6 py-3 text-lg"
};

export function Button({
  variant = "primary",
  size = "md",
  disabled = false,
  loading = false,
  onClick,
  children,
  type = "button",
  className = "",
  "aria-label": ariaLabel,
}: ButtonProps): JSX.Element {
  const isDisabled = disabled || loading;

  return (
    <button
      type={type}
      disabled={isDisabled}
      onClick={onClick}
      aria-label={ariaLabel}
      aria-disabled={isDisabled}
      className={[
        "inline-flex items-center justify-center",
        "font-medium rounded-md",
        "transition-colors duration-150",
        "focus:outline-none focus:ring-2 focus:ring-offset-2",
        variantClasses[variant],
        sizeClasses[size],
        isDisabled ? "opacity-50 cursor-not-allowed" : "cursor-pointer",
        className
      ].join(" ")}
    >
      {loading && (
        <svg
          className="animate-spin -ml-1 mr-2 h-4 w-4"
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          aria-hidden="true"
        >
          <circle
            className="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            strokeWidth="4"
          />
          <path
            className="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
          />
        </svg>
      )}
      {children}
    </button>
  );
}
```

### packages/ui/src/components/Input.tsx

```tsx
// packages/ui/src/components/Input.tsx
import React, { forwardRef } from "react";

export interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
  hint?: string;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  fullWidth?: boolean;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  function Input(
    {
      label,
      error,
      hint,
      leftIcon,
      rightIcon,
      fullWidth = false,
      className = "",
      id,
      ...props
    },
    ref
  ) {
    const inputId = id ?? label?.toLowerCase().replace(/\s+/g, "-");

    return (
      <div className={fullWidth ? "w-full" : "inline-block"}>
        {label && (
          <label
            htmlFor={inputId}
            className="block text-sm font-medium text-gray-700 mb-1"
          >
            {label}
          </label>
        )}
        <div className="relative">
          {leftIcon && (
            <div className="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
              {leftIcon}
            </div>
          )}
          <input
            ref={ref}
            id={inputId}
            className={[
              "block rounded-md border shadow-sm",
              "focus:outline-none focus:ring-2 focus:ring-offset-0",
              fullWidth ? "w-full" : "",
              error
                ? "border-red-300 focus:ring-red-500 focus:border-red-500"
                : "border-gray-300 focus:ring-blue-500 focus:border-blue-500",
              leftIcon ? "pl-10" : "pl-3",
              rightIcon ? "pr-10" : "pr-3",
              "py-2 text-sm",
              className
            ].join(" ")}
            aria-invalid={!!error}
            aria-describedby={
              error ? `${inputId}-error` : hint ? `${inputId}-hint` : undefined
            }
            {...props}
          />
          {rightIcon && (
            <div className="absolute inset-y-0 right-0 pr-3 flex items-center">
              {rightIcon}
            </div>
          )}
        </div>
        {error && (
          <p id={`${inputId}-error`} className="mt-1 text-sm text-red-600" role="alert">
            {error}
          </p>
        )}
        {hint && !error && (
          <p id={`${inputId}-hint`} className="mt-1 text-sm text-gray-500">
            {hint}
          </p>
        )}
      </div>
    );
  }
);
```

### packages/ui/src/components/Card.tsx

```tsx
// packages/ui/src/components/Card.tsx
import React from "react";

export interface CardProps {
  title?: string;
  description?: string;
  footer?: React.ReactNode;
  children?: React.ReactNode;
  className?: string;
  padding?: "none" | "sm" | "md" | "lg";
  shadow?: "none" | "sm" | "md" | "lg";
  border?: boolean;
  onClick?: () => void;
}

const paddingClasses: Record<NonNullable<CardProps["padding"]>, string> = {
  none: "p-0",
  sm: "p-3",
  md: "p-5",
  lg: "p-8"
};

const shadowClasses: Record<NonNullable<CardProps["shadow"]>, string> = {
  none: "",
  sm: "shadow-sm",
  md: "shadow-md",
  lg: "shadow-lg"
};

export function Card({
  title,
  description,
  footer,
  children,
  className = "",
  padding = "md",
  shadow = "sm",
  border = true,
  onClick,
}: CardProps): JSX.Element {
  return (
    <div
      className={[
        "bg-white rounded-lg",
        paddingClasses[padding],
        shadowClasses[shadow],
        border ? "border border-gray-200" : "",
        onClick ? "cursor-pointer hover:shadow-md transition-shadow" : "",
        className
      ].join(" ")}
      onClick={onClick}
      role={onClick ? "button" : undefined}
      tabIndex={onClick ? 0 : undefined}
    >
      {(title || description) && (
        <div className="mb-4">
          {title && (
            <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
          )}
          {description && (
            <p className="mt-1 text-sm text-gray-500">{description}</p>
          )}
        </div>
      )}
      {children}
      {footer && (
        <div className="mt-4 pt-4 border-t border-gray-100">{footer}</div>
      )}
    </div>
  );
}
```

### packages/ui/src/index.ts

```typescript
// packages/ui/src/index.ts
export { Button } from "./components/Button";
export type { ButtonProps } from "./components/Button";

export { Input } from "./components/Input";
export type { InputProps } from "./components/Input";

export { Card } from "./components/Card";
export type { CardProps } from "./components/Card";
```

---

## 45.7 Shared Utilities Package

### packages/utils/package.json

```json
{
  "name": "@my-monorepo/utils",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./string": "./src/string.ts",
    "./date": "./src/date.ts",
    "./number": "./src/number.ts"
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts",
    "dev": "tsup src/index.ts --format esm,cjs --dts --watch",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "lint": "eslint src/ --ext .ts",
    "clean": "rm -rf dist"
  },
  "devDependencies": {
    "@my-monorepo/config": "workspace:*",
    "@types/jest": "^29.5.0",
    "jest": "^29.5.0",
    "ts-jest": "^29.1.0",
    "typescript": "^5.3.0"
  }
}
```

### packages/utils/src/string.ts

```typescript
// packages/utils/src/string.ts

/**
 * แปลง string เป็น slug
 */
export function slugify(str: string): string {
  return str
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, "")
    .replace(/[\s_-]+/g, "-")
    .replace(/^-+|-+$/g, "");
}

/**
 * ตัด string ให้มีความยาวไม่เกินที่กำหนด
 */
export function truncate(str: string, maxLength: number, suffix = "..."): string {
  if (str.length <= maxLength) return str;
  return str.slice(0, maxLength - suffix.length) + suffix;
}

/**
 * Capitalize ตัวแรก
 */
export function capitalize(str: string): string {
  if (!str) return str;
  return str.charAt(0).toUpperCase() + str.slice(1);
}

/**
 * แปลงเป็น camelCase
 */
export function toCamelCase(str: string): string {
  return str
    .replace(/[-_\s]+(.)?/g, (_, char) => char ? char.toUpperCase() : "")
    .replace(/^(.)/, char => char.toLowerCase());
}

/**
 * แปลงเป็น PascalCase
 */
export function toPascalCase(str: string): string {
  const camel = toCamelCase(str);
  return capitalize(camel);
}

/**
 * แปลงเป็น snake_case
 */
export function toSnakeCase(str: string): string {
  return str
    .replace(/\s+/g, "_")
    .replace(/([A-Z])/g, "_$1")
    .toLowerCase()
    .replace(/^_/, "");
}

/**
 * ลบ HTML tags
 */
export function stripHtml(html: string): string {
  return html.replace(/<[^>]*>/g, "");
}

/**
 * ตรวจสอบว่า string เป็น URL ที่ถูกต้อง
 */
export function isValidUrl(str: string): boolean {
  try {
    new URL(str);
    return true;
  } catch {
    return false;
  }
}

// ตัวอย่างการใช้งาน
console.log(slugify("Hello World! สวัสดี")); // "hello-world"
console.log(truncate("This is a long text", 10)); // "This is..."
console.log(toCamelCase("hello-world")); // "helloWorld"
console.log(toPascalCase("hello-world")); // "HelloWorld"
console.log(toSnakeCase("helloWorld")); // "hello_world"
```

### packages/utils/src/date.ts

```typescript
// packages/utils/src/date.ts

/**
 * Format วันที่เป็น string ภาษาไทย
 */
export function formatDateThai(date: Date): string {
  return date.toLocaleDateString("th-TH", {
    year: "numeric",
    month: "long",
    day: "numeric"
  });
}

/**
 * ตรวจสอบว่าเป็นวันเดียวกัน
 */
export function isSameDay(date1: Date, date2: Date): boolean {
  return (
    date1.getFullYear() === date2.getFullYear() &&
    date1.getMonth() === date2.getMonth() &&
    date1.getDate() === date2.getDate()
  );
}

/**
 * คำนวณความแตกต่างของวัน
 */
export function daysBetween(date1: Date, date2: Date): number {
  const oneDay = 24 * 60 * 60 * 1000;
  return Math.round(Math.abs((date1.getTime() - date2.getTime()) / oneDay));
}

/**
 * เพิ่มวัน
 */
export function addDays(date: Date, days: number): Date {
  const result = new Date(date);
  result.setDate(result.getDate() + days);
  return result;
}

/**
 * เช็คว่าวันที่ผ่านมาแล้วหรือไม่
 */
export function isPast(date: Date): boolean {
  return date < new Date();
}

/**
 * เช็คว่าวันที่ยังไม่มาถึง
 */
export function isFuture(date: Date): boolean {
  return date > new Date();
}

/**
 * แปลงเวลาเป็น relative time เช่น "5 นาทีที่แล้ว"
 */
export function relativeTime(date: Date): string {
  const now = new Date();
  const diff = now.getTime() - date.getTime();
  const seconds = Math.floor(diff / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours = Math.floor(minutes / 60);
  const days = Math.floor(hours / 24);

  if (seconds < 60) return "เมื่อสักครู่";
  if (minutes < 60) return `${minutes} นาทีที่แล้ว`;
  if (hours < 24) return `${hours} ชั่วโมงที่แล้ว`;
  if (days < 7) return `${days} วันที่แล้ว`;
  
  return formatDateThai(date);
}
```

### packages/utils/src/index.ts

```typescript
// packages/utils/src/index.ts
export * from "./string";
export * from "./date";
export * from "./number";
```

---

## 45.8 Turborepo Pipeline Configuration

### turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"],
      "env": ["NODE_ENV"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "outputs": []
    },
    "type-check": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "env": ["NODE_ENV", "TEST_DATABASE_URL"]
    },
    "test:coverage": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    },
    "clean": {
      "cache": false
    }
  }
}
```

### อธิบาย Pipeline Configuration

```typescript
// การอธิบาย turbo.json fields

interface TurboTask {
  // dependsOn: tasks ที่ต้องทำก่อน
  // "^build" = build ของ packages ที่เป็น dependency ต้องเสร็จก่อน
  // "build" = build ของ package เดียวกันต้องเสร็จก่อน
  dependsOn?: string[];
  
  // outputs: ไฟล์ที่ถูก cache
  outputs?: string[];
  
  // cache: เปิด/ปิด caching (default: true)
  cache?: boolean;
  
  // persistent: สำหรับ long-running tasks เช่น dev server
  persistent?: boolean;
  
  // env: environment variables ที่ใช้ใน task (สำหรับ cache key)
  env?: string[];
}
```

---

## 45.9 Remote Caching

### การตั้งค่า Remote Cache กับ Vercel

```bash
# Login กับ Vercel
npx turbo login

# Link project กับ Vercel Remote Cache
npx turbo link
```

### การตั้งค่า Remote Cache ด้วย Self-hosted

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "remoteCache": {
    "enabled": true,
    "apiUrl": "https://your-cache-server.com"
  },
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    }
  }
}
```

```bash
# ใช้ remote cache token
TURBO_TOKEN=your-token TURBO_TEAM=your-team turbo run build

# หรือตั้งใน .env
TURBO_TOKEN=your-token
TURBO_TEAM=your-team
```

---

## 45.10 Apps Configuration

### apps/web/package.json

```json
{
  "name": "@my-monorepo/web",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "clean": "rm -rf .next"
  },
  "dependencies": {
    "@my-monorepo/ui": "workspace:*",
    "@my-monorepo/utils": "workspace:*",
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@my-monorepo/config": "workspace:*",
    "@types/node": "^20.0.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "typescript": "^5.3.0"
  }
}
```

### apps/web/tsconfig.json

```json
{
  "extends": "@my-monorepo/config/typescript/nextjs",
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## 45.11 Publishing Packages

### packages/ui/package.json (publishable)

```json
{
  "name": "@my-org/ui",
  "version": "1.0.0",
  "description": "Shared UI components",
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts --external react",
    "prepublishOnly": "pnpm build"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "keywords": ["react", "ui", "components"],
  "repository": {
    "type": "git",
    "url": "https://github.com/my-org/my-monorepo.git",
    "directory": "packages/ui"
  }
}
```

### tsup.config.ts สำหรับ Build

```typescript
// packages/ui/tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],
  dts: true,
  splitting: false,
  sourcemap: true,
  clean: true,
  treeshake: true,
  external: ["react", "react-dom"],
  esbuildOptions(options) {
    options.banner = {
      js: '"use client"'
    };
  }
});
```

---

## 45.12 Changesets สำหรับ Versioning

```bash
# ติดตั้ง changesets
pnpm add -D @changesets/cli -w

# Initialize
pnpm changeset init
```

### .changeset/config.json

```json
{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

### การใช้งาน Changesets

```bash
# สร้าง changeset ใหม่
pnpm changeset

# อัปเดต versions
pnpm changeset version

# Publish packages
pnpm changeset publish
```

---

## 45.13 CI/CD Pipeline

### .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Build
        run: pnpm build
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
      
      - name: Lint
        run: pnpm lint
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
      
      - name: Type Check
        run: pnpm type-check
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
      
      - name: Test
        run: pnpm test
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
```

---

## 45.14 Shared Types Package

### packages/types/package.json

```json
{
  "name": "@my-monorepo/types",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  },
  "devDependencies": {
    "@my-monorepo/config": "workspace:*",
    "typescript": "^5.3.0"
  }
}
```

### packages/types/src/index.ts

```typescript
// packages/types/src/index.ts

// User types
export interface User {
  id: string;
  email: string;
  name: string;
  role: UserRole;
  createdAt: Date;
  updatedAt: Date;
}

export type UserRole = "admin" | "user" | "moderator";

// API Response types
export interface ApiResponse<T> {
  data: T;
  message?: string;
  success: boolean;
}

export interface ApiError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
  };
}

// Product types
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  category: string;
  images: string[];
  stock: number;
  createdAt: Date;
}

// Order types
export type OrderStatus = "pending" | "processing" | "shipped" | "delivered" | "cancelled";

export interface Order {
  id: string;
  userId: string;
  items: OrderItem[];
  status: OrderStatus;
  total: number;
  createdAt: Date;
  updatedAt: Date;
}

export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

// Utility types
export type Nullable<T> = T | null;
export type Optional<T> = T | undefined;
export type Maybe<T> = T | null | undefined;

export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

export type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
```

---

## สรุปบทที่ 45

ในบทนี้เราได้เรียนรู้:

1. **Monorepo Benefits** - ข้อดีของการใช้ monorepo
2. **Turborepo Setup** - การตั้งค่า turborepo
3. **Workspace Configuration** - pnpm workspaces
4. **Shared TypeScript Configs** - config ที่ใช้ร่วมกัน
5. **Shared ESLint Configs** - eslint config ร่วมกัน
6. **Shared UI Library** - component library ร่วมกัน
7. **Pipeline Configuration** - การตั้งค่า build pipeline
8. **Remote Caching** - การใช้ remote cache
9. **Publishing** - การ publish packages
10. **CI/CD** - การตั้งค่า continuous integration

Turborepo ช่วยให้:
- Build เร็วขึ้นด้วย caching
- ทำงาน parallel ได้
- จัดการ dependencies ระหว่าง packages
- Scale ได้ดีเมื่อ codebase ใหญ่ขึ้น
