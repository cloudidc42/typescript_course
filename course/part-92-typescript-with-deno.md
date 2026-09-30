# Part 92: TypeScript กับ Deno

## บทนำ

Deno คือ JavaScript/TypeScript runtime ที่สร้างโดย Ryan Dahl (ผู้สร้าง Node.js) เพื่อแก้ไขปัญหาที่เขาพบใน Node.js โดย Deno รองรับ TypeScript แบบ native โดยไม่ต้องติดตั้งหรือตั้งค่าอะไรเพิ่มเติม

## ส่วนที่ 1: Deno vs Node.js

### 1.1 ความแตกต่างหลัก

| คุณสมบัติ | Deno | Node.js |
|-----------|------|---------|
| TypeScript | Native | ต้องติดตั้ง ts-node หรือ compile |
| Security | Sandbox by default | ไม่มี sandbox |
| Module System | ES Modules (URL imports) | CommonJS + ESM |
| Package Manager | ไม่ต้องใช้ npm | npm/yarn/pnpm |
| Standard Library | Built-in, stable | ต้องพึ่ง third-party |
| Top-level await | รองรับ | รองรับบางส่วน |
| Browser APIs | รองรับ (fetch, WebSocket, etc.) | ต้องติดตั้งเพิ่ม |

### 1.2 การติดตั้ง Deno

```bash
# macOS/Linux
curl -fsSL https://deno.land/install.sh | sh

# Windows (PowerShell)
irm https://deno.land/install.ps1 | iex

# ตรวจสอบเวอร์ชัน
deno --version
# deno 1.40.0
# v8 12.1.285.6
# typescript 5.3.3
```

### 1.3 รันไฟล์ TypeScript โดยตรง

```typescript
// hello.ts
const message: string = "สวัสดี Deno!";
const numbers: number[] = [1, 2, 3, 4, 5];
const sum: number = numbers.reduce((acc, n) => acc + n, 0);

console.log(message);
console.log(`ผลรวม: ${sum}`);

interface User {
  name: string;
  age: number;
}

const users: User[] = [
  { name: "สมชาย", age: 30 },
  { name: "สมหญิง", age: 25 },
];

users.forEach((user) => {
  console.log(`${user.name} อายุ ${user.age} ปี`);
});
```

```bash
# รันโดยตรง ไม่ต้อง compile!
deno run hello.ts
```

## ส่วนที่ 2: ระบบ Permissions

### 2.1 Deno Permissions System

Deno ใช้ "deny by default" - ไม่มีสิทธิ์อะไรเลยจนกว่าจะให้สิทธิ์

```bash
# ให้สิทธิ์อ่านไฟล์
deno run --allow-read script.ts

# ให้สิทธิ์เขียนไฟล์
deno run --allow-write script.ts

# ให้สิทธิ์เครือข่าย
deno run --allow-net script.ts

# ให้สิทธิ์ environment variables
deno run --allow-env script.ts

# ให้สิทธิ์รันโปรแกรมอื่น
deno run --allow-run script.ts

# ให้สิทธิ์ทั้งหมด (ไม่แนะนำ)
deno run --allow-all script.ts

# จำกัดสิทธิ์เฉพาะ path หรือ domain
deno run --allow-read=/tmp --allow-net=api.example.com script.ts
```

### 2.2 ตัวอย่างการใช้ Permissions

```typescript
// file-operations.ts
// ต้องรันด้วย: deno run --allow-read --allow-write file-operations.ts

import { exists } from "https://deno.land/std@0.208.0/fs/exists.ts";

async function readConfig(path: string): Promise<Record<string, string>> {
  const fileExists = await exists(path);
  
  if (!fileExists) {
    console.log(`ไม่พบไฟล์: ${path}`);
    return {};
  }

  const content = await Deno.readTextFile(path);
  const lines = content.split('\n');
  
  return lines.reduce((config, line) => {
    const [key, value] = line.split('=');
    if (key && value) {
      config[key.trim()] = value.trim();
    }
    return config;
  }, {} as Record<string, string>);
}

async function writeConfig(path: string, config: Record<string, string>): Promise<void> {
  const content = Object.entries(config)
    .map(([k, v]) => `${k}=${v}`)
    .join('\n');
  
  await Deno.writeTextFile(path, content);
  console.log(`บันทึกการตั้งค่าไปยัง ${path}`);
}

const config = await readConfig('./config.env');
console.log('การตั้งค่า:', config);

await writeConfig('./config.env', {
  ...config,
  UPDATED_AT: new Date().toISOString(),
});
```

### 2.3 Permission API

```typescript
// check-permissions.ts
// ตรวจสอบสิทธิ์ก่อนดำเนินการ

async function checkAndRequestPermission(
  descriptor: Deno.PermissionDescriptor
): Promise<boolean> {
  const status = await Deno.permissions.query(descriptor);
  
  if (status.state === 'granted') {
    return true;
  }
  
  if (status.state === 'prompt') {
    const result = await Deno.permissions.request(descriptor);
    return result.state === 'granted';
  }
  
  return false;
}

// ตรวจสอบสิทธิ์อ่านไฟล์
const canRead = await checkAndRequestPermission({
  name: 'read',
  path: '/home/user/data',
});

if (canRead) {
  console.log('มีสิทธิ์อ่านไฟล์');
} else {
  console.log('ไม่มีสิทธิ์อ่านไฟล์');
  Deno.exit(1);
}

// ตรวจสอบสิทธิ์เครือข่าย
const canNet = await checkAndRequestPermission({
  name: 'net',
  host: 'api.example.com',
});

console.log(`สิทธิ์เครือข่าย: ${canNet}`);
```

## ส่วนที่ 3: Deno Standard Library

### 3.1 HTTP Module

```typescript
// http-client.ts
// deno run --allow-net http-client.ts

interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

async function fetchPosts(): Promise<Post[]> {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  
  if (!response.ok) {
    throw new Error(`HTTP Error: ${response.status}`);
  }
  
  return response.json();
}

async function fetchPost(id: number): Promise<Post> {
  const response = await fetch(
    `https://jsonplaceholder.typicode.com/posts/${id}`
  );
  
  if (!response.ok) {
    throw new Error(`ไม่พบโพสต์ ID: ${id}`);
  }
  
  return response.json();
}

// ใช้งาน
const posts = await fetchPosts();
console.log(`จำนวนโพสต์: ${posts.length}`);

const firstPost = await fetchPost(1);
console.log(`หัวข้อโพสต์แรก: ${firstPost.title}`);
```

### 3.2 File System

```typescript
// filesystem.ts
// deno run --allow-read --allow-write filesystem.ts

import { ensureDir, ensureFile, copy, move } from "https://deno.land/std@0.208.0/fs/mod.ts";
import { join, basename, extname } from "https://deno.land/std@0.208.0/path/mod.ts";

// สร้างโครงสร้างโฟลเดอร์
async function createProjectStructure(projectName: string): Promise<void> {
  const dirs = [
    `${projectName}/src`,
    `${projectName}/tests`,
    `${projectName}/docs`,
    `${projectName}/dist`,
  ];

  for (const dir of dirs) {
    await ensureDir(dir);
    console.log(`สร้างโฟลเดอร์: ${dir}`);
  }

  // สร้างไฟล์ตั้งต้น
  await ensureFile(join(projectName, 'src', 'main.ts'));
  await ensureFile(join(projectName, 'README.md'));
  
  // เขียนเนื้อหาลงไฟล์
  await Deno.writeTextFile(
    join(projectName, 'README.md'),
    `# ${projectName}\n\nโปรเจกต์ TypeScript กับ Deno\n`
  );
  
  console.log(`สร้างโปรเจกต์ ${projectName} เรียบร้อย`);
}

// อ่านไฟล์ทั้งหมดในโฟลเดอร์
async function listFiles(dirPath: string): Promise<string[]> {
  const files: string[] = [];
  
  for await (const entry of Deno.readDir(dirPath)) {
    if (entry.isFile) {
      files.push(join(dirPath, entry.name));
    } else if (entry.isDirectory) {
      const subFiles = await listFiles(join(dirPath, entry.name));
      files.push(...subFiles);
    }
  }
  
  return files;
}

// ใช้งาน
await createProjectStructure('my-deno-project');

const files = await listFiles('my-deno-project');
console.log('ไฟล์ทั้งหมด:');
files.forEach(f => console.log(`  ${f}`));
```

### 3.3 Path Module

```typescript
// path-examples.ts
import {
  join,
  resolve,
  relative,
  dirname,
  basename,
  extname,
  parse,
  format,
} from "https://deno.land/std@0.208.0/path/mod.ts";

// การจัดการ path
const filePath = '/home/user/projects/my-app/src/main.ts';

console.log('dirname:', dirname(filePath));
// /home/user/projects/my-app/src

console.log('basename:', basename(filePath));
// main.ts

console.log('extname:', extname(filePath));
// .ts

console.log('basename (no ext):', basename(filePath, '.ts'));
// main

const parsed = parse(filePath);
console.log('parse:', parsed);
// { root: '/', dir: '/home/user/...', base: 'main.ts', ext: '.ts', name: 'main' }

// join paths
const joined = join('/home/user', 'projects', 'my-app', 'src', 'main.ts');
console.log('join:', joined);

// resolve (absolute path)
const resolved = resolve('src', 'utils', 'helper.ts');
console.log('resolve:', resolved);
```

### 3.4 Datetime Formatting

```typescript
// datetime.ts
import { format, parse } from "https://deno.land/std@0.208.0/datetime/mod.ts";

const now = new Date();

// Format datetime
console.log(format(now, "yyyy-MM-dd HH:mm:ss"));
// 2024-01-15 14:30:00

console.log(format(now, "dd/MM/yyyy"));
// 15/01/2024

// Parse datetime
const dateStr = "15/01/2024";
const parsed = parse(dateStr, "dd/MM/yyyy");
console.log(parsed);

// คำนวณความแตกต่าง
function dateDiffInDays(date1: Date, date2: Date): number {
  const msPerDay = 1000 * 60 * 60 * 24;
  return Math.round((date2.getTime() - date1.getTime()) / msPerDay);
}

const startDate = new Date('2024-01-01');
const endDate = new Date('2024-12-31');
console.log(`จำนวนวัน: ${dateDiffInDays(startDate, endDate)} วัน`);
```

## ส่วนที่ 4: HTTP Server ด้วย Deno

### 4.1 Basic HTTP Server

```typescript
// server.ts
// deno run --allow-net server.ts

interface RequestContext {
  req: Request;
  params: Record<string, string>;
}

type Handler = (ctx: RequestContext) => Response | Promise<Response>;

function createServer() {
  const routes: Map<string, Map<string, Handler>> = new Map();

  function addRoute(method: string, path: string, handler: Handler) {
    if (!routes.has(method)) {
      routes.set(method, new Map());
    }
    routes.get(method)!.set(path, handler);
  }

  function get(path: string, handler: Handler) {
    addRoute('GET', path, handler);
  }

  function post(path: string, handler: Handler) {
    addRoute('POST', path, handler);
  }

  function matchRoute(method: string, url: string): { handler: Handler; params: Record<string, string> } | null {
    const methodRoutes = routes.get(method);
    if (!methodRoutes) return null;

    for (const [pattern, handler] of methodRoutes) {
      const params = matchPattern(pattern, url);
      if (params !== null) {
        return { handler, params };
      }
    }
    return null;
  }

  function matchPattern(pattern: string, url: string): Record<string, string> | null {
    const patternParts = pattern.split('/');
    const urlParts = url.split('/');

    if (patternParts.length !== urlParts.length) return null;

    const params: Record<string, string> = {};

    for (let i = 0; i < patternParts.length; i++) {
      if (patternParts[i].startsWith(':')) {
        params[patternParts[i].slice(1)] = urlParts[i];
      } else if (patternParts[i] !== urlParts[i]) {
        return null;
      }
    }

    return params;
  }

  async function handle(req: Request): Promise<Response> {
    const url = new URL(req.url);
    const match = matchRoute(req.method, url.pathname);

    if (!match) {
      return new Response(JSON.stringify({ error: 'Not Found' }), {
        status: 404,
        headers: { 'Content-Type': 'application/json' },
      });
    }

    try {
      return await match.handler({ req, params: match.params });
    } catch (error) {
      console.error(error);
      return new Response(JSON.stringify({ error: 'Internal Server Error' }), {
        status: 500,
        headers: { 'Content-Type': 'application/json' },
      });
    }
  }

  function listen(port: number) {
    console.log(`Server กำลังทำงานที่ http://localhost:${port}`);
    Deno.serve({ port }, handle);
  }

  return { get, post, listen };
}

// สร้าง Server
const server = createServer();

// Mock database
interface User {
  id: number;
  name: string;
  email: string;
}

const users: User[] = [
  { id: 1, name: "สมชาย", email: "somchai@example.com" },
  { id: 2, name: "สมหญิง", email: "somying@example.com" },
];

// Routes
server.get('/api/users', () => {
  return new Response(JSON.stringify(users), {
    headers: { 'Content-Type': 'application/json' },
  });
});

server.get('/api/users/:id', ({ params }) => {
  const user = users.find(u => u.id === parseInt(params.id));

  if (!user) {
    return new Response(JSON.stringify({ error: 'ไม่พบผู้ใช้' }), {
      status: 404,
      headers: { 'Content-Type': 'application/json' },
    });
  }

  return new Response(JSON.stringify(user), {
    headers: { 'Content-Type': 'application/json' },
  });
});

server.post('/api/users', async ({ req }) => {
  const body = await req.json() as { name: string; email: string };
  const newUser: User = {
    id: users.length + 1,
    ...body,
  };
  users.push(newUser);

  return new Response(JSON.stringify(newUser), {
    status: 201,
    headers: { 'Content-Type': 'application/json' },
  });
});

server.listen(8080);
```

### 4.2 Middleware Pattern

```typescript
// middleware.ts
// deno run --allow-net middleware.ts

type Middleware = (
  req: Request,
  next: () => Promise<Response>
) => Promise<Response>;

function compose(middlewares: Middleware[]) {
  return async function (req: Request): Promise<Response> {
    let index = -1;

    async function dispatch(i: number): Promise<Response> {
      if (i <= index) {
        throw new Error('next() ถูกเรียกหลายครั้ง');
      }
      index = i;

      if (i === middlewares.length) {
        return new Response('Not Found', { status: 404 });
      }

      const middleware = middlewares[i];
      return middleware(req, () => dispatch(i + 1));
    }

    return dispatch(0);
  };
}

// Logger Middleware
const logger: Middleware = async (req, next) => {
  const start = Date.now();
  console.log(`→ ${req.method} ${new URL(req.url).pathname}`);

  const response = await next();

  console.log(
    `← ${req.method} ${new URL(req.url).pathname} ${response.status} (${Date.now() - start}ms)`
  );
  return response;
};

// CORS Middleware
const cors: Middleware = async (req, next) => {
  if (req.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      },
    });
  }

  const response = await next();
  const newHeaders = new Headers(response.headers);
  newHeaders.set('Access-Control-Allow-Origin', '*');

  return new Response(response.body, {
    status: response.status,
    headers: newHeaders,
  });
};

// Rate Limiter Middleware
const rateLimiter: Middleware = (() => {
  const requests = new Map<string, number[]>();
  const limit = 100;
  const window = 60 * 1000; // 1 นาที

  return async (req, next) => {
    const ip = req.headers.get('x-forwarded-for') || 'unknown';
    const now = Date.now();

    if (!requests.has(ip)) {
      requests.set(ip, []);
    }

    const ipRequests = requests.get(ip)!;
    const validRequests = ipRequests.filter(t => now - t < window);

    if (validRequests.length >= limit) {
      return new Response(JSON.stringify({ error: 'Too Many Requests' }), {
        status: 429,
        headers: { 'Content-Type': 'application/json' },
      });
    }

    validRequests.push(now);
    requests.set(ip, validRequests);

    return next();
  };
})();

// Application Handler
const appHandler: Middleware = async (req) => {
  const url = new URL(req.url);

  if (url.pathname === '/api/hello') {
    return new Response(JSON.stringify({ message: 'สวัสดี Deno!' }), {
      headers: { 'Content-Type': 'application/json' },
    });
  }

  return new Response('Not Found', { status: 404 });
};

const handler = compose([logger, cors, rateLimiter, appHandler]);

Deno.serve({ port: 8080 }, handler);
console.log('Server กำลังทำงานที่ http://localhost:8080');
```

## ส่วนที่ 5: Third-party Modules

### 5.1 Oak Framework (Express-like)

```typescript
// oak-server.ts
// deno run --allow-net --allow-read oak-server.ts

import { Application, Router, Context } from "https://deno.land/x/oak@v13.0.0/mod.ts";
import { oakCors } from "https://deno.land/x/cors@v1.2.2/mod.ts";

interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
  createdAt: Date;
}

const posts: Post[] = [
  {
    id: 1,
    title: "TypeScript กับ Deno",
    content: "เนื้อหาบทความ...",
    authorId: 1,
    createdAt: new Date(),
  },
];

const router = new Router();

// GET /posts
router.get('/posts', (ctx: Context) => {
  ctx.response.body = posts;
});

// GET /posts/:id
router.get('/posts/:id', (ctx: Context) => {
  const id = parseInt(ctx.params.id);
  const post = posts.find(p => p.id === id);

  if (!post) {
    ctx.response.status = 404;
    ctx.response.body = { error: 'ไม่พบโพสต์' };
    return;
  }

  ctx.response.body = post;
});

// POST /posts
router.post('/posts', async (ctx: Context) => {
  const body = await ctx.request.body.json();

  const newPost: Post = {
    id: posts.length + 1,
    title: body.title,
    content: body.content,
    authorId: body.authorId,
    createdAt: new Date(),
  };

  posts.push(newPost);

  ctx.response.status = 201;
  ctx.response.body = newPost;
});

// DELETE /posts/:id
router.delete('/posts/:id', (ctx: Context) => {
  const id = parseInt(ctx.params.id);
  const index = posts.findIndex(p => p.id === id);

  if (index === -1) {
    ctx.response.status = 404;
    ctx.response.body = { error: 'ไม่พบโพสต์' };
    return;
  }

  posts.splice(index, 1);
  ctx.response.status = 204;
});

const app = new Application();

// Middleware
app.use(oakCors());

app.use(async (ctx, next) => {
  await next();
  ctx.response.headers.set('X-Response-Time', `${Date.now()}ms`);
});

// Error handling
app.addEventListener('error', (evt) => {
  console.error(evt.error);
});

app.use(router.routes());
app.use(router.allowedMethods());

console.log('Oak server กำลังทำงานที่ http://localhost:8080');
await app.listen({ port: 8080 });
```

### 5.2 Database ด้วย Deno

```typescript
// database.ts
// deno run --allow-net --allow-env database.ts

import { Pool } from "https://deno.land/x/postgres@v0.19.3/mod.ts";

interface User {
  id: number;
  email: string;
  username: string;
  created_at: Date;
}

class Database {
  private pool: Pool;

  constructor() {
    this.pool = new Pool({
      hostname: Deno.env.get('DB_HOST') || 'localhost',
      port: parseInt(Deno.env.get('DB_PORT') || '5432'),
      user: Deno.env.get('DB_USER') || 'postgres',
      password: Deno.env.get('DB_PASSWORD') || '',
      database: Deno.env.get('DB_NAME') || 'blog',
    }, 3); // pool size = 3
  }

  async query<T>(sql: string, params?: unknown[]): Promise<T[]> {
    const client = await this.pool.connect();

    try {
      const result = await client.queryObject<T>(sql, params);
      return result.rows;
    } finally {
      client.release();
    }
  }

  async findUser(id: number): Promise<User | null> {
    const users = await this.query<User>(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return users[0] || null;
  }

  async createUser(email: string, username: string): Promise<User> {
    const users = await this.query<User>(
      'INSERT INTO users (email, username) VALUES ($1, $2) RETURNING *',
      [email, username]
    );
    return users[0];
  }

  async close(): Promise<void> {
    await this.pool.end();
  }
}

// ใช้งาน
const db = new Database();

const user = await db.createUser('test@example.com', 'testuser');
console.log('สร้างผู้ใช้:', user);

const found = await db.findUser(user.id);
console.log('ค้นหาผู้ใช้:', found);

await db.close();
```

## ส่วนที่ 6: Testing ใน Deno

### 6.1 Built-in Test Runner

```typescript
// calculator.ts
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
    throw new Error('ไม่สามารถหารด้วยศูนย์ได้');
  }
  return a / b;
}
```

```typescript
// calculator.test.ts
// deno test calculator.test.ts

import { assertEquals, assertThrows } from "https://deno.land/std@0.208.0/assert/mod.ts";
import { add, subtract, multiply, divide } from "./calculator.ts";

Deno.test("add", () => {
  assertEquals(add(2, 3), 5);
  assertEquals(add(-1, 1), 0);
  assertEquals(add(0, 0), 0);
});

Deno.test("subtract", () => {
  assertEquals(subtract(5, 3), 2);
  assertEquals(subtract(0, 5), -5);
});

Deno.test("multiply", () => {
  assertEquals(multiply(3, 4), 12);
  assertEquals(multiply(-2, 3), -6);
  assertEquals(multiply(0, 100), 0);
});

Deno.test("divide", () => {
  assertEquals(divide(10, 2), 5);
  assertEquals(divide(7, 2), 3.5);
});

Deno.test("divide by zero throws error", () => {
  assertThrows(
    () => divide(10, 0),
    Error,
    'ไม่สามารถหารด้วยศูนย์ได้'
  );
});

// Test group
Deno.test({
  name: "คำนวณเลขซับซ้อน",
  fn() {
    const result = add(multiply(2, 3), divide(10, 2));
    assertEquals(result, 11);
  },
});

// Async test
Deno.test("async operations", async () => {
  const delay = (ms: number) =>
    new Promise((resolve) => setTimeout(resolve, ms));

  const start = Date.now();
  await delay(100);
  const elapsed = Date.now() - start;

  // ต้องใช้เวลาอย่างน้อย 100ms
  assertEquals(elapsed >= 100, true);
});
```

### 6.2 Advanced Testing

```typescript
// advanced-tests.ts

import {
  assertEquals,
  assertExists,
  assertRejects,
  assertMatch,
} from "https://deno.land/std@0.208.0/assert/mod.ts";
import { spy, stub } from "https://deno.land/std@0.208.0/testing/mock.ts";

// Mock HTTP request
Deno.test("fetch API mock", async () => {
  const originalFetch = globalThis.fetch;

  // Mock fetch
  globalThis.fetch = () =>
    Promise.resolve(
      new Response(JSON.stringify({ userId: 1, id: 1, title: "Test" }), {
        headers: { 'Content-Type': 'application/json' },
      })
    );

  try {
    const response = await fetch('https://api.example.com/posts/1');
    const post = await response.json();

    assertEquals(post.id, 1);
    assertEquals(post.title, "Test");
  } finally {
    globalThis.fetch = originalFetch;
  }
});

// Test with spy
Deno.test("spy on function calls", () => {
  const logger = {
    log: (message: string) => console.log(message),
  };

  const logSpy = spy(logger, 'log');

  logger.log('ข้อความแรก');
  logger.log('ข้อความสอง');

  assertEquals(logSpy.calls.length, 2);
  assertEquals(logSpy.calls[0].args[0], 'ข้อความแรก');
});

// Parameterized tests
interface TestCase {
  input: number[];
  expected: number;
  description: string;
}

const addTestCases: TestCase[] = [
  { input: [1, 2], expected: 3, description: 'บวกเลขบวก' },
  { input: [-1, -2], expected: -3, description: 'บวกเลขลบ' },
  { input: [0, 0], expected: 0, description: 'บวกศูนย์' },
  { input: [100, -50], expected: 50, description: 'บวกเลขผสม' },
];

for (const { input, expected, description } of addTestCases) {
  Deno.test(`add: ${description}`, () => {
    const [a, b] = input;
    assertEquals(a + b, expected);
  });
}
```

## ส่วนที่ 7: Deno Deploy

### 7.1 Deploy ขึ้น Deno Deploy

```typescript
// main.ts - สำหรับ Deno Deploy

interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
}

function jsonResponse<T>(
  data: ApiResponse<T>,
  status = 200
): Response {
  return new Response(JSON.stringify(data), {
    status,
    headers: {
      'Content-Type': 'application/json',
      'X-Powered-By': 'Deno Deploy',
    },
  });
}

Deno.serve((req: Request) => {
  const url = new URL(req.url);

  // Health check
  if (url.pathname === '/health') {
    return jsonResponse({
      success: true,
      data: {
        status: 'ok',
        timestamp: new Date().toISOString(),
        region: Deno.env.get('DENO_REGION') || 'unknown',
      },
    });
  }

  // API routes
  if (url.pathname === '/api/hello') {
    const name = url.searchParams.get('name') || 'โลก';
    return jsonResponse({
      success: true,
      data: { message: `สวัสดี ${name} จาก Deno Deploy!` },
    });
  }

  // 404
  return jsonResponse({ success: false, error: 'ไม่พบหน้าที่ต้องการ' }, 404);
});
```

```yaml
# deno.json
{
  "tasks": {
    "dev": "deno run --allow-net --allow-env --watch main.ts",
    "test": "deno test --allow-net",
    "deploy": "deployctl deploy --project=my-blog --prod main.ts"
  },
  "imports": {
    "oak/": "https://deno.land/x/oak@v13.0.0/",
    "std/": "https://deno.land/std@0.208.0/"
  }
}
```

### 7.2 Environment Variables ใน Deno Deploy

```typescript
// config.ts

interface AppConfig {
  port: number;
  dbUrl: string;
  jwtSecret: string;
  isDevelopment: boolean;
}

function loadConfig(): AppConfig {
  const port = parseInt(Deno.env.get('PORT') || '8080');
  const dbUrl = Deno.env.get('DATABASE_URL');
  const jwtSecret = Deno.env.get('JWT_SECRET');

  if (!dbUrl) {
    throw new Error('DATABASE_URL is required');
  }

  if (!jwtSecret) {
    throw new Error('JWT_SECRET is required');
  }

  return {
    port,
    dbUrl,
    jwtSecret,
    isDevelopment: Deno.env.get('NODE_ENV') !== 'production',
  };
}

export const config = loadConfig();
```

## ส่วนที่ 8: Fresh Framework

### 8.1 การตั้งค่า Fresh

```bash
# สร้าง Fresh project
deno run -A -r https://fresh.deno.dev my-fresh-blog

cd my-fresh-blog
deno task start
```

### 8.2 Fresh Route Component

```tsx
// routes/index.tsx

import { Head } from "$fresh/runtime.ts";
import { Handlers, PageProps } from "$fresh/server.ts";

interface Post {
  id: number;
  title: string;
  excerpt: string;
  slug: string;
}

interface HomePageData {
  posts: Post[];
}

export const handler: Handlers<HomePageData> = {
  async GET(_, ctx) {
    const response = await fetch('https://api.example.com/posts');
    const posts = await response.json();

    return ctx.render({ posts });
  },
};

export default function Home({ data }: PageProps<HomePageData>) {
  return (
    <>
      <Head>
        <title>Blog Home</title>
        <meta name="description" content="บทความทั้งหมด" />
      </Head>
      <main class="max-w-4xl mx-auto px-4 py-8">
        <h1 class="text-3xl font-bold mb-8">บทความล่าสุด</h1>

        <div class="grid gap-6">
          {data.posts.map((post) => (
            <article key={post.id} class="bg-white rounded-lg shadow p-6">
              <h2 class="text-xl font-bold mb-2">
                <a href={`/posts/${post.slug}`} class="hover:text-blue-600">
                  {post.title}
                </a>
              </h2>
              <p class="text-gray-600">{post.excerpt}</p>
            </article>
          ))}
        </div>
      </main>
    </>
  );
}
```

### 8.3 Fresh Island Component (Interactive)

```tsx
// islands/LikeButton.tsx
import { useState } from "preact/hooks";

interface LikeButtonProps {
  initialCount: number;
  postId: number;
}

export default function LikeButton({ initialCount, postId }: LikeButtonProps) {
  const [count, setCount] = useState(initialCount);
  const [liked, setLiked] = useState(false);
  const [loading, setLoading] = useState(false);

  const handleLike = async () => {
    if (loading) return;

    setLoading(true);
    try {
      const response = await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
      });

      const data = await response.json();
      setCount(data.likesCount);
      setLiked(data.liked);
    } catch (error) {
      console.error('เกิดข้อผิดพลาด:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <button
      onClick={handleLike}
      disabled={loading}
      class={`flex items-center gap-2 px-4 py-2 rounded-full transition-colors ${
        liked
          ? 'bg-red-100 text-red-600'
          : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
      }`}
    >
      <span>{liked ? '❤️' : '🤍'}</span>
      <span>{count}</span>
    </button>
  );
}
```

## ส่วนที่ 9: WebSocket ใน Deno

```typescript
// websocket-server.ts
// deno run --allow-net websocket-server.ts

interface Client {
  id: string;
  socket: WebSocket;
  username: string;
}

interface Message {
  type: 'message' | 'join' | 'leave' | 'users';
  username?: string;
  content?: string;
  users?: string[];
  timestamp?: string;
}

const clients = new Map<string, Client>();

function broadcast(message: Message, excludeId?: string): void {
  const data = JSON.stringify(message);

  clients.forEach((client, id) => {
    if (id !== excludeId && client.socket.readyState === WebSocket.OPEN) {
      client.socket.send(data);
    }
  });
}

function getOnlineUsers(): string[] {
  return Array.from(clients.values()).map(c => c.username);
}

Deno.serve({ port: 8080 }, (req) => {
  if (req.headers.get('upgrade') !== 'websocket') {
    return new Response('WebSocket endpoint', { status: 200 });
  }

  const { socket, response } = Deno.upgradeWebSocket(req);
  const clientId = crypto.randomUUID();

  socket.addEventListener('open', () => {
    console.log(`Client เชื่อมต่อ: ${clientId}`);
  });

  socket.addEventListener('message', (event) => {
    try {
      const message: Message & { username: string } = JSON.parse(event.data);

      if (message.type === 'join') {
        const client: Client = {
          id: clientId,
          socket,
          username: message.username,
        };
        clients.set(clientId, client);

        // แจ้งทุกคนว่ามีคนเข้ามาใหม่
        broadcast({
          type: 'join',
          username: message.username,
          timestamp: new Date().toISOString(),
        }, clientId);

        // ส่งรายชื่อผู้ใช้ออนไลน์ให้ client ใหม่
        socket.send(JSON.stringify({
          type: 'users',
          users: getOnlineUsers(),
        }));
      }

      if (message.type === 'message') {
        const client = clients.get(clientId);
        if (client) {
          broadcast({
            type: 'message',
            username: client.username,
            content: message.content,
            timestamp: new Date().toISOString(),
          });
        }
      }
    } catch (error) {
      console.error('Invalid message:', error);
    }
  });

  socket.addEventListener('close', () => {
    const client = clients.get(clientId);
    if (client) {
      clients.delete(clientId);
      broadcast({
        type: 'leave',
        username: client.username,
        timestamp: new Date().toISOString(),
      });
      console.log(`Client ตัดการเชื่อมต่อ: ${client.username}`);
    }
  });

  return response;
});

console.log('WebSocket server กำลังทำงานที่ ws://localhost:8080');
```

## ส่วนที่ 10: Import Maps และ deno.json

```json
// deno.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "lib": ["deno.ns", "dom", "dom.iterable"]
  },
  "tasks": {
    "dev": "deno run --allow-all --watch src/main.ts",
    "test": "deno test --allow-all --coverage=coverage",
    "test:watch": "deno test --allow-all --watch",
    "lint": "deno lint",
    "fmt": "deno fmt",
    "check": "deno check src/**/*.ts",
    "coverage": "deno coverage coverage"
  },
  "imports": {
    "oak": "https://deno.land/x/oak@v13.0.0/mod.ts",
    "postgres": "https://deno.land/x/postgres@v0.19.3/mod.ts",
    "std/": "https://deno.land/std@0.208.0/",
    "zod": "https://deno.land/x/zod@v3.22.4/mod.ts"
  },
  "lint": {
    "include": ["src/"],
    "exclude": ["node_modules/"],
    "rules": {
      "tags": ["recommended"],
      "include": ["ban-untagged-todo"],
      "exclude": ["no-explicit-any"]
    }
  },
  "fmt": {
    "useTabs": false,
    "lineWidth": 80,
    "indentWidth": 2,
    "singleQuote": true
  }
}
```

## สรุป

Deno มีข้อดีหลายอย่างสำหรับนักพัฒนา TypeScript:

1. **TypeScript Native** - ไม่ต้อง compile หรือตั้งค่าอะไร
2. **Security First** - ระบบ permissions ที่ปลอดภัยโดย default
3. **Standard Library** - มี built-in library ที่ครบครัน
4. **ES Modules** - ใช้ URL imports แทน node_modules
5. **Built-in Tools** - มี formatter, linter, test runner ในตัว
6. **Browser Compatible APIs** - ใช้ fetch, WebSocket ได้เลย
7. **Deno Deploy** - Deploy ได้ง่ายมาก

Deno เหมาะสำหรับ:
- Serverless functions
- API servers
- CLI tools
- Scripts ขนาดเล็ก
- Projects ที่ต้องการความปลอดภัยสูง
