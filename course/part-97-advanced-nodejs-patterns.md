# Part 97: Advanced Node.js Patterns กับ TypeScript

## บทนำ

Node.js เป็น runtime ที่ทรงพลังสำหรับการพัฒนา server-side JavaScript/TypeScript การเข้าใจ internals ของ Node.js ช่วยให้เราเขียนโค้ดที่มีประสิทธิภาพสูง บทนี้จะครอบคลุม event loop, streams, worker threads, child processes, และ memory management

---

## 97.1 Event Loop Deep Dive

Event loop คือหัวใจของ Node.js ที่ทำให้ non-blocking I/O เป็นไปได้

```typescript
// src/event-loop/understanding-phases.ts

/**
 * Event Loop มี 6 phases:
 * 1. timers - setTimeout, setInterval callbacks
 * 2. pending callbacks - I/O callbacks deferred to next iteration
 * 3. idle, prepare - internal use only
 * 4. poll - retrieve new I/O events
 * 5. check - setImmediate callbacks
 * 6. close callbacks - close events
 */

// ทดสอบ order ของ callbacks
function demonstrateEventLoop(): void {
  console.log('1. Synchronous code starts');

  // Timer phase
  setTimeout(() => console.log('4. setTimeout (timer phase)'), 0);

  // Check phase
  setImmediate(() => console.log('5. setImmediate (check phase)'));

  // Microtasks (before each phase)
  Promise.resolve().then(() => console.log('3. Promise microtask'));
  
  // queueMicrotask (like Promise but more direct)
  queueMicrotask(() => console.log('3b. queueMicrotask'));

  process.nextTick(() => console.log('2. process.nextTick (highest priority)'));

  console.log('1b. More synchronous code');

  /**
   * Expected output:
   * 1. Synchronous code starts
   * 1b. More synchronous code
   * 2. process.nextTick (highest priority)
   * 3. Promise microtask
   * 3b. queueMicrotask
   * 4. setTimeout (timer phase)
   * 5. setImmediate (check phase)
   */
}

demonstrateEventLoop();
```

```typescript
// src/event-loop/blocking-analysis.ts

// ตัวอย่าง blocking vs non-blocking
class EventLoopMonitor {
  private lastCheck: bigint = process.hrtime.bigint();
  private interval: NodeJS.Timeout | null = null;

  start(checkIntervalMs = 100): void {
    this.lastCheck = process.hrtime.bigint();
    
    this.interval = setInterval(() => {
      const now = process.hrtime.bigint();
      const elapsed = Number(now - this.lastCheck) / 1_000_000; // convert to ms
      
      if (elapsed > checkIntervalMs * 1.5) {
        console.warn(`⚠️ Event loop lag detected: ${elapsed.toFixed(2)}ms (expected ${checkIntervalMs}ms)`);
      }
      
      this.lastCheck = now;
    }, checkIntervalMs);
    
    this.interval.unref(); // Don't prevent process from exiting
  }

  stop(): void {
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
    }
  }
}

// Blocking operation - BAD
function blockingOperation(ms: number): void {
  const start = Date.now();
  while (Date.now() - start < ms) {
    // Busy waiting - blocks event loop!
  }
}

// Non-blocking alternative
function nonBlockingDelay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// CPU-intensive work ที่ไม่ block event loop
async function cpuIntensiveNonBlocking(
  data: number[],
  chunkSize = 1000
): Promise<number> {
  let sum = 0;
  
  for (let i = 0; i < data.length; i += chunkSize) {
    const chunk = data.slice(i, i + chunkSize);
    sum += chunk.reduce((a, b) => a + b, 0);
    
    // ยอมให้ event loop ทำงาน
    await new Promise<void>((resolve) => setImmediate(resolve));
  }
  
  return sum;
}

async function demo() {
  const monitor = new EventLoopMonitor();
  monitor.start();

  const largeData = Array.from({ length: 1_000_000 }, (_, i) => i);
  
  console.log('Starting non-blocking computation...');
  const result = await cpuIntensiveNonBlocking(largeData);
  console.log('Sum:', result);
  
  monitor.stop();
}
```

---

## 97.2 libuv และ Async I/O

```typescript
// src/libuv/thread-pool.ts

import * as fs from 'fs';
import * as crypto from 'crypto';
import * as os from 'os';

/**
 * libuv thread pool ใช้สำหรับ:
 * - File system operations
 * - DNS lookups  
 * - Crypto operations
 * - User-defined C++ addons
 *
 * Default size: 4 threads (UV_THREADPOOL_SIZE env var)
 */

// ทดสอบ thread pool saturation
async function testThreadPoolSaturation(): Promise<void> {
  const NUM_OPERATIONS = 8; // เกิน default pool size ของ 4
  
  console.log(`CPU cores: ${os.cpus().length}`);
  console.log(`Thread pool size: ${process.env.UV_THREADPOOL_SIZE ?? 4}`);
  
  const startTime = Date.now();
  
  // ทำ crypto operations พร้อมกัน (ใช้ thread pool)
  const promises = Array.from({ length: NUM_OPERATIONS }, (_, i) =>
    new Promise<void>((resolve) => {
      crypto.pbkdf2(
        `password${i}`,
        `salt${i}`,
        100000,
        64,
        'sha512',
        () => {
          console.log(`Operation ${i + 1} done at ${Date.now() - startTime}ms`);
          resolve();
        }
      );
    })
  );

  await Promise.all(promises);
  console.log(`Total time: ${Date.now() - startTime}ms`);
}

// เพิ่ม thread pool size
function increaseThreadPool(): void {
  // ต้องตั้งก่อน require libuv
  process.env.UV_THREADPOOL_SIZE = String(os.cpus().length * 2);
  console.log('Thread pool size set to:', process.env.UV_THREADPOOL_SIZE);
}

// I/O operations
async function demonstrateAsyncIO(): Promise<void> {
  const tmpFile = '/tmp/test-io.txt';
  
  // Non-blocking file write
  await fs.promises.writeFile(tmpFile, 'Hello from async I/O!');
  
  // Non-blocking file read
  const content = await fs.promises.readFile(tmpFile, 'utf-8');
  console.log('File content:', content);
  
  // File stats
  const stats = await fs.promises.stat(tmpFile);
  console.log('File size:', stats.size, 'bytes');
  
  // Cleanup
  await fs.promises.unlink(tmpFile);
}
```

---

## 97.3 Streams กับ TypeScript

```typescript
// src/streams/readable-stream.ts
import { Readable, ReadableOptions } from 'stream';

// Custom Readable Stream
class NumberStream extends Readable {
  private current: number;
  private max: number;

  constructor(
    private start: number,
    max: number,
    options?: ReadableOptions
  ) {
    super({ ...options, objectMode: true });
    this.current = start;
    this.max = max;
  }

  override _read(): void {
    if (this.current <= this.max) {
      this.push(this.current);
      this.current++;
    } else {
      this.push(null); // Signal end of stream
    }
  }
}

// Async Generator Readable Stream
function createAsyncGeneratorStream<T>(
  generator: AsyncGenerator<T>
): Readable {
  return new Readable({
    objectMode: true,
    async read() {
      const { value, done } = await generator.next();
      if (done) {
        this.push(null);
      } else {
        this.push(value);
      }
    },
  });
}

// ตัวอย่าง async generator
async function* databaseQueryGenerator(
  batchSize = 100
): AsyncGenerator<Record<string, unknown>[]> {
  // Mock database query
  let offset = 0;
  let total = 500; // total records
  
  while (offset < total) {
    const batch = Array.from({ length: Math.min(batchSize, total - offset) }, (_, i) => ({
      id: offset + i + 1,
      name: `Record ${offset + i + 1}`,
      value: Math.random(),
    }));
    
    yield batch;
    offset += batchSize;
    
    // Simulate async query
    await new Promise((resolve) => setTimeout(resolve, 10));
  }
}

// ใช้งาน
async function readableDemo(): Promise<void> {
  // Number stream
  const numbers = new NumberStream(1, 10);
  const collected: number[] = [];
  
  for await (const num of numbers) {
    collected.push(num as number);
  }
  console.log('Numbers:', collected);

  // Database stream
  const dbStream = createAsyncGeneratorStream(databaseQueryGenerator(100));
  let totalRecords = 0;
  
  for await (const batch of dbStream) {
    totalRecords += (batch as unknown[]).length;
    process.stdout.write(`\rLoaded ${totalRecords} records`);
  }
  
  console.log('\nDone!');
}
```

```typescript
// src/streams/writable-stream.ts
import { Writable, WritableOptions } from 'stream';
import * as fs from 'fs';

// Custom Writable Stream ที่ buffer ข้อมูล
class BufferedWritable extends Writable {
  private buffer: Buffer[] = [];
  private bufferSize = 0;
  private readonly flushSize: number;
  private flushCount = 0;

  constructor(
    private outputPath: string,
    options: WritableOptions & { flushSize?: number } = {}
  ) {
    super(options);
    this.flushSize = options.flushSize ?? 1024 * 64; // 64KB default
  }

  override _write(
    chunk: Buffer | string,
    _encoding: BufferEncoding,
    callback: (error?: Error | null) => void
  ): void {
    const buffer = Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk);
    this.buffer.push(buffer);
    this.bufferSize += buffer.length;

    if (this.bufferSize >= this.flushSize) {
      this.flush(callback);
    } else {
      callback();
    }
  }

  private flush(callback: (error?: Error | null) => void): void {
    const combined = Buffer.concat(this.buffer);
    this.buffer = [];
    this.bufferSize = 0;
    this.flushCount++;

    fs.appendFile(this.outputPath, combined, callback);
    console.log(`Flushed chunk #${this.flushCount} (${combined.length} bytes)`);
  }

  override _final(callback: (error?: Error | null) => void): void {
    if (this.buffer.length > 0) {
      this.flush(callback);
    } else {
      callback();
    }
  }
}

// Database-style writable
interface DatabaseRecord {
  id: number;
  data: unknown;
}

class DatabaseWritable extends Writable {
  private insertCount = 0;
  private records: DatabaseRecord[] = [];

  constructor() {
    super({ objectMode: true });
  }

  override _write(
    record: DatabaseRecord,
    _encoding: BufferEncoding,
    callback: (error?: Error | null) => void
  ): void {
    this.records.push(record);
    this.insertCount++;

    if (this.insertCount % 100 === 0) {
      console.log(`Inserted ${this.insertCount} records`);
    }

    // Simulate async DB insert
    setImmediate(callback);
  }

  getRecords(): DatabaseRecord[] {
    return this.records;
  }
}
```

```typescript
// src/streams/transform-stream.ts
import { Transform, TransformOptions } from 'stream';

// CSV Parser Transform Stream
class CSVParserTransform extends Transform {
  private headers: string[] | null = null;
  private lineBuffer = '';
  private separator: string;

  constructor(options: TransformOptions & { separator?: string } = {}) {
    super({ ...options, objectMode: true });
    this.separator = options.separator ?? ',';
  }

  override _transform(
    chunk: Buffer | string,
    _encoding: BufferEncoding,
    callback: (error?: Error | null, data?: unknown) => void
  ): void {
    const text = chunk.toString();
    this.lineBuffer += text;

    const lines = this.lineBuffer.split('\n');
    this.lineBuffer = lines.pop() ?? '';

    for (const line of lines) {
      if (line.trim() === '') continue;

      const values = this.parseLine(line);

      if (!this.headers) {
        this.headers = values;
      } else {
        const record: Record<string, string> = {};
        this.headers.forEach((header, i) => {
          record[header] = values[i] ?? '';
        });
        this.push(record);
      }
    }

    callback();
  }

  override _flush(callback: (error?: Error | null) => void): void {
    if (this.lineBuffer.trim()) {
      const values = this.parseLine(this.lineBuffer);
      if (this.headers) {
        const record: Record<string, string> = {};
        this.headers.forEach((header, i) => {
          record[header] = values[i] ?? '';
        });
        this.push(record);
      }
    }
    callback();
  }

  private parseLine(line: string): string[] {
    const result: string[] = [];
    let current = '';
    let inQuotes = false;

    for (const char of line) {
      if (char === '"') {
        inQuotes = !inQuotes;
      } else if (char === this.separator && !inQuotes) {
        result.push(current.trim());
        current = '';
      } else {
        current += char;
      }
    }

    result.push(current.trim());
    return result;
  }
}

// JSON Stringify Transform Stream
class JSONStringifyTransform extends Transform {
  constructor() {
    super({ objectMode: true });
  }

  override _transform(
    chunk: unknown,
    _encoding: BufferEncoding,
    callback: (error?: Error | null, data?: unknown) => void
  ): void {
    this.push(JSON.stringify(chunk) + '\n');
    callback();
  }
}

// Compression Transform Stream (without external deps)
class GZipTransform extends Transform {
  constructor() {
    super();
  }
  
  override _transform(
    chunk: Buffer,
    _encoding: BufferEncoding,
    callback: (error?: Error | null, data?: unknown) => void
  ): void {
    // In production, use zlib.createGzip()
    // This is a mock that just passes through
    this.push(chunk);
    callback();
  }
}

// ตัวอย่าง pipeline
import { pipeline } from 'stream/promises';
import * as fs from 'fs';

async function csvProcessingPipeline(
  inputFile: string,
  outputFile: string
): Promise<void> {
  await pipeline(
    fs.createReadStream(inputFile),
    new CSVParserTransform(),
    new JSONStringifyTransform(),
    fs.createWriteStream(outputFile)
  );
  
  console.log('Pipeline complete!');
}
```

---

## 97.4 Duplex Streams

```typescript
// src/streams/duplex-stream.ts
import { Duplex } from 'stream';
import * as net from 'net';

// Custom Duplex Stream สำหรับ protocol handling
class ProtocolStream extends Duplex {
  private messageBuffer = Buffer.alloc(0);
  private readonly HEADER_SIZE = 4; // 4 bytes for message length

  constructor() {
    super({ allowHalfOpen: false });
  }

  // Readable side
  override _read(_size: number): void {
    // Data is pushed via pushMessage()
  }

  // Writable side
  override _write(
    chunk: Buffer,
    _encoding: BufferEncoding,
    callback: (error?: Error | null) => void
  ): void {
    this.messageBuffer = Buffer.concat([this.messageBuffer, chunk]);
    this.processBuffer();
    callback();
  }

  private processBuffer(): void {
    while (this.messageBuffer.length >= this.HEADER_SIZE) {
      const messageLength = this.messageBuffer.readUInt32BE(0);
      
      if (this.messageBuffer.length < this.HEADER_SIZE + messageLength) {
        break; // Wait for more data
      }

      const message = this.messageBuffer.slice(
        this.HEADER_SIZE,
        this.HEADER_SIZE + messageLength
      );
      
      this.messageBuffer = this.messageBuffer.slice(
        this.HEADER_SIZE + messageLength
      );

      try {
        const parsed = JSON.parse(message.toString()) as unknown;
        this.push(parsed);
      } catch {
        this.emit('error', new Error('Invalid message format'));
      }
    }
  }

  // Send a message
  sendMessage(data: unknown): boolean {
    const messageStr = JSON.stringify(data);
    const messageBuffer = Buffer.from(messageStr);
    
    const header = Buffer.alloc(this.HEADER_SIZE);
    header.writeUInt32BE(messageBuffer.length, 0);
    
    return this.write(Buffer.concat([header, messageBuffer]));
  }
}

// TCP Server กับ TypeScript Types
interface ServerMessage {
  type: 'ping' | 'pong' | 'data' | 'error';
  payload?: unknown;
  timestamp: number;
}

function createMessageServer(port: number): net.Server {
  const server = net.createServer((socket) => {
    const protocol = new ProtocolStream();
    
    socket.pipe(protocol);
    
    protocol.on('data', (message: ServerMessage) => {
      console.log('Received:', message);
      
      if (message.type === 'ping') {
        protocol.sendMessage({
          type: 'pong',
          timestamp: Date.now(),
        } as ServerMessage);
      }
    });

    protocol.on('error', (err) => {
      console.error('Protocol error:', err);
      socket.destroy();
    });

    socket.on('error', (err) => {
      console.error('Socket error:', err);
    });
  });

  server.listen(port, () => {
    console.log(`Server listening on port ${port}`);
  });

  return server;
}
```

---

## 97.5 Back-pressure Handling

```typescript
// src/streams/backpressure.ts
import { Readable, Writable } from 'stream';

// Producer ที่เร็วกว่า Consumer
class FastProducer extends Readable {
  private counter = 0;
  private producedCount = 0;
  private pauseCount = 0;

  constructor(private totalItems: number) {
    super({ objectMode: true, highWaterMark: 16 });
  }

  override _read(): void {
    while (this.counter < this.totalItems) {
      const data = { id: this.counter, timestamp: Date.now() };
      this.counter++;
      this.producedCount++;

      // push() returns false when buffer is full (back-pressure)
      const canContinue = this.push(data);
      
      if (!canContinue) {
        this.pauseCount++;
        console.log(`Back-pressure applied at item ${this.counter} (paused ${this.pauseCount} times)`);
        return; // Stop producing until _read() is called again
      }
    }
    
    this.push(null);
    console.log(`Producer finished. Total produced: ${this.producedCount}, Pauses: ${this.pauseCount}`);
  }
}

// Slow Consumer
class SlowConsumer extends Writable {
  private consumedCount = 0;

  constructor(private delayMs: number) {
    super({ objectMode: true, highWaterMark: 4 });
  }

  override _write(
    chunk: unknown,
    _encoding: BufferEncoding,
    callback: (error?: Error | null) => void
  ): void {
    this.consumedCount++;
    
    // Simulate slow processing
    setTimeout(() => {
      process.stdout.write(`\rConsumed: ${this.consumedCount} items`);
      callback();
    }, this.delayMs);
  }
}

// Manual back-pressure implementation
class BackPressureController {
  private isPaused = false;
  private processQueue: Array<() => Promise<void>> = [];
  private processing = false;

  constructor(
    private readonly maxQueueSize: number,
    private readonly processItem: (item: unknown) => Promise<void>
  ) {}

  async push(item: unknown): Promise<void> {
    if (this.processQueue.length >= this.maxQueueSize) {
      // Apply back-pressure - wait until there's space
      await new Promise<void>((resolve) => {
        const checkSpace = setInterval(() => {
          if (this.processQueue.length < this.maxQueueSize) {
            clearInterval(checkSpace);
            resolve();
          }
        }, 10);
      });
    }

    this.processQueue.push(() => this.processItem(item));
    
    if (!this.processing) {
      void this.drainQueue();
    }
  }

  private async drainQueue(): Promise<void> {
    this.processing = true;
    
    while (this.processQueue.length > 0) {
      const task = this.processQueue.shift();
      if (task) {
        await task();
      }
    }
    
    this.processing = false;
  }
}

async function backpressureDemo(): Promise<void> {
  console.log('Demonstrating back-pressure...\n');
  
  const producer = new FastProducer(100);
  const consumer = new SlowConsumer(50); // 50ms per item

  await new Promise<void>((resolve, reject) => {
    producer.pipe(consumer);
    consumer.on('finish', resolve);
    consumer.on('error', reject);
  });
  
  console.log('\nBack-pressure demo complete!');
}
```

---

## 97.6 Cluster Module กับ TypeScript

```typescript
// src/cluster/cluster-server.ts
import cluster from 'cluster';
import * as http from 'http';
import * as os from 'os';
import process from 'process';

interface WorkerMessage {
  type: 'stats' | 'shutdown';
  workerId?: number;
  requestCount?: number;
}

interface MasterMessage {
  type: 'ping' | 'restart';
}

// Master process
function startMaster(): void {
  const numCPUs = os.cpus().length;
  console.log(`Master ${process.pid} is running`);
  console.log(`Starting ${numCPUs} workers...`);

  const workerStats = new Map<number, { requestCount: number; pid: number }>();

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    const worker = cluster.fork();
    workerStats.set(worker.id, { requestCount: 0, pid: worker.process.pid ?? 0 });
    
    worker.on('message', (message: WorkerMessage) => {
      if (message.type === 'stats' && message.workerId !== undefined) {
        const stats = workerStats.get(message.workerId);
        if (stats) {
          stats.requestCount = message.requestCount ?? 0;
        }
      }
    });
  }

  // Handle worker exit
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    workerStats.delete(worker.id);
    
    const newWorker = cluster.fork();
    workerStats.set(newWorker.id, { requestCount: 0, pid: newWorker.process.pid ?? 0 });
  });

  // Print stats every 10 seconds
  setInterval(() => {
    console.log('\n=== Worker Stats ===');
    for (const [id, stats] of workerStats) {
      console.log(`Worker ${id} (PID: ${stats.pid}): ${stats.requestCount} requests`);
    }
  }, 10000);
}

// Worker process
function startWorker(): void {
  let requestCount = 0;

  const server = http.createServer((req, res) => {
    requestCount++;
    
    // Report stats to master
    if (process.send && requestCount % 100 === 0) {
      const message: WorkerMessage = {
        type: 'stats',
        workerId: cluster.worker?.id,
        requestCount,
      };
      process.send(message);
    }

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      workerId: cluster.worker?.id,
      pid: process.pid,
      requestCount,
    }));
  });

  server.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
  });

  // Graceful shutdown
  process.on('message', (message: MasterMessage) => {
    if (message.type === 'shutdown') {
      server.close(() => {
        process.exit(0);
      });
    }
  });
}

// Entry point
if (cluster.isPrimary) {
  startMaster();
} else {
  startWorker();
}
```

---

## 97.7 Worker Threads สำหรับ CPU-bound Tasks

```typescript
// src/workers/worker-thread.ts
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';
import * as path from 'path';

// ===== Worker Code =====
if (!isMainThread) {
  interface WorkerData {
    type: 'fibonacci' | 'primes' | 'sort';
    input: number | number[];
  }

  const data = workerData as WorkerData;

  switch (data.type) {
    case 'fibonacci': {
      const n = data.input as number;
      
      function fibonacci(n: number): bigint {
        if (n <= 1) return BigInt(n);
        let a = BigInt(0);
        let b = BigInt(1);
        for (let i = 2; i <= n; i++) {
          [a, b] = [b, a + b];
        }
        return b;
      }
      
      const result = fibonacci(n);
      parentPort?.postMessage({ result: result.toString() });
      break;
    }

    case 'primes': {
      const limit = data.input as number;
      
      function sieveOfEratosthenes(n: number): number[] {
        const sieve = new Uint8Array(n + 1).fill(1);
        sieve[0] = sieve[1] = 0;
        
        for (let i = 2; i * i <= n; i++) {
          if (sieve[i]) {
            for (let j = i * i; j <= n; j += i) {
              sieve[j] = 0;
            }
          }
        }
        
        return Array.from(sieve.entries())
          .filter(([, isPrime]) => isPrime)
          .map(([num]) => num);
      }
      
      const primes = sieveOfEratosthenes(limit);
      parentPort?.postMessage({ count: primes.length, largest: primes[primes.length - 1] });
      break;
    }

    case 'sort': {
      const arr = [...(data.input as number[])];
      arr.sort((a, b) => a - b);
      parentPort?.postMessage({ sorted: arr.slice(0, 10) }); // Return first 10
      break;
    }
  }
}
```

```typescript
// src/workers/worker-pool.ts
import { Worker } from 'worker_threads';
import * as path from 'path';

interface WorkerTask<TInput, TOutput> {
  data: TInput;
  resolve: (result: TOutput) => void;
  reject: (error: Error) => void;
}

interface WorkerState {
  worker: Worker;
  busy: boolean;
}

export class WorkerPool<TInput, TOutput> {
  private workers: WorkerState[] = [];
  private queue: WorkerTask<TInput, TOutput>[] = [];

  constructor(
    private workerScript: string,
    private poolSize: number
  ) {
    for (let i = 0; i < poolSize; i++) {
      this.addWorker();
    }
  }

  private addWorker(): void {
    const worker = new Worker(this.workerScript);
    const state: WorkerState = { worker, busy: false };

    worker.on('message', (result: TOutput) => {
      state.busy = false;
      const task = this.currentTasks.get(worker.threadId);
      
      if (task) {
        this.currentTasks.delete(worker.threadId);
        task.resolve(result);
      }
      
      this.processQueue();
    });

    worker.on('error', (error) => {
      state.busy = false;
      const task = this.currentTasks.get(worker.threadId);
      
      if (task) {
        this.currentTasks.delete(worker.threadId);
        task.reject(error);
      }
    });

    this.workers.push(state);
  }

  private currentTasks = new Map<number, WorkerTask<TInput, TOutput>>();

  private processQueue(): void {
    const freeWorker = this.workers.find((w) => !w.busy);
    
    if (!freeWorker || this.queue.length === 0) return;

    const task = this.queue.shift()!;
    freeWorker.busy = true;
    this.currentTasks.set(freeWorker.worker.threadId, task);
    freeWorker.worker.postMessage(task.data);
  }

  run(data: TInput): Promise<TOutput> {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.processQueue();
    });
  }

  async terminate(): Promise<void> {
    await Promise.all(this.workers.map((w) => w.worker.terminate()));
  }
}

// ตัวอย่างการใช้ Worker Pool
async function workerPoolDemo(): Promise<void> {
  const WORKER_SCRIPT = path.join(__dirname, 'worker-thread.js');
  const pool = new WorkerPool<
    { type: string; input: number },
    { result?: string; count?: number }
  >(WORKER_SCRIPT, 4);

  const tasks = Array.from({ length: 20 }, (_, i) => ({
    type: 'fibonacci' as const,
    input: 40 + i,
  }));

  console.time('Worker pool computation');
  
  const results = await Promise.all(
    tasks.map((task) => pool.run(task))
  );
  
  console.timeEnd('Worker pool computation');
  console.log('Results:', results.slice(0, 5));

  await pool.terminate();
}
```

---

## 97.8 Shared Memory กับ SharedArrayBuffer

```typescript
// src/shared-memory/shared-counter.ts
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

// Worker code
if (!isMainThread) {
  interface WorkerInit {
    sharedBuffer: SharedArrayBuffer;
    workerId: number;
    iterations: number;
  }
  
  const { sharedBuffer, workerId, iterations } = workerData as WorkerInit;
  const counter = new Int32Array(sharedBuffer);

  for (let i = 0; i < iterations; i++) {
    // Atomic increment (thread-safe)
    Atomics.add(counter, 0, 1);
  }
  
  parentPort?.postMessage({ workerId, done: true });
}

// Main thread code
async function sharedMemoryDemo(): Promise<void> {
  const NUM_WORKERS = 4;
  const ITERATIONS_PER_WORKER = 100000;

  // Create shared buffer
  const sharedBuffer = new SharedArrayBuffer(4); // 4 bytes for Int32
  const counter = new Int32Array(sharedBuffer);

  console.log('Starting workers with shared memory...');
  
  // Create workers
  const workers = Array.from({ length: NUM_WORKERS }, (_, i) => {
    return new Promise<void>((resolve, reject) => {
      const worker = new Worker(__filename, {
        workerData: {
          sharedBuffer,
          workerId: i,
          iterations: ITERATIONS_PER_WORKER,
        },
      });

      worker.on('message', () => resolve());
      worker.on('error', reject);
    });
  });

  await Promise.all(workers);
  
  const expected = NUM_WORKERS * ITERATIONS_PER_WORKER;
  const actual = Atomics.load(counter, 0);
  
  console.log(`Expected: ${expected}`);
  console.log(`Actual: ${actual}`);
  console.log(`Correct: ${expected === actual}`);
}

// Ring buffer สำหรับ inter-thread communication
class RingBuffer {
  private buffer: SharedArrayBuffer;
  private view: Int32Array;
  private data: Uint8Array;
  private readonly HEAD_OFFSET = 0;
  private readonly TAIL_OFFSET = 1;
  private readonly SIZE_OFFSET = 2;

  constructor(size: number) {
    // 3 Int32 values for head, tail, size + data area
    this.buffer = new SharedArrayBuffer(12 + size);
    this.view = new Int32Array(this.buffer, 0, 3);
    this.data = new Uint8Array(this.buffer, 12, size);
    
    Atomics.store(this.view, this.SIZE_OFFSET, size);
  }

  write(byte: number): boolean {
    const size = Atomics.load(this.view, this.SIZE_OFFSET);
    const tail = Atomics.load(this.view, this.TAIL_OFFSET);
    const head = Atomics.load(this.view, this.HEAD_OFFSET);
    
    const nextTail = (tail + 1) % size;
    
    if (nextTail === head) {
      return false; // Buffer full
    }
    
    this.data[tail] = byte;
    Atomics.store(this.view, this.TAIL_OFFSET, nextTail);
    return true;
  }

  read(): number | null {
    const size = Atomics.load(this.view, this.SIZE_OFFSET);
    const head = Atomics.load(this.view, this.HEAD_OFFSET);
    const tail = Atomics.load(this.view, this.TAIL_OFFSET);
    
    if (head === tail) {
      return null; // Buffer empty
    }
    
    const byte = this.data[head] ?? 0;
    Atomics.store(this.view, this.HEAD_OFFSET, (head + 1) % size);
    return byte;
  }

  getSharedBuffer(): SharedArrayBuffer {
    return this.buffer;
  }
}
```

---

## 97.9 Child Processes

```typescript
// src/child-process/process-manager.ts
import {
  spawn,
  exec,
  execFile,
  fork,
  ChildProcess,
  SpawnOptions,
} from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

interface ProcessResult {
  stdout: string;
  stderr: string;
  exitCode: number | null;
}

export class ProcessManager {
  // Run command และรอผลลัพธ์
  async run(
    command: string,
    args: string[] = [],
    options: SpawnOptions = {}
  ): Promise<ProcessResult> {
    return new Promise((resolve, reject) => {
      const child = spawn(command, args, {
        ...options,
        stdio: 'pipe',
      });

      let stdout = '';
      let stderr = '';

      child.stdout?.on('data', (data: Buffer) => {
        stdout += data.toString();
      });

      child.stderr?.on('data', (data: Buffer) => {
        stderr += data.toString();
      });

      child.on('close', (exitCode) => {
        if (exitCode === 0) {
          resolve({ stdout, stderr, exitCode });
        } else {
          reject(
            Object.assign(new Error(`Command failed: ${command} ${args.join(' ')}`), {
              stdout,
              stderr,
              exitCode,
            })
          );
        }
      });

      child.on('error', reject);
    });
  }

  // Streaming output
  stream(
    command: string,
    args: string[] = [],
    onData: (data: string, stream: 'stdout' | 'stderr') => void
  ): Promise<number | null> {
    return new Promise((resolve, reject) => {
      const child = spawn(command, args, { stdio: 'pipe' });

      child.stdout?.on('data', (data: Buffer) => {
        onData(data.toString(), 'stdout');
      });

      child.stderr?.on('data', (data: Buffer) => {
        onData(data.toString(), 'stderr');
      });

      child.on('close', resolve);
      child.on('error', reject);
    });
  }

  // Execute shell command
  async exec(command: string): Promise<{ stdout: string; stderr: string }> {
    return execAsync(command);
  }

  // Fork a Node.js module
  forkModule(
    modulePath: string,
    args: string[] = []
  ): ChildProcess {
    return fork(modulePath, args, {
      stdio: 'pipe',
    });
  }
}

// ตัวอย่างการใช้งาน
async function processDemo(): Promise<void> {
  const manager = new ProcessManager();

  // Run TypeScript compiler
  await manager.stream(
    'npx',
    ['tsc', '--noEmit'],
    (data, stream) => {
      if (stream === 'stdout') {
        process.stdout.write(data);
      } else {
        process.stderr.write(data);
      }
    }
  );

  // Run tests
  const result = await manager.run('npx', ['jest', '--json']);
  const testResults = JSON.parse(result.stdout) as { numPassedTests: number };
  console.log(`Tests passed: ${testResults.numPassedTests}`);
}

// Process pool
class ChildProcessPool {
  private processes: ChildProcess[] = [];
  private taskQueue: Array<{
    message: unknown;
    resolve: (result: unknown) => void;
    reject: (error: Error) => void;
  }> = [];
  private pendingMap = new Map<ChildProcess, {
    resolve: (result: unknown) => void;
    reject: (error: Error) => void;
  }>();

  constructor(
    modulePath: string,
    size: number
  ) {
    for (let i = 0; i < size; i++) {
      const child = fork(modulePath);
      
      child.on('message', (result: unknown) => {
        const pending = this.pendingMap.get(child);
        if (pending) {
          this.pendingMap.delete(child);
          pending.resolve(result);
          this.processQueue(child);
        }
      });

      child.on('error', (error) => {
        const pending = this.pendingMap.get(child);
        if (pending) {
          this.pendingMap.delete(child);
          pending.reject(error);
        }
      });

      this.processes.push(child);
    }
  }

  private processQueue(child: ChildProcess): void {
    if (this.taskQueue.length > 0) {
      const task = this.taskQueue.shift()!;
      this.pendingMap.set(child, { resolve: task.resolve, reject: task.reject });
      child.send(task.message);
    }
  }

  send(message: unknown): Promise<unknown> {
    return new Promise((resolve, reject) => {
      const availableChild = this.processes.find(
        (p) => !this.pendingMap.has(p)
      );

      if (availableChild) {
        this.pendingMap.set(availableChild, { resolve, reject });
        availableChild.send(message);
      } else {
        this.taskQueue.push({ message, resolve, reject });
      }
    });
  }

  destroy(): void {
    this.processes.forEach((p) => p.kill());
  }
}
```

---

## 97.10 IPC Communication

```typescript
// src/ipc/ipc-protocol.ts
import { EventEmitter } from 'events';
import * as net from 'net';

// Type-safe IPC Protocol
interface IPCMessage<T = unknown> {
  id: string;
  type: string;
  payload: T;
  timestamp: number;
}

interface IPCResponse<T = unknown> {
  id: string;
  success: boolean;
  data?: T;
  error?: string;
}

type MessageHandler<TPayload, TResult> = (
  payload: TPayload
) => Promise<TResult> | TResult;

export class IPCServer extends EventEmitter {
  private server: net.Server;
  private handlers = new Map<string, MessageHandler<unknown, unknown>>();
  private clients = new Set<net.Socket>();

  constructor(private socketPath: string) {
    super();
    this.server = net.createServer((socket) => {
      this.clients.add(socket);
      this.handleConnection(socket);
    });
  }

  register<TPayload, TResult>(
    type: string,
    handler: MessageHandler<TPayload, TResult>
  ): this {
    this.handlers.set(type, handler as MessageHandler<unknown, unknown>);
    return this;
  }

  private handleConnection(socket: net.Socket): void {
    let buffer = '';

    socket.on('data', (data) => {
      buffer += data.toString();
      
      const lines = buffer.split('\n');
      buffer = lines.pop() ?? '';

      for (const line of lines) {
        if (!line.trim()) continue;
        
        try {
          const message = JSON.parse(line) as IPCMessage;
          void this.handleMessage(socket, message);
        } catch {
          socket.write(JSON.stringify({ error: 'Invalid JSON' }) + '\n');
        }
      }
    });

    socket.on('close', () => {
      this.clients.delete(socket);
    });
  }

  private async handleMessage(
    socket: net.Socket,
    message: IPCMessage
  ): Promise<void> {
    const handler = this.handlers.get(message.type);
    
    const response: IPCResponse = {
      id: message.id,
      success: false,
    };

    try {
      if (!handler) {
        throw new Error(`No handler for message type: ${message.type}`);
      }
      
      const data = await handler(message.payload);
      response.success = true;
      response.data = data;
    } catch (error) {
      response.error = String(error);
    }

    socket.write(JSON.stringify(response) + '\n');
  }

  broadcast(type: string, payload: unknown): void {
    const message: IPCMessage = {
      id: crypto.randomUUID(),
      type,
      payload,
      timestamp: Date.now(),
    };

    const data = JSON.stringify(message) + '\n';
    for (const client of this.clients) {
      client.write(data);
    }
  }

  listen(): Promise<void> {
    return new Promise((resolve) => {
      this.server.listen(this.socketPath, resolve);
    });
  }

  close(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.server.close((err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }
}
```

---

## 97.11 Memory Management

```typescript
// src/memory/memory-monitor.ts
import * as v8 from 'v8';

interface MemoryStats {
  heapUsed: number;
  heapTotal: number;
  external: number;
  arrayBuffers: number;
  rss: number;
  heapUsedMB: number;
  heapTotalMB: number;
  rssMB: number;
}

interface HeapSnapshot {
  timestamp: Date;
  stats: MemoryStats;
}

export class MemoryMonitor {
  private snapshots: HeapSnapshot[] = [];
  private intervalId: NodeJS.Timeout | null = null;
  private thresholdMB: number;
  private onThresholdExceeded?: (stats: MemoryStats) => void;

  constructor(options: {
    thresholdMB?: number;
    onThresholdExceeded?: (stats: MemoryStats) => void;
  } = {}) {
    this.thresholdMB = options.thresholdMB ?? 500;
    this.onThresholdExceeded = options.onThresholdExceeded;
  }

  getStats(): MemoryStats {
    const mem = process.memoryUsage();
    return {
      heapUsed: mem.heapUsed,
      heapTotal: mem.heapTotal,
      external: mem.external,
      arrayBuffers: mem.arrayBuffers,
      rss: mem.rss,
      heapUsedMB: Math.round(mem.heapUsed / 1024 / 1024),
      heapTotalMB: Math.round(mem.heapTotal / 1024 / 1024),
      rssMB: Math.round(mem.rss / 1024 / 1024),
    };
  }

  snapshot(): HeapSnapshot {
    const snap = {
      timestamp: new Date(),
      stats: this.getStats(),
    };
    this.snapshots.push(snap);
    return snap;
  }

  startMonitoring(intervalMs = 5000): void {
    this.intervalId = setInterval(() => {
      const stats = this.getStats();
      
      if (stats.heapUsedMB > this.thresholdMB) {
        console.warn(`⚠️ Memory threshold exceeded: ${stats.heapUsedMB}MB > ${this.thresholdMB}MB`);
        this.onThresholdExceeded?.(stats);
      }
    }, intervalMs);

    this.intervalId.unref();
  }

  stopMonitoring(): void {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  // Force garbage collection (requires --expose-gc flag)
  forceGC(): void {
    if (global.gc) {
      global.gc();
      console.log('Garbage collection forced');
    } else {
      console.warn('GC not exposed. Run with --expose-gc flag');
    }
  }

  // Heap statistics
  getHeapStatistics() {
    return v8.getHeapStatistics();
  }

  // Print memory report
  printReport(): void {
    const stats = this.getStats();
    const heapInfo = v8.getHeapStatistics();
    
    console.log('\n=== Memory Report ===');
    console.log(`Heap Used: ${stats.heapUsedMB} MB`);
    console.log(`Heap Total: ${stats.heapTotalMB} MB`);
    console.log(`RSS: ${stats.rssMB} MB`);
    console.log(`External: ${Math.round(stats.external / 1024 / 1024)} MB`);
    console.log(`Heap Size Limit: ${Math.round(heapInfo.heap_size_limit / 1024 / 1024)} MB`);
    console.log('====================\n');
  }
}

// Memory leak detection
class WeakRefCache<K, V extends object> {
  private cache = new Map<K, WeakRef<V>>();
  private registry = new FinalizationRegistry<K>((key) => {
    // Called when value is garbage collected
    this.cache.delete(key);
    console.log(`Cache entry ${String(key)} was garbage collected`);
  });

  set(key: K, value: V): void {
    const ref = new WeakRef(value);
    this.registry.register(value, key);
    this.cache.set(key, ref);
  }

  get(key: K): V | undefined {
    const ref = this.cache.get(key);
    if (!ref) return undefined;
    
    const value = ref.deref();
    if (!value) {
      this.cache.delete(key); // Clean up dead reference
      return undefined;
    }
    
    return value;
  }

  has(key: K): boolean {
    return this.get(key) !== undefined;
  }

  size(): number {
    return this.cache.size; // May include dead references
  }
}
```

---

## 97.12 Advanced Error Handling Patterns

```typescript
// src/error-handling/error-patterns.ts

// Domain-specific errors
export class DatabaseError extends Error {
  constructor(
    message: string,
    public readonly query?: string,
    public readonly code?: string,
    public override readonly cause?: Error
  ) {
    super(message);
    this.name = 'DatabaseError';
  }
}

export class NetworkError extends Error {
  constructor(
    message: string,
    public readonly statusCode?: number,
    public readonly url?: string,
    public override readonly cause?: Error
  ) {
    super(message);
    this.name = 'NetworkError';
  }
}

// Result type pattern
type Result<T, E extends Error = Error> =
  | { success: true; value: T }
  | { success: false; error: E };

function ok<T>(value: T): Result<T> {
  return { success: true, value };
}

function err<E extends Error>(error: E): Result<never, E> {
  return { success: false, error };
}

// Error boundary for async operations
async function withErrorBoundary<T>(
  fn: () => Promise<T>,
  fallback?: T
): Promise<Result<T>> {
  try {
    const value = await fn();
    return ok(value);
  } catch (error) {
    if (error instanceof Error) {
      return err(error);
    }
    return err(new Error(String(error)));
  }
}

// Retry with exponential backoff
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    initialDelay?: number;
    maxDelay?: number;
    backoffFactor?: number;
    shouldRetry?: (error: Error) => boolean;
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    initialDelay = 1000,
    maxDelay = 30000,
    backoffFactor = 2,
    shouldRetry = () => true,
  } = options;

  let lastError: Error;
  let delay = initialDelay;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));
      
      if (attempt === maxRetries || !shouldRetry(lastError)) {
        throw lastError;
      }

      console.log(`Attempt ${attempt + 1} failed. Retrying in ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
      delay = Math.min(delay * backoffFactor, maxDelay);
    }
  }

  throw lastError!;
}

// Circuit breaker
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

class CircuitBreaker<T> {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime = 0;
  private successCount = 0;

  constructor(
    private fn: (...args: unknown[]) => Promise<T>,
    private options: {
      failureThreshold?: number;
      resetTimeout?: number;
      successThreshold?: number;
    } = {}
  ) {}

  async execute(...args: unknown[]): Promise<T> {
    const {
      failureThreshold = 5,
      resetTimeout = 30000,
      successThreshold = 2,
    } = this.options;

    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > resetTimeout) {
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await this.fn(...args);
      
      if (this.state === 'HALF_OPEN') {
        this.successCount++;
        if (this.successCount >= successThreshold) {
          this.reset();
        }
      }
      
      return result;
    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();
      
      if (this.failureCount >= failureThreshold) {
        this.state = 'OPEN';
        console.error(`Circuit breaker OPENED after ${this.failureCount} failures`);
      }
      
      throw error;
    }
  }

  private reset(): void {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    console.log('Circuit breaker CLOSED');
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

---

## บทสรุป Part 97

ในบทนี้เราได้เรียนรู้:

1. **Event Loop** - การทำงานของ phases และ microtasks
2. **libuv** - Thread pool สำหรับ async I/O
3. **Streams** - Readable, Writable, Transform, Duplex กับ TypeScript
4. **Back-pressure** - การจัดการเมื่อ producer เร็วกว่า consumer
5. **Cluster Module** - Scale across CPU cores
6. **Worker Threads** - CPU-bound tasks
7. **Shared Memory** - SharedArrayBuffer และ Atomics
8. **Child Processes** - spawn, exec, fork
9. **IPC** - Inter-process communication
10. **Memory Management** - Monitoring และ leak detection
11. **Error Handling** - Circuit breaker, retry patterns

Node.js มี primitives ที่ทรงพลังมาก TypeScript ช่วยให้ใช้งานสิ่งเหล่านี้ได้อย่างปลอดภัยและ maintainable มากขึ้น

---

## แบบฝึกหัด

1. สร้าง streaming CSV processor ที่รองรับไฟล์ขนาด GB โดยใช้ back-pressure
2. สร้าง worker pool สำหรับ image resizing ด้วย Worker Threads
3. Implement distributed cache โดยใช้ Cluster + SharedArrayBuffer
4. สร้าง process supervisor ที่ monitor และ restart workers อัตโนมัติ
5. Implement circuit breaker สำหรับ database connections
