# บทที่ 2: การติดตั้งและตั้งค่า TypeScript

> ตั้งค่า Environment ให้พร้อมสำหรับการพัฒนา TypeScript อย่างมืออาชีพ

---

## สารบัญบทนี้

1. [การติดตั้ง Node.js และ npm](#1-การติดตั้ง-nodejs-และ-npm)
2. [การติดตั้ง TypeScript](#2-การติดตั้ง-typescript)
3. [การสร้างและตั้งค่า tsconfig.json](#3-การสร้างและตั้งค่า-tsconfigjson)
4. [การตั้งค่า VS Code](#4-การตั้งค่า-vs-code)
5. [ts-node สำหรับรัน TypeScript โดยตรง](#5-ts-node-สำหรับรัน-typescript-โดยตรง)
6. [โครงสร้างโปรเจกต์ที่ดี](#6-โครงสร้างโปรเจกต์ที่ดี)
7. [การตั้งค่า package.json Scripts](#7-การตั้งค่า-packagejson-scripts)
8. [โปรเจกต์ตัวอย่างสมบูรณ์](#8-โปรเจกต์ตัวอย่างสมบูรณ์)

---

## 1. การติดตั้ง Node.js และ npm

### ทำไมต้องมี Node.js?

TypeScript Compiler (tsc) ทำงานบน Node.js ดังนั้นก่อนติดตั้ง TypeScript เราต้องติดตั้ง Node.js ก่อน

### ดาวน์โหลดและติดตั้ง Node.js

**วิธีที่ 1: ดาวน์โหลดจาก Official Website**

1. ไปที่ [https://nodejs.org](https://nodejs.org)
2. เลือก **LTS version** (Long Term Support) สำหรับการใช้งานจริง
3. ดาวน์โหลดและรัน installer

**วิธีที่ 2: ใช้ Node Version Manager (แนะนำ)**

```bash
# ติดตั้ง nvm บน macOS/Linux
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# หรือ wget
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# รีสตาร์ท Terminal แล้วรัน
nvm install --lts          # ติดตั้ง LTS version
nvm use --lts              # ใช้ LTS version
nvm alias default --lts    # ตั้งเป็น default

# ดูเวอร์ชันที่ติดตั้ง
nvm list
```

```powershell
# ติดตั้ง nvm-windows
# ดาวน์โหลดจาก https://github.com/coreybutler/nvm-windows/releases

# หลังติดตั้ง nvm-windows
nvm install lts
nvm use lts
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Node.js
node --version
# ควรได้: v20.x.x (หรือสูงกว่า)

# ตรวจสอบ npm
npm --version
# ควรได้: 10.x.x (หรือสูงกว่า)

# ทดสอบ Node.js
node -e "console.log('Node.js ทำงานได้แล้ว!')"
```

### ทำความเข้าใจ npm

npm (Node Package Manager) คือตัวจัดการ packages ของ Node.js

```bash
# คำสั่ง npm พื้นฐาน
npm init                    # สร้าง package.json ใหม่
npm init -y                 # สร้าง package.json โดยไม่ถาม
npm install <package>       # ติดตั้ง package (อักษรย่อ: npm i)
npm install -g <package>    # ติดตั้ง package แบบ global
npm install --save-dev <package>  # ติดตั้งเป็น devDependency
npm uninstall <package>     # ถอนการติดตั้ง package
npm update                  # อัปเดต packages ทั้งหมด
npm list                    # แสดง packages ที่ติดตั้งแล้ว
npm run <script>            # รัน script ใน package.json
```

---

## 2. การติดตั้ง TypeScript

### ติดตั้ง TypeScript แบบ Global

```bash
# ติดตั้ง TypeScript compiler ทั่วทั้ง system
npm install -g typescript

# ตรวจสอบเวอร์ชัน
tsc --version
# ควรได้: Version 5.x.x
```

### ติดตั้ง TypeScript แบบ Local (แนะนำ)

การติดตั้งแบบ local ในแต่ละโปรเจกต์ดีกว่า เพราะ:
- แต่ละโปรเจกต์สามารถใช้ TypeScript เวอร์ชันต่างกันได้
- ป้องกันปัญหาที่เกิดจาก version mismatch
- Team ทุกคนใช้ TypeScript เวอร์ชันเดียวกัน

```bash
# สร้าง project ใหม่
mkdir my-typescript-project
cd my-typescript-project

# สร้าง package.json
npm init -y

# ติดตั้ง TypeScript เป็น devDependency
npm install --save-dev typescript

# ตรวจสอบ
npx tsc --version
```

### ตั้งค่า TypeScript ครั้งแรก

```bash
# สร้าง tsconfig.json อัตโนมัติ
npx tsc --init

# ผลลัพธ์จะสร้างไฟล์ tsconfig.json ใน directory ปัจจุบัน
```

### ทดสอบการ Compile TypeScript

```bash
# สร้างไฟล์ test
mkdir src
```

```typescript
// src/index.ts
const message: string = "สวัสดี TypeScript!";
console.log(message);

const add = (a: number, b: number): number => a + b;
console.log(`1 + 2 = ${add(1, 2)}`);
```

```bash
# Compile TypeScript
npx tsc src/index.ts

# จะได้ไฟล์ src/index.js
# รัน JavaScript ที่ได้
node src/index.js
# สวัสดี TypeScript!
# 1 + 2 = 3
```

---

## 3. การสร้างและตั้งค่า tsconfig.json

### ไฟล์ tsconfig.json คืออะไร?

`tsconfig.json` คือไฟล์ตั้งค่าสำหรับ TypeScript Compiler บอกให้ compiler รู้ว่า:
- ไฟล์ TypeScript อยู่ที่ไหน
- ต้องการ output เป็น JavaScript เวอร์ชันอะไร
- ตั้งค่า strict mode หรือเปล่า
- และอื่นๆ อีกมาก

### tsconfig.json สำหรับงาน Node.js

```json
{
    "compilerOptions": {
        // ==========================================
        // Language and Environment
        // ==========================================
        
        // JavaScript version ที่ต้องการ output
        // "ES3" | "ES5" | "ES6"/"ES2015" | "ES2016" | ... | "ES2023" | "ESNext"
        "target": "ES2020",
        
        // Libraries ที่รวม type definitions
        // "ES5" | "ES6" | "DOM" | "DOM.Iterable" | "ESNext" | etc.
        "lib": ["ES2020"],
        
        // ==========================================
        // Module Settings
        // ==========================================
        
        // ระบบ Module ที่ใช้
        // "CommonJS" (Node.js) | "ES6"/"ES2015" | "ESNext" | "Node16" | "NodeNext"
        "module": "commonjs",
        
        // วิธีที่ TypeScript ค้นหา modules
        // "node" | "bundler" | "node10" | "node16" | "nodenext"
        "moduleResolution": "node",
        
        // ==========================================
        // Type Checking (Strictness)
        // ==========================================
        
        // เปิด strict mode ทั้งหมด (แนะนำสูงสุด)
        "strict": true,
        
        // ตัวเลือก strict แต่ละตัว (ถ้าไม่ใช้ "strict": true)
        // "noImplicitAny": true,      // ห้าม any โดยปริยาย
        // "strictNullChecks": true,   // ตรวจสอบ null/undefined
        // "strictFunctionTypes": true, // ตรวจสอบ function types
        // "strictPropertyInitialization": true, // ต้อง initialize properties
        // "noImplicitThis": true,     // ห้าม this โดยปริยาย
        
        // ==========================================
        // Output Settings
        // ==========================================
        
        // ที่เก็บ JavaScript output
        "outDir": "./dist",
        
        // ที่เก็บ TypeScript source files
        "rootDir": "./src",
        
        // สร้าง .d.ts declaration files
        "declaration": true,
        
        // สร้าง .d.ts.map files
        "declarationMap": true,
        
        // สร้าง source map สำหรับ debugging
        "sourceMap": true,
        
        // ลบ comments ใน output
        "removeComments": false,
        
        // ==========================================
        // Interop Settings
        // ==========================================
        
        // ทำให้ CommonJS modules ทำงานกับ ES modules ได้
        "esModuleInterop": true,
        
        // อนุญาต default imports จาก modules ที่ไม่มี default export
        "allowSyntheticDefaultImports": true,
        
        // ==========================================
        // Additional Checks
        // ==========================================
        
        // Error ถ้า import แต่ไม่ใช้
        "noUnusedLocals": true,
        
        // Error ถ้า parameter ไม่ถูกใช้
        "noUnusedParameters": true,
        
        // Error ถ้า return path ไม่ครบ
        "noImplicitReturns": true,
        
        // Error ถ้า switch case ไม่มี break
        "noFallthroughCasesInSwitch": true,
        
        // ==========================================
        // Paths and Module Aliases
        // ==========================================
        
        // Path mapping สำหรับ imports
        "baseUrl": "./src",
        "paths": {
            "@/*": ["./*"],
            "@services/*": ["./services/*"],
            "@models/*": ["./models/*"],
            "@utils/*": ["./utils/*"]
        }
    },
    
    // ไฟล์ที่จะ compile
    "include": ["src/**/*"],
    
    // ไฟล์ที่ไม่ต้อง compile
    "exclude": [
        "node_modules",
        "dist",
        "**/*.test.ts",
        "**/*.spec.ts"
    ]
}
```

### tsconfig.json สำหรับ React + TypeScript

```json
{
    "compilerOptions": {
        "target": "ES2020",
        "lib": ["DOM", "DOM.Iterable", "ES2020"],
        "module": "ESNext",
        "moduleResolution": "bundler",
        "jsx": "react-jsx",
        "strict": true,
        "outDir": "./dist",
        "rootDir": "./src",
        "declaration": true,
        "sourceMap": true,
        "esModuleInterop": true,
        "allowSyntheticDefaultImports": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "noImplicitReturns": true,
        "baseUrl": "./src",
        "paths": {
            "@components/*": ["components/*"],
            "@hooks/*": ["hooks/*"],
            "@pages/*": ["pages/*"],
            "@services/*": ["services/*"],
            "@types/*": ["types/*"],
            "@utils/*": ["utils/*"]
        }
    },
    "include": ["src"],
    "exclude": ["node_modules", "build"]
}
```

### tsconfig.json Options ที่สำคัญ

```bash
# ดู options ทั้งหมด
npx tsc --help --all

# หรือดูใน tsconfig.json ที่สร้างอัตโนมัติ
npx tsc --init
```

**ตารางสรุป Options ที่ใช้บ่อย:**

| Option | Default | คำอธิบาย |
|--------|---------|-----------|
| `target` | ES3 | JavaScript version ที่ output |
| `module` | CommonJS | ระบบ Module |
| `strict` | false | เปิด strict checks ทั้งหมด |
| `outDir` | - | ที่เก็บ output files |
| `rootDir` | - | ที่เก็บ source files |
| `sourceMap` | false | สร้าง source map |
| `declaration` | false | สร้าง .d.ts files |
| `noImplicitAny` | false | ห้าม any โดยปริยาม |
| `strictNullChecks` | false | ตรวจสอบ null/undefined |
| `esModuleInterop` | false | ช่วย CommonJS + ESM |

---

## 4. การตั้งค่า VS Code

### ทำไมต้องใช้ VS Code?

VS Code มี built-in support สำหรับ TypeScript ที่ดีที่สุด เพราะ:
- TypeScript Language Server ทำงานอยู่ใน background
- Auto-completion (IntelliSense) สำหรับ TypeScript
- Real-time error highlighting
- Refactoring tools
- Debugging support

### Extensions ที่แนะนำ

**Essential Extensions:**

```
1. TypeScript (built-in) - มาพร้อม VS Code แล้ว

2. ESLint (dbaeumer.vscode-eslint)
   - ตรวจสอบ code style และ errors
   
3. Prettier - Code formatter (esbenp.prettier-vscode)
   - จัด format โค้ดให้สวยงาม

4. Error Lens (usernamehw.errorlens)
   - แสดง error โดยตรงในบรรทัดโค้ด
   
5. TypeScript Importer (pmneo.tsimporter)
   - auto-import TypeScript modules
```

**Productivity Extensions:**

```
6. GitLens (eamodio.gitlens)
   - ดู git history และ blame

7. Thunder Client (rangav.vscode-thunder-client)
   - ทดสอบ API โดยตรงใน VS Code
   
8. REST Client (humao.rest-client)
   - ทดสอบ API ด้วย .http files

9. Docker (ms-azuretools.vscode-docker)
   - รองรับ Docker files
   
10. TODO Highlight (wayou.vscode-todo-highlight)
    - Highlight TODO comments
```

### VS Code Settings สำหรับ TypeScript

สร้างไฟล์ `.vscode/settings.json` ในโปรเจกต์:

```json
{
    // Editor settings
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.codeActionsOnSave": {
        "source.fixAll.eslint": "explicit",
        "source.organizeImports": "explicit"
    },
    
    // TypeScript settings
    "typescript.updateImportsOnFileMove.enabled": "always",
    "typescript.suggest.autoImports": true,
    "typescript.preferences.includePackageJsonAutoImports": "auto",
    "typescript.inlayHints.parameterNames.enabled": "all",
    "typescript.inlayHints.variableTypes.enabled": true,
    "typescript.inlayHints.returnTypes.enabled": true,
    
    // File associations
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    "[typescriptreact]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    
    // Search settings
    "search.exclude": {
        "**/node_modules": true,
        "**/dist": true,
        "**/.git": true
    },
    
    // File explorer
    "files.exclude": {
        "**/.git": true,
        "**/node_modules": true
    }
}
```

### การตั้งค่า ESLint สำหรับ TypeScript

```bash
# ติดตั้ง ESLint และ TypeScript ESLint
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

สร้างไฟล์ `.eslintrc.js`:

```javascript
// .eslintrc.js
module.exports = {
    parser: "@typescript-eslint/parser",
    extends: [
        "eslint:recommended",
        "plugin:@typescript-eslint/recommended",
        "plugin:@typescript-eslint/recommended-requiring-type-checking"
    ],
    plugins: ["@typescript-eslint"],
    parserOptions: {
        ecmaVersion: 2020,
        sourceType: "module",
        project: "./tsconfig.json"
    },
    rules: {
        // TypeScript-specific rules
        "@typescript-eslint/explicit-function-return-type": "warn",
        "@typescript-eslint/no-explicit-any": "warn",
        "@typescript-eslint/no-unused-vars": "error",
        "@typescript-eslint/prefer-const": "error",
        
        // General rules
        "no-console": "warn",
        "semi": ["error", "always"],
        "quotes": ["error", "double"]
    },
    env: {
        node: true,
        es2020: true
    }
};
```

### การตั้งค่า Prettier

สร้างไฟล์ `.prettierrc`:

```json
{
    "printWidth": 80,
    "tabWidth": 4,
    "useTabs": false,
    "semi": true,
    "singleQuote": false,
    "quoteProps": "as-needed",
    "jsxSingleQuote": false,
    "trailingComma": "es5",
    "bracketSpacing": true,
    "arrowParens": "always",
    "endOfLine": "lf"
}
```

---

## 5. ts-node สำหรับรัน TypeScript โดยตรง

### ts-node คืออะไร?

ts-node คือ TypeScript execution engine ที่ช่วยให้รัน TypeScript ได้โดยตรง โดยไม่ต้อง compile เป็น JavaScript ก่อน

```bash
# ติดตั้ง ts-node
npm install --save-dev ts-node

# หรือติดตั้ง global
npm install -g ts-node
```

### การใช้งาน ts-node

```bash
# รัน TypeScript file โดยตรง
npx ts-node src/index.ts

# Interactive REPL
npx ts-node

# ใน REPL
> const x: number = 42
> x * 2
84
> .exit
```

### tsx - รุ่นใหม่ที่เร็วกว่า

```bash
# ติดตั้ง tsx (เร็วกว่า ts-node)
npm install --save-dev tsx

# รัน TypeScript
npx tsx src/index.ts

# Watch mode (รัน auto เมื่อไฟล์เปลี่ยน)
npx tsx watch src/index.ts
```

### ตั้งค่า ts-node ใน tsconfig.json

```json
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "commonjs",
        "strict": true
    },
    "ts-node": {
        "transpileOnly": true,
        "files": true,
        "esm": false
    }
}
```

---

## 6. โครงสร้างโปรเจกต์ที่ดี

### โครงสร้างพื้นฐาน

```
my-typescript-project/
├── src/                        # TypeScript source files
│   ├── index.ts                # Entry point
│   ├── types/                  # Type definitions
│   │   ├── index.ts
│   │   └── models.ts
│   ├── services/               # Business logic
│   │   ├── userService.ts
│   │   └── productService.ts
│   ├── utils/                  # Utility functions
│   │   ├── helpers.ts
│   │   └── validators.ts
│   └── config/                 # Configuration
│       └── database.ts
├── tests/                      # Test files
│   ├── unit/
│   └── integration/
├── dist/                       # Compiled JavaScript (gitignored)
├── node_modules/               # Dependencies (gitignored)
├── .vscode/                    # VS Code settings
│   └── settings.json
├── .eslintrc.js                # ESLint config
├── .prettierrc                 # Prettier config
├── .gitignore                  # Git ignore file
├── package.json                # Project manifest
├── tsconfig.json               # TypeScript config
└── README.md                   # Documentation
```

### โครงสร้างสำหรับโปรเจกต์ขนาดใหญ่

```
large-project/
├── src/
│   ├── api/                    # API layer
│   │   ├── controllers/
│   │   ├── middlewares/
│   │   └── routes/
│   ├── core/                   # Core business logic
│   │   ├── entities/
│   │   ├── repositories/
│   │   └── use-cases/
│   ├── infrastructure/         # External interfaces
│   │   ├── database/
│   │   ├── email/
│   │   └── storage/
│   ├── shared/                 # Shared utilities
│   │   ├── constants/
│   │   ├── errors/
│   │   ├── types/
│   │   └── utils/
│   └── main.ts                 # Entry point
├── tests/
│   ├── unit/
│   │   ├── core/
│   │   └── shared/
│   ├── integration/
│   │   └── api/
│   └── e2e/
└── ...
```

### ไฟล์ .gitignore สำหรับ TypeScript Project

```gitignore
# Node.js
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# TypeScript
dist/
build/
*.js.map
*.d.ts.map

# Environment
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# IDE
.vscode/*
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Coverage
coverage/
.nyc_output/

# Logs
logs/
*.log
```

---

## 7. การตั้งค่า package.json Scripts

### scripts พื้นฐาน

```json
{
    "name": "my-typescript-project",
    "version": "1.0.0",
    "description": "โปรเจกต์ TypeScript ของฉัน",
    "main": "dist/index.js",
    "scripts": {
        "build": "tsc",
        "build:watch": "tsc --watch",
        "start": "node dist/index.js",
        "dev": "tsx watch src/index.ts",
        "lint": "eslint src --ext .ts",
        "lint:fix": "eslint src --ext .ts --fix",
        "format": "prettier --write src/**/*.ts",
        "typecheck": "tsc --noEmit",
        "clean": "rm -rf dist"
    },
    "devDependencies": {
        "typescript": "^5.0.0",
        "ts-node": "^10.9.0",
        "tsx": "^4.0.0",
        "eslint": "^8.0.0",
        "@typescript-eslint/parser": "^6.0.0",
        "@typescript-eslint/eslint-plugin": "^6.0.0",
        "prettier": "^3.0.0"
    }
}
```

### scripts สำหรับโปรเจกต์ที่มี Testing

```json
{
    "scripts": {
        "build": "tsc",
        "build:watch": "tsc --watch",
        "start": "node dist/index.js",
        "dev": "tsx watch src/index.ts",
        "test": "jest",
        "test:watch": "jest --watch",
        "test:coverage": "jest --coverage",
        "test:ci": "jest --ci --coverage --runInBand",
        "lint": "eslint src --ext .ts",
        "lint:fix": "eslint src --ext .ts --fix",
        "format": "prettier --write \"src/**/*.ts\"",
        "format:check": "prettier --check \"src/**/*.ts\"",
        "typecheck": "tsc --noEmit",
        "validate": "npm run typecheck && npm run lint && npm run test",
        "clean": "rm -rf dist coverage",
        "prebuild": "npm run clean",
        "prepare": "npm run build"
    }
}
```

---

## 8. โปรเจกต์ตัวอย่างสมบูรณ์

### สร้างโปรเจกต์ TypeScript ตั้งแต่ต้น

```bash
# 1. สร้าง directory
mkdir typescript-todo-app
cd typescript-todo-app

# 2. สร้าง package.json
npm init -y

# 3. ติดตั้ง dependencies
npm install --save-dev typescript ts-node tsx
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install --save-dev prettier

# 4. สร้าง tsconfig.json
npx tsc --init

# 5. สร้างโครงสร้าง directories
mkdir -p src/types src/models src/services src/utils
```

### ไฟล์ทั้งหมดในโปรเจกต์

**package.json:**
```json
{
    "name": "typescript-todo-app",
    "version": "1.0.0",
    "description": "Todo App ด้วย TypeScript",
    "main": "dist/index.js",
    "scripts": {
        "build": "tsc",
        "start": "node dist/index.js",
        "dev": "tsx watch src/index.ts",
        "typecheck": "tsc --noEmit"
    },
    "devDependencies": {
        "@typescript-eslint/eslint-plugin": "^6.0.0",
        "@typescript-eslint/parser": "^6.0.0",
        "eslint": "^8.0.0",
        "prettier": "^3.0.0",
        "ts-node": "^10.9.0",
        "tsx": "^4.0.0",
        "typescript": "^5.0.0"
    }
}
```

**tsconfig.json:**
```json
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "commonjs",
        "lib": ["ES2020"],
        "outDir": "./dist",
        "rootDir": "./src",
        "strict": true,
        "esModuleInterop": true,
        "allowSyntheticDefaultImports": true,
        "declaration": true,
        "sourceMap": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "noImplicitReturns": true,
        "baseUrl": "./src",
        "paths": {
            "@types/*": ["types/*"],
            "@models/*": ["models/*"],
            "@services/*": ["services/*"],
            "@utils/*": ["utils/*"]
        }
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

**src/types/index.ts:**
```typescript
// Type definitions ทั้งหมด

export type Priority = "low" | "medium" | "high";
export type Status = "pending" | "in-progress" | "completed" | "cancelled";

export interface Todo {
    id: string;
    title: string;
    description?: string;
    priority: Priority;
    status: Status;
    createdAt: Date;
    updatedAt: Date;
    completedAt?: Date;
    tags: string[];
}

export interface CreateTodoInput {
    title: string;
    description?: string;
    priority?: Priority;
    tags?: string[];
}

export interface UpdateTodoInput {
    title?: string;
    description?: string;
    priority?: Priority;
    status?: Status;
    tags?: string[];
}

export interface TodoFilter {
    status?: Status;
    priority?: Priority;
    tag?: string;
}

export interface PaginationOptions {
    page: number;
    limit: number;
}

export interface PaginatedResult<T> {
    data: T[];
    total: number;
    page: number;
    totalPages: number;
}
```

**src/utils/helpers.ts:**
```typescript
// Utility functions

import { v4 as uuidv4 } from "uuid";

// สร้าง unique ID
export function generateId(): string {
    return uuidv4();
}

// Format date เป็น Thai locale
export function formatDate(date: Date): string {
    return date.toLocaleDateString("th-TH", {
        year: "numeric",
        month: "long",
        day: "numeric",
        hour: "2-digit",
        minute: "2-digit"
    });
}

// ตรวจสอบว่า string ว่างเปล่าหรือไม่
export function isEmpty(str: string | undefined | null): boolean {
    return !str || str.trim().length === 0;
}

// ลบ whitespace และ validate ชื่อ
export function validateTitle(title: string): string {
    const trimmed = title.trim();
    if (isEmpty(trimmed)) {
        throw new Error("ชื่อ Todo ต้องไม่ว่างเปล่า");
    }
    if (trimmed.length > 200) {
        throw new Error("ชื่อ Todo ต้องไม่เกิน 200 ตัวอักษร");
    }
    return trimmed;
}
```

**src/services/todoService.ts:**
```typescript
// Business logic สำหรับจัดการ Todos

import {
    Todo,
    CreateTodoInput,
    UpdateTodoInput,
    TodoFilter,
    PaginationOptions,
    PaginatedResult,
    Priority,
    Status
} from "@types/index";
import { generateId, validateTitle } from "@utils/helpers";

export class TodoService {
    private todos: Map<string, Todo> = new Map();
    
    // สร้าง Todo ใหม่
    create(input: CreateTodoInput): Todo {
        const title = validateTitle(input.title);
        
        const now = new Date();
        const todo: Todo = {
            id: generateId(),
            title,
            description: input.description,
            priority: input.priority ?? "medium",
            status: "pending",
            createdAt: now,
            updatedAt: now,
            tags: input.tags ?? []
        };
        
        this.todos.set(todo.id, todo);
        return todo;
    }
    
    // อัปเดต Todo
    update(id: string, input: UpdateTodoInput): Todo {
        const todo = this.findById(id);
        
        const updatedTodo: Todo = {
            ...todo,
            ...(input.title && { title: validateTitle(input.title) }),
            ...(input.description !== undefined && { description: input.description }),
            ...(input.priority && { priority: input.priority }),
            ...(input.tags && { tags: input.tags }),
            updatedAt: new Date()
        };
        
        if (input.status) {
            updatedTodo.status = input.status;
            if (input.status === "completed") {
                updatedTodo.completedAt = new Date();
            }
        }
        
        this.todos.set(id, updatedTodo);
        return updatedTodo;
    }
    
    // ลบ Todo
    delete(id: string): void {
        if (!this.todos.has(id)) {
            throw new Error(`ไม่พบ Todo ID: ${id}`);
        }
        this.todos.delete(id);
    }
    
    // ค้นหาด้วย ID
    findById(id: string): Todo {
        const todo = this.todos.get(id);
        if (!todo) {
            throw new Error(`ไม่พบ Todo ID: ${id}`);
        }
        return todo;
    }
    
    // ดู Todos ทั้งหมดพร้อม filter
    findAll(
        filter?: TodoFilter,
        pagination?: PaginationOptions
    ): PaginatedResult<Todo> {
        let todos = Array.from(this.todos.values());
        
        // Apply filters
        if (filter) {
            if (filter.status) {
                todos = todos.filter(t => t.status === filter.status);
            }
            if (filter.priority) {
                todos = todos.filter(t => t.priority === filter.priority);
            }
            if (filter.tag) {
                todos = todos.filter(t => t.tags.includes(filter.tag!));
            }
        }
        
        // Sort by createdAt descending
        todos.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
        
        const total = todos.length;
        
        // Apply pagination
        if (pagination) {
            const { page, limit } = pagination;
            const start = (page - 1) * limit;
            todos = todos.slice(start, start + limit);
            
            return {
                data: todos,
                total,
                page,
                totalPages: Math.ceil(total / limit)
            };
        }
        
        return {
            data: todos,
            total,
            page: 1,
            totalPages: 1
        };
    }
    
    // สถิติ
    getStats(): {
        total: number;
        byStatus: Record<Status, number>;
        byPriority: Record<Priority, number>;
    } {
        const todos = Array.from(this.todos.values());
        
        const byStatus: Record<Status, number> = {
            pending: 0,
            "in-progress": 0,
            completed: 0,
            cancelled: 0
        };
        
        const byPriority: Record<Priority, number> = {
            low: 0,
            medium: 0,
            high: 0
        };
        
        todos.forEach(todo => {
            byStatus[todo.status]++;
            byPriority[todo.priority]++;
        });
        
        return {
            total: todos.length,
            byStatus,
            byPriority
        };
    }
}
```

**src/index.ts:**
```typescript
// Entry point ของโปรแกรม

import { TodoService } from "@services/todoService";

async function main(): Promise<void> {
    const service = new TodoService();
    
    console.log("🚀 เริ่มต้น TypeScript Todo App\n");
    
    // สร้าง Todos
    const todo1 = service.create({
        title: "เรียน TypeScript",
        description: "เรียนตั้งแต่ Part 1 ถึง Part 10",
        priority: "high",
        tags: ["education", "programming"]
    });
    
    const todo2 = service.create({
        title: "สร้าง Portfolio Website",
        priority: "medium",
        tags: ["project", "web"]
    });
    
    const todo3 = service.create({
        title: "อ่านหนังสือ Clean Code",
        priority: "low",
        tags: ["education", "books"]
    });
    
    // อัปเดต status
    service.update(todo1.id, { status: "in-progress" });
    
    // ดูสถิติ
    const stats = service.getStats();
    console.log("📊 สถิติ Todos:");
    console.log(`   รวมทั้งหมด: ${stats.total}`);
    console.log(`   กำลังทำ: ${stats.byStatus["in-progress"]}`);
    console.log(`   รอทำ: ${stats.byStatus.pending}`);
    
    // ดู Todos ที่มี priority สูง
    const highPriority = service.findAll({ priority: "high" });
    console.log("\n🔴 งานสำคัญ:");
    highPriority.data.forEach(t => {
        console.log(`   - [${t.status}] ${t.title}`);
    });
    
    console.log("\n✅ โปรแกรมทำงานสำเร็จ!");
}

main().catch(console.error);
```

### รัน โปรเจกต์

```bash
# Development mode (auto-reload เมื่อไฟล์เปลี่ยน)
npm run dev

# Build สำหรับ production
npm run build

# รัน production build
npm start

# Type check โดยไม่ build
npm run typecheck
```

---

### สรุปบทเรียน

ในบทนี้คุณได้เรียนรู้:

1. **การติดตั้ง Node.js** และเครื่องมือที่จำเป็น
2. **การติดตั้ง TypeScript** ทั้งแบบ global และ local
3. **tsconfig.json** และ options ที่สำคัญ
4. **VS Code setup** สำหรับพัฒนา TypeScript
5. **ts-node และ tsx** สำหรับรัน TypeScript โดยตรง
6. **โครงสร้างโปรเจกต์** ที่ดีและ scalable
7. **package.json scripts** สำหรับ workflow
8. **โปรเจกต์ตัวอย่างสมบูรณ์** ที่ใช้งานได้จริง

---

*บทต่อไป: [Part 03 - Types พื้นฐาน](part-03-basic-types.md)*
