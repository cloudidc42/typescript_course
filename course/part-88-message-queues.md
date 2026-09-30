# Part 88: Message Queues กับ TypeScript

## บทนำ

Message Queue คือระบบที่ช่วยให้ services สื่อสารกันแบบ asynchronous ทำให้ระบบมีความยืดหยุ่น รองรับโหลดสูง และทนทานต่อความผิดพลาด

---

## 1. ทำไมต้องใช้ Message Queues

### 1.1 ปัญหาที่แก้ได้

```
ปัญหา                       Message Queue แก้ได้อย่างไร
──────────────────────────── ────────────────────────────────────
Service ล่ม → ข้อมูลหาย    Message ถูกเก็บใน queue จนกว่า consumer จะพร้อม
Traffic spike → system ล่ม  Queue รับโหลดไว้ consumer ทยอยประมวลผล
Service ช้า → timeout       Async processing ไม่ต้องรอ response
Tight coupling              Services สื่อสารผ่าน messages ไม่รู้จักกัน
```

### 1.2 Use Cases ใน FinTech

```typescript
// ตัวอย่าง: ระบบแจ้งเตือนการโอนเงิน
// แทนที่จะเรียก notification service โดยตรง
// ส่ง message ไปที่ queue

// ❌ แบบ synchronous (เสี่ยง)
async function transfer(from: string, to: string, amount: number) {
  await debitAccount(from, amount);    // ถ้า step นี้สำเร็จ
  await creditAccount(to, amount);     // แต่ step นี้ล้มเหลว
  await sendSMSNotification(from);     // แล้วส่ง SMS ไม่ได้ → ลูกค้าไม่รู้
  await sendPushNotification(from);    // กว่าจะได้รับ response นาน
}

// ✅ แบบ async ด้วย message queue
async function transferWithQueue(from: string, to: string, amount: number) {
  const result = await processTransfer(from, to, amount);
  
  // ส่ง message ไป queue แล้ว return ทันที
  await messageQueue.publish('transfer.completed', {
    transactionId: result.transactionId,
    fromAccount: from,
    toAccount: to,
    amount,
  });
  
  return result; // ตอบลูกค้าทันทีโดยไม่ต้องรอ SMS
}
```

---

## 2. RabbitMQ กับ TypeScript

### 2.1 ติดตั้งและกำหนดค่า

```json
{
  "dependencies": {
    "amqplib": "^0.10.3",
    "amqp-connection-manager": "^4.1.14"
  },
  "devDependencies": {
    "@types/amqplib": "^0.10.4"
  }
}
```

### 2.2 Connection Manager

```typescript
// src/messaging/rabbitmq.connection.ts
import amqp from 'amqplib';
import { AmqpConnectionManager, connect } from 'amqp-connection-manager';

export interface RabbitMQConfig {
  url: string | string[]; // รองรับ cluster
  heartbeat?: number;
  vhost?: string;
  reconnectTimeInSeconds?: number;
}

export class RabbitMQConnection {
  private connection: AmqpConnectionManager;

  constructor(private readonly config: RabbitMQConfig) {
    this.connection = connect(
      Array.isArray(config.url) ? config.url : [config.url],
      {
        heartbeatIntervalInSeconds: config.heartbeat ?? 60,
        reconnectTimeInSeconds: config.reconnectTimeInSeconds ?? 5,
      }
    );

    this.connection.on('connect', () => {
      console.log('เชื่อมต่อ RabbitMQ สำเร็จ');
    });

    this.connection.on('disconnect', ({ err }) => {
      console.error('การเชื่อมต่อ RabbitMQ หลุด:', err?.message);
    });

    this.connection.on('connectFailed', ({ err }) => {
      console.error('เชื่อมต่อ RabbitMQ ไม่ได้:', err?.message);
    });
  }

  getConnection(): AmqpConnectionManager {
    return this.connection;
  }

  async close(): Promise<void> {
    await this.connection.close();
  }
}

// สร้าง singleton connection
export const rabbitMQConnection = new RabbitMQConnection({
  url: process.env.RABBITMQ_URL ?? 'amqp://localhost:5672',
  heartbeat: 30,
  reconnectTimeInSeconds: 5,
});
```

### 2.3 Direct Exchange

```typescript
// src/messaging/exchanges/direct.exchange.ts
import { ChannelWrapper } from 'amqp-connection-manager';
import { Channel, ConsumeMessage } from 'amqplib';

// Type-safe message interface
export interface PaymentMessage {
  transactionId: string;
  fromAccount: string;
  toAccount: string;
  amount: string;
  currency: string;
  timestamp: string;
}

export class DirectExchange {
  private channelWrapper: ChannelWrapper;
  private readonly exchangeName = 'payment.direct';

  constructor(connection: ReturnType<typeof import('amqp-connection-manager').connect>) {
    this.channelWrapper = connection.createChannel({
      json: true, // auto serialize/deserialize JSON
      setup: async (channel: Channel) => {
        // สร้าง exchange
        await channel.assertExchange(this.exchangeName, 'direct', {
          durable: true, // รอดเมื่อ RabbitMQ restart
        });

        // สร้าง queue สำหรับ transfer notifications
        await channel.assertQueue('transfer.notifications', {
          durable: true,
          arguments: {
            'x-dead-letter-exchange': 'payment.dlx', // Dead Letter Exchange
            'x-dead-letter-routing-key': 'transfer.notifications.dead',
            'x-message-ttl': 86400000, // 24 ชั่วโมง
          },
        });

        // สร้าง queue สำหรับ SMS
        await channel.assertQueue('transfer.sms', {
          durable: true,
        });

        // Bind queues กับ routing keys
        await channel.bindQueue(
          'transfer.notifications',
          this.exchangeName,
          'notify' // routing key
        );
        await channel.bindQueue(
          'transfer.sms',
          this.exchangeName,
          'sms'
        );

        console.log('Direct Exchange setup เสร็จแล้ว');
      },
    });
  }

  // Publish message
  async publish(routingKey: string, message: PaymentMessage): Promise<void> {
    await this.channelWrapper.publish(
      this.exchangeName,
      routingKey,
      message,
      {
        persistent: true,     // บันทึกลง disk (รอดเมื่อ restart)
        contentType: 'application/json',
        timestamp: Date.now(),
        messageId: `msg-${Date.now()}-${Math.random().toString(36).slice(2)}`,
        headers: {
          'x-retry-count': 0,
          'x-source-service': 'payment-service',
        },
      }
    );
  }

  // Consume messages
  async consume(
    queue: string,
    handler: (message: PaymentMessage) => Promise<void>
  ): Promise<void> {
    await this.channelWrapper.addSetup(async (channel: Channel) => {
      // Prefetch: รับทีละ 10 messages (backpressure)
      await channel.prefetch(10);

      await channel.consume(queue, async (msg: ConsumeMessage | null) => {
        if (!msg) return;

        try {
          const content = JSON.parse(msg.content.toString()) as PaymentMessage;
          console.log(`รับ message: ${msg.properties.messageId}`);

          await handler(content);

          // Acknowledge: บอก RabbitMQ ว่า message ถูก process แล้ว
          channel.ack(msg);
        } catch (error) {
          console.error('เกิดข้อผิดพลาดขณะประมวลผล message:', error);
          
          // Nack: ส่ง message กลับ queue (หรือ dead letter)
          const retryCount = (msg.properties.headers?.['x-retry-count'] as number) || 0;
          
          if (retryCount < 3) {
            // Requeue พร้อมเพิ่ม retry count
            channel.nack(msg, false, false); // ส่งไป dead letter
          } else {
            channel.nack(msg, false, false); // discard
          }
        }
      });
    });
  }
}
```

### 2.4 Fanout Exchange

```typescript
// src/messaging/exchanges/fanout.exchange.ts
export class FanoutExchange {
  private channelWrapper: ChannelWrapper;
  private readonly exchangeName = 'payment.broadcast';

  constructor(connection: any) {
    this.channelWrapper = connection.createChannel({
      json: true,
      setup: async (channel: Channel) => {
        // Fanout exchange ส่งไปทุก queue ที่ bind
        await channel.assertExchange(this.exchangeName, 'fanout', {
          durable: true,
        });

        // แต่ละ subscriber มี queue ของตัวเอง
        const services = ['audit-service', 'notification-service', 'analytics-service'];
        
        for (const service of services) {
          const queueName = `payment.broadcast.${service}`;
          await channel.assertQueue(queueName, { durable: true });
          
          // Fanout ไม่ใช้ routing key (ใช้ '' แทน)
          await channel.bindQueue(queueName, this.exchangeName, '');
        }
      },
    });
  }

  // Broadcast ไปทุก service
  async broadcast(event: {
    eventType: string;
    payload: Record<string, unknown>;
  }): Promise<void> {
    await this.channelWrapper.publish(
      this.exchangeName,
      '', // Fanout ไม่ใช้ routing key
      event,
      { persistent: true }
    );
    console.log(`Broadcast event: ${event.eventType} ไปทุก subscribers`);
  }
}
```

### 2.5 Topic Exchange

```typescript
// src/messaging/exchanges/topic.exchange.ts
export class TopicExchange {
  private channelWrapper: ChannelWrapper;
  private readonly exchangeName = 'payment.topic';

  constructor(connection: any) {
    this.channelWrapper = connection.createChannel({
      json: true,
      setup: async (channel: Channel) => {
        await channel.assertExchange(this.exchangeName, 'topic', {
          durable: true,
        });

        // Queue สำหรับ notification ทุกประเภท
        await channel.assertQueue('notifications.all', { durable: true });
        await channel.bindQueue(
          'notifications.all',
          this.exchangeName,
          'payment.#' // # = wildcard หลายคำ
        );

        // Queue สำหรับ transfer เท่านั้น
        await channel.assertQueue('notifications.transfer', { durable: true });
        await channel.bindQueue(
          'notifications.transfer',
          this.exchangeName,
          'payment.transfer.*' // * = wildcard คำเดียว
        );

        // Queue สำหรับ high-value transactions
        await channel.assertQueue('notifications.highvalue', { durable: true });
        await channel.bindQueue(
          'notifications.highvalue',
          this.exchangeName,
          'payment.*.highvalue'
        );
      },
    });
  }

  async publish(routingKey: string, data: unknown): Promise<void> {
    // routing key ตัวอย่าง:
    // "payment.transfer.completed"
    // "payment.transfer.highvalue"
    // "payment.deposit.completed"
    await this.channelWrapper.publish(
      this.exchangeName,
      routingKey,
      data,
      { persistent: true }
    );
  }
}

// ใช้งาน topic exchange
const topicExchange = new TopicExchange(rabbitMQConnection.getConnection());

// ส่ง notification ธุรกรรมโอนเงินมูลค่าสูง
await topicExchange.publish('payment.transfer.highvalue', {
  transactionId: 'TXN001',
  amount: '500000',
  currency: 'THB',
});
```

---

## 3. Dead Letter Queue

### 3.1 DLQ Setup

```typescript
// src/messaging/dlq.handler.ts
export class DeadLetterQueueHandler {
  private channelWrapper: ChannelWrapper;

  constructor(connection: any) {
    this.channelWrapper = connection.createChannel({
      json: true,
      setup: async (channel: Channel) => {
        // Dead Letter Exchange
        await channel.assertExchange('payment.dlx', 'direct', {
          durable: true,
        });

        // Dead Letter Queue
        await channel.assertQueue('payment.dlq', {
          durable: true,
          arguments: {
            'x-message-ttl': 7 * 24 * 60 * 60 * 1000, // เก็บ 7 วัน
          },
        });

        await channel.bindQueue(
          'payment.dlq',
          'payment.dlx',
          'transfer.notifications.dead'
        );
      },
    });
  }

  // ประมวลผล failed messages ใน DLQ
  async processDLQ(): Promise<void> {
    await this.channelWrapper.addSetup(async (channel: Channel) => {
      await channel.consume('payment.dlq', async (msg) => {
        if (!msg) return;

        const content = JSON.parse(msg.content.toString());
        const headers = msg.properties.headers ?? {};
        const retryCount = headers['x-retry-count'] as number || 0;
        const originalQueue = headers['x-original-queue'] as string;
        const failureReason = headers['x-failure-reason'] as string;

        console.log(`DLQ Message:`, {
          messageId: msg.properties.messageId,
          retryCount,
          failureReason,
          content,
        });

        // บันทึกลง alert system
        await this.alertFailedMessage({
          messageId: msg.properties.messageId,
          content,
          failureReason,
          retryCount,
        });

        channel.ack(msg); // ลบออกจาก DLQ หลัง log
      });
    });
  }

  private async alertFailedMessage(info: {
    messageId: string | undefined;
    content: unknown;
    failureReason: string;
    retryCount: number;
  }): Promise<void> {
    console.error('🚨 Dead Letter Message:', info);
    // TODO: ส่ง alert ไปยัง ops team
  }
}
```

### 3.2 Retry กับ Exponential Backoff

```typescript
// src/messaging/retry.handler.ts
export class RetryHandler {
  private readonly maxRetries = 5;
  private readonly baseDelayMs = 1000; // 1 วินาที

  async handleWithRetry<T>(
    operation: () => Promise<T>,
    context: { messageId: string; queue: string }
  ): Promise<T> {
    let attempt = 0;
    
    while (attempt <= this.maxRetries) {
      try {
        return await operation();
      } catch (error) {
        attempt++;
        
        if (attempt > this.maxRetries) {
          console.error(`Message ${context.messageId} ล้มเหลวหลัง ${this.maxRetries} ครั้ง`);
          throw error;
        }

        const delay = this.calculateDelay(attempt);
        console.warn(
          `Message ${context.messageId} ล้มเหลว (attempt ${attempt}/${this.maxRetries}). ` +
          `Retry ใน ${delay}ms...`
        );
        
        await this.sleep(delay);
      }
    }

    throw new Error('ไม่ควรมาถึงจุดนี้');
  }

  private calculateDelay(attempt: number): number {
    // Exponential backoff พร้อม jitter
    const exponential = this.baseDelayMs * Math.pow(2, attempt - 1);
    const jitter = Math.random() * 0.3 * exponential; // ±30% randomness
    return Math.min(exponential + jitter, 30000); // สูงสุด 30 วินาที
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ใช้งาน
const retryHandler = new RetryHandler();

async function processPaymentMessage(message: PaymentMessage): Promise<void> {
  await retryHandler.handleWithRetry(
    async () => {
      // ประมวลผล message
      await sendSMSNotification(message.fromAccount, message.amount);
    },
    { messageId: 'msg-123', queue: 'transfer.sms' }
  );
}
```

---

## 4. BullMQ - Redis-based Job Queues

### 4.1 ติดตั้งและกำหนดค่า

```json
{
  "dependencies": {
    "bullmq": "^5.1.1",
    "ioredis": "^5.3.2"
  }
}
```

### 4.2 Queue และ Worker

```typescript
// src/jobs/payment.queue.ts
import { Queue, Worker, QueueEvents, Job } from 'bullmq';
import Redis from 'ioredis';

// Job data types
export interface TransferJobData {
  fromAccountId: string;
  toAccountId: string;
  amount: string;
  currency: string;
  initiatedBy: string;
  idempotencyKey: string;
}

export interface NotificationJobData {
  userId: string;
  type: 'SMS' | 'PUSH' | 'EMAIL';
  message: string;
  template?: string;
  data?: Record<string, string>;
}

// Redis connection
const redisConnection = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: parseInt(process.env.REDIS_PORT ?? '6379'),
  maxRetriesPerRequest: null, // ต้องตั้งค่านี้สำหรับ BullMQ
});

// สร้าง queues
export const transferQueue = new Queue<TransferJobData>('transfer', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,                    // retry 3 ครั้ง
    backoff: {
      type: 'exponential',
      delay: 2000,                  // เริ่มต้น 2 วินาที
    },
    removeOnComplete: {
      count: 1000,                  // เก็บ completed jobs 1000 รายการ
      age: 24 * 60 * 60,           // หรือ 24 ชั่วโมง
    },
    removeOnFail: {
      count: 500,                   // เก็บ failed jobs 500 รายการ
    },
  },
});

export const notificationQueue = new Queue<NotificationJobData>('notification', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
  },
});

console.log('BullMQ Queues พร้อมใช้งาน');
```

### 4.3 Job Types และ Options

```typescript
// src/jobs/job.types.ts

// Priority Queue: งานสำคัญทำก่อน
await transferQueue.add(
  'high-priority-transfer',
  { fromAccountId: 'acc-1', toAccountId: 'acc-2', amount: '500000', currency: 'THB', initiatedBy: 'user-1', idempotencyKey: 'key-1' },
  { priority: 1 } // 1 = สูงสุด
);

await transferQueue.add(
  'normal-transfer',
  { fromAccountId: 'acc-3', toAccountId: 'acc-4', amount: '100', currency: 'THB', initiatedBy: 'user-2', idempotencyKey: 'key-2' },
  { priority: 10 } // ยิ่งสูง ยิ่งต่ำกว่า
);

// Delayed Job: ทำงานใน 30 นาทีข้างหน้า
await notificationQueue.add(
  'send-statement',
  { userId: 'user-123', type: 'EMAIL', message: 'Statement ประจำเดือน' },
  { delay: 30 * 60 * 1000 } // 30 นาที
);

// Recurring Job: ทุกวันเที่ยงคืน (cron)
await notificationQueue.add(
  'daily-summary',
  { userId: 'all', type: 'PUSH', message: 'สรุปยอดประจำวัน' },
  {
    repeat: {
      cron: '0 0 * * *', // เที่ยงคืนทุกวัน
      tz: 'Asia/Bangkok',
    },
  }
);

// Job พร้อม custom ID (idempotency)
await transferQueue.add(
  'transfer',
  { fromAccountId: 'acc-1', toAccountId: 'acc-2', amount: '1000', currency: 'THB', initiatedBy: 'user-1', idempotencyKey: 'unique-transfer-001' },
  { jobId: 'transfer-unique-001' } // ถ้า job id ซ้ำ จะไม่ add ใหม่
);
```

### 4.4 Worker Implementation

```typescript
// src/jobs/workers/transfer.worker.ts
import { Worker, Job, UnrecoverableError } from 'bullmq';

export class TransferWorker {
  private worker: Worker<TransferJobData>;

  constructor(private readonly transferService: any) {
    this.worker = new Worker<TransferJobData>(
      'transfer',
      this.processJob.bind(this),
      {
        connection: redisConnection,
        concurrency: 5, // ประมวลผล 5 jobs พร้อมกัน
        limiter: {
          max: 100,       // สูงสุด 100 jobs
          duration: 60000, // ต่อนาที
        },
      }
    );

    this.setupEventHandlers();
  }

  private async processJob(job: Job<TransferJobData>): Promise<void> {
    const { fromAccountId, toAccountId, amount, currency, idempotencyKey } = job.data;
    
    console.log(`ประมวลผล Transfer Job #${job.id}: ${amount} ${currency}`);

    // อัพเดท progress
    await job.updateProgress(10);

    // ตรวจสอบ idempotency
    const existing = await this.checkIdempotency(idempotencyKey);
    if (existing) {
      console.log(`Job ${job.id} เคยทำแล้ว (idempotency key: ${idempotencyKey})`);
      return;
    }

    await job.updateProgress(30);

    try {
      // ดำเนินการโอน
      const result = await this.transferService.transfer(
        fromAccountId,
        toAccountId,
        amount,
        currency
      );

      await job.updateProgress(80);

      // บันทึก idempotency result
      await this.saveIdempotencyResult(idempotencyKey, result);

      await job.updateProgress(100);

      // เพิ่ม notification job
      await notificationQueue.add('transfer-notification', {
        userId: fromAccountId,
        type: 'PUSH',
        message: `โอนเงิน ${amount} ${currency} สำเร็จ`,
      });

    } catch (error) {
      // ถ้าเป็น unrecoverable error ไม่ต้อง retry
      if ((error as Error).name === 'InsufficientFundsError') {
        throw new UnrecoverableError(`ยอดเงินไม่พอ: ${(error as Error).message}`);
      }
      // ข้อผิดพลาดอื่น retry ตาม defaultJobOptions
      throw error;
    }
  }

  private setupEventHandlers(): void {
    this.worker.on('completed', (job: Job) => {
      console.log(`✅ Job #${job.id} เสร็จสิ้น`);
    });

    this.worker.on('failed', (job: Job | undefined, err: Error) => {
      console.error(`❌ Job #${job?.id} ล้มเหลว:`, err.message);
      
      // ถ้าหมด attempts บันทึก alert
      if (job && job.attemptsMade >= (job.opts.attempts ?? 3)) {
        this.alertFailedJob(job, err);
      }
    });

    this.worker.on('progress', (job: Job, progress: number | object) => {
      console.log(`⏳ Job #${job.id} progress: ${progress}%`);
    });

    this.worker.on('stalled', (jobId: string) => {
      console.warn(`⚠️ Job #${jobId} stalled (worker crash?)`)
    });
  }

  private async checkIdempotency(key: string): Promise<boolean> {
    // TODO: ตรวจสอบใน Redis
    return false;
  }

  private async saveIdempotencyResult(key: string, result: unknown): Promise<void> {
    // TODO: บันทึกใน Redis พร้อม TTL
  }

  private async alertFailedJob(job: Job, error: Error): Promise<void> {
    console.error(`🚨 Job ${job.id} failed permanently:`, {
      data: job.data,
      error: error.message,
      attempts: job.attemptsMade,
    });
  }

  async close(): Promise<void> {
    await this.worker.close();
  }
}
```

### 4.5 Queue Events Monitoring

```typescript
// src/jobs/queue.monitor.ts
import { QueueEvents } from 'bullmq';

export class QueueMonitor {
  private transferEvents: QueueEvents;
  private notificationEvents: QueueEvents;

  constructor() {
    this.transferEvents = new QueueEvents('transfer', {
      connection: redisConnection,
    });

    this.notificationEvents = new QueueEvents('notification', {
      connection: redisConnection,
    });

    this.setupMonitoring();
  }

  private setupMonitoring(): void {
    // Transfer queue events
    this.transferEvents.on('completed', ({ jobId, returnvalue }) => {
      console.log(`Transfer job ${jobId} completed`, returnvalue);
      metrics.increment('transfer.jobs.completed');
    });

    this.transferEvents.on('failed', ({ jobId, failedReason }) => {
      console.error(`Transfer job ${jobId} failed: ${failedReason}`);
      metrics.increment('transfer.jobs.failed');
    });

    this.transferEvents.on('waiting', ({ jobId }) => {
      console.log(`Transfer job ${jobId} waiting`);
      metrics.gauge('transfer.queue.waiting', 1);
    });

    this.transferEvents.on('delayed', ({ jobId, delay }) => {
      console.log(`Transfer job ${jobId} delayed ${delay}ms`);
    });

    this.transferEvents.on('active', ({ jobId }) => {
      console.log(`Transfer job ${jobId} active`);
    });
  }

  async getQueueStats(): Promise<{
    waiting: number;
    active: number;
    completed: number;
    failed: number;
    delayed: number;
  }> {
    const [waiting, active, completed, failed, delayed] = await Promise.all([
      transferQueue.getWaitingCount(),
      transferQueue.getActiveCount(),
      transferQueue.getCompletedCount(),
      transferQueue.getFailedCount(),
      transferQueue.getDelayedCount(),
    ]);

    return { waiting, active, completed, failed, delayed };
  }

  async close(): Promise<void> {
    await Promise.all([
      this.transferEvents.close(),
      this.notificationEvents.close(),
    ]);
  }
}

// Placeholder metrics object
const metrics = {
  increment: (name: string) => {},
  gauge: (name: string, value: number) => {},
};
```

### 4.6 Concurrency Control

```typescript
// Worker พร้อม concurrency control
const worker = new Worker<TransferJobData>(
  'transfer',
  async (job: Job<TransferJobData>) => {
    // ใช้ distributed lock เพื่อป้องกัน race condition
    const lockKey = `transfer:${job.data.fromAccountId}`;
    const locked = await acquireDistributedLock(lockKey, 30); // lock 30 วินาที
    
    if (!locked) {
      throw new Error('ไม่สามารถ lock บัญชีได้ กรุณาลองใหม่');
    }

    try {
      await processTransfer(job.data);
    } finally {
      await releaseDistributedLock(lockKey);
    }
  },
  {
    connection: redisConnection,
    concurrency: 10, // ประมวลผลพร้อมกัน 10 jobs
    lockDuration: 30000, // job lock timeout
  }
);

async function acquireDistributedLock(key: string, ttlSeconds: number): Promise<boolean> {
  // TODO: implement Redis SET NX EX
  return true;
}

async function releaseDistributedLock(key: string): Promise<void> {
  // TODO: implement Redis DEL
}

async function processTransfer(data: TransferJobData): Promise<void> {
  console.log(`Processing transfer: ${data.amount} ${data.currency}`);
}
```

---

## 5. Apache Kafka กับ TypeScript

### 5.1 ติดตั้ง KafkaJS

```json
{
  "dependencies": {
    "kafkajs": "^2.2.4"
  }
}
```

### 5.2 Kafka Connection

```typescript
// src/messaging/kafka/kafka.connection.ts
import { Kafka, logLevel } from 'kafkajs';

export const kafka = new Kafka({
  clientId: 'payment-service',
  brokers: (process.env.KAFKA_BROKERS ?? 'localhost:9092').split(','),
  logLevel: logLevel.INFO,
  
  // SSL สำหรับ production
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: true,
    ca: [process.env.KAFKA_CA_CERT ?? ''],
    key: process.env.KAFKA_CLIENT_KEY,
    cert: process.env.KAFKA_CLIENT_CERT,
  } : undefined,

  // SASL authentication
  sasl: process.env.KAFKA_USERNAME ? {
    mechanism: 'plain',
    username: process.env.KAFKA_USERNAME,
    password: process.env.KAFKA_PASSWORD ?? '',
  } : undefined,

  retry: {
    initialRetryTime: 1000,
    retries: 10,
  },
});
```

### 5.3 Producer

```typescript
// src/messaging/kafka/payment.producer.ts
import { Producer, ProducerRecord, RecordMetadata } from 'kafkajs';

export interface KafkaPaymentEvent {
  eventType: 'transfer.completed' | 'transfer.failed' | 'deposit.completed' | 'withdrawal.completed';
  transactionId: string;
  accountId: string;
  amount: string;
  currency: string;
  timestamp: string;
  metadata?: Record<string, string>;
}

export class PaymentProducer {
  private producer: Producer;
  private isConnected = false;

  constructor() {
    this.producer = kafka.producer({
      allowAutoTopicCreation: true,
      transactionTimeout: 30000, // สำหรับ exactly-once
    });
  }

  async connect(): Promise<void> {
    await this.producer.connect();
    this.isConnected = true;
    console.log('Kafka Producer เชื่อมต่อแล้ว');
  }

  async publish(event: KafkaPaymentEvent): Promise<RecordMetadata[]> {
    if (!this.isConnected) {
      await this.connect();
    }

    const record: ProducerRecord = {
      topic: `payment.events.${event.eventType.split('.')[0]}`,
      messages: [
        {
          key: event.accountId, // ใช้ account ID เป็น partition key
          value: JSON.stringify(event),
          headers: {
            'event-type': event.eventType,
            'event-version': '1.0',
            'source-service': 'payment-service',
          },
          timestamp: new Date(event.timestamp).getTime().toString(),
        },
      ],
    };

    return this.producer.send(record);
  }

  // Batch publish สำหรับ bulk operations
  async publishBatch(events: KafkaPaymentEvent[]): Promise<RecordMetadata[][]> {
    if (!this.isConnected) await this.connect();

    // จัดกลุ่มตาม topic
    const byTopic = events.reduce((acc, event) => {
      const topic = `payment.events.${event.eventType.split('.')[0]}`;
      if (!acc[topic]) acc[topic] = [];
      acc[topic].push({
        key: event.accountId,
        value: JSON.stringify(event),
        headers: { 'event-type': event.eventType },
        timestamp: new Date(event.timestamp).getTime().toString(),
      });
      return acc;
    }, {} as Record<string, any[]>);

    const results = await Promise.all(
      Object.entries(byTopic).map(([topic, messages]) =>
        this.producer.send({ topic, messages })
      )
    );

    return results;
  }

  async disconnect(): Promise<void> {
    await this.producer.disconnect();
    this.isConnected = false;
  }
}
```

### 5.4 Consumer และ Consumer Groups

```typescript
// src/messaging/kafka/payment.consumer.ts
import { Consumer, EachMessagePayload, EachBatchPayload } from 'kafkajs';

export class PaymentConsumer {
  private consumer: Consumer;

  constructor(private readonly groupId: string) {
    this.consumer = kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
      maxBytesPerPartition: 1048576, // 1MB
    });
  }

  async start(
    topics: string[],
    handler: (event: KafkaPaymentEvent) => Promise<void>
  ): Promise<void> {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topics,
      fromBeginning: false, // อ่านแต่ messages ใหม่
    });

    console.log(`Kafka Consumer (${this.groupId}) subscribed to: ${topics.join(', ')}`);

    await this.consumer.run({
      eachMessage: async ({
        topic,
        partition,
        message,
        heartbeat,
      }: EachMessagePayload) => {
        if (!message.value) return;

        try {
          const event = JSON.parse(message.value.toString()) as KafkaPaymentEvent;
          
          console.log(`รับ event: ${event.eventType} (partition: ${partition})`);
          
          await handler(event);
          
          // ส่ง heartbeat เพื่อป้องกัน session timeout สำหรับ long-running tasks
          await heartbeat();
          
        } catch (error) {
          console.error('เกิดข้อผิดพลาดขณะประมวลผล Kafka message:', error);
          // ใน production ควรส่งไป DLQ หรือ error topic
        }
      },
    });
  }

  // Batch processing สำหรับ throughput สูง
  async startBatch(
    topics: string[],
    batchHandler: (events: KafkaPaymentEvent[]) => Promise<void>
  ): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({ topics, fromBeginning: false });

    await this.consumer.run({
      eachBatch: async ({
        batch,
        resolveOffset,
        heartbeat,
        isRunning,
        isStale,
      }: EachBatchPayload) => {
        const events: KafkaPaymentEvent[] = [];
        
        for (const message of batch.messages) {
          if (!isRunning() || isStale()) break;
          
          if (message.value) {
            const event = JSON.parse(message.value.toString()) as KafkaPaymentEvent;
            events.push(event);
          }
        }

        if (events.length > 0) {
          await batchHandler(events);
          
          // Commit offset หลัง process เสร็จ
          resolveOffset(batch.messages[batch.messages.length - 1].offset);
          await heartbeat();
        }
      },
    });
  }

  async disconnect(): Promise<void> {
    await this.consumer.disconnect();
  }
}
```

### 5.5 Exactly-Once Semantics

```typescript
// src/messaging/kafka/exactly-once.producer.ts
export class ExactlyOnceProducer {
  private producer: Producer;

  constructor() {
    // ต้องการ transactional producer
    this.producer = kafka.producer({
      transactionalId: `payment-service-${process.pid}`, // unique per instance
      idempotent: true,           // ป้องกัน duplicate messages
      maxInFlightRequests: 1,     // ต้องตั้ง 1 สำหรับ idempotent
    });
  }

  async publishWithTransaction(
    events: KafkaPaymentEvent[],
    dbOperation: () => Promise<void>
  ): Promise<void> {
    const transaction = await this.producer.transaction();

    try {
      // ทำ DB operation และ publish Kafka message ใน transaction เดียวกัน
      await dbOperation(); // บันทึก DB
      
      // Publish ไป Kafka (ยังไม่ commit)
      await transaction.send({
        topic: 'payment.events',
        messages: events.map(event => ({
          key: event.accountId,
          value: JSON.stringify(event),
        })),
      });

      // Commit ทั้ง DB และ Kafka พร้อมกัน (2-phase commit แบบง่าย)
      await transaction.commit();
      console.log('Transaction committed สำเร็จ');
      
    } catch (error) {
      await transaction.abort();
      console.error('Transaction aborted:', error);
      throw error;
    }
  }
}
```

---

## 6. Message Serialization

### 6.1 Type-safe Serialization

```typescript
// src/messaging/serialization.ts
export interface MessageEnvelope<T> {
  id: string;
  version: string;
  type: string;
  payload: T;
  timestamp: string;
  correlationId?: string;
  causationId?: string;
}

export class MessageSerializer {
  // Serialize with schema validation
  static serialize<T>(type: string, payload: T, correlationId?: string): string {
    const envelope: MessageEnvelope<T> = {
      id: `msg-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
      version: '1.0',
      type,
      payload,
      timestamp: new Date().toISOString(),
      correlationId,
    };

    return JSON.stringify(envelope);
  }

  // Deserialize with type checking
  static deserialize<T>(raw: string): MessageEnvelope<T> {
    const envelope = JSON.parse(raw) as MessageEnvelope<T>;
    
    // Validate required fields
    if (!envelope.id || !envelope.type || !envelope.payload) {
      throw new Error('Message ไม่ครบรูปแบบ');
    }

    return envelope;
  }
}

// Type-safe event definitions
export const PaymentEvents = {
  TRANSFER_COMPLETED: 'payment.transfer.completed',
  TRANSFER_FAILED: 'payment.transfer.failed',
  DEPOSIT_COMPLETED: 'payment.deposit.completed',
} as const;

export type PaymentEventType = typeof PaymentEvents[keyof typeof PaymentEvents];

export interface TransferCompletedPayload {
  transactionId: string;
  fromAccount: string;
  toAccount: string;
  amount: string;
  currency: string;
  fee: string;
}

// ใช้งาน
const message = MessageSerializer.serialize<TransferCompletedPayload>(
  PaymentEvents.TRANSFER_COMPLETED,
  {
    transactionId: 'TXN001',
    fromAccount: 'acc-1',
    toAccount: 'acc-2',
    amount: '1000.00',
    currency: 'THB',
    fee: '0.00',
  }
);

const envelope = MessageSerializer.deserialize<TransferCompletedPayload>(message);
console.log('Received:', envelope.type, envelope.payload.amount);
```

---

## 7. Complete Integration

### 7.1 Payment Event System

```typescript
// src/messaging/payment.event.system.ts
export class PaymentEventSystem {
  private producer: PaymentProducer;
  private consumer: PaymentConsumer;
  private directExchange: DirectExchange;

  constructor() {
    this.producer = new PaymentProducer();
    this.consumer = new PaymentConsumer('payment-processors');
    this.directExchange = new DirectExchange(rabbitMQConnection.getConnection());
  }

  async start(): Promise<void> {
    await this.producer.connect();

    // Start Kafka consumer สำหรับ external events
    await this.consumer.start(
      ['payment.events.transfer', 'payment.events.deposit'],
      async (event) => {
        await this.handlePaymentEvent(event);
      }
    );

    // Start RabbitMQ consumer สำหรับ internal notifications
    await this.directExchange.consume(
      'transfer.notifications',
      async (message) => {
        await this.sendNotification(message);
      }
    );

    console.log('Payment Event System พร้อมทำงาน');
  }

  async publishTransferCompleted(data: {
    transactionId: string;
    fromAccount: string;
    toAccount: string;
    amount: string;
    currency: string;
  }): Promise<void> {
    // ส่งทั้ง Kafka (external) และ RabbitMQ (internal)
    await Promise.all([
      this.producer.publish({
        eventType: 'transfer.completed',
        transactionId: data.transactionId,
        accountId: data.fromAccount,
        amount: data.amount,
        currency: data.currency,
        timestamp: new Date().toISOString(),
      }),
      this.directExchange.publish('notify', {
        transactionId: data.transactionId,
        fromAccount: data.fromAccount,
        toAccount: data.toAccount,
        amount: data.amount,
        currency: data.currency,
        timestamp: new Date().toISOString(),
      }),
    ]);
  }

  private async handlePaymentEvent(event: KafkaPaymentEvent): Promise<void> {
    switch (event.eventType) {
      case 'transfer.completed':
        await this.processTransferCompleted(event);
        break;
      case 'transfer.failed':
        await this.processTransferFailed(event);
        break;
      default:
        console.warn(`Unknown event type: ${event.eventType}`);
    }
  }

  private async processTransferCompleted(event: KafkaPaymentEvent): Promise<void> {
    console.log(`Process transfer completed: ${event.transactionId}`);
    // บันทึกลง analytics, update dashboard, etc.
  }

  private async processTransferFailed(event: KafkaPaymentEvent): Promise<void> {
    console.error(`Transfer failed: ${event.transactionId}`);
    // Alert fraud team, reverse holds, etc.
  }

  private async sendNotification(message: PaymentMessage): Promise<void> {
    console.log(`ส่ง notification ไปยัง ${message.fromAccount}`);
    // TODO: ส่ง SMS/Push notification จริง
  }

  async stop(): Promise<void> {
    await Promise.all([
      this.producer.disconnect(),
      this.consumer.disconnect(),
    ]);
  }
}
```

---

## สรุปบทที่ 88

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Message Queues** - ประโยชน์ของ async messaging
2. **RabbitMQ กับ amqplib** - Direct, Fanout, Topic exchanges
3. **Dead Letter Queue** - จัดการ failed messages
4. **Retry กับ Exponential Backoff** - ลดผลกระทบจากความผิดพลาด
5. **BullMQ** - Redis-based job queues พร้อม priority, delay, cron
6. **Concurrency Control** - จัดการ race conditions ด้วย distributed lock
7. **Apache Kafka** - Producer, Consumer, Consumer Groups
8. **Exactly-Once Semantics** - ป้องกัน duplicate processing
9. **Type-safe Serialization** - Message envelope pattern
10. **Integration** - รวม RabbitMQ + Kafka ในระบบเดียวกัน
