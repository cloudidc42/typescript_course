# Part 93: TypeScript กับ Bun

## บทนำ

Bun คือ JavaScript runtime ใหม่ที่เขียนด้วย Zig ออกแบบมาให้เร็วมากและ all-in-one - รวม runtime, package manager, bundler, และ test runner ไว้ในเครื่องมือเดียว Bun รองรับ TypeScript แบบ zero-config เช่นเดียวกับ Deno

## ส่วนที่ 1: Bun Overview

### 1.1 ความสามารถหลักของ Bun

| ความสามารถ | รายละเอียด |
|-----------|-----------|
| Runtime | เร็วกว่า Node.js 2-3x |
| TypeScript | Zero-config, ไม่ต้อง tsconfig |
| Package Manager | เร็วกว่า npm 10-30x |
| Bundler | เร็วกว่า webpack/esbuild |
| Test Runner | Built-in, ไม่ต้อง Jest |
| Node.js compat | 90%+ compatible |
| npm packages | ใช้ได้ทั้งหมด |

### 1.2 การติดตั้ง Bun

```bash
# macOS/Linux
curl -fsSL https://bun.sh/install | bash

# Windows
powershell -c "irm bun.sh/install.ps1 | iex"

# ผ่าน npm
npm install -g bun

# ตรวจสอบเวอร์ชัน
bun --version
# 1.0.25
```

### 1.3 รันไฟล์ TypeScript

```typescript
// hello.ts
interface Greeting {
  language: string;
  message: string;
}

function greet(name: string): Greeting {
  return {
    language: 'TypeScript',
    message: `สวัสดี ${name}! ยินดีต้อนรับสู่ Bun`,
  };
}

const result = greet('นักพัฒนา');
console.log(result);

// Type-safe array operations
const numbers: number[] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const evenSquares = numbers
  .filter((n): n is number => n % 2 === 0)
  .map((n) => n ** 2);

console.log('กำลังสองของเลขคู่:', evenSquares);
```

```bash
# รันโดยตรง ไม่ต้อง compile!
bun hello.ts

# หรือรันแบบ watch mode
bun --watch hello.ts
```

## ส่วนที่ 2: Bun Runtime APIs

### 2.1 Bun Built-in APIs

```typescript
// bun-apis.ts

// Bun.file - อ่านไฟล์ (เร็วมาก)
const file = Bun.file('./package.json');
const pkg = await file.json();
console.log('Package name:', pkg.name);

// อ่านเป็น text
const textContent = await Bun.file('./README.md').text();
console.log('README length:', textContent.length);

// อ่านเป็น ArrayBuffer
const buffer = await Bun.file('./image.png').arrayBuffer();
console.log('Image size:', buffer.byteLength, 'bytes');

// เขียนไฟล์
await Bun.write('./output.txt', 'Hello from Bun!');
await Bun.write('./data.json', JSON.stringify({ hello: 'world' }, null, 2));

// เขียนจาก Response
const response = await fetch('https://example.com/data.json');
await Bun.write('./downloaded.json', response);

// Bun.env - Environment variables
const dbUrl = Bun.env.DATABASE_URL || 'localhost:5432';
const port = parseInt(Bun.env.PORT || '3000');
console.log(`Server: ${dbUrl}, Port: ${port}`);
```

### 2.2 Bun.hash และ Crypto

```typescript
// crypto-utils.ts

// Hash
const hash1 = Bun.hash("Hello World");
console.log('Hash:', hash1);

// SHA-256
const sha256 = new Bun.CryptoHasher("sha256");
sha256.update("Hello");
sha256.update(" World");
const digest = sha256.digest("hex");
console.log('SHA256:', digest);

// Password hashing
const password = "mySecretPassword123";
const hashed = await Bun.password.hash(password);
console.log('Hashed:', hashed);

// Verify password
const isValid = await Bun.password.verify(password, hashed);
console.log('Password valid:', isValid);

// UUID
const id = crypto.randomUUID();
console.log('UUID:', id);
```

### 2.3 Bun.spawn - รัน Process

```typescript
// spawn.ts

// รัน command
const ls = Bun.spawn(['ls', '-la', '/tmp']);
const output = await new Response(ls.stdout).text();
console.log('Files:', output);

// Shell ด้วย $ template literal
import { $ } from 'bun';

const result = await $`echo "Hello from shell"`.text();
console.log(result);

// Build TypeScript project
const build = await $`tsc --noEmit`.quiet();
if (build.exitCode !== 0) {
  console.error('TypeScript errors found!');
} else {
  console.log('TypeScript OK');
}

// Git operations
const gitLog = await $`git log --oneline -5`.text();
console.log('Recent commits:\n', gitLog);
```

## ส่วนที่ 3: Bun HTTP Server

### 3.1 Basic Server

```typescript
// server.ts

const server = Bun.serve({
  port: 3000,
  hostname: '0.0.0.0',

  fetch(req: Request): Response | Promise<Response> {
    const url = new URL(req.url);

    // Static file serving
    if (url.pathname.startsWith('/static/')) {
      const filePath = `.${url.pathname}`;
      const file = Bun.file(filePath);
      return new Response(file);
    }

    // API routes
    if (url.pathname === '/api/hello') {
      return Response.json({
        message: 'สวัสดีจาก Bun!',
        timestamp: new Date().toISOString(),
      });
    }

    // 404
    return new Response('Not Found', { status: 404 });
  },

  error(err: Error): Response {
    console.error(err);
    return new Response('Internal Server Error', { status: 500 });
  },
});

console.log(`Server กำลังทำงานที่ http://localhost:${server.port}`);
```

### 3.2 Type-safe Router

```typescript
// typed-router.ts

type Method = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';

interface RouteParams {
  [key: string]: string;
}

interface RequestContext {
  req: Request;
  params: RouteParams;
  query: URLSearchParams;
  url: URL;
}

type RouteHandler = (
  ctx: RequestContext
) => Response | Promise<Response>;

interface Route {
  method: Method;
  pattern: URLPattern;
  handler: RouteHandler;
}

class Router {
  private routes: Route[] = [];

  private addRoute(method: Method, path: string, handler: RouteHandler) {
    this.routes.push({
      method,
      pattern: new URLPattern({ pathname: path }),
      handler,
    });
  }

  get(path: string, handler: RouteHandler) {
    this.addRoute('GET', path, handler);
    return this;
  }

  post(path: string, handler: RouteHandler) {
    this.addRoute('POST', path, handler);
    return this;
  }

  put(path: string, handler: RouteHandler) {
    this.addRoute('PUT', path, handler);
    return this;
  }

  delete(path: string, handler: RouteHandler) {
    this.addRoute('DELETE', path, handler);
    return this;
  }

  async handle(req: Request): Promise<Response> {
    const url = new URL(req.url);

    for (const route of this.routes) {
      if (route.method !== req.method) continue;

      const match = route.pattern.exec(url);
      if (!match) continue;

      const ctx: RequestContext = {
        req,
        params: match.pathname.groups as RouteParams,
        query: url.searchParams,
        url,
      };

      try {
        return await route.handler(ctx);
      } catch (err) {
        console.error('Route error:', err);
        return Response.json(
          { error: 'Internal Server Error' },
          { status: 500 }
        );
      }
    }

    return Response.json({ error: 'Not Found' }, { status: 404 });
  }
}

// ใช้งาน Router
const router = new Router();

// Types
interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
}

const posts: Post[] = [
  { id: 1, title: 'TypeScript กับ Bun', content: '...', authorId: 1 },
  { id: 2, title: 'Bun Performance', content: '...', authorId: 1 },
];

// Routes
router
  .get('/api/posts', ({ query }) => {
    const page = parseInt(query.get('page') || '1');
    const limit = parseInt(query.get('limit') || '10');
    const start = (page - 1) * limit;
    const paginated = posts.slice(start, start + limit);

    return Response.json({
      data: paginated,
      meta: {
        page,
        limit,
        total: posts.length,
        totalPages: Math.ceil(posts.length / limit),
      },
    });
  })
  .get('/api/posts/:id', ({ params }) => {
    const post = posts.find(p => p.id === parseInt(params.id));

    if (!post) {
      return Response.json({ error: 'ไม่พบโพสต์' }, { status: 404 });
    }

    return Response.json(post);
  })
  .post('/api/posts', async ({ req }) => {
    const body = await req.json() as Omit<Post, 'id'>;
    const newPost: Post = { id: posts.length + 1, ...body };
    posts.push(newPost);

    return Response.json(newPost, { status: 201 });
  });

Bun.serve({
  port: 3000,
  fetch: (req) => router.handle(req),
});

console.log('Server at http://localhost:3000');
```

### 3.3 WebSocket Server

```typescript
// ws-server.ts

interface Client {
  id: string;
  name: string;
  ws: ServerWebSocket<{ id: string; name: string }>;
}

const clients = new Map<string, Client>();

const server = Bun.serve<{ id: string; name: string }>({
  port: 3000,

  fetch(req, server) {
    const url = new URL(req.url);

    // Upgrade to WebSocket
    if (url.pathname === '/ws') {
      const name = url.searchParams.get('name') || `User_${Date.now()}`;
      const id = crypto.randomUUID();

      const upgraded = server.upgrade(req, { data: { id, name } });

      if (upgraded) {
        return undefined;
      }
    }

    return new Response('WebSocket Server', { status: 200 });
  },

  websocket: {
    open(ws) {
      const { id, name } = ws.data;

      clients.set(id, { id, name, ws });

      // แจ้งทุกคน
      broadcast({
        type: 'join',
        name,
        userCount: clients.size,
      });

      console.log(`${name} เชื่อมต่อแล้ว (${clients.size} คน)`);
    },

    message(ws, message) {
      const { id, name } = ws.data;

      try {
        const data = JSON.parse(message as string);

        broadcast({
          type: 'message',
          from: name,
          content: data.content,
          timestamp: new Date().toISOString(),
        });
      } catch (e) {
        ws.send(JSON.stringify({ error: 'Invalid message format' }));
      }
    },

    close(ws) {
      const { id, name } = ws.data;
      clients.delete(id);

      broadcast({
        type: 'leave',
        name,
        userCount: clients.size,
      });

      console.log(`${name} ตัดการเชื่อมต่อ`);
    },
  },
});

function broadcast(data: Record<string, unknown>) {
  const message = JSON.stringify(data);
  for (const client of clients.values()) {
    client.ws.send(message);
  }
}

console.log(`WebSocket server at ws://localhost:${server.port}`);
```

## ส่วนที่ 4: File I/O

### 4.1 อ่าน/เขียนไฟล์ขั้นสูง

```typescript
// file-io.ts

import { join } from 'path';

// อ่านไฟล์ใหญ่แบบ streaming
async function processLargeFile(filePath: string): Promise<void> {
  const file = Bun.file(filePath);
  const stream = file.stream();
  const reader = stream.getReader();
  const decoder = new TextDecoder();

  let processedLines = 0;

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const chunk = decoder.decode(value);
    const lines = chunk.split('\n');
    processedLines += lines.length;
  }

  console.log(`ประมวลผลแล้ว: ${processedLines} บรรทัด`);
}

// เขียนหลายไฟล์พร้อมกัน
async function writeMultipleFiles(
  files: Array<{ path: string; content: string }>
): Promise<void> {
  await Promise.all(
    files.map(({ path, content }) => Bun.write(path, content))
  );
  console.log(`เขียน ${files.length} ไฟล์เรียบร้อย`);
}

// Copy file
async function copyFile(src: string, dest: string): Promise<void> {
  const sourceFile = Bun.file(src);

  if (!(await sourceFile.exists())) {
    throw new Error(`ไม่พบไฟล์: ${src}`);
  }

  await Bun.write(dest, sourceFile);
  console.log(`คัดลอก ${src} → ${dest}`);
}

// แปลง JSON files
async function transformJsonFile<T, U>(
  inputPath: string,
  outputPath: string,
  transform: (data: T) => U
): Promise<void> {
  const inputData = await Bun.file(inputPath).json() as T;
  const outputData = transform(inputData);
  await Bun.write(outputPath, JSON.stringify(outputData, null, 2));
  console.log(`แปลง ${inputPath} → ${outputPath}`);
}

// ใช้งาน
await writeMultipleFiles([
  { path: '/tmp/file1.txt', content: 'เนื้อหาไฟล์ 1' },
  { path: '/tmp/file2.txt', content: 'เนื้อหาไฟล์ 2' },
  { path: '/tmp/file3.txt', content: 'เนื้อหาไฟล์ 3' },
]);
```

### 4.2 SQLite ด้วย Bun

```typescript
// sqlite.ts

import { Database } from 'bun:sqlite';

interface User {
  id: number;
  email: string;
  username: string;
  createdAt: string;
}

interface Post {
  id: number;
  title: string;
  content: string;
  userId: number;
  createdAt: string;
}

class BlogDatabase {
  private db: Database;

  constructor(path: string) {
    this.db = new Database(path, { create: true });
    this.init();
  }

  private init(): void {
    this.db.run(`
      CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        email TEXT UNIQUE NOT NULL,
        username TEXT UNIQUE NOT NULL,
        created_at TEXT NOT NULL DEFAULT (datetime('now'))
      )
    `);

    this.db.run(`
      CREATE TABLE IF NOT EXISTS posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT NOT NULL,
        user_id INTEGER NOT NULL,
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        FOREIGN KEY (user_id) REFERENCES users(id)
      )
    `);

    // Create indexes
    this.db.run(`CREATE INDEX IF NOT EXISTS idx_posts_user ON posts(user_id)`);
  }

  // Prepared statements สำหรับ performance
  private readonly insertUser = this.db.prepare<unknown, [string, string]>(
    'INSERT INTO users (email, username) VALUES (?, ?) RETURNING *'
  );

  private readonly findUserById = this.db.prepare<User, [number]>(
    'SELECT * FROM users WHERE id = ?'
  );

  private readonly insertPost = this.db.prepare<unknown, [string, string, number]>(
    'INSERT INTO posts (title, content, user_id) VALUES (?, ?, ?) RETURNING *'
  );

  createUser(email: string, username: string): User {
    const result = this.insertUser.get(email, username) as User;
    return result;
  }

  getUser(id: number): User | null {
    return this.findUserById.get(id) || null;
  }

  createPost(title: string, content: string, userId: number): Post {
    const result = this.insertPost.get(title, content, userId) as Post;
    return result;
  }

  getUserPosts(userId: number): Post[] {
    return this.db.prepare<Post, [number]>(
      'SELECT * FROM posts WHERE user_id = ? ORDER BY created_at DESC'
    ).all(userId);
  }

  searchPosts(query: string): Post[] {
    return this.db.prepare<Post, [string]>(
      "SELECT * FROM posts WHERE title LIKE ? OR content LIKE ?"
    ).all(`%${query}%`, `%${query}%`);
  }

  // Transaction
  createUserWithPost(
    email: string,
    username: string,
    postTitle: string,
    postContent: string
  ): { user: User; post: Post } {
    const transaction = this.db.transaction(() => {
      const user = this.createUser(email, username);
      const post = this.createPost(postTitle, postContent, user.id);
      return { user, post };
    });

    return transaction();
  }

  close(): void {
    this.db.close();
  }
}

// ใช้งาน
const db = new BlogDatabase('./blog.sqlite');

const { user, post } = db.createUserWithPost(
  'john@example.com',
  'john_doe',
  'บทความแรก',
  'เนื้อหาบทความ...'
);

console.log('ผู้ใช้:', user);
console.log('โพสต์:', post);

const userPosts = db.getUserPosts(user.id);
console.log('โพสต์ของผู้ใช้:', userPosts);

db.close();
```

## ส่วนที่ 5: Bun Package Manager

### 5.1 คำสั่ง bun install

```bash
# ติดตั้ง dependencies (เร็วกว่า npm มาก)
bun install

# เพิ่ม package
bun add express
bun add -d @types/express

# ลบ package
bun remove express

# อัพเดท packages
bun update

# รัน scripts
bun run dev
bun run build
bun run test

# รัน script โดยตรง (ไม่ต้องพิมพ์ run)
bun dev
bun build

# Lock file
# bun.lockb - binary lock file (เล็กและเร็ว)
```

### 5.2 package.json

```json
{
  "name": "my-bun-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "bun --watch src/index.ts",
    "build": "bun build src/index.ts --outdir=dist --target=bun",
    "test": "bun test",
    "lint": "bunx eslint src/",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "hono": "^3.12.0",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "@types/bun": "^1.0.0",
    "typescript": "^5.3.0"
  }
}
```

## ส่วนที่ 6: Bun Test Runner

### 6.1 การเขียน Tests

```typescript
// math.ts
export const add = (a: number, b: number): number => a + b;
export const multiply = (a: number, b: number): number => a * b;
export const factorial = (n: number): number => n <= 1 ? 1 : n * factorial(n - 1);
```

```typescript
// math.test.ts
import { describe, test, expect, beforeEach, afterEach, mock } from 'bun:test';
import { add, multiply, factorial } from './math';

describe('คณิตศาสตร์พื้นฐาน', () => {
  test('บวกเลข', () => {
    expect(add(2, 3)).toBe(5);
    expect(add(-1, 1)).toBe(0);
    expect(add(0, 0)).toBe(0);
  });

  test('คูณเลข', () => {
    expect(multiply(3, 4)).toBe(12);
    expect(multiply(-2, 3)).toBe(-6);
  });

  test('factorial', () => {
    expect(factorial(0)).toBe(1);
    expect(factorial(1)).toBe(1);
    expect(factorial(5)).toBe(120);
    expect(factorial(10)).toBe(3628800);
  });
});

// Mock functions
describe('Mock Functions', () => {
  test('mock fetch', async () => {
    const originalFetch = global.fetch;
    global.fetch = mock(() =>
      Promise.resolve(
        new Response(JSON.stringify({ id: 1, name: 'Test' }))
      )
    );

    const response = await fetch('/api/user');
    const data = await response.json();

    expect(data.id).toBe(1);
    expect(fetch).toHaveBeenCalledWith('/api/user');

    global.fetch = originalFetch;
  });

  test('spy on method', () => {
    const calculator = {
      add: (a: number, b: number) => a + b,
    };

    const spy = mock(calculator, 'add');

    calculator.add(2, 3);
    calculator.add(10, 20);

    expect(spy).toHaveBeenCalledTimes(2);
    expect(spy).toHaveBeenCalledWith(2, 3);
    expect(spy).toHaveBeenCalledWith(10, 20);
  });
});

// Async tests
describe('Async Operations', () => {
  test('async fetch data', async () => {
    // Mock API call
    const fetchData = async (id: number) => {
      return { id, name: `Item ${id}` };
    };

    const data = await fetchData(42);
    expect(data.id).toBe(42);
    expect(data.name).toBe('Item 42');
  });

  test('promise rejection', async () => {
    const failingFn = () => Promise.reject(new Error('API Error'));

    await expect(failingFn()).rejects.toThrow('API Error');
  });
});

// Snapshot testing
describe('Snapshot Tests', () => {
  test('user object snapshot', () => {
    const user = {
      id: 1,
      name: 'John',
      email: 'john@example.com',
      role: 'admin',
    };

    expect(user).toMatchSnapshot();
  });
});
```

```bash
# รัน tests
bun test

# รันด้วย watch mode
bun test --watch

# รันเฉพาะไฟล์
bun test math.test.ts

# Coverage report
bun test --coverage
```

## ส่วนที่ 7: Bun Bundler

### 7.1 Build Configuration

```typescript
// build.ts

await Bun.build({
  entrypoints: ['./src/index.ts'],
  outdir: './dist',
  target: 'browser', // หรือ 'bun', 'node'
  format: 'esm', // หรือ 'cjs', 'iife'
  minify: true,
  sourcemap: 'external',
  splitting: true, // Code splitting
  naming: {
    // Custom naming
    chunk: '[name]-[hash].[ext]',
    entry: '[dir]/[name].[ext]',
    asset: '[name]-[hash].[ext]',
  },
  define: {
    'process.env.NODE_ENV': JSON.stringify('production'),
    'API_URL': JSON.stringify('https://api.example.com'),
  },
  external: ['react', 'react-dom'], // Don't bundle these
  plugins: [
    // Custom plugin
    {
      name: 'my-plugin',
      setup(build) {
        // Transform .txt files
        build.onLoad({ filter: /\.txt$/ }, async (args) => {
          const content = await Bun.file(args.path).text();
          return {
            contents: `export default ${JSON.stringify(content)}`,
            loader: 'js',
          };
        });
      },
    },
  ],
});

console.log('Build เสร็จสมบูรณ์!');
```

### 7.2 Bundle Analysis

```typescript
// analyze-bundle.ts

const result = await Bun.build({
  entrypoints: ['./src/index.ts'],
  outdir: './dist',
  target: 'browser',
  minify: false,
});

// ดูขนาด outputs
for (const output of result.outputs) {
  const size = output.size;
  const sizeKB = (size / 1024).toFixed(2);
  console.log(`${output.path}: ${sizeKB} KB (${output.kind})`);
}

if (!result.success) {
  console.error('Build failed:');
  result.logs.forEach(log => console.error(log));
}
```

## ส่วนที่ 8: Elysia Framework

### 8.1 Elysia Setup

```bash
bun add elysia
```

```typescript
// elysia-server.ts

import { Elysia, t } from 'elysia';
import { jwt } from '@elysiajs/jwt';
import { cors } from '@elysiajs/cors';

interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
}

const posts: Post[] = [];
let nextId = 1;

const app = new Elysia()
  // Plugins
  .use(cors())
  .use(jwt({
    name: 'jwt',
    secret: 'my-secret-key',
    exp: '7d',
  }))

  // State
  .state('posts', posts)

  // Global error handling
  .onError(({ code, error, set }) => {
    switch (code) {
      case 'NOT_FOUND':
        set.status = 404;
        return { error: 'ไม่พบทรัพยากรที่ต้องการ' };

      case 'VALIDATION':
        set.status = 400;
        return { error: 'ข้อมูลไม่ถูกต้อง', details: error.message };

      default:
        set.status = 500;
        return { error: 'เกิดข้อผิดพลาดภายใน' };
    }
  })

  // Health check
  .get('/health', () => ({
    status: 'ok',
    timestamp: new Date().toISOString(),
  }))

  // Posts routes with TypeScript validation
  .group('/api/posts', (app) =>
    app
      .get('/', ({ store: { posts }, query }) => {
        const page = parseInt(query.page as string || '1');
        const limit = parseInt(query.limit as string || '10');
        const start = (page - 1) * limit;

        return {
          data: posts.slice(start, start + limit),
          meta: { page, limit, total: posts.length },
        };
      })

      .get('/:id', ({ params: { id }, store: { posts } }) => {
        const post = posts.find(p => p.id === parseInt(id));
        if (!post) throw new Error('Post not found');
        return post;
      })

      .post('/', ({ body, store, jwt }) => {
        const newPost: Post = {
          id: nextId++,
          ...body as { title: string; content: string; authorId: number },
        };
        store.posts.push(newPost);
        return { status: 201, data: newPost };
      }, {
        body: t.Object({
          title: t.String({ minLength: 5 }),
          content: t.String({ minLength: 20 }),
          authorId: t.Number(),
        }),
      })

      .delete('/:id', ({ params: { id }, store }) => {
        const index = store.posts.findIndex(p => p.id === parseInt(id));
        if (index === -1) throw new Error('Post not found');
        store.posts.splice(index, 1);
        return { message: 'ลบโพสต์เรียบร้อย' };
      })
  )

  // Auth routes
  .group('/api/auth', (app) =>
    app
      .post('/login', async ({ body, jwt }) => {
        const { email, password } = body as { email: string; password: string };

        // Validate credentials (simplified)
        if (email !== 'admin@example.com' || password !== 'password') {
          throw new Error('Unauthorized');
        }

        const token = await jwt.sign({ email, role: 'admin' });
        return { token };
      })
  )

  .listen(3000);

console.log(`Elysia server กำลังทำงานที่ http://${app.server?.hostname}:${app.server?.port}`);
```

### 8.2 Elysia Middleware และ Guards

```typescript
// elysia-advanced.ts

import { Elysia, t } from 'elysia';

// Plugin สำหรับ logging
const logger = new Elysia({ name: 'logger' })
  .onBeforeHandle(({ request }) => {
    console.log(`→ ${request.method} ${new URL(request.url).pathname}`);
  })
  .onAfterHandle(({ request, response }) => {
    console.log(`← ${request.method} ${new URL(request.url).pathname} OK`);
  });

// Plugin สำหรับ rate limiting
const rateLimiter = new Elysia({ name: 'rate-limiter' })
  .state('requestCounts', new Map<string, number[]>())
  .onBeforeHandle(({ request, store, set }) => {
    const ip = request.headers.get('x-forwarded-for') || 'unknown';
    const now = Date.now();
    const window = 60 * 1000;
    const limit = 100;

    const counts = store.requestCounts.get(ip) || [];
    const valid = counts.filter(t => now - t < window);

    if (valid.length >= limit) {
      set.status = 429;
      return { error: 'Too Many Requests' };
    }

    valid.push(now);
    store.requestCounts.set(ip, valid);
  });

// Auth plugin
const auth = new Elysia({ name: 'auth' })
  .derive(({ request }) => {
    const token = request.headers.get('authorization')?.replace('Bearer ', '');
    
    return {
      isAuthenticated: !!token,
      userId: token ? 'user-123' : null,
    };
  });

// Main app
const app = new Elysia()
  .use(logger)
  .use(rateLimiter)
  .use(auth)

  .get('/protected', ({ isAuthenticated, userId }) => {
    if (!isAuthenticated) {
      return { error: 'Unauthorized' };
    }
    return { message: `สวัสดี User ${userId}` };
  })

  .listen(3000);
```

## ส่วนที่ 9: Migration จาก Node.js ไป Bun

### 9.1 ตรวจสอบ Compatibility

```bash
# ตรวจสอบว่า project สามารถใช้ Bun ได้
bun pm untrusted

# รัน Node.js app ด้วย Bun
bun start  # แทน node index.js
```

### 9.2 แปลง CommonJS เป็น ESM

```typescript
// เดิม (CommonJS)
// const express = require('express');
// module.exports = { something };

// ใหม่ (ESM - Bun รองรับแบบ native)
import express from 'express';
export { something };

// Bun รองรับทั้งสองแบบ แต่ ESM แนะนำ
```

### 9.3 ใช้ Bun APIs แทน Node APIs

```typescript
// การเปลี่ยนแปลงที่พบบ่อย

// Node.js: fs.readFileSync
// import { readFileSync } from 'fs';
// const content = readFileSync('./file.txt', 'utf-8');

// Bun: Bun.file (เร็วกว่ามาก)
const content = await Bun.file('./file.txt').text();

// Node.js: path.join
// import { join } from 'path';
// Bun รองรับ path module เช่นเดิม
import { join } from 'path';

// Node.js: process.env
// Bun รองรับ process.env แต่มี Bun.env ด้วย
const port = process.env.PORT || Bun.env.PORT || '3000';

// Node.js: Buffer
// Bun รองรับ Buffer
const buffer = Buffer.from('Hello World');

// Node.js: crypto
// Bun รองรับ crypto module
import { createHash } from 'crypto';
const hash = createHash('sha256').update('test').digest('hex');
```

## ส่วนที่ 10: Performance Comparison

### 10.1 Benchmark

```typescript
// benchmark.ts

function measureTime<T>(
  name: string,
  fn: () => T | Promise<T>
): Promise<{ name: string; time: number; result: T }> {
  return new Promise(async (resolve) => {
    const start = Bun.nanoseconds();
    const result = await fn();
    const end = Bun.nanoseconds();
    const time = (end - start) / 1_000_000; // ms

    resolve({ name, time, result });
  });
}

// Benchmark JSON parsing
const iterations = 100_000;
const jsonData = JSON.stringify({ 
  id: 1, 
  name: 'test', 
  data: Array.from({ length: 100 }, (_, i) => i) 
});

const jsonBench = await measureTime('JSON.parse (100k iterations)', () => {
  for (let i = 0; i < iterations; i++) {
    JSON.parse(jsonData);
  }
  return 'done';
});

console.log(`${jsonBench.name}: ${jsonBench.time.toFixed(2)}ms`);

// Benchmark string operations
const strBench = await measureTime('String concatenation', () => {
  let result = '';
  for (let i = 0; i < iterations; i++) {
    result += `item_${i}`;
  }
  return result.length;
});

console.log(`${strBench.name}: ${strBench.time.toFixed(2)}ms`);

// Benchmark file operations
const fileBench = await measureTime('File write (1000 times)', async () => {
  const promises = Array.from({ length: 1000 }, (_, i) =>
    Bun.write(`/tmp/bench_${i}.txt`, `content_${i}`)
  );
  await Promise.all(promises);
  return 'done';
});

console.log(`${fileBench.name}: ${fileBench.time.toFixed(2)}ms`);
```

## สรุป

Bun เป็น runtime ที่น่าสนใจสำหรับ TypeScript เพราะ:

1. **ความเร็ว** - เร็วกว่า Node.js อย่างมีนัยสำคัญ
2. **Zero Config** - TypeScript ทำงานได้เลยโดยไม่ต้องตั้งค่า
3. **All-in-one** - Runtime + Package Manager + Bundler + Test Runner
4. **Node.js compat** - ใช้ npm packages ได้เกือบทั้งหมด
5. **SQLite built-in** - มี SQLite อยู่ในตัว
6. **Hot reload** - --watch flag ในตัว

เมื่อควรใช้ Bun:
- Projects ใหม่ที่ต้องการ performance
- API servers ที่ต้องการ throughput สูง
- CLI tools
- Build scripts
- ต้องการ all-in-one tooling

เมื่อควรระวัง:
- Production workloads ที่ critical (ยังอยู่ในการพัฒนา)
- Packages บางตัวที่ต้องใช้ Node.js specifics
- Team ที่คุ้นเคยกับ Node.js ecosystem แล้ว
