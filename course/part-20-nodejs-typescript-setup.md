# ตอนที่ 20: การตั้งค่า Node.js กับ TypeScript

## บทนำ

Node.js เป็น JavaScript runtime ที่ทรงพลัง และเมื่อนำมาใช้ร่วมกับ TypeScript จะทำให้การพัฒนาแอพพลิเคชันฝั่ง server มีความปลอดภัยด้านประเภทข้อมูล (type-safe) มากยิ่งขึ้น ในบทนี้เราจะเรียนรู้วิธีการตั้งค่าโปรเจกต์ Node.js กับ TypeScript ตั้งแต่ต้น

---

## 20.1 การสร้างโปรเจกต์ใหม่

### ขั้นตอนเริ่มต้น

```bash
# สร้างโฟลเดอร์โปรเจกต์
mkdir my-node-ts-project
cd my-node-ts-project

# เริ่มต้น npm project
npm init -y

# ติดตั้ง TypeScript และ Node types
npm install --save-dev typescript @types/node

# ติดตั้ง ts-node สำหรับรันโค้ด TypeScript โดยตรง
npm install --save-dev ts-node ts-node-dev

# สร้างไฟล์ tsconfig.json
npx tsc --init
```

### package.json พื้นฐาน

```json
{
  "name": "my-node-ts-project",
  "version": "1.0.0",
  "description": "Node.js project with TypeScript",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "dev:debug": "ts-node-dev --respawn --transpile-only --inspect src/index.ts",
    "clean": "rm -rf dist",
    "prebuild": "npm run clean",
    "lint": "eslint src --ext .ts",
    "test": "jest"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "ts-node": "^10.9.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## 20.2 การตั้งค่า tsconfig.json สำหรับ Node.js

### tsconfig.json พื้นฐาน

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
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### tsconfig.json แบบขั้นสูง พร้อม path aliases

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
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "strictNullChecks": true,
    "baseUrl": "./src",
    "paths": {
      "@controllers/*": ["controllers/*"],
      "@services/*": ["services/*"],
      "@models/*": ["models/*"],
      "@middleware/*": ["middleware/*"],
      "@utils/*": ["utils/*"],
      "@config/*": ["config/*"],
      "@types/*": ["types/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

---

## 20.3 โครงสร้างโฟลเดอร์สำหรับโปรเจกต์ Node.js

```
my-node-ts-project/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── environment.ts
│   │   └── index.ts
│   ├── controllers/
│   │   ├── userController.ts
│   │   └── productController.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── errorHandler.ts
│   │   └── validation.ts
│   ├── models/
│   │   ├── User.ts
│   │   └── Product.ts
│   ├── routes/
│   │   ├── userRoutes.ts
│   │   └── productRoutes.ts
│   ├── services/
│   │   ├── userService.ts
│   │   └── productService.ts
│   ├── types/
│   │   ├── express.d.ts
│   │   └── index.ts
│   ├── utils/
│   │   ├── logger.ts
│   │   ├── helpers.ts
│   │   └── validators.ts
│   └── index.ts
├── tests/
│   ├── unit/
│   └── integration/
├── .env
├── .env.example
├── .gitignore
├── package.json
└── tsconfig.json
```

---

## 20.4 ts-node และ ts-node-dev

### การใช้งาน ts-node

```bash
# รันไฟล์ TypeScript โดยตรง
npx ts-node src/index.ts

# รันพร้อม transpile-only (เร็วกว่า ไม่ตรวจสอบ types)
npx ts-node --transpile-only src/index.ts

# รันด้วย esm module
npx ts-node --esm src/index.ts

# รันพร้อม custom tsconfig
npx ts-node --project tsconfig.dev.json src/index.ts
```

### การใช้งาน ts-node-dev

```bash
# รันแบบ watch mode (restart เมื่อไฟล์เปลี่ยน)
npx ts-node-dev src/index.ts

# รันพร้อม options
npx ts-node-dev --respawn --transpile-only src/index.ts

# รันพร้อม ignore บางไฟล์
npx ts-node-dev --respawn --ignore-watch node_modules src/index.ts
```

### package.json scripts สำหรับ development

```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only --exit-child src/index.ts",
    "dev:watch": "ts-node-dev --respawn --transpile-only --watch src src/index.ts",
    "debug": "ts-node-dev --respawn --transpile-only --inspect=0.0.0.0:9229 src/index.ts",
    "build": "tsc --build",
    "build:watch": "tsc --watch",
    "start": "node dist/index.js",
    "start:prod": "NODE_ENV=production node dist/index.js"
  }
}
```

---

## 20.5 การตั้งค่า nodemon

### ติดตั้ง nodemon

```bash
npm install --save-dev nodemon
```

### nodemon.json

```json
{
  "watch": ["src"],
  "ext": "ts,json",
  "ignore": ["src/**/*.test.ts", "src/**/*.spec.ts"],
  "exec": "ts-node --transpile-only src/index.ts",
  "env": {
    "NODE_ENV": "development"
  }
}
```

### package.json กับ nodemon

```json
{
  "scripts": {
    "dev:nodemon": "nodemon",
    "dev:nodemon-script": "nodemon --exec 'ts-node --transpile-only' src/index.ts"
  }
}
```

---

## 20.6 Path Aliases

### การตั้งค่า path aliases ใน tsconfig.json

```json
{
  "compilerOptions": {
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"],
      "@config/*": ["config/*"],
      "@controllers/*": ["controllers/*"],
      "@services/*": ["services/*"],
      "@models/*": ["models/*"],
      "@middleware/*": ["middleware/*"],
      "@utils/*": ["utils/*"],
      "@types/*": ["types/*"]
    }
  }
}
```

### ติดตั้ง module-alias สำหรับ runtime

```bash
npm install module-alias
npm install --save-dev @types/module-alias
```

### การใช้งาน module-alias

```typescript
// src/index.ts - ต้องเรียกก่อน import อื่น ๆ
import 'module-alias/register';

// หรือใน package.json
// "_moduleAliases": {
//   "@": "dist",
//   "@config": "dist/config",
//   "@controllers": "dist/controllers"
// }
```

### package.json พร้อม moduleAliases

```json
{
  "_moduleAliases": {
    "@": "dist",
    "@config": "dist/config",
    "@controllers": "dist/controllers",
    "@services": "dist/services",
    "@models": "dist/models",
    "@middleware": "dist/middleware",
    "@utils": "dist/utils",
    "@types": "dist/types"
  }
}
```

### ตัวอย่างการ import ด้วย path aliases

```typescript
// แทนที่จะใช้ relative path แบบนี้:
import { UserService } from '../../../services/userService';
import { logger } from '../../utils/logger';

// ใช้ path aliases แทน:
import { UserService } from '@services/userService';
import { logger } from '@utils/logger';
```

### ทางเลือก: tsconfig-paths

```bash
npm install --save-dev tsconfig-paths
```

```json
{
  "scripts": {
    "dev": "ts-node-dev -r tsconfig-paths/register --respawn src/index.ts",
    "start": "node -r tsconfig-paths/register dist/index.js"
  }
}
```

---

## 20.7 Environment Variables กับ dotenv

### ติดตั้ง dotenv

```bash
npm install dotenv
npm install --save-dev @types/node
```

### ไฟล์ .env

```bash
# .env
NODE_ENV=development
PORT=3000
HOST=localhost
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=my-super-secret-key
JWT_EXPIRES_IN=7d
REDIS_URL=redis://localhost:6379
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=user@gmail.com
SMTP_PASS=password
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=my-bucket
LOG_LEVEL=debug
```

### ไฟล์ .env.example

```bash
# .env.example - ไฟล์ตัวอย่าง (ควร commit ขึ้น git)
NODE_ENV=development
PORT=3000
HOST=localhost
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=
JWT_EXPIRES_IN=7d
REDIS_URL=redis://localhost:6379
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=
LOG_LEVEL=debug
```

### src/config/environment.ts - Type-safe environment variables

```typescript
import dotenv from 'dotenv';
import path from 'path';

// โหลด .env ไฟล์
dotenv.config({ path: path.resolve(process.cwd(), '.env') });

// กำหนด interface สำหรับ environment variables
interface EnvironmentVariables {
  NODE_ENV: 'development' | 'production' | 'test';
  PORT: number;
  HOST: string;
  DATABASE_URL: string;
  JWT_SECRET: string;
  JWT_EXPIRES_IN: string;
  REDIS_URL?: string;
  SMTP_HOST?: string;
  SMTP_PORT?: number;
  SMTP_USER?: string;
  SMTP_PASS?: string;
  AWS_ACCESS_KEY_ID?: string;
  AWS_SECRET_ACCESS_KEY?: string;
  AWS_REGION?: string;
  AWS_S3_BUCKET?: string;
  LOG_LEVEL: 'error' | 'warn' | 'info' | 'debug';
}

// ฟังก์ชันตรวจสอบและแปลงค่า environment variables
function getEnvironmentVariables(): EnvironmentVariables {
  const {
    NODE_ENV,
    PORT,
    HOST,
    DATABASE_URL,
    JWT_SECRET,
    JWT_EXPIRES_IN,
    REDIS_URL,
    SMTP_HOST,
    SMTP_PORT,
    SMTP_USER,
    SMTP_PASS,
    AWS_ACCESS_KEY_ID,
    AWS_SECRET_ACCESS_KEY,
    AWS_REGION,
    AWS_S3_BUCKET,
    LOG_LEVEL,
  } = process.env;

  // ตรวจสอบ required variables
  const requiredVars = ['DATABASE_URL', 'JWT_SECRET'];
  for (const varName of requiredVars) {
    if (!process.env[varName]) {
      throw new Error(`Environment variable ${varName} is required but not set`);
    }
  }

  return {
    NODE_ENV: (NODE_ENV as EnvironmentVariables['NODE_ENV']) || 'development',
    PORT: PORT ? parseInt(PORT, 10) : 3000,
    HOST: HOST || 'localhost',
    DATABASE_URL: DATABASE_URL!,
    JWT_SECRET: JWT_SECRET!,
    JWT_EXPIRES_IN: JWT_EXPIRES_IN || '7d',
    REDIS_URL,
    SMTP_HOST,
    SMTP_PORT: SMTP_PORT ? parseInt(SMTP_PORT, 10) : undefined,
    SMTP_USER,
    SMTP_PASS,
    AWS_ACCESS_KEY_ID,
    AWS_SECRET_ACCESS_KEY,
    AWS_REGION,
    AWS_S3_BUCKET,
    LOG_LEVEL: (LOG_LEVEL as EnvironmentVariables['LOG_LEVEL']) || 'info',
  };
}

export const env = getEnvironmentVariables();
export type { EnvironmentVariables };
```

### src/config/index.ts - Configuration object

```typescript
import { env } from './environment';

export const config = {
  app: {
    env: env.NODE_ENV,
    port: env.PORT,
    host: env.HOST,
    isDevelopment: env.NODE_ENV === 'development',
    isProduction: env.NODE_ENV === 'production',
    isTest: env.NODE_ENV === 'test',
  },
  database: {
    url: env.DATABASE_URL,
  },
  jwt: {
    secret: env.JWT_SECRET,
    expiresIn: env.JWT_EXPIRES_IN,
  },
  redis: {
    url: env.REDIS_URL,
  },
  smtp: {
    host: env.SMTP_HOST,
    port: env.SMTP_PORT,
    user: env.SMTP_USER,
    pass: env.SMTP_PASS,
  },
  aws: {
    accessKeyId: env.AWS_ACCESS_KEY_ID,
    secretAccessKey: env.AWS_SECRET_ACCESS_KEY,
    region: env.AWS_REGION,
    s3Bucket: env.AWS_S3_BUCKET,
  },
  log: {
    level: env.LOG_LEVEL,
  },
} as const;

export type Config = typeof config;
```

---

## 20.8 Common Node.js Types (@types/node)

### การทำงานกับ File System

```typescript
import fs from 'fs';
import { promises as fsPromises } from 'fs';
import path from 'path';

// อ่านไฟล์แบบ synchronous
function readFileSync(filePath: string): string {
  const absolutePath = path.resolve(filePath);
  return fs.readFileSync(absolutePath, 'utf-8');
}

// อ่านไฟล์แบบ asynchronous ด้วย Promise
async function readFileAsync(filePath: string): Promise<string> {
  const absolutePath = path.resolve(filePath);
  return fsPromises.readFile(absolutePath, 'utf-8');
}

// เขียนไฟล์
async function writeFileAsync(filePath: string, content: string): Promise<void> {
  const absolutePath = path.resolve(filePath);
  const dir = path.dirname(absolutePath);
  
  // สร้างโฟลเดอร์ถ้ายังไม่มี
  await fsPromises.mkdir(dir, { recursive: true });
  await fsPromises.writeFile(absolutePath, content, 'utf-8');
}

// ตรวจสอบว่าไฟล์/โฟลเดอร์มีอยู่
async function fileExists(filePath: string): Promise<boolean> {
  try {
    await fsPromises.access(filePath);
    return true;
  } catch {
    return false;
  }
}

// รายการไฟล์ในโฟลเดอร์
async function listDirectory(dirPath: string): Promise<string[]> {
  const entries = await fsPromises.readdir(dirPath, { withFileTypes: true });
  return entries.map(entry => entry.name);
}

// ลบไฟล์
async function deleteFile(filePath: string): Promise<void> {
  await fsPromises.unlink(filePath);
}

// คัดลอกไฟล์
async function copyFile(source: string, destination: string): Promise<void> {
  await fsPromises.copyFile(source, destination);
}

// ข้อมูล stats ของไฟล์
interface FileStats {
  size: number;
  isFile: boolean;
  isDirectory: boolean;
  createdAt: Date;
  modifiedAt: Date;
}

async function getFileStats(filePath: string): Promise<FileStats> {
  const stats = await fsPromises.stat(filePath);
  return {
    size: stats.size,
    isFile: stats.isFile(),
    isDirectory: stats.isDirectory(),
    createdAt: stats.birthtime,
    modifiedAt: stats.mtime,
  };
}

// การใช้งาน
async function main(): Promise<void> {
  // เขียนไฟล์
  await writeFileAsync('./output/hello.txt', 'Hello, TypeScript!');
  
  // ตรวจสอบว่ามีไฟล์
  const exists = await fileExists('./output/hello.txt');
  console.log('File exists:', exists);
  
  // อ่านไฟล์
  const content = await readFileAsync('./output/hello.txt');
  console.log('Content:', content);
  
  // ข้อมูล stats
  const stats = await getFileStats('./output/hello.txt');
  console.log('File stats:', stats);
}

main().catch(console.error);
```

---

## 20.9 การทำงานกับ Streams

### ReadStream และ WriteStream

```typescript
import fs from 'fs';
import path from 'path';
import { Transform, Readable, Writable, pipeline } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// อ่านไฟล์ขนาดใหญ่ด้วย stream
async function processLargeFile(inputPath: string, outputPath: string): Promise<void> {
  const readStream = fs.createReadStream(inputPath, { encoding: 'utf-8' });
  const writeStream = fs.createWriteStream(outputPath);
  
  // Transform stream สำหรับแปลงข้อมูล
  const transformStream = new Transform({
    transform(chunk: Buffer, encoding: string, callback: () => void) {
      // แปลงข้อความเป็นตัวพิมพ์ใหญ่
      const transformedChunk = chunk.toString().toUpperCase();
      this.push(transformedChunk);
      callback();
    },
  });
  
  // ใช้ pipeline เพื่อ pipe streams
  await pipelineAsync(readStream, transformStream, writeStream);
  console.log('File processing completed');
}

// สร้าง Readable stream จาก array
function createReadableFromArray<T>(items: T[]): Readable {
  let index = 0;
  
  return new Readable({
    objectMode: true,
    read() {
      if (index < items.length) {
        this.push(items[index++]);
      } else {
        this.push(null); // สัญญาณว่าข้อมูลหมดแล้ว
      }
    },
  });
}

// Transform stream สำหรับแปลงข้อมูล JSON
class JsonTransformStream extends Transform {
  constructor() {
    super({ objectMode: true });
  }

  _transform(
    chunk: Record<string, unknown>,
    encoding: string,
    callback: () => void
  ): void {
    // เพิ่ม timestamp ให้กับข้อมูล
    const transformed = {
      ...chunk,
      processedAt: new Date().toISOString(),
    };
    this.push(transformed);
    callback();
  }
}

// Writable stream สำหรับรับข้อมูล
class ConsoleWritableStream extends Writable {
  constructor() {
    super({ objectMode: true });
  }

  _write(
    chunk: Record<string, unknown>,
    encoding: string,
    callback: () => void
  ): void {
    console.log('Received:', JSON.stringify(chunk, null, 2));
    callback();
  }
}

// การใช้งาน
async function streamExample(): Promise<void> {
  const data = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
    { id: 3, name: 'Charlie' },
  ];
  
  const readable = createReadableFromArray(data);
  const transform = new JsonTransformStream();
  const writable = new ConsoleWritableStream();
  
  await pipelineAsync(readable, transform, writable);
}

streamExample().catch(console.error);
```

---

## 20.10 การทำงานกับ Buffers

```typescript
// การสร้าง Buffer
const buf1 = Buffer.from('Hello, World!', 'utf-8');
const buf2 = Buffer.alloc(10); // สร้าง buffer ขนาด 10 bytes เต็มด้วย zeros
const buf3 = Buffer.allocUnsafe(10); // สร้าง buffer โดยไม่ initialize

// แปลง Buffer เป็น string
const str = buf1.toString('utf-8');
console.log('String:', str);

// แปลง Buffer เป็น base64
const base64 = buf1.toString('base64');
console.log('Base64:', base64);

// แปลง base64 กลับเป็น Buffer
const fromBase64 = Buffer.from(base64, 'base64');
console.log('From Base64:', fromBase64.toString('utf-8'));

// รวม Buffers
const combined = Buffer.concat([buf1, Buffer.from(' TypeScript!')]);
console.log('Combined:', combined.toString('utf-8'));

// ตรวจสอบขนาด Buffer
console.log('Buffer length:', buf1.length);
console.log('Byte length:', buf1.byteLength);

// การเปรียบเทียบ Buffers
const buf4 = Buffer.from('Hello');
const buf5 = Buffer.from('Hello');
const buf6 = Buffer.from('World');

console.log('buf4 equals buf5:', buf4.equals(buf5)); // true
console.log('buf4 equals buf6:', buf4.equals(buf6)); // false

// ฟังก์ชัน utility สำหรับ Buffer
function encodeToBase64(text: string): string {
  return Buffer.from(text, 'utf-8').toString('base64');
}

function decodeFromBase64(base64String: string): string {
  return Buffer.from(base64String, 'base64').toString('utf-8');
}

function hexToBuffer(hex: string): Buffer {
  return Buffer.from(hex, 'hex');
}

function bufferToHex(buffer: Buffer): string {
  return buffer.toString('hex');
}

// การใช้งาน
const original = 'Hello, TypeScript!';
const encoded = encodeToBase64(original);
const decoded = decodeFromBase64(encoded);

console.log('Original:', original);
console.log('Encoded:', encoded);
console.log('Decoded:', decoded);
console.log('Match:', original === decoded);
```

---

## 20.11 การทำงานกับ Path Module

```typescript
import path from 'path';

// ข้อมูลเส้นทางพื้นฐาน
console.log('__dirname:', __dirname);
console.log('__filename:', __filename);

// ฟังก์ชัน path ที่ใช้บ่อย
const filePath = '/home/user/projects/myapp/src/index.ts';

// แยกส่วนต่าง ๆ ของเส้นทาง
console.log('basename:', path.basename(filePath));        // index.ts
console.log('basename no ext:', path.basename(filePath, '.ts')); // index
console.log('dirname:', path.dirname(filePath));          // /home/user/projects/myapp/src
console.log('extname:', path.extname(filePath));          // .ts
console.log('parse:', path.parse(filePath));              // { root, dir, base, ext, name }

// รวมเส้นทาง
const projectRoot = path.join('/home/user', 'projects', 'myapp');
console.log('join:', projectRoot); // /home/user/projects/myapp

// แปลงเป็น absolute path
const relativePath = './src/utils';
const absolutePath = path.resolve(relativePath);
console.log('resolve:', absolutePath);

// ตรวจสอบว่าเป็น absolute path
console.log('isAbsolute:', path.isAbsolute('/home/user')); // true
console.log('isAbsolute:', path.isAbsolute('./src'));       // false

// หา relative path ระหว่างสองเส้นทาง
const from = '/home/user/projects/myapp/src';
const to = '/home/user/projects/myapp/dist';
console.log('relative:', path.relative(from, to)); // ../dist

// ตัวอย่างการใช้งานจริง
class PathUtils {
  private readonly rootDir: string;

  constructor(rootDir: string = process.cwd()) {
    this.rootDir = rootDir;
  }

  getSourceDir(): string {
    return path.join(this.rootDir, 'src');
  }

  getDistDir(): string {
    return path.join(this.rootDir, 'dist');
  }

  getPublicDir(): string {
    return path.join(this.rootDir, 'public');
  }

  getUploadsDir(): string {
    return path.join(this.rootDir, 'uploads');
  }

  resolvePath(...segments: string[]): string {
    return path.resolve(this.rootDir, ...segments);
  }

  getFileExtension(filePath: string): string {
    return path.extname(filePath).toLowerCase();
  }

  isImageFile(filePath: string): boolean {
    const imageExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.webp', '.svg'];
    return imageExtensions.includes(this.getFileExtension(filePath));
  }
}

const pathUtils = new PathUtils();
console.log('Source dir:', pathUtils.getSourceDir());
console.log('Is image:', pathUtils.isImageFile('photo.jpg')); // true
```

---

## 20.12 การทำงานกับ OS Module

```typescript
import os from 'os';

// ข้อมูลระบบ
interface SystemInfo {
  platform: string;
  arch: string;
  hostname: string;
  username: string;
  homeDir: string;
  tempDir: string;
  totalMemory: number;
  freeMemory: number;
  cpuCount: number;
  uptime: number;
}

function getSystemInfo(): SystemInfo {
  return {
    platform: os.platform(),
    arch: os.arch(),
    hostname: os.hostname(),
    username: os.userInfo().username,
    homeDir: os.homedir(),
    tempDir: os.tmpdir(),
    totalMemory: os.totalmem(),
    freeMemory: os.freemem(),
    cpuCount: os.cpus().length,
    uptime: os.uptime(),
  };
}

function formatBytes(bytes: number): string {
  const units = ['B', 'KB', 'MB', 'GB', 'TB'];
  let size = bytes;
  let unitIndex = 0;
  
  while (size >= 1024 && unitIndex < units.length - 1) {
    size /= 1024;
    unitIndex++;
  }
  
  return `${size.toFixed(2)} ${units[unitIndex]}`;
}

function printSystemInfo(): void {
  const info = getSystemInfo();
  
  console.log('=== System Information ===');
  console.log(`Platform: ${info.platform}`);
  console.log(`Architecture: ${info.arch}`);
  console.log(`Hostname: ${info.hostname}`);
  console.log(`Username: ${info.username}`);
  console.log(`Home Directory: ${info.homeDir}`);
  console.log(`Temp Directory: ${info.tempDir}`);
  console.log(`Total Memory: ${formatBytes(info.totalMemory)}`);
  console.log(`Free Memory: ${formatBytes(info.freeMemory)}`);
  console.log(`CPU Count: ${info.cpuCount}`);
  console.log(`Uptime: ${(info.uptime / 3600).toFixed(2)} hours`);
}

printSystemInfo();
```

---

## 20.13 Process Object

```typescript
// การเข้าถึง process information
console.log('Node.js version:', process.version);
console.log('Platform:', process.platform);
console.log('Architecture:', process.arch);
console.log('Process ID:', process.pid);
console.log('Current directory:', process.cwd());

// Environment variables
const nodeEnv = process.env.NODE_ENV || 'development';
console.log('NODE_ENV:', nodeEnv);

// Command line arguments
const args = process.argv.slice(2);
console.log('Command line args:', args);

// Memory usage
const memUsage = process.memoryUsage();
console.log('Memory usage:', {
  rss: `${(memUsage.rss / 1024 / 1024).toFixed(2)} MB`,
  heapTotal: `${(memUsage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
  heapUsed: `${(memUsage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
  external: `${(memUsage.external / 1024 / 1024).toFixed(2)} MB`,
});

// CPU usage
const cpuUsage = process.cpuUsage();
console.log('CPU usage:', cpuUsage);

// Exit handlers
process.on('exit', (code: number) => {
  console.log(`Process exiting with code: ${code}`);
});

process.on('SIGTERM', () => {
  console.log('SIGTERM signal received');
  // cleanup resources
  process.exit(0);
});

process.on('SIGINT', () => {
  console.log('SIGINT signal received (Ctrl+C)');
  // cleanup resources
  process.exit(0);
});

process.on('uncaughtException', (error: Error) => {
  console.error('Uncaught Exception:', error);
  process.exit(1);
});

process.on('unhandledRejection', (reason: unknown, promise: Promise<unknown>) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  process.exit(1);
});

// ฟังก์ชัน graceful shutdown
class GracefulShutdown {
  private isShuttingDown = false;
  private cleanupFunctions: Array<() => Promise<void>> = [];

  register(cleanupFn: () => Promise<void>): void {
    this.cleanupFunctions.push(cleanupFn);
  }

  async shutdown(signal: string): Promise<void> {
    if (this.isShuttingDown) return;
    
    this.isShuttingDown = true;
    console.log(`\nReceived ${signal}, starting graceful shutdown...`);
    
    for (const cleanupFn of this.cleanupFunctions) {
      try {
        await cleanupFn();
      } catch (error) {
        console.error('Error during cleanup:', error);
      }
    }
    
    console.log('Graceful shutdown completed');
    process.exit(0);
  }

  setup(): void {
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
  }
}

const gracefulShutdown = new GracefulShutdown();

// ลงทะเบียน cleanup functions
gracefulShutdown.register(async () => {
  console.log('Closing database connection...');
  // await database.close();
});

gracefulShutdown.register(async () => {
  console.log('Closing HTTP server...');
  // await server.close();
});

gracefulShutdown.setup();
```

---

## 20.14 Logger Utility

```typescript
// src/utils/logger.ts
import { config } from '../config';

type LogLevel = 'error' | 'warn' | 'info' | 'debug';

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  data?: unknown;
  stack?: string;
}

const LOG_LEVELS: Record<LogLevel, number> = {
  error: 0,
  warn: 1,
  info: 2,
  debug: 3,
};

class Logger {
  private readonly level: LogLevel;
  private readonly prefix: string;

  constructor(level: LogLevel = 'info', prefix: string = '') {
    this.level = level;
    this.prefix = prefix;
  }

  private shouldLog(level: LogLevel): boolean {
    return LOG_LEVELS[level] <= LOG_LEVELS[this.level];
  }

  private formatEntry(level: LogLevel, message: string, data?: unknown): LogEntry {
    return {
      timestamp: new Date().toISOString(),
      level,
      message: this.prefix ? `[${this.prefix}] ${message}` : message,
      data,
      stack: data instanceof Error ? data.stack : undefined,
    };
  }

  private log(entry: LogEntry): void {
    const output = JSON.stringify(entry);
    
    switch (entry.level) {
      case 'error':
        console.error(output);
        break;
      case 'warn':
        console.warn(output);
        break;
      default:
        console.log(output);
    }
  }

  error(message: string, data?: unknown): void {
    if (this.shouldLog('error')) {
      this.log(this.formatEntry('error', message, data));
    }
  }

  warn(message: string, data?: unknown): void {
    if (this.shouldLog('warn')) {
      this.log(this.formatEntry('warn', message, data));
    }
  }

  info(message: string, data?: unknown): void {
    if (this.shouldLog('info')) {
      this.log(this.formatEntry('info', message, data));
    }
  }

  debug(message: string, data?: unknown): void {
    if (this.shouldLog('debug')) {
      this.log(this.formatEntry('debug', message, data));
    }
  }

  child(prefix: string): Logger {
    return new Logger(this.level, prefix);
  }
}

export const logger = new Logger(
  (process.env.LOG_LEVEL as LogLevel) || 'info'
);

export { Logger };
export type { LogLevel, LogEntry };
```

---

## 20.15 Complete Working Server Example

### src/types/index.ts

```typescript
export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  message?: string;
  errors?: string[];
  meta?: {
    total?: number;
    page?: number;
    limit?: number;
  };
}

export interface PaginationQuery {
  page?: number;
  limit?: number;
  sort?: string;
  order?: 'asc' | 'desc';
}

export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateUserInput {
  name: string;
  email: string;
  password: string;
  role?: 'admin' | 'user';
}

export interface UpdateUserInput {
  name?: string;
  email?: string;
  role?: 'admin' | 'user';
}
```

### src/data/users.ts - In-memory data store

```typescript
import { User } from '../types';
import { v4 as uuidv4 } from 'uuid';

// ข้อมูล users จำลอง
const users: User[] = [
  {
    id: uuidv4(),
    name: 'Alice Johnson',
    email: 'alice@example.com',
    role: 'admin',
    createdAt: new Date('2024-01-01'),
    updatedAt: new Date('2024-01-01'),
  },
  {
    id: uuidv4(),
    name: 'Bob Smith',
    email: 'bob@example.com',
    role: 'user',
    createdAt: new Date('2024-01-02'),
    updatedAt: new Date('2024-01-02'),
  },
];

export function getUsers(): User[] {
  return [...users];
}

export function getUserById(id: string): User | undefined {
  return users.find(user => user.id === id);
}

export function getUserByEmail(email: string): User | undefined {
  return users.find(user => user.email === email);
}

export function createUser(data: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): User {
  const newUser: User = {
    id: uuidv4(),
    ...data,
    createdAt: new Date(),
    updatedAt: new Date(),
  };
  users.push(newUser);
  return newUser;
}

export function updateUser(id: string, data: Partial<Omit<User, 'id' | 'createdAt'>>): User | undefined {
  const index = users.findIndex(user => user.id === id);
  if (index === -1) return undefined;
  
  users[index] = {
    ...users[index],
    ...data,
    updatedAt: new Date(),
  };
  
  return users[index];
}

export function deleteUser(id: string): boolean {
  const index = users.findIndex(user => user.id === id);
  if (index === -1) return false;
  
  users.splice(index, 1);
  return true;
}
```

### src/index.ts - Main server file

```typescript
import http from 'http';
import { IncomingMessage, ServerResponse } from 'http';
import url from 'url';
import { getUsers, getUserById, createUser, updateUser, deleteUser } from './data/users';
import { ApiResponse, CreateUserInput, UpdateUserInput } from './types';
import { logger } from './utils/logger';

const PORT = parseInt(process.env.PORT || '3000', 10);
const HOST = process.env.HOST || 'localhost';

// Helper function สำหรับส่ง JSON response
function sendJsonResponse<T>(
  res: ServerResponse,
  statusCode: number,
  data: ApiResponse<T>
): void {
  res.writeHead(statusCode, {
    'Content-Type': 'application/json',
    'X-Powered-By': 'Node.js TypeScript',
  });
  res.end(JSON.stringify(data));
}

// Helper function สำหรับ parse request body
async function parseRequestBody<T>(req: IncomingMessage): Promise<T> {
  return new Promise((resolve, reject) => {
    let body = '';
    
    req.on('data', (chunk: Buffer) => {
      body += chunk.toString();
    });
    
    req.on('end', () => {
      try {
        resolve(JSON.parse(body) as T);
      } catch (error) {
        reject(new Error('Invalid JSON'));
      }
    });
    
    req.on('error', reject);
  });
}

// Route handler
async function handleRequest(req: IncomingMessage, res: ServerResponse): Promise<void> {
  const parsedUrl = url.parse(req.url || '/', true);
  const pathname = parsedUrl.pathname || '/';
  const method = req.method || 'GET';
  
  logger.info(`${method} ${pathname}`);
  
  // Health check
  if (pathname === '/health' && method === 'GET') {
    sendJsonResponse(res, 200, {
      success: true,
      data: {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
      },
    });
    return;
  }
  
  // GET /users
  if (pathname === '/users' && method === 'GET') {
    const users = getUsers();
    sendJsonResponse(res, 200, {
      success: true,
      data: users,
      meta: { total: users.length },
    });
    return;
  }
  
  // GET /users/:id
  const userByIdMatch = pathname.match(/^\/users\/([^\/]+)$/);
  
  if (userByIdMatch && method === 'GET') {
    const id = userByIdMatch[1];
    const user = getUserById(id);
    
    if (!user) {
      sendJsonResponse(res, 404, {
        success: false,
        message: 'User not found',
      });
      return;
    }
    
    sendJsonResponse(res, 200, {
      success: true,
      data: user,
    });
    return;
  }
  
  // POST /users
  if (pathname === '/users' && method === 'POST') {
    try {
      const body = await parseRequestBody<CreateUserInput>(req);
      
      if (!body.name || !body.email) {
        sendJsonResponse(res, 400, {
          success: false,
          message: 'Name and email are required',
        });
        return;
      }
      
      const newUser = createUser({
        name: body.name,
        email: body.email,
        role: body.role || 'user',
      });
      
      sendJsonResponse(res, 201, {
        success: true,
        data: newUser,
        message: 'User created successfully',
      });
    } catch (error) {
      sendJsonResponse(res, 400, {
        success: false,
        message: 'Invalid request body',
      });
    }
    return;
  }
  
  // PUT /users/:id
  if (userByIdMatch && method === 'PUT') {
    const id = userByIdMatch[1];
    
    try {
      const body = await parseRequestBody<UpdateUserInput>(req);
      const updatedUser = updateUser(id, body);
      
      if (!updatedUser) {
        sendJsonResponse(res, 404, {
          success: false,
          message: 'User not found',
        });
        return;
      }
      
      sendJsonResponse(res, 200, {
        success: true,
        data: updatedUser,
        message: 'User updated successfully',
      });
    } catch (error) {
      sendJsonResponse(res, 400, {
        success: false,
        message: 'Invalid request body',
      });
    }
    return;
  }
  
  // DELETE /users/:id
  if (userByIdMatch && method === 'DELETE') {
    const id = userByIdMatch[1];
    const deleted = deleteUser(id);
    
    if (!deleted) {
      sendJsonResponse(res, 404, {
        success: false,
        message: 'User not found',
      });
      return;
    }
    
    sendJsonResponse(res, 200, {
      success: true,
      message: 'User deleted successfully',
    });
    return;
  }
  
  // 404 Not Found
  sendJsonResponse(res, 404, {
    success: false,
    message: 'Route not found',
  });
}

// สร้าง HTTP Server
const server = http.createServer(async (req, res) => {
  try {
    await handleRequest(req, res);
  } catch (error) {
    logger.error('Unhandled error:', error);
    sendJsonResponse(res, 500, {
      success: false,
      message: 'Internal server error',
    });
  }
});

// เริ่มต้น server
server.listen(PORT, HOST, () => {
  logger.info(`Server is running on http://${HOST}:${PORT}`);
  logger.info(`Environment: ${process.env.NODE_ENV || 'development'}`);
});

// Graceful shutdown
const shutdown = async (): Promise<void> => {
  logger.info('Shutting down server...');
  
  server.close((err) => {
    if (err) {
      logger.error('Error closing server:', err);
      process.exit(1);
    }
    logger.info('Server closed successfully');
    process.exit(0);
  });
};

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

export { server };
```

---

## 20.16 สรุป Scripts ทั้งหมด

### package.json สมบูรณ์

```json
{
  "name": "node-typescript-server",
  "version": "1.0.0",
  "description": "Complete Node.js TypeScript server",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "start": "node dist/index.js",
    "start:prod": "NODE_ENV=production node dist/index.js",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "dev:debug": "ts-node-dev --respawn --transpile-only --inspect src/index.ts",
    "clean": "rm -rf dist",
    "prebuild": "npm run clean",
    "lint": "eslint src --ext .ts --fix",
    "typecheck": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "dependencies": {
    "dotenv": "^16.0.0",
    "module-alias": "^2.2.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "ts-node": "^10.9.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.0.0"
  },
  "_moduleAliases": {
    "@": "dist",
    "@config": "dist/config",
    "@controllers": "dist/controllers",
    "@services": "dist/services",
    "@models": "dist/models",
    "@middleware": "dist/middleware",
    "@utils": "dist/utils"
  }
}
```

---

## สรุปบทที่ 20

ในบทนี้เราได้เรียนรู้:

1. **การตั้งค่าโปรเจกต์** - สร้าง package.json, tsconfig.json สำหรับ Node.js
2. **ts-node และ ts-node-dev** - เครื่องมือสำหรับพัฒนา TypeScript บน Node.js
3. **nodemon** - การตั้งค่า auto-reload ระหว่าง development
4. **Path Aliases** - การใช้ shortcut paths แทน relative paths ที่ยาว
5. **Environment Variables** - การจัดการ env vars อย่าง type-safe ด้วย dotenv
6. **Node.js Core Modules** - การใช้งาน fs, path, os, process
7. **Streams และ Buffers** - การทำงานกับข้อมูลขนาดใหญ่
8. **Logger Utility** - การสร้าง logging system
9. **Complete Server** - ตัวอย่าง HTTP server ที่ครบสมบูรณ์

ในบทถัดไปเราจะเรียนรู้การใช้งาน Express.js กับ TypeScript เพื่อสร้าง REST API ที่มีประสิทธิภาพ
