# ตอนที่ 25: การตั้งค่า React + TypeScript

## บทนำ

React และ TypeScript เป็นคู่ที่ทำงานร่วมกันได้อย่างดีเยี่ยม TypeScript ช่วยให้โค้ด React มีความปลอดภัยทางประเภทข้อมูล ลดข้อผิดพลาดขณะรันไทม์ และช่วยให้ IDE สามารถให้ความช่วยเหลือที่ดีขึ้น ในบทนี้เราจะเรียนรู้วิธีการตั้งค่า React กับ TypeScript ทั้งด้วย Create React App และ Vite

---

## 25.1 Create React App กับ TypeScript

### การสร้าง project ใหม่

```bash
# สร้าง project ใหม่ด้วย TypeScript template
npx create-react-app my-app --template typescript

# หรือด้วย Yarn
yarn create react-app my-app --template typescript

# เข้าไปใน directory
cd my-app

# รัน development server
npm start
```

### โครงสร้างไฟล์ที่ได้จาก CRA

```
my-app/
├── public/
│   ├── favicon.ico
│   ├── index.html
│   ├── manifest.json
│   └── robots.txt
├── src/
│   ├── App.css
│   ├── App.test.tsx
│   ├── App.tsx
│   ├── index.css
│   ├── index.tsx
│   ├── logo.svg
│   ├── react-app-env.d.ts
│   ├── reportWebVitals.ts
│   └── setupTests.ts
├── .gitignore
├── package.json
├── README.md
└── tsconfig.json
```

### การเพิ่ม TypeScript ใน project ที่มีอยู่แล้ว

```bash
# ติดตั้ง TypeScript และ types
npm install --save typescript @types/node @types/react @types/react-dom

# เปลี่ยนชื่อไฟล์ .js เป็น .ts หรือ .tsx
# .ts สำหรับไฟล์ที่ไม่มี JSX
# .tsx สำหรับไฟล์ที่มี JSX

# รัน build เพื่อสร้าง tsconfig.json อัตโนมัติ
npm start  # CRA จะสร้าง tsconfig.json ให้
```

---

## 25.2 Vite + React + TypeScript

### การสร้าง project ด้วย Vite

```bash
# สร้าง project ด้วย Vite
npm create vite@latest my-vite-app -- --template react-ts

# หรือเลือก template แบบ interactive
npm create vite@latest my-vite-app
# แล้วเลือก: React > TypeScript

# ติดตั้ง dependencies
cd my-vite-app
npm install

# รัน development server
npm run dev
```

### โครงสร้างไฟล์จาก Vite

```
my-vite-app/
├── public/
│   └── vite.svg
├── src/
│   ├── assets/
│   │   └── react.svg
│   ├── App.css
│   ├── App.tsx
│   ├── index.css
│   ├── main.tsx
│   └── vite-env.d.ts
├── .gitignore
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.node.json
└── vite.config.ts
```

### การตั้งค่า vite.config.ts

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  
  // Path aliases
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
      '@utils': path.resolve(__dirname, './src/utils'),
      '@types': path.resolve(__dirname, './src/types'),
      '@services': path.resolve(__dirname, './src/services'),
      '@assets': path.resolve(__dirname, './src/assets')
    }
  },
  
  // Development server
  server: {
    port: 3000,
    open: true,
    cors: true,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  },
  
  // Build options
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom']
        }
      }
    }
  },
  
  // Test configuration
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/setupTests.ts',
    css: true
  }
});
```

---

## 25.3 tsconfig สำหรับ React

### tsconfig.json แบบสมบูรณ์สำหรับ React

```json
{
  "compilerOptions": {
    // Target และ Module
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    
    // JSX
    "jsx": "react-jsx",
    
    // Output
    "outDir": "./dist",
    "rootDir": "./src",
    "noEmit": true,
    
    // Strict mode
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    
    // Module resolution
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    
    // Path aliases
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"],
      "@services/*": ["src/services/*"],
      "@assets/*": ["src/assets/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### tsconfig.node.json (สำหรับ Vite)

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "strict": true
  },
  "include": ["vite.config.ts"]
}
```

### การตั้งค่า JSX

```typescript
// ความแตกต่างระหว่าง jsx options:

// "preserve" - ไม่แปลง JSX, เหมาะสำหรับ tools อื่นที่จัดการ JSX
// "react" - แปลงเป็น React.createElement() (React 16 และเก่ากว่า)
// "react-jsx" - แปลงโดยไม่ต้อง import React (React 17+)
// "react-jsxdev" - เหมือน react-jsx แต่มี debug info เพิ่มเติม
// "react-native" - ไม่แปลง JSX (สำหรับ React Native)

// ตัวอย่างการใช้ JSX โดยไม่ต้อง import React (react-jsx mode)
// src/components/Hello.tsx
const Hello = ({ name }: { name: string }) => {
  return <h1>สวัสดี {name}!</h1>;
  // แปลงเป็น: import { jsx as _jsx } from 'react/jsx-runtime';
  // _jsx("h1", { children: "สวัสดี " + name + "!" })
};

// ตัวอย่างการใช้ JSX กับ React 16 (react mode)
// ต้อง import React เสมอ
import React from 'react';
const Hello16 = ({ name }: { name: string }) => {
  return React.createElement('h1', null, `สวัสดี ${name}!`);
};
```

---

## 25.4 โครงสร้าง Project ที่แนะนำ

### โครงสร้างขนาดกลาง-ใหญ่

```
src/
├── assets/
│   ├── images/
│   ├── fonts/
│   └── icons/
├── components/
│   ├── common/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.test.tsx
│   │   │   ├── Button.module.css
│   │   │   └── index.ts
│   │   ├── Input/
│   │   ├── Modal/
│   │   └── index.ts
│   ├── layout/
│   │   ├── Header/
│   │   ├── Footer/
│   │   ├── Sidebar/
│   │   └── index.ts
│   └── features/
│       ├── auth/
│       ├── products/
│       └── cart/
├── hooks/
│   ├── useAuth.ts
│   ├── useFetch.ts
│   ├── useLocalStorage.ts
│   └── index.ts
├── pages/
│   ├── Home/
│   │   ├── Home.tsx
│   │   ├── Home.test.tsx
│   │   └── index.ts
│   ├── About/
│   └── NotFound/
├── services/
│   ├── api.ts
│   ├── auth.service.ts
│   └── product.service.ts
├── store/
│   ├── index.ts
│   └── slices/
├── types/
│   ├── api.types.ts
│   ├── component.types.ts
│   └── index.ts
├── utils/
│   ├── format.ts
│   ├── validation.ts
│   └── index.ts
├── styles/
│   ├── globals.css
│   ├── variables.css
│   └── mixins.css
├── App.tsx
├── main.tsx
└── vite-env.d.ts
```

### Index files pattern

```typescript
// src/components/common/Button/index.ts
export { Button } from './Button';
export type { ButtonProps } from './Button';

// src/components/common/index.ts
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';

// src/components/index.ts
export * from './common';
export * from './layout';
```

---

## 25.5 React Types Overview (@types/react)

### Types ที่สำคัญใน @types/react

```typescript
// src/types/react-overview.ts
import React from 'react';

// ===== Component Types =====

// FunctionComponent (FC)
type FC<P = {}> = React.FunctionComponent<P>;

// Component ที่ไม่รับ props
const SimpleComponent: React.FC = () => <div>Hello</div>;

// Component ที่รับ props
interface GreetProps {
  name: string;
  age?: number;
}
const Greet: React.FC<GreetProps> = ({ name, age }) => (
  <div>{name} {age && `(${age})`}</div>
);

// ===== Element Types =====

// JSX.Element - ผลลัพธ์จาก JSX expression
const element: JSX.Element = <div>Hello</div>;

// React.ReactElement - เฉพาะเจาะจงกว่า JSX.Element
const reactElement: React.ReactElement = React.createElement('div', null, 'Hello');

// React.ReactNode - union type ที่ครอบคลุมมากที่สุด
type ReactNodeExample = 
  | React.ReactElement   // JSX elements
  | string               // strings
  | number               // numbers
  | boolean              // booleans (render nothing)
  | null                 // null (render nothing)
  | undefined            // undefined (render nothing)
  | React.ReactPortal   // portals
  | React.ReactFragment; // fragments

// ===== Event Types =====

// Mouse events
const handleClick: React.MouseEventHandler<HTMLButtonElement> = (event) => {
  event.preventDefault();
  console.log(event.currentTarget.textContent);
};

// Keyboard events
const handleKeyDown: React.KeyboardEventHandler<HTMLInputElement> = (event) => {
  if (event.key === 'Enter') {
    console.log(event.currentTarget.value);
  }
};

// Change events
const handleChange: React.ChangeEventHandler<HTMLInputElement> = (event) => {
  console.log(event.target.value);
};

// Form events
const handleSubmit: React.FormEventHandler<HTMLFormElement> = (event) => {
  event.preventDefault();
};

// ===== Ref Types =====

// useRef types
const inputRef = React.useRef<HTMLInputElement>(null);
const valueRef = React.useRef<number>(0);

// RefObject vs MutableRefObject
// RefObject<T>: readonly current (จาก useRef(null))
// MutableRefObject<T>: mutable current (จาก useRef(initialValue))

// ===== CSS Types =====

const style: React.CSSProperties = {
  backgroundColor: 'blue',
  fontSize: '16px',
  marginTop: 8,
  display: 'flex'
};
```

### Component children types

```typescript
// src/components/examples/ChildrenTypes.tsx
import React from 'react';

// ReactNode - รับ children ได้หลายแบบ
interface WithReactNode {
  children: React.ReactNode;
}

const ContainerNode: React.FC<WithReactNode> = ({ children }) => (
  <div className="container">{children}</div>
);

// ReactElement - ต้องการ JSX element เท่านั้น
interface WithReactElement {
  children: React.ReactElement;
}

const ContainerElement: React.FC<WithReactElement> = ({ children }) => (
  <div className="container">{children}</div>
);

// ReactChild - ReactElement หรือ string หรือ number
interface WithReactChild {
  children: React.ReactChild;
}

// PropsWithChildren - เพิ่ม children ให้ props interface อื่น
interface BaseProps {
  title: string;
  className?: string;
}

type CardProps = React.PropsWithChildren<BaseProps>;

const Card: React.FC<CardProps> = ({ title, className, children }) => (
  <div className={`card ${className || ''}`}>
    <h2>{title}</h2>
    <div className="card-body">{children}</div>
  </div>
);

// การใช้งาน
const Usage: React.FC = () => (
  <>
    <ContainerNode>
      <span>Text</span>
      <div>Element</div>
      {"String node"}
      {42}
    </ContainerNode>
    
    <ContainerElement>
      <div>Must be element</div>
    </ContainerElement>
    
    <Card title="My Card" className="featured">
      <p>Card content here</p>
    </Card>
  </>
);

export { ContainerNode, ContainerElement, Card };
```

---

## 25.6 JSX Typing

### การเข้าใจ JSX types

```typescript
// src/examples/jsxTyping.tsx
import React from 'react';

// ===== Intrinsic Elements =====
// HTML elements ต่างๆ มี type เป็น JSX.IntrinsicElements

// ตัวอย่าง intrinsic element types
const htmlElements: JSX.Element[] = [
  <div key="1" className="container" onClick={() => {}}>content</div>,
  <input key="2" type="text" value="" onChange={() => {}} />,
  <button key="3" disabled={false} onClick={() => {}}>Click</button>,
  <img key="4" src="/image.png" alt="description" />,
  <a key="5" href="/link" rel="noopener">Link</a>
];

// ===== Custom Component Types =====
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

const CustomButton: React.FC<ButtonProps> = ({
  variant,
  size = 'md',
  disabled = false,
  onClick,
  children
}) => (
  <button
    className={`btn btn-${variant} btn-${size}`}
    disabled={disabled}
    onClick={onClick}
  >
    {children}
  </button>
);

// TypeScript จะ check types ตรงนี้
const buttonUsage: JSX.Element = (
  <CustomButton
    variant="primary"
    size="lg"
    onClick={() => console.log('clicked')}
  >
    Click Me
  </CustomButton>
);

// ===== Generic Components =====
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function TypedList<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}

// การใช้งาน TypedList
interface Product {
  id: string;
  name: string;
  price: number;
}

const products: Product[] = [
  { id: '1', name: 'สินค้า A', price: 100 },
  { id: '2', name: 'สินค้า B', price: 200 }
];

const ProductList: React.FC = () => (
  <TypedList
    items={products}
    keyExtractor={(p) => p.id}
    renderItem={(p) => (
      <span>{p.name} - {p.price} บาท</span>
    )}
  />
);

export { CustomButton, TypedList, ProductList };
```

---

## 25.7 Component File Structure

### โครงสร้างไฟล์ component ที่แนะนำ

```typescript
// src/components/common/Button/Button.tsx

// 1. Imports
import React, { useCallback } from 'react';
import styles from './Button.module.css';

// 2. Type definitions
export type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost';
export type ButtonSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

export interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  loading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  fullWidth?: boolean;
}

// 3. Helper functions / constants
const SIZE_CLASSES: Record<ButtonSize, string> = {
  xs: styles.xs,
  sm: styles.sm,
  md: styles.md,
  lg: styles.lg,
  xl: styles.xl
};

const VARIANT_CLASSES: Record<ButtonVariant, string> = {
  primary: styles.primary,
  secondary: styles.secondary,
  danger: styles.danger,
  ghost: styles.ghost
};

// 4. Component
export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  (
    {
      variant = 'primary',
      size = 'md',
      loading = false,
      leftIcon,
      rightIcon,
      fullWidth = false,
      disabled,
      className,
      children,
      onClick,
      ...rest
    },
    ref
  ) => {
    const handleClick = useCallback(
      (event: React.MouseEvent<HTMLButtonElement>) => {
        if (!loading && onClick) {
          onClick(event);
        }
      },
      [loading, onClick]
    );

    const classes = [
      styles.button,
      VARIANT_CLASSES[variant],
      SIZE_CLASSES[size],
      fullWidth ? styles.fullWidth : '',
      loading ? styles.loading : '',
      className || ''
    ]
      .filter(Boolean)
      .join(' ');

    return (
      <button
        ref={ref}
        className={classes}
        disabled={disabled || loading}
        onClick={handleClick}
        {...rest}
      >
        {loading && <span className={styles.spinner} aria-hidden="true" />}
        {leftIcon && <span className={styles.leftIcon}>{leftIcon}</span>}
        <span className={styles.label}>{children}</span>
        {rightIcon && <span className={styles.rightIcon}>{rightIcon}</span>}
      </button>
    );
  }
);

Button.displayName = 'Button';

// 5. Default export (optional)
export default Button;
```

```typescript
// src/components/common/Button/index.ts
export { Button } from './Button';
export type { ButtonProps, ButtonVariant, ButtonSize } from './Button';
```

---

## 25.8 CSS Modules กับ TypeScript

### การตั้งค่า CSS Modules

```typescript
// vite-env.d.ts หรือ global.d.ts
/// <reference types="vite/client" />

// Declaration สำหรับ CSS Modules
declare module '*.module.css' {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module '*.module.scss' {
  const classes: { readonly [key: string]: string };
  export default classes;
}
```

### การใช้งาน CSS Modules

```css
/* src/components/common/Button/Button.module.css */
.button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: none;
  cursor: pointer;
  font-family: inherit;
  font-weight: 500;
  border-radius: 6px;
  transition: all 0.2s ease;
  gap: 8px;
}

.button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Variants */
.primary {
  background-color: #3b82f6;
  color: white;
}

.primary:hover:not(:disabled) {
  background-color: #2563eb;
}

.secondary {
  background-color: #e5e7eb;
  color: #374151;
}

.danger {
  background-color: #ef4444;
  color: white;
}

.ghost {
  background-color: transparent;
  color: #374151;
  border: 1px solid #d1d5db;
}

/* Sizes */
.xs { padding: 4px 8px; font-size: 12px; }
.sm { padding: 6px 12px; font-size: 14px; }
.md { padding: 8px 16px; font-size: 16px; }
.lg { padding: 10px 20px; font-size: 18px; }
.xl { padding: 12px 24px; font-size: 20px; }

/* Full width */
.fullWidth { width: 100%; }

/* Loading */
.loading { pointer-events: none; }

/* Icons */
.leftIcon, .rightIcon { display: flex; align-items: center; }
.label { flex: 1; }
.spinner { width: 16px; height: 16px; border: 2px solid rgba(255,255,255,0.3); border-top-color: white; border-radius: 50%; animation: spin 0.8s linear infinite; }

@keyframes spin { to { transform: rotate(360deg); } }
```

### Typed CSS Modules ด้วย typescript-plugin-css-modules

```bash
# ติดตั้ง plugin
npm install --save-dev typescript-plugin-css-modules
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "typescript-plugin-css-modules"
      }
    ]
  }
}
```

### การใช้ clsx สำหรับ class composition

```typescript
// ติดตั้ง: npm install clsx
import clsx from 'clsx';

interface ButtonProps {
  variant: 'primary' | 'secondary';
  size: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  className?: string;
}

const Button: React.FC<ButtonProps> = ({ variant, size, disabled, className, children }) => {
  const buttonClass = clsx(
    'btn',
    `btn-${variant}`,
    `btn-${size}`,
    {
      'btn-disabled': disabled,
      'btn-active': !disabled
    },
    className
  );

  return (
    <button className={buttonClass} disabled={disabled}>
      {children}
    </button>
  );
};
```

---

## 25.9 Environment Variables ใน React

### การตั้งค่า Environment Variables

```bash
# .env (base - ใช้สำหรับทุก environment)
REACT_APP_APP_NAME=My TypeScript App

# .env.development (development เท่านั้น)
REACT_APP_API_URL=http://localhost:3001
REACT_APP_DEBUG=true

# .env.production (production เท่านั้น)
REACT_APP_API_URL=https://api.example.com
REACT_APP_DEBUG=false

# .env.test (test environment)
REACT_APP_API_URL=http://localhost:3002
```

### การใช้งาน Environment Variables กับ TypeScript

```typescript
// src/config/env.ts

// สำหรับ Create React App
interface CRAEnvironment {
  NODE_ENV: 'development' | 'test' | 'production';
  REACT_APP_API_URL: string;
  REACT_APP_APP_NAME: string;
  REACT_APP_DEBUG: string;
}

// สำหรับ Vite
interface ViteEnvironment {
  VITE_API_URL: string;
  VITE_APP_NAME: string;
  VITE_DEBUG: string;
}

// Type-safe environment access สำหรับ Vite
const getEnvVar = (key: keyof ViteEnvironment): string => {
  const value = import.meta.env[key];
  if (!value) {
    throw new Error(`Environment variable ${key} is not defined`);
  }
  return value;
};

// การ parse environment variables
const parseBoolean = (value: string): boolean => value === 'true';
const parseNumber = (value: string): number => parseInt(value, 10);

// Config object
export const config = {
  apiUrl: import.meta.env.VITE_API_URL || 'http://localhost:3001',
  appName: import.meta.env.VITE_APP_NAME || 'My App',
  debug: parseBoolean(import.meta.env.VITE_DEBUG || 'false'),
  isDevelopment: import.meta.env.DEV,
  isProduction: import.meta.env.PROD,
  mode: import.meta.env.MODE as 'development' | 'production' | 'test'
} as const;

export type Config = typeof config;
```

### Type declarations สำหรับ Vite environment

```typescript
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_NAME: string;
  readonly VITE_DEBUG: string;
  readonly VITE_GOOGLE_ANALYTICS_ID?: string;
  // เพิ่ม environment variables อื่นๆ ที่นี่
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## 25.10 ESLint + TypeScript Setup

### การติดตั้ง ESLint

```bash
# ติดตั้ง ESLint และ plugins สำหรับ TypeScript
npm install --save-dev \
  eslint \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  eslint-plugin-react \
  eslint-plugin-react-hooks \
  eslint-plugin-import \
  eslint-config-prettier \
  prettier

# หรือใช้ eslint init
npx eslint --init
```

### การตั้งค่า .eslintrc.cjs

```javascript
// .eslintrc.cjs
module.exports = {
  root: true,
  
  env: {
    browser: true,
    es2020: true,
    node: true
  },
  
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:@typescript-eslint/recommended-requiring-type-checking',
    'plugin:react/recommended',
    'plugin:react/jsx-runtime',  // สำหรับ React 17+ (ไม่ต้อง import React)
    'plugin:react-hooks/recommended',
    'plugin:import/recommended',
    'plugin:import/typescript',
    'prettier'  // ต้องอยู่ท้ายสุดเพื่อ override formatting rules
  ],
  
  parser: '@typescript-eslint/parser',
  
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
    project: ['./tsconfig.json', './tsconfig.node.json'],
    tsconfigRootDir: __dirname,
    ecmaFeatures: {
      jsx: true
    }
  },
  
  plugins: [
    '@typescript-eslint',
    'react',
    'react-hooks',
    'import'
  ],
  
  settings: {
    react: {
      version: 'detect'
    },
    'import/resolver': {
      typescript: {
        alwaysTryTypes: true,
        project: './tsconfig.json'
      }
    }
  },
  
  rules: {
    // TypeScript rules
    '@typescript-eslint/no-unused-vars': ['error', {
      argsIgnorePattern: '^_',
      varsIgnorePattern: '^_'
    }],
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/explicit-module-boundary-types': 'off',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-non-null-assertion': 'warn',
    '@typescript-eslint/prefer-nullish-coalescing': 'error',
    '@typescript-eslint/prefer-optional-chain': 'error',
    
    // React rules
    'react/prop-types': 'off',  // ใช้ TypeScript แทน
    'react/display-name': 'warn',
    'react-hooks/rules-of-hooks': 'error',
    'react-hooks/exhaustive-deps': 'warn',
    
    // Import rules
    'import/order': ['error', {
      groups: [
        'builtin',
        'external',
        'internal',
        ['parent', 'sibling'],
        'index',
        'type'
      ],
      'newlines-between': 'always',
      alphabetize: { order: 'asc', caseInsensitive: true }
    }],
    'import/no-duplicates': 'error',
    
    // General rules
    'no-console': ['warn', { allow: ['warn', 'error'] }],
    'prefer-const': 'error',
    'eqeqeq': ['error', 'always']
  },
  
  ignorePatterns: [
    'dist',
    '.eslintrc.cjs',
    'vite.config.ts',
    '*.test.ts',
    '*.test.tsx'
  ]
};
```

### การตั้งค่า Prettier

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "es5",
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
      "files": "*.tsx",
      "options": {
        "printWidth": 120
      }
    }
  ]
}
```

```
// .prettierignore
dist
node_modules
coverage
*.min.js
```

### Scripts ใน package.json

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext ts,tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,json}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,css,json}\"",
    "type-check": "tsc --noEmit",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage",
    "validate": "npm run type-check && npm run lint && npm run format:check"
  }
}
```

---

## 25.11 Declaration Files

### การสร้าง global.d.ts

```typescript
// src/global.d.ts หรือ src/types/global.d.ts

// ประกาศ global variables
declare global {
  interface Window {
    analytics: {
      track: (event: string, properties?: Record<string, unknown>) => void;
      identify: (userId: string, traits?: Record<string, unknown>) => void;
    };
    __APP_VERSION__: string;
  }
}

// ประกาศ module สำหรับ file types
declare module '*.svg' {
  import React from 'react';
  export const ReactComponent: React.FC<React.SVGProps<SVGSVGElement>>;
  const src: string;
  export default src;
}

declare module '*.png' {
  const content: string;
  export default content;
}

declare module '*.jpg' {
  const content: string;
  export default content;
}

declare module '*.webp' {
  const content: string;
  export default content;
}

declare module '*.json' {
  const content: Record<string, unknown>;
  export default content;
}

// ส่งออกว่างเพื่อให้ TypeScript รู้ว่าเป็น module
export {};
```

---

## 25.12 ตัวอย่าง App.tsx สมบูรณ์

```typescript
// src/App.tsx
import React, { Suspense, lazy } from 'react';
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom';

// Lazy loading สำหรับ code splitting
const HomePage = lazy(() => import('./pages/Home'));
const AboutPage = lazy(() => import('./pages/About'));
const ProductsPage = lazy(() => import('./pages/Products'));
const NotFoundPage = lazy(() => import('./pages/NotFound'));

// Loading component
const LoadingSpinner: React.FC = () => (
  <div className="loading-container">
    <div className="spinner" role="status">
      <span className="sr-only">กำลังโหลด...</span>
    </div>
  </div>
);

// Error Boundary
interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<
  React.PropsWithChildren<{}>,
  ErrorBoundaryState
> {
  constructor(props: React.PropsWithChildren<{}>) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('Error caught by boundary:', error, errorInfo);
  }

  render(): React.ReactNode {
    if (this.state.hasError) {
      return (
        <div className="error-container">
          <h2>เกิดข้อผิดพลาดที่ไม่คาดคิด</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            ลองใหม่
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

const App: React.FC = () => {
  return (
    <ErrorBoundary>
      <Router>
        <Suspense fallback={<LoadingSpinner />}>
          <Routes>
            <Route path="/" element={<HomePage />} />
            <Route path="/about" element={<AboutPage />} />
            <Route path="/products" element={<ProductsPage />} />
            <Route path="/404" element={<NotFoundPage />} />
            <Route path="*" element={<Navigate to="/404" replace />} />
          </Routes>
        </Suspense>
      </Router>
    </ErrorBoundary>
  );
};

export default App;
```

---

## 25.13 Vitest Configuration (ทางเลือกแทน Jest)

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/setupTests.ts'],
    
    // CSS support
    css: {
      modules: {
        classNameStrategy: 'non-scoped'
      }
    },
    
    // Coverage
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/setupTests.ts',
        '**/*.d.ts',
        '**/*.test.{ts,tsx}'
      ]
    },
    
    // Include patterns
    include: ['src/**/*.{test,spec}.{ts,tsx}']
  },
  
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  }
});
```

```typescript
// src/setupTests.ts
import '@testing-library/jest-dom';
import { afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';

// Cleanup after each test
afterEach(() => {
  cleanup();
});

// Mock global objects
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: (query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: () => {},
    removeListener: () => {},
    addEventListener: () => {},
    removeEventListener: () => {},
    dispatchEvent: () => {}
  })
});
```

---

## สรุปบทที่ 25

ในบทนี้เราได้เรียนรู้:
- การสร้าง React project ด้วย TypeScript ทั้งแบบ CRA และ Vite
- การตั้งค่า tsconfig.json สำหรับ React
- โครงสร้าง project ที่แนะนำ
- React types สำคัญจาก @types/react
- การใช้ JSX กับ TypeScript
- CSS Modules กับ TypeScript
- Environment variables กับ TypeScript
- ESLint และ Prettier configuration
- Declaration files สำหรับ assets

**ขั้นตอนต่อไป**: ในบทที่ 26 เราจะเรียนรู้เกี่ยวกับ React Components และ Props กับ TypeScript อย่างละเอียด
