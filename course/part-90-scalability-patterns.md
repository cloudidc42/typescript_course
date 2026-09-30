# Part 90: Scalability Patterns กับ TypeScript

## บทนำ

Scalability คือความสามารถของระบบในการรองรับโหลดที่เพิ่มขึ้นโดยยังคงประสิทธิภาพที่ดี ในบทนี้เราจะเรียนรู้ patterns ต่างๆ สำหรับสร้างระบบที่ scale ได้ด้วย TypeScript และ Node.js

---

## 1. Horizontal vs Vertical Scaling

### 1.1 ความแตกต่าง

```
Vertical Scaling (Scale Up)      Horizontal Scaling (Scale Out)
─────────────────────────────    ───────────────────────────────
เพิ่ม RAM/CPU ให้เซิร์ฟเวอร์     เพิ่มจำนวนเซิร์ฟเวอร์
มี limit (ฮาร์ดแวร์)              ไม่มี limit ทางทฤษฎี
ง่ายกว่า (ไม่ต้องเปลี่ยน code)   ต้องออกแบบ stateless
downtime ขณะ upgrade             ไม่มี downtime (rolling deploy)
ราคาแพงขึ้นเรื่อยๆ               ราคาต่อ unit ลดลง
```

### 1.2 Stateless Service Design

```typescript
// ❌ Stateful: เก็บ state ใน memory → scale ไม่ได้
class BadSessionManager {
  private sessions = new Map<string, Session>(); // ข้อมูลอยู่ใน process เดียว!

  getSession(id: string): Session | undefined {
    return this.sessions.get(id); // process อื่นไม่รู้
  }
}

// ✅ Stateless: เก็บ state ใน shared storage
class StatelessSessionManager {
  constructor(private readonly redis: Redis) {}

  async getSession(id: string): Promise<Session | null> {
    const raw = await this.redis.get(`session:${id}`);
    return raw ? JSON.parse(raw) : null;
  }

  async setSession(id: string, session: Session): Promise<void> {
    await this.redis.setex(`session:${id}`, 3600, JSON.stringify(session));
  }
}
```

### 1.3 12-Factor App Principles ที่เกี่ยวกับ Scalability

```typescript
// Factor 3: Config - ใช้ environment variables
const config = {
  port: parseInt(process.env.PORT ?? '3000'),
  dbUrl: process.env.DATABASE_URL!,
  redisUrl: process.env.REDIS_URL ?? 'redis://localhost:6379',
  workers: parseInt(process.env.WEB_CONCURRENCY ?? '1'),
};

// Factor 6: Processes - process stateless
// Factor 8: Concurrency - scale via process model
// Factor 9: Disposability - fast startup/graceful shutdown
```

---

## 2. Node.js Cluster Module

### 2.1 Basic Cluster Setup

```typescript
// src/cluster.ts
import cluster from 'cluster';
import os from 'os';
import process from 'process';

const NUM_WORKERS = parseInt(process.env.WEB_CONCURRENCY ?? '') || os.cpus().length;

export function startCluster(workerFn: () => void): void {
  if (cluster.isPrimary) {
    console.log(`Master process ${process.pid} กำลังทำงาน`);
    console.log(`เริ่ม ${NUM_WORKERS} workers...`);

    // Fork workers
    for (let i = 0; i < NUM_WORKERS; i++) {
      cluster.fork();
    }

    // ถ้า worker ตาย ให้ fork ใหม่
    cluster.on('exit', (worker, code, signal) => {
      console.warn(
        `Worker ${worker.process.pid} ตาย (code: ${code}, signal: ${signal})`
      );

      if (!worker.exitedAfterDisconnect) {
        console.log('กำลัง restart worker...');
        cluster.fork();
      }
    });

    cluster.on('online', (worker) => {
      console.log(`Worker ${worker.process.pid} online`);
    });

    // Graceful shutdown
    process.on('SIGTERM', () => {
      console.log('Master ได้รับ SIGTERM กำลังปิด workers...');
      
      for (const worker of Object.values(cluster.workers ?? {})) {
        worker?.send('shutdown');
        worker?.disconnect();
      }

      setTimeout(() => {
        console.log('Force kill remaining workers');
        process.exit(0);
      }, 10000).unref();
    });

  } else {
    // Worker process
    console.log(`Worker ${process.pid} เริ่มทำงาน`);
    workerFn();

    process.on('message', (msg) => {
      if (msg === 'shutdown') {
        console.log(`Worker ${process.pid} กำลังปิด gracefully...`);
        process.exit(0);
      }
    });
  }
}

// ใช้งาน
import express from 'express';

startCluster(() => {
  const app = express();
  
  app.get('/health', (req, res) => {
    res.json({ 
      status: 'ok', 
      pid: process.pid,
      uptime: process.uptime()
    });
  });

  app.listen(3000, () => {
    console.log(`Worker ${process.pid} listening on port 3000`);
  });
});
```

### 2.2 Zero-Downtime Reload

```typescript
// src/cluster.manager.ts
export class ClusterManager {
  private workers: Map<number, cluster.Worker> = new Map();

  async rollingReload(): Promise<void> {
    const workerIds = Array.from(this.workers.keys());
    
    for (const workerId of workerIds) {
      const oldWorker = this.workers.get(workerId);
      if (!oldWorker) continue;

      console.log(`Reloading worker ${oldWorker.process.pid}...`);

      // Fork ใหม่ก่อน
      const newWorker = cluster.fork();
      
      // รอ worker ใหม่ online
      await new Promise<void>((resolve) => {
        newWorker.on('online', () => {
          console.log(`New worker ${newWorker.process.pid} online`);
          
          // ปิด worker เก่า gracefully
          oldWorker.send('shutdown');
          oldWorker.disconnect();
          
          resolve();
        });
      });

      // รอสักครู่ก่อน reload worker ถัดไป
      await new Promise(resolve => setTimeout(resolve, 1000));
    }

    console.log('Rolling reload เสร็จสิ้น');
  }
}
```

---

## 3. Worker Threads สำหรับ CPU-Intensive Tasks

### 3.1 Worker Thread Setup

```typescript
// src/workers/crypto.worker.ts
import { parentPort, workerData } from 'worker_threads';
import crypto from 'crypto';

interface WorkerInput {
  type: 'hash' | 'encrypt' | 'decrypt';
  data: string;
  key?: string;
}

interface WorkerOutput {
  success: boolean;
  result?: string;
  error?: string;
}

// Worker logic
function processTask(input: WorkerInput): WorkerOutput {
  try {
    switch (input.type) {
      case 'hash': {
        // CPU-intensive hashing (e.g., bcrypt-like)
        let result = input.data;
        for (let i = 0; i < 10000; i++) {
          result = crypto.createHash('sha256').update(result).digest('hex');
        }
        return { success: true, result };
      }
      case 'encrypt': {
        const key = crypto.scryptSync(input.key ?? 'default', 'salt', 32);
        const iv = crypto.randomBytes(16);
        const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
        const encrypted = Buffer.concat([
          cipher.update(input.data, 'utf8'),
          cipher.final()
        ]);
        return { 
          success: true, 
          result: iv.toString('hex') + ':' + encrypted.toString('hex') 
        };
      }
      default:
        return { success: false, error: 'Unknown task type' };
    }
  } catch (error) {
    return { success: false, error: (error as Error).message };
  }
}

// Run task and send result back
const result = processTask(workerData as WorkerInput);
parentPort?.postMessage(result);
```

### 3.2 Worker Thread Pool

```typescript
// src/workers/thread.pool.ts
import { Worker } from 'worker_threads';
import path from 'path';

interface QueuedTask {
  data: unknown;
  resolve: (value: unknown) => void;
  reject: (reason: unknown) => void;
}

export class WorkerThreadPool {
  private workers: Worker[] = [];
  private freeWorkers: Worker[] = [];
  private taskQueue: QueuedTask[] = [];
  private workerTaskMap = new Map<Worker, QueuedTask>();

  constructor(
    private readonly workerScript: string,
    private readonly poolSize: number = 4
  ) {
    this.initializeWorkers();
  }

  private initializeWorkers(): void {
    for (let i = 0; i < this.poolSize; i++) {
      this.addWorker();
    }
  }

  private addWorker(): void {
    const worker = new Worker(
      path.resolve(this.workerScript),
      { workerData: null } // จะ override เมื่อส่ง task
    );

    worker.on('error', (err) => {
      console.error('Worker error:', err);
      const task = this.workerTaskMap.get(worker);
      if (task) {
        task.reject(err);
        this.workerTaskMap.delete(worker);
      }
      // Replace broken worker
      this.workers = this.workers.filter(w => w !== worker);
      this.freeWorkers = this.freeWorkers.filter(w => w !== worker);
      this.addWorker();
    });

    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }

  async execute<TInput, TOutput>(data: TInput): Promise<TOutput> {
    return new Promise((resolve, reject) => {
      const task: QueuedTask = { data, resolve, reject };
      
      if (this.freeWorkers.length > 0) {
        this.runTask(this.freeWorkers.pop()!, task);
      } else {
        // Queue งานรอ
        this.taskQueue.push(task);
      }
    });
  }

  private runTask(worker: Worker, task: QueuedTask): void {
    this.workerTaskMap.set(worker, task);

    // สร้าง worker ใหม่พร้อม data
    const newWorker = new Worker(
      path.resolve(this.workerScript),
      { workerData: task.data }
    );

    newWorker.once('message', (result) => {
      task.resolve(result);
      newWorker.terminate();
      
      // เอา worker กลับมา pool
      this.workerTaskMap.delete(worker);
      this.freeWorkers.push(worker);
      
      // ทำงาน task ถัดไปถ้ามี
      if (this.taskQueue.length > 0) {
        const nextTask = this.taskQueue.shift()!;
        this.runTask(worker, nextTask);
      }
    });

    newWorker.once('error', (err) => {
      task.reject(err);
      newWorker.terminate();
      this.workerTaskMap.delete(worker);
      this.freeWorkers.push(worker);
    });
  }

  async terminate(): Promise<void> {
    await Promise.all(this.workers.map(w => w.terminate()));
    this.workers = [];
    this.freeWorkers = [];
  }
}

// ใช้งาน
const cryptoPool = new WorkerThreadPool('./src/workers/crypto.worker.js', 4);

async function hashPasswordAsync(password: string): Promise<string> {
  const result = await cryptoPool.execute<
    { type: string; data: string },
    { success: boolean; result?: string }
  >({ type: 'hash', data: password });
  
  if (!result.success || !result.result) {
    throw new Error('Hash ล้มเหลว');
  }
  return result.result;
}
```

---

## 4. Connection Pooling

### 4.1 Database Connection Pool

```typescript
// src/db/pool.ts
import { Pool, PoolConfig, PoolClient } from 'pg';

export class DatabasePool {
  private pool: Pool;
  private metrics = {
    queries: 0,
    errors: 0,
    totalTime: 0,
  };

  constructor(config: PoolConfig) {
    this.pool = new Pool({
      max: parseInt(process.env.DB_POOL_MAX ?? '10'),       // max connections
      min: parseInt(process.env.DB_POOL_MIN ?? '2'),        // min connections
      idleTimeoutMillis: 30000,                              // ปิด idle connection 30s
      connectionTimeoutMillis: 2000,                         // timeout รอ connection 2s
      statement_timeout: 30000,                              // query timeout 30s
      ...config,
    });

    this.pool.on('connect', (client: PoolClient) => {
      console.log('Database connection เปิดแล้ว');
    });

    this.pool.on('error', (err: Error) => {
      console.error('Database pool error:', err);
    });

    this.pool.on('acquire', () => {
      // connection ถูก checkout
    });

    this.pool.on('remove', () => {
      // connection ถูก remove จาก pool
    });
  }

  async query<T = unknown>(
    sql: string,
    params?: unknown[]
  ): Promise<{ rows: T[]; rowCount: number }> {
    const start = Date.now();
    this.metrics.queries++;

    try {
      const result = await this.pool.query(sql, params);
      this.metrics.totalTime += Date.now() - start;
      return { rows: result.rows as T[], rowCount: result.rowCount ?? 0 };
    } catch (error) {
      this.metrics.errors++;
      throw error;
    }
  }

  // Transaction
  async withTransaction<T>(
    fn: (client: PoolClient) => Promise<T>
  ): Promise<T> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  getPoolStats(): {
    totalCount: number;
    idleCount: number;
    waitingCount: number;
    queriesPerSecond: number;
  } {
    return {
      totalCount: this.pool.totalCount,
      idleCount: this.pool.idleCount,
      waitingCount: this.pool.waitingCount,
      queriesPerSecond: this.metrics.queries / (process.uptime() || 1),
    };
  }

  async healthCheck(): Promise<boolean> {
    try {
      await this.query('SELECT 1');
      return true;
    } catch {
      return false;
    }
  }

  async close(): Promise<void> {
    await this.pool.end();
  }
}
```

### 4.2 Redis Connection Pool

```typescript
// src/cache/redis.pool.ts
import Redis, { Cluster } from 'ioredis';

export class RedisPool {
  private clients: Redis[] = [];
  private currentIndex = 0;

  constructor(
    private readonly url: string,
    private readonly poolSize = 5
  ) {
    for (let i = 0; i < poolSize; i++) {
      this.clients.push(new Redis(url, {
        maxRetriesPerRequest: 3,
        enableReadyCheck: true,
      }));
    }
  }

  // Round-robin distribution
  getClient(): Redis {
    const client = this.clients[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.clients.length;
    return client;
  }

  async close(): Promise<void> {
    await Promise.all(this.clients.map(c => c.quit()));
  }
}
```

---

## 5. Rate Limiting Implementation

### 5.1 Sliding Window Rate Limiter

```typescript
// src/middleware/sliding-window.rate-limiter.ts
import Redis from 'ioredis';

export interface RateLimitConfig {
  windowMs: number;    // ขนาด window (ms)
  maxRequests: number; // requests สูงสุดใน window
  keyPrefix?: string;
}

export class SlidingWindowRateLimiter {
  constructor(
    private readonly redis: Redis,
    private readonly config: RateLimitConfig
  ) {}

  async isAllowed(identifier: string): Promise<{
    allowed: boolean;
    remaining: number;
    resetAt: number;
    retryAfter?: number;
  }> {
    const key = `${this.config.keyPrefix ?? 'ratelimit'}:${identifier}`;
    const now = Date.now();
    const windowStart = now - this.config.windowMs;

    // Lua script เพื่อ atomic operation
    const luaScript = `
      local key = KEYS[1]
      local now = tonumber(ARGV[1])
      local window_start = tonumber(ARGV[2])
      local max_requests = tonumber(ARGV[3])
      local window_ms = tonumber(ARGV[4])
      
      -- ลบ entries เก่าที่นอก window
      redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
      
      -- นับ requests ปัจจุบัน
      local count = redis.call('ZCARD', key)
      
      if count < max_requests then
        -- อนุญาต: เพิ่ม request ปัจจุบัน
        redis.call('ZADD', key, now, tostring(now) .. '-' .. math.random())
        redis.call('PEXPIRE', key, window_ms)
        return {1, max_requests - count - 1, 0}
      else
        -- ปฏิเสธ: หา retry after
        local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
        local retry_after = 0
        if #oldest >= 2 then
          retry_after = tonumber(oldest[2]) + window_ms - now
        end
        return {0, 0, retry_after}
      end
    `;

    const result = await this.redis.eval(
      luaScript, 1, key,
      now.toString(),
      windowStart.toString(),
      this.config.maxRequests.toString(),
      this.config.windowMs.toString()
    ) as [number, number, number];

    const [allowed, remaining, retryAfterMs] = result;

    return {
      allowed: allowed === 1,
      remaining,
      resetAt: now + this.config.windowMs,
      retryAfter: retryAfterMs > 0 ? Math.ceil(retryAfterMs / 1000) : undefined,
    };
  }
}

// Express middleware
export function slidingWindowMiddleware(
  limiter: SlidingWindowRateLimiter,
  keyExtractor: (req: any) => string = (req) => req.ip
) {
  return async (req: any, res: any, next: any) => {
    const key = keyExtractor(req);
    const result = await limiter.isAllowed(key);
    
    res.setHeader('X-RateLimit-Limit', '100');
    res.setHeader('X-RateLimit-Remaining', result.remaining);
    res.setHeader('X-RateLimit-Reset', new Date(result.resetAt).toISOString());

    if (!result.allowed) {
      if (result.retryAfter) {
        res.setHeader('Retry-After', result.retryAfter);
      }
      return res.status(429).json({
        error: 'Too Many Requests',
        message: `กรุณารอ ${result.retryAfter} วินาทีก่อนลองใหม่`,
        retryAfter: result.retryAfter,
      });
    }

    next();
  };
}
```

### 5.2 Token Bucket Rate Limiter

```typescript
// src/middleware/token-bucket.rate-limiter.ts
export class TokenBucketRateLimiter {
  constructor(
    private readonly redis: Redis,
    private readonly options: {
      capacity: number;       // ความจุของ bucket
      refillRate: number;     // token ต่อวินาที
      keyPrefix?: string;
    }
  ) {}

  async consume(identifier: string, tokens = 1): Promise<{
    allowed: boolean;
    remaining: number;
    nextRefillAt: number;
  }> {
    const key = `${this.options.keyPrefix ?? 'tokenbucket'}:${identifier}`;
    const now = Date.now() / 1000; // วินาที

    const luaScript = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      local requested = tonumber(ARGV[4])
      
      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local tokens = tonumber(bucket[1]) or capacity
      local last_refill = tonumber(bucket[2]) or now
      
      -- เติม tokens ตาม elapsed time
      local elapsed = now - last_refill
      local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)
      
      if new_tokens >= requested then
        -- อนุญาต
        redis.call('HMSET', key, 'tokens', new_tokens - requested, 'last_refill', now)
        redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 1)
        return {1, new_tokens - requested}
      else
        -- ปฏิเสธ
        redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
        redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 1)
        return {0, new_tokens}
      end
    `;

    const result = await this.redis.eval(
      luaScript, 1, key,
      this.options.capacity.toString(),
      this.options.refillRate.toString(),
      now.toString(),
      tokens.toString()
    ) as [number, number];

    const tokensUntilRefill = tokens - result[1];
    const secondsUntilRefill = tokensUntilRefill / this.options.refillRate;

    return {
      allowed: result[0] === 1,
      remaining: result[1],
      nextRefillAt: Date.now() + secondsUntilRefill * 1000,
    };
  }
}
```

---

## 6. Backpressure Handling ใน Streams

### 6.1 Readable Stream กับ Backpressure

```typescript
// src/streams/backpressure.ts
import { Readable, Writable, Transform, pipeline } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// Custom Readable Stream ที่รองรับ backpressure
class PaymentDataSource extends Readable {
  private currentIndex = 0;
  private readonly totalRecords: number;

  constructor(totalRecords = 100000) {
    super({ objectMode: true, highWaterMark: 100 }); // buffer 100 objects
    this.totalRecords = totalRecords;
  }

  _read(size: number): void {
    // ถ้า push คืน false → หยุดผลิต (backpressure)
    let canPush = true;
    
    while (canPush && this.currentIndex < this.totalRecords) {
      const payment = {
        id: `PAY-${this.currentIndex}`,
        amount: Math.random() * 10000,
        currency: 'THB',
        timestamp: new Date().toISOString(),
      };
      
      canPush = this.push(payment);
      this.currentIndex++;
    }

    if (this.currentIndex >= this.totalRecords) {
      this.push(null); // สัญญาณ end of stream
    }
  }
}

// Transform Stream สำหรับ process
class PaymentProcessor extends Transform {
  constructor() {
    super({ objectMode: true, highWaterMark: 50 });
  }

  _transform(payment: any, encoding: string, callback: (error?: Error | null, data?: any) => void): void {
    // ประมวลผล payment
    const processed = {
      ...payment,
      processedAt: new Date().toISOString(),
      status: 'PROCESSED',
      taxAmount: payment.amount * 0.07, // VAT 7%
    };

    callback(null, processed);
  }
}

// Writable Stream สำหรับบันทึก
class DatabaseWriter extends Writable {
  private buffer: unknown[] = [];
  private readonly batchSize = 100;

  constructor() {
    super({ objectMode: true, highWaterMark: 200 });
  }

  _write(
    chunk: unknown,
    encoding: string,
    callback: (error?: Error | null) => void
  ): void {
    this.buffer.push(chunk);
    
    if (this.buffer.length >= this.batchSize) {
      this.flushBatch()
        .then(() => callback())
        .catch(callback);
    } else {
      callback();
    }
  }

  _final(callback: (error?: Error | null) => void): void {
    if (this.buffer.length > 0) {
      this.flushBatch()
        .then(() => callback())
        .catch(callback);
    } else {
      callback();
    }
  }

  private async flushBatch(): Promise<void> {
    console.log(`Saving batch of ${this.buffer.length} records...`);
    // TODO: batch insert to database
    this.buffer = [];
  }
}

// ใช้งาน pipeline พร้อม backpressure
async function processPaymentStream(): Promise<void> {
  const source = new PaymentDataSource(100000);
  const processor = new PaymentProcessor();
  const writer = new DatabaseWriter();

  console.log('เริ่ม stream processing...');
  
  await pipelineAsync(source, processor, writer);
  
  console.log('Stream processing เสร็จสิ้น');
}
```

---

## 7. Job Queue สำหรับ Async Processing

### 7.1 Priority Job Queue

```typescript
// src/jobs/priority.queue.ts
interface Job<T> {
  id: string;
  priority: number;
  data: T;
  createdAt: number;
  maxAttempts: number;
  currentAttempts: number;
}

export class PriorityJobQueue<T> {
  private queue: Job<T>[] = [];
  private processing = false;
  private workers: number = 0;

  constructor(
    private readonly handler: (data: T) => Promise<void>,
    private readonly options: {
      concurrency?: number;
      maxAttempts?: number;
    } = {}
  ) {}

  enqueue(
    data: T,
    priority = 5,
    maxAttempts?: number
  ): string {
    const job: Job<T> = {
      id: `job-${Date.now()}-${Math.random().toString(36).slice(2)}`,
      priority,
      data,
      createdAt: Date.now(),
      maxAttempts: maxAttempts ?? this.options.maxAttempts ?? 3,
      currentAttempts: 0,
    };

    // Insert ตาม priority (สูงกว่า = ทำก่อน)
    const insertIndex = this.queue.findIndex(j => j.priority < priority);
    if (insertIndex === -1) {
      this.queue.push(job);
    } else {
      this.queue.splice(insertIndex, 0, job);
    }

    this.processNext();
    return job.id;
  }

  private async processNext(): Promise<void> {
    const maxConcurrency = this.options.concurrency ?? 5;
    
    if (this.workers >= maxConcurrency || this.queue.length === 0) return;

    const job = this.queue.shift();
    if (!job) return;

    this.workers++;

    try {
      job.currentAttempts++;
      await this.handler(job.data);
    } catch (error) {
      if (job.currentAttempts < job.maxAttempts) {
        // Re-enqueue กับ delay
        const delay = Math.pow(2, job.currentAttempts) * 1000;
        setTimeout(() => this.enqueue(job.data, job.priority), delay);
      } else {
        console.error(`Job ${job.id} ล้มเหลวหลัง ${job.maxAttempts} attempts`);
      }
    } finally {
      this.workers--;
      this.processNext(); // ทำ job ถัดไป
    }
  }

  getStats(): { queued: number; processing: number } {
    return { queued: this.queue.length, processing: this.workers };
  }
}
```

---

## 8. Feature Flags

### 8.1 Feature Flag Service

```typescript
// src/features/feature-flag.service.ts
export type FeatureFlagValue = boolean | string | number | Record<string, unknown>;

export interface FeatureFlag {
  key: string;
  enabled: boolean;
  value?: FeatureFlagValue;
  percentage?: number;    // A/B testing: เปิดให้ N% ของ users
  userAllowlist?: string[]; // เปิดเฉพาะ users เหล่านี้
  userBlocklist?: string[]; // ปิดสำหรับ users เหล่านี้
  startDate?: Date;       // เปิดตั้งแต่วันนี้
  endDate?: Date;         // ปิดหลังวันนี้
}

export class FeatureFlagService {
  private flags = new Map<string, FeatureFlag>();
  private readonly cache: Map<string, { value: boolean; expiresAt: number }> = new Map();

  async isEnabled(flagKey: string, userId?: string): Promise<boolean> {
    const cacheKey = `${flagKey}:${userId ?? 'global'}`;
    const cached = this.cache.get(cacheKey);
    
    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }

    const flag = await this.getFlag(flagKey);
    if (!flag) return false;

    const result = this.evaluateFlag(flag, userId);
    
    // Cache result สำหรับ 60 วินาที
    this.cache.set(cacheKey, {
      value: result,
      expiresAt: Date.now() + 60000,
    });

    return result;
  }

  private evaluateFlag(flag: FeatureFlag, userId?: string): boolean {
    if (!flag.enabled) return false;

    const now = new Date();
    if (flag.startDate && now < flag.startDate) return false;
    if (flag.endDate && now > flag.endDate) return false;

    if (userId) {
      if (flag.userBlocklist?.includes(userId)) return false;
      if (flag.userAllowlist?.includes(userId)) return true;
    }

    // Percentage rollout
    if (flag.percentage !== undefined && userId) {
      const hash = this.hashUser(userId + flagKey);
      return (hash % 100) < flag.percentage;
    }

    return flag.enabled;
  }

  private hashUser(input: string): number {
    let hash = 0;
    for (let i = 0; i < input.length; i++) {
      hash = ((hash << 5) - hash) + input.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  private async getFlag(key: string): Promise<FeatureFlag | undefined> {
    return this.flags.get(key);
  }

  setFlag(flag: FeatureFlag): void {
    this.flags.set(flag.key, flag);
    // Clear cache for this flag
    for (const key of this.cache.keys()) {
      if (key.startsWith(flag.key + ':')) {
        this.cache.delete(key);
      }
    }
  }

  // ใช้กับ decorator
  flag(flagKey: string) {
    return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
      const originalMethod = descriptor.value;
      const flagService = this;

      descriptor.value = async function (...args: unknown[]) {
        const isEnabled = await flagService.isEnabled(flagKey);
        if (!isEnabled) {
          throw new Error(`Feature "${flagKey}" ยังไม่เปิดใช้งาน`);
        }
        return originalMethod.apply(this, args);
      };

      return descriptor;
    };
  }
}

// ใช้งาน
const featureFlags = new FeatureFlagService();

featureFlags.setFlag({
  key: 'new-transfer-ui',
  enabled: true,
  percentage: 10, // เปิดให้ 10% ของ users ก่อน
});

featureFlags.setFlag({
  key: 'instant-transfer',
  enabled: true,
  userAllowlist: ['user-vip-001', 'user-vip-002'],
});

// ตรวจสอบก่อนใช้งาน
async function getTransferUI(userId: string): Promise<string> {
  const useNewUI = await featureFlags.isEnabled('new-transfer-ui', userId);
  return useNewUI ? 'new-ui' : 'old-ui';
}
```

---

## 9. Health Check Endpoints

### 9.1 Comprehensive Health Check

```typescript
// src/health/health.service.ts
export type HealthStatus = 'healthy' | 'degraded' | 'unhealthy';

export interface HealthCheckResult {
  name: string;
  status: HealthStatus;
  responseTime: number;
  message?: string;
  metadata?: Record<string, unknown>;
}

export interface HealthReport {
  status: HealthStatus;
  timestamp: string;
  version: string;
  uptime: number;
  checks: HealthCheckResult[];
}

export class HealthService {
  private checks: Map<string, () => Promise<HealthCheckResult>> = new Map();

  registerCheck(
    name: string,
    check: () => Promise<HealthCheckResult>
  ): void {
    this.checks.set(name, check);
  }

  async runAll(): Promise<HealthReport> {
    const results = await Promise.allSettled(
      Array.from(this.checks.entries()).map(async ([name, check]) => {
        const start = Date.now();
        try {
          return await check();
        } catch (error) {
          return {
            name,
            status: 'unhealthy' as HealthStatus,
            responseTime: Date.now() - start,
            message: (error as Error).message,
          };
        }
      })
    );

    const checks = results.map(r => 
      r.status === 'fulfilled' ? r.value : {
        name: 'unknown',
        status: 'unhealthy' as HealthStatus,
        responseTime: 0,
        message: 'Check ล้มเหลว',
      }
    );

    const overallStatus: HealthStatus = 
      checks.some(c => c.status === 'unhealthy') ? 'unhealthy' :
      checks.some(c => c.status === 'degraded') ? 'degraded' : 'healthy';

    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION ?? '1.0.0',
      uptime: Math.floor(process.uptime()),
      checks,
    };
  }
}

// Register checks
export function setupHealthChecks(
  healthService: HealthService,
  dbPool: DatabasePool,
  redis: Redis
): void {
  // Database check
  healthService.registerCheck('database', async () => {
    const start = Date.now();
    const healthy = await dbPool.healthCheck();
    const stats = dbPool.getPoolStats();
    
    return {
      name: 'database',
      status: healthy ? 'healthy' : 'unhealthy',
      responseTime: Date.now() - start,
      metadata: stats,
    };
  });

  // Redis check
  healthService.registerCheck('redis', async () => {
    const start = Date.now();
    try {
      await redis.ping();
      const info = await redis.info('server');
      return {
        name: 'redis',
        status: 'healthy',
        responseTime: Date.now() - start,
      };
    } catch (error) {
      return {
        name: 'redis',
        status: 'unhealthy',
        responseTime: Date.now() - start,
        message: (error as Error).message,
      };
    }
  });

  // Memory check
  healthService.registerCheck('memory', async () => {
    const used = process.memoryUsage();
    const heapUsedPercent = (used.heapUsed / used.heapTotal) * 100;
    
    return {
      name: 'memory',
      status: heapUsedPercent > 90 ? 'unhealthy' :
              heapUsedPercent > 75 ? 'degraded' : 'healthy',
      responseTime: 0,
      metadata: {
        heapUsedMB: Math.round(used.heapUsed / 1024 / 1024),
        heapTotalMB: Math.round(used.heapTotal / 1024 / 1024),
        heapUsedPercent: Math.round(heapUsedPercent),
        rssMB: Math.round(used.rss / 1024 / 1024),
      },
    };
  });
}

// Express route
import express from 'express';

function createHealthRouter(healthService: HealthService) {
  const router = express.Router();

  // Liveness probe: process ยังมีชีวิตอยู่
  router.get('/live', (req, res) => {
    res.status(200).json({ status: 'ok' });
  });

  // Readiness probe: พร้อมรับ traffic
  router.get('/ready', async (req, res) => {
    const report = await healthService.runAll();
    const httpStatus = report.status === 'unhealthy' ? 503 : 200;
    res.status(httpStatus).json(report);
  });

  // Full health report
  router.get('/health', async (req, res) => {
    const report = await healthService.runAll();
    res.json(report);
  });

  return router;
}
```

---

## 10. Graceful Shutdown

### 10.1 Graceful Shutdown Manager

```typescript
// src/lifecycle/graceful-shutdown.ts
type ShutdownHook = () => Promise<void>;

export class GracefulShutdown {
  private hooks: Array<{ name: string; fn: ShutdownHook; order: number }> = [];
  private isShuttingDown = false;
  private readonly shutdownTimeout: number;

  constructor(timeoutMs = 30000) {
    this.shutdownTimeout = timeoutMs;
    this.setupSignalHandlers();
  }

  // Register shutdown hook
  register(name: string, fn: ShutdownHook, order = 100): void {
    this.hooks.push({ name, fn, order });
    this.hooks.sort((a, b) => a.order - b.order);
  }

  private setupSignalHandlers(): void {
    const signals = ['SIGTERM', 'SIGINT', 'SIGUSR2'] as const;
    
    signals.forEach(signal => {
      process.once(signal, async () => {
        console.log(`\nได้รับสัญญาณ ${signal} กำลัง shutdown...`);
        await this.shutdown();
      });
    });

    process.once('unhandledRejection', async (reason) => {
      console.error('Unhandled Promise Rejection:', reason);
      await this.shutdown(1);
    });

    process.once('uncaughtException', async (err) => {
      console.error('Uncaught Exception:', err);
      await this.shutdown(1);
    });
  }

  async shutdown(exitCode = 0): Promise<never> {
    if (this.isShuttingDown) {
      console.log('Shutdown ดำเนินการอยู่แล้ว...');
      process.exit(exitCode);
    }

    this.isShuttingDown = true;
    console.log('เริ่ม graceful shutdown...');

    // Force exit หลัง timeout
    const forceExit = setTimeout(() => {
      console.error(`Graceful shutdown ไม่เสร็จใน ${this.shutdownTimeout}ms, force exit`);
      process.exit(1);
    }, this.shutdownTimeout);
    forceExit.unref();

    // ทำ hooks ตาม order
    for (const hook of this.hooks) {
      console.log(`กำลัง: ${hook.name}...`);
      try {
        await hook.fn();
        console.log(`✓ ${hook.name}`);
      } catch (error) {
        console.error(`✗ ${hook.name}:`, (error as Error).message);
      }
    }

    console.log('Shutdown เสร็จสิ้น');
    process.exit(exitCode);
  }
}

// ใช้งาน
const shutdown = new GracefulShutdown(30000);

// Register hooks ตาม order (น้อย = ทำก่อน)
shutdown.register('stop-accepting-requests', async () => {
  // หยุดรับ request ใหม่ แต่ยังประมวลผล request ที่มีอยู่
  httpServer.close();
}, 10);

shutdown.register('drain-job-queue', async () => {
  // รอให้ job queue ทำงานที่ active ให้เสร็จ
  await jobQueue.close();
}, 20);

shutdown.register('close-database', async () => {
  await dbPool.close();
}, 30);

shutdown.register('close-redis', async () => {
  await redis.quit();
}, 40);
```

---

## 11. Load Shedding

### 11.1 Load Shedder

```typescript
// src/middleware/load-shedder.ts
export class LoadShedder {
  private requestQueue: Array<{
    resolve: () => void;
    reject: (err: Error) => void;
    priority: number;
    enqueuedAt: number;
  }> = [];

  private activeRequests = 0;
  private readonly maxQueue = 1000;

  constructor(
    private readonly options: {
      maxConcurrent: number;    // request พร้อมกันสูงสุด
      maxQueueSize: number;     // queue สูงสุด
      requestTimeout: number;   // timeout สำหรับ request ใน queue
    }
  ) {}

  async acquire(priority = 5): Promise<void> {
    if (this.activeRequests < this.options.maxConcurrent) {
      this.activeRequests++;
      return;
    }

    if (this.requestQueue.length >= this.options.maxQueueSize) {
      throw new Error('ระบบมีโหลดสูงเกินไป กรุณาลองใหม่ภายหลัง');
    }

    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        const index = this.requestQueue.findIndex(r => r.resolve === resolve);
        if (index !== -1) {
          this.requestQueue.splice(index, 1);
        }
        reject(new Error('Request timeout ขณะรอคิว'));
      }, this.options.requestTimeout);

      this.requestQueue.push({
        resolve: () => {
          clearTimeout(timeout);
          resolve();
        },
        reject,
        priority,
        enqueuedAt: Date.now(),
      });

      // Sort by priority
      this.requestQueue.sort((a, b) => b.priority - a.priority);
    });
  }

  release(): void {
    this.activeRequests--;
    
    if (this.requestQueue.length > 0) {
      const next = this.requestQueue.shift()!;
      this.activeRequests++;
      next.resolve();
    }
  }

  // Express middleware
  middleware() {
    return async (req: any, res: any, next: any) => {
      const priority = this.getRequestPriority(req);
      
      try {
        await this.acquire(priority);
        
        res.on('finish', () => this.release());
        res.on('close', () => this.release());
        
        next();
      } catch (error) {
        res.status(503).json({
          error: 'Service Unavailable',
          message: (error as Error).message,
          retryAfter: 5,
        });
      }
    };
  }

  private getRequestPriority(req: any): number {
    // Premium users ได้ priority สูงกว่า
    if (req.user?.tier === 'premium') return 10;
    if (req.user?.tier === 'business') return 8;
    if (req.path.startsWith('/health')) return 15; // Health checks ทำเสมอ
    return 5;
  }

  getStats(): {
    activeRequests: number;
    queuedRequests: number;
    utilizationPercent: number;
  } {
    return {
      activeRequests: this.activeRequests,
      queuedRequests: this.requestQueue.length,
      utilizationPercent: (this.activeRequests / this.options.maxConcurrent) * 100,
    };
  }
}
```

---

## 12. Circuit Breaker Pattern

### 12.1 Circuit Breaker Implementation

```typescript
// src/resilience/circuit-breaker.ts
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

export class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime?: number;
  private readonly name: string;

  constructor(
    name: string,
    private readonly options: {
      failureThreshold: number;     // failures ก่อน open
      successThreshold: number;     // successes ใน half-open ก่อน close
      timeout: number;              // ms ก่อน ลอง half-open
      resetTimeout?: number;        // ms ก่อน reset counters
    }
  ) {
    this.name = name;
  }

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (this.shouldAttemptReset()) {
        this.state = 'HALF_OPEN';
        console.log(`Circuit Breaker "${this.name}": OPEN → HALF_OPEN`);
      } else {
        throw new Error(`Circuit Breaker "${this.name}" is OPEN`);
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.successCount = 0;
        console.log(`Circuit Breaker "${this.name}": HALF_OPEN → CLOSED`);
      }
    } else {
      this.failureCount = 0;
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
      this.successCount = 0;
      console.log(`Circuit Breaker "${this.name}": HALF_OPEN → OPEN (ยังไม่พร้อม)`);
    } else if (this.failureCount >= this.options.failureThreshold) {
      this.state = 'OPEN';
      console.log(
        `Circuit Breaker "${this.name}": CLOSED → OPEN ` +
        `(${this.failureCount} failures)`
      );
    }
  }

  private shouldAttemptReset(): boolean {
    return this.lastFailureTime !== undefined &&
      Date.now() - this.lastFailureTime >= this.options.timeout;
  }

  getState(): CircuitState {
    return this.state;
  }

  getMetrics(): {
    state: CircuitState;
    failureCount: number;
    successCount: number;
  } {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
    };
  }
}

// ใช้งาน
const externalPaymentCB = new CircuitBreaker('external-payment', {
  failureThreshold: 5,
  successThreshold: 3,
  timeout: 60000, // 1 นาที
});

async function processExternalPayment(data: unknown): Promise<unknown> {
  return externalPaymentCB.execute(async () => {
    // เรียก external payment API
    const response = await fetch('https://payment.example.com/api', {
      method: 'POST',
      body: JSON.stringify(data),
    });
    
    if (!response.ok) {
      throw new Error(`Payment API error: ${response.status}`);
    }
    
    return response.json();
  });
}
```

---

## สรุปบทที่ 90

ในบทนี้เราได้เรียนรู้ Scalability Patterns ที่สำคัญ:

1. **Horizontal vs Vertical Scaling** - เลือกวิธี scale ที่เหมาะสม
2. **Stateless Service Design** - ออกแบบ service ที่ scale ออกได้ง่าย
3. **Node.js Cluster Module** - ใช้ CPU หลาย cores พร้อมกัน
4. **Worker Threads** - จัดการ CPU-intensive tasks
5. **Connection Pooling** - ใช้ database/Redis connections อย่างมีประสิทธิภาพ
6. **Rate Limiting** - Sliding Window และ Token Bucket algorithms
7. **Backpressure** - จัดการ stream ที่มีข้อมูลมากเกินไป
8. **Job Queue** - Async processing พร้อม priority
9. **Feature Flags** - Rollout features อย่างปลอดภัย
10. **Health Checks** - Liveness และ Readiness probes
11. **Graceful Shutdown** - ปิด service โดยไม่สูญเสียข้อมูล
12. **Load Shedding** - ปฏิเสธ request เมื่อโหลดสูงเกินไป
13. **Circuit Breaker** - ป้องกัน cascade failure

---

## จบหลักสูตร TypeScript

ตลอด 90 บทที่ผ่านมา เราได้เรียนรู้ TypeScript อย่างครบถ้วนตั้งแต่พื้นฐานจนถึงการใช้งานจริงในระดับ Production รวมถึง:

- **Core TypeScript** - Types, Generics, Decorators
- **Design Patterns** - SOLID, Factory, Observer, Strategy
- **Architecture** - DDD, CQRS, Event Sourcing, Microservices
- **Testing** - Unit, Integration, E2E
- **Real-world Projects** - FinTech, E-commerce
- **Advanced Topics** - gRPC, Message Queues, Caching, Scalability
