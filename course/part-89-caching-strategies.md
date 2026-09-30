# Part 89: Caching Strategies กับ TypeScript

## บทนำ

Caching คือเทคนิคการเก็บข้อมูลไว้ในที่ที่เข้าถึงได้เร็ว เพื่อลดเวลาในการดึงข้อมูลจากแหล่งที่ช้ากว่า เช่น database หรือ API ภายนอก

---

## 1. ทำไม Caching ถึงสำคัญ

### 1.1 ผลกระทบต่อ Performance

```
ระดับความเร็วของ storage (โดยประมาณ)
─────────────────────────────────────────────
L1 Cache (CPU)    :  0.5 ns    1x
L2 Cache (CPU)    :  7 ns      14x
RAM               :  100 ns    200x
Redis (local)     :  0.1 ms    200,000x
SSD               :  0.1 ms    200,000x
Network (LAN)     :  1 ms      2,000,000x
HDD               :  10 ms     20,000,000x
Database query    :  1-100 ms  2M-200Mx
Network (WAN)     :  100 ms    200,000,000x
```

### 1.2 Cache Hit Rate

```typescript
// ตัวอย่างการวัด cache performance
class CacheMetrics {
  private hits = 0;
  private misses = 0;
  private totalLatency = 0;

  recordHit(latencyMs: number): void {
    this.hits++;
    this.totalLatency += latencyMs;
  }

  recordMiss(latencyMs: number): void {
    this.misses++;
    this.totalLatency += latencyMs;
  }

  getStats(): {
    hitRate: number;
    missRate: number;
    avgLatency: number;
    totalRequests: number;
  } {
    const total = this.hits + this.misses;
    return {
      hitRate: total > 0 ? this.hits / total : 0,
      missRate: total > 0 ? this.misses / total : 0,
      avgLatency: total > 0 ? this.totalLatency / total : 0,
      totalRequests: total,
    };
  }
}
```

---

## 2. Redis กับ TypeScript (ioredis)

### 2.1 ติดตั้งและกำหนดค่า

```json
{
  "dependencies": {
    "ioredis": "^5.3.2",
    "node-cache": "^5.1.2"
  },
  "devDependencies": {
    "@types/ioredis": "^4.28.10"
  }
}
```

### 2.2 Redis Connection

```typescript
// src/cache/redis.client.ts
import Redis, { RedisOptions } from 'ioredis';

export class RedisClient {
  private client: Redis;
  private readonly defaultTTL: number;

  constructor(options?: RedisOptions, defaultTTL = 3600) {
    this.defaultTTL = defaultTTL;
    
    this.client = new Redis({
      host: process.env.REDIS_HOST ?? 'localhost',
      port: parseInt(process.env.REDIS_PORT ?? '6379'),
      password: process.env.REDIS_PASSWORD,
      db: parseInt(process.env.REDIS_DB ?? '0'),
      keyPrefix: 'app:',        // prefix ทุก key อัตโนมัติ
      retryStrategy: (times: number) => {
        if (times > 10) {
          console.error('Redis reconnect attempts exceeded');
          return null; // หยุด retry
        }
        return Math.min(times * 100, 3000); // exponential backoff สูงสุด 3 วินาที
      },
      reconnectOnError: (err: Error) => {
        // Reconnect เมื่อ READONLY error (failover)
        return err.message.includes('READONLY');
      },
      lazyConnect: true,        // เชื่อมต่อเมื่อมีการใช้งานครั้งแรก
      enableReadyCheck: true,
      maxRetriesPerRequest: 3,
      ...options,
    });

    this.client.on('connect', () => console.log('Redis เชื่อมต่อแล้ว'));
    this.client.on('error', (err) => console.error('Redis error:', err));
    this.client.on('ready', () => console.log('Redis พร้อมใช้งาน'));
    this.client.on('close', () => console.log('Redis connection ปิดแล้ว'));
  }

  getClient(): Redis {
    return this.client;
  }

  async disconnect(): Promise<void> {
    await this.client.quit();
  }
}

// Singleton
export const redisClient = new RedisClient();
```

---

## 3. Redis Data Structures

### 3.1 String Operations

```typescript
// src/cache/string.cache.ts
export class StringCache {
  constructor(private readonly redis: Redis) {}

  // SET กับ TTL
  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.redis.set(key, value, 'EX', ttlSeconds);
    } else {
      await this.redis.set(key, value);
    }
  }

  // SET ถ้ายังไม่มี key (SETNX)
  async setIfNotExists(
    key: string,
    value: string,
    ttlSeconds: number
  ): Promise<boolean> {
    const result = await this.redis.set(key, value, 'EX', ttlSeconds, 'NX');
    return result === 'OK';
  }

  // Atomic increment
  async increment(key: string, by = 1): Promise<number> {
    return this.redis.incrby(key, by);
  }

  // GET
  async get(key: string): Promise<string | null> {
    return this.redis.get(key);
  }

  // GET แล้ว refresh TTL
  async getAndRefresh(key: string, ttlSeconds: number): Promise<string | null> {
    const pipeline = this.redis.pipeline();
    pipeline.get(key);
    pipeline.expire(key, ttlSeconds);
    const results = await pipeline.exec();
    return (results?.[0]?.[1] as string | null) ?? null;
  }

  // Cache JSON objects
  async setJSON<T>(key: string, value: T, ttlSeconds?: number): Promise<void> {
    const json = JSON.stringify(value);
    await this.set(key, json, ttlSeconds);
  }

  async getJSON<T>(key: string): Promise<T | null> {
    const json = await this.get(key);
    if (!json) return null;
    try {
      return JSON.parse(json) as T;
    } catch {
      return null;
    }
  }
}
```

### 3.2 Hash Operations

```typescript
// src/cache/hash.cache.ts
export class HashCache {
  constructor(private readonly redis: Redis) {}

  // บันทึก object เป็น Hash (ประหยัด memory กว่า JSON string)
  async setObject(key: string, obj: Record<string, string | number | boolean>): Promise<void> {
    const stringified = Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, String(v)])
    );
    await this.redis.hmset(key, stringified);
  }

  // ดึง field เดียว
  async getField(key: string, field: string): Promise<string | null> {
    return this.redis.hget(key, field);
  }

  // ดึงทั้ง object
  async getObject(key: string): Promise<Record<string, string> | null> {
    const result = await this.redis.hgetall(key);
    return Object.keys(result).length > 0 ? result : null;
  }

  // อัพเดท field เดียวโดยไม่ต้องดึงทั้ง object
  async updateField(key: string, field: string, value: string): Promise<void> {
    await this.redis.hset(key, field, value);
  }

  // Increment numeric field
  async incrementField(key: string, field: string, by = 1): Promise<number> {
    return this.redis.hincrbyfloat(key, field, by);
  }

  // User profile caching
  async cacheUserProfile(userId: string, profile: {
    name: string;
    email: string;
    tier: string;
    balance: string;
  }): Promise<void> {
    const key = `user:profile:${userId}`;
    await this.setObject(key, profile);
    await this.redis.expire(key, 1800); // 30 นาที
  }

  async getUserProfile(userId: string): Promise<{
    name: string;
    email: string;
    tier: string;
    balance: string;
  } | null> {
    const result = await this.getObject(`user:profile:${userId}`);
    if (!result) return null;
    return result as { name: string; email: string; tier: string; balance: string };
  }
}
```

### 3.3 List Operations

```typescript
// src/cache/list.cache.ts
export class ListCache {
  constructor(private readonly redis: Redis) {}

  // เพิ่มที่ต้น list (LPUSH)
  async prepend(key: string, ...values: string[]): Promise<number> {
    return this.redis.lpush(key, ...values);
  }

  // เพิ่มที่ท้าย list (RPUSH)
  async append(key: string, ...values: string[]): Promise<number> {
    return this.redis.rpush(key, ...values);
  }

  // ดึง range (0 to -1 = ทั้งหมด)
  async getRange(key: string, start = 0, end = -1): Promise<string[]> {
    return this.redis.lrange(key, start, end);
  }

  // ตัด list ให้เหลือ N รายการล่าสุด
  async trimToLastN(key: string, n: number): Promise<void> {
    await this.redis.ltrim(key, 0, n - 1);
  }

  // Recent transactions cache
  async addRecentTransaction(
    accountId: string,
    transaction: { id: string; amount: string; type: string; date: string }
  ): Promise<void> {
    const key = `account:${accountId}:recent_txns`;
    
    // LPUSH แล้ว LTRIM เหลือ 50 รายการล่าสุด
    await this.redis.lpush(key, JSON.stringify(transaction));
    await this.trimToLastN(key, 50);
    await this.redis.expire(key, 86400); // 24 ชั่วโมง
  }

  async getRecentTransactions(accountId: string, limit = 10): Promise<Array<{
    id: string;
    amount: string;
    type: string;
    date: string;
  }>> {
    const raw = await this.getRange(`account:${accountId}:recent_txns`, 0, limit - 1);
    return raw.map(item => JSON.parse(item));
  }
}
```

### 3.4 Set Operations

```typescript
// src/cache/set.cache.ts
export class SetCache {
  constructor(private readonly redis: Redis) {}

  async addMembers(key: string, ...members: string[]): Promise<number> {
    return this.redis.sadd(key, ...members);
  }

  async isMember(key: string, member: string): Promise<boolean> {
    return (await this.redis.sismember(key, member)) === 1;
  }

  async getMembers(key: string): Promise<string[]> {
    return this.redis.smembers(key);
  }

  async removeMember(key: string, member: string): Promise<number> {
    return this.redis.srem(key, member);
  }

  // กรณีใช้งาน: Blacklist tokens
  async blacklistToken(token: string, ttlSeconds: number): Promise<void> {
    const key = 'auth:blacklisted_tokens';
    await this.addMembers(key, token);
    // NOTE: Set ทั้งอันมี TTL เดียว ถ้าต้องการ TTL รายชิ้นใช้ Sorted Set แทน
  }

  async isTokenBlacklisted(token: string): Promise<boolean> {
    return this.isMember('auth:blacklisted_tokens', token);
  }

  // กรณีใช้งาน: Online users
  async setUserOnline(userId: string): Promise<void> {
    await this.addMembers('users:online', userId);
  }

  async setUserOffline(userId: string): Promise<void> {
    await this.removeMember('users:online', userId);
  }

  async getOnlineUsers(): Promise<string[]> {
    return this.getMembers('users:online');
  }
}
```

### 3.5 Sorted Set Operations

```typescript
// src/cache/sorted-set.cache.ts
export class SortedSetCache {
  constructor(private readonly redis: Redis) {}

  // เพิ่มสมาชิกพร้อม score
  async addWithScore(key: string, score: number, member: string): Promise<number> {
    return this.redis.zadd(key, score, member);
  }

  // ดึง top N (คะแนนสูงสุด)
  async getTopN(key: string, n: number): Promise<Array<{ member: string; score: number }>> {
    const results = await this.redis.zrevrangebyscore(
      key, '+inf', '-inf',
      'WITHSCORES',
      'LIMIT', 0, n
    );
    
    const parsed: Array<{ member: string; score: number }> = [];
    for (let i = 0; i < results.length; i += 2) {
      parsed.push({
        member: results[i],
        score: parseFloat(results[i + 1]),
      });
    }
    return parsed;
  }

  // Leaderboard
  async updateLeaderboard(userId: string, score: number): Promise<void> {
    await this.addWithScore('leaderboard:transfers', score, userId);
    await this.redis.expire('leaderboard:transfers', 86400);
  }

  async getLeaderboard(top = 10): Promise<Array<{ userId: string; score: number }>> {
    const results = await this.getTopN('leaderboard:transfers', top);
    return results.map(r => ({ userId: r.member, score: r.score }));
  }

  // Rate limiting ด้วย Sorted Set (Sliding Window)
  async checkRateLimit(
    key: string,
    windowMs: number,
    maxRequests: number
  ): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
    const now = Date.now();
    const windowStart = now - windowMs;

    const pipeline = this.redis.pipeline();
    // ลบ requests เก่าที่นอก window
    pipeline.zremrangebyscore(key, '-inf', windowStart);
    // นับ requests ปัจจุบัน
    pipeline.zcard(key);
    // เพิ่ม request ปัจจุบัน
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    // ตั้ง TTL
    pipeline.pexpire(key, windowMs);

    const results = await pipeline.exec();
    const currentCount = (results?.[1]?.[1] as number) ?? 0;

    const allowed = currentCount < maxRequests;
    const remaining = Math.max(0, maxRequests - currentCount - 1);
    const oldestRequest = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
    const resetAt = oldestRequest.length >= 2 
      ? parseInt(oldestRequest[1]) + windowMs 
      : now + windowMs;

    return { allowed, remaining, resetAt };
  }
}
```

---

## 4. Cache Patterns

### 4.1 Cache-Aside Pattern

```typescript
// src/cache/patterns/cache-aside.ts
export class CacheAsidePattern<T> {
  constructor(
    private readonly cache: StringCache,
    private readonly ttlSeconds: number = 300
  ) {}

  async get(
    key: string,
    fetchFn: () => Promise<T>
  ): Promise<T> {
    // 1. ลองดึงจาก cache ก่อน
    const cached = await this.cache.getJSON<T>(key);
    
    if (cached !== null) {
      metrics.cacheHit(key);
      return cached;
    }

    // 2. Cache miss: ดึงจาก source
    metrics.cacheMiss(key);
    const data = await fetchFn();
    
    // 3. บันทึกลง cache
    await this.cache.setJSON(key, data, this.ttlSeconds);
    
    return data;
  }

  async invalidate(key: string): Promise<void> {
    await this.cache.getClient().del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.cache.getClient().keys(pattern);
    if (keys.length > 0) {
      await this.cache.getClient().del(...keys);
    }
  }
}

// ใช้งาน
const accountCache = new CacheAsidePattern<Account>(stringCache, 300);

async function getAccount(accountId: string): Promise<Account> {
  return accountCache.get(
    `account:${accountId}`,
    () => accountRepository.findById(accountId) as Promise<Account>
  );
}

// เมื่ออัพเดท account ให้ invalidate cache
async function updateAccount(accountId: string, data: Partial<Account>): Promise<void> {
  await accountRepository.update(accountId, data);
  await accountCache.invalidate(`account:${accountId}`);
}
```

### 4.2 Write-Through Pattern

```typescript
// src/cache/patterns/write-through.ts
export class WriteThroughPattern<T> {
  constructor(
    private readonly cache: StringCache,
    private readonly ttlSeconds: number = 300
  ) {}

  // เขียนทั้ง DB และ cache พร้อมกัน
  async set(
    key: string,
    value: T,
    writeFn: (value: T) => Promise<void>
  ): Promise<void> {
    // เขียน DB ก่อน
    await writeFn(value);
    
    // แล้วเขียน cache
    await this.cache.setJSON(key, value, this.ttlSeconds);
  }

  async get(
    key: string,
    fetchFn: () => Promise<T | null>
  ): Promise<T | null> {
    const cached = await this.cache.getJSON<T>(key);
    if (cached !== null) return cached;
    
    const data = await fetchFn();
    if (data !== null) {
      await this.cache.setJSON(key, data, this.ttlSeconds);
    }
    return data;
  }
}

// ใช้งาน: Account balance cache
const balanceCache = new WriteThroughPattern<string>(stringCache, 60);

async function updateBalance(accountId: string, newBalance: string): Promise<void> {
  await balanceCache.set(
    `balance:${accountId}`,
    newBalance,
    async (balance) => {
      await accountRepository.updateBalance(accountId, balance);
    }
  );
}
```

### 4.3 Write-Behind (Write-Back) Pattern

```typescript
// src/cache/patterns/write-behind.ts
export class WriteBehindPattern<T> {
  private pendingWrites = new Map<string, { value: T; timestamp: number }>();
  private flushInterval: NodeJS.Timeout;

  constructor(
    private readonly cache: StringCache,
    private readonly writeFn: (key: string, value: T) => Promise<void>,
    private readonly flushIntervalMs = 5000,
    private readonly ttlSeconds = 300
  ) {
    // Flush pending writes ทุก 5 วินาที
    this.flushInterval = setInterval(
      () => this.flush(),
      this.flushIntervalMs
    );
  }

  // เขียน cache ทันที แต่ DB ทำทีหลัง
  async set(key: string, value: T): Promise<void> {
    // เขียน cache ทันที
    await this.cache.setJSON(key, value, this.ttlSeconds);
    
    // Queue DB write
    this.pendingWrites.set(key, { value, timestamp: Date.now() });
  }

  async get(key: string): Promise<T | null> {
    return this.cache.getJSON<T>(key);
  }

  private async flush(): Promise<void> {
    if (this.pendingWrites.size === 0) return;
    
    const writes = new Map(this.pendingWrites);
    this.pendingWrites.clear();

    console.log(`Flushing ${writes.size} pending writes...`);
    
    await Promise.allSettled(
      Array.from(writes.entries()).map(async ([key, { value }]) => {
        try {
          await this.writeFn(key, value);
        } catch (error) {
          // คืน pending write ถ้า flush ล้มเหลว
          this.pendingWrites.set(key, { value, timestamp: Date.now() });
          console.error(`Failed to flush write for ${key}:`, error);
        }
      })
    );
  }

  // Force flush เมื่อ shutdown
  async destroy(): Promise<void> {
    clearInterval(this.flushInterval);
    await this.flush();
  }
}
```

### 4.4 Read-Through Pattern

```typescript
// src/cache/patterns/read-through.ts
export class ReadThroughPattern<T> {
  constructor(
    private readonly cache: StringCache,
    private readonly loader: (key: string) => Promise<T | null>,
    private readonly ttlSeconds: number = 300
  ) {}

  // Cache โปร่งใส: ผู้ใช้ไม่รู้ว่าอยู่ใน cache หรือเปล่า
  async get(key: string): Promise<T | null> {
    const cached = await this.cache.getJSON<T>(key);
    if (cached !== null) return cached;

    const data = await this.loader(key);
    if (data !== null) {
      await this.cache.setJSON(key, data, this.ttlSeconds);
    }
    return data;
  }

  // Preload cache
  async preload(keys: string[]): Promise<void> {
    const results = await Promise.allSettled(
      keys.map(key => this.get(key))
    );
    
    const failed = results.filter(r => r.status === 'rejected').length;
    console.log(`Preloaded ${keys.length - failed}/${keys.length} keys`);
  }
}
```

---

## 5. Cache Stampede Prevention

### 5.1 Mutex Lock

```typescript
// src/cache/mutex.ts
export class RedisMutex {
  constructor(
    private readonly redis: Redis,
    private readonly defaultTTL = 30
  ) {}

  // Acquire lock
  async acquire(lockKey: string, ttlSeconds = this.defaultTTL): Promise<string | null> {
    const token = `lock:${Date.now()}:${Math.random().toString(36).slice(2)}`;
    
    // SET NX EX: set ถ้ายังไม่มี key พร้อม TTL
    const result = await this.redis.set(
      `mutex:${lockKey}`,
      token,
      'EX', ttlSeconds,
      'NX'
    );

    return result === 'OK' ? token : null;
  }

  // Release lock (ตรวจสอบว่า token ตรงกันก่อน)
  async release(lockKey: string, token: string): Promise<boolean> {
    // Lua script เพื่อให้เป็น atomic operation
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(
      script, 1, `mutex:${lockKey}`, token
    ) as number;
    
    return result === 1;
  }

  // Execute with lock
  async withLock<T>(
    lockKey: string,
    fn: () => Promise<T>,
    options: { ttlSeconds?: number; maxWaitMs?: number } = {}
  ): Promise<T> {
    const { ttlSeconds = 30, maxWaitMs = 5000 } = options;
    const startTime = Date.now();
    let token: string | null = null;

    // รอ lock ด้วย polling
    while (!token) {
      token = await this.acquire(lockKey, ttlSeconds);
      
      if (!token) {
        if (Date.now() - startTime > maxWaitMs) {
          throw new Error(`ไม่สามารถ acquire lock "${lockKey}" ได้ภายใน ${maxWaitMs}ms`);
        }
        await new Promise(resolve => setTimeout(resolve, 100)); // รอ 100ms แล้วลองใหม่
      }
    }

    try {
      return await fn();
    } finally {
      await this.release(lockKey, token);
    }
  }
}

// แก้ Cache Stampede
export class AntiStampedeCache<T> {
  private readonly mutex: RedisMutex;
  private readonly cache: StringCache;

  constructor(redis: Redis, private readonly ttlSeconds = 300) {
    this.mutex = new RedisMutex(redis);
    this.cache = new StringCache(redis);
  }

  async get(
    key: string,
    fetchFn: () => Promise<T>
  ): Promise<T> {
    // ตรวจ cache ก่อน
    const cached = await this.cache.getJSON<T>(key);
    if (cached !== null) return cached;

    // ถ้า cache miss ใช้ mutex ป้องกัน stampede
    return this.mutex.withLock(
      `fetch:${key}`,
      async () => {
        // ตรวจ cache อีกครั้งหลัง acquire lock
        // (อาจมี process อื่น fetch ไปแล้วระหว่างรอ lock)
        const doubleCheck = await this.cache.getJSON<T>(key);
        if (doubleCheck !== null) return doubleCheck;

        // Fetch จาก source
        const data = await fetchFn();
        await this.cache.setJSON(key, data, this.ttlSeconds);
        return data;
      },
      { maxWaitMs: 3000 }
    );
  }
}
```

---

## 6. TTL Management

### 6.1 Smart TTL

```typescript
// src/cache/ttl.manager.ts
export class TTLManager {
  // TTL ตาม data type
  static readonly TTLPresets = {
    USER_SESSION: 86400,          // 24 ชั่วโมง
    USER_PROFILE: 1800,           // 30 นาที
    ACCOUNT_BALANCE: 60,          // 1 นาที (ต้อง fresh)
    EXCHANGE_RATE: 300,           // 5 นาที
    TRANSACTION_HISTORY: 600,     // 10 นาที
    STATIC_CONFIG: 3600,          // 1 ชั่วโมง
    PRODUCT_CATALOG: 7200,        // 2 ชั่วโมง
    RATE_LIMIT_WINDOW: 60,        // 1 นาที
  } as const;

  // TTL แบบ fuzzy (เพิ่ม jitter ป้องกัน mass expiry)
  static withJitter(baseTTL: number, jitterPercent = 0.1): number {
    const jitter = baseTTL * jitterPercent;
    return Math.floor(baseTTL + (Math.random() * 2 - 1) * jitter);
  }

  // คำนวณ TTL ตาม time of day
  static getAdaptiveTTL(baseKey: string): number {
    const hour = new Date().getHours();
    
    // กลางคืน (23:00-06:00) cache นานกว่า
    if (hour >= 23 || hour < 6) {
      return TTLManager.withJitter(7200); // 2 ชั่วโมง
    }
    // ชั่วโมง rush (08:00-10:00, 17:00-19:00) cache น้อยลง
    if ((hour >= 8 && hour <= 10) || (hour >= 17 && hour <= 19)) {
      return TTLManager.withJitter(300); // 5 นาที
    }
    return TTLManager.withJitter(1800); // 30 นาที (default)
  }
}

// ใช้งาน
const balance = await antiStampedeCache.get(
  `balance:${accountId}`,
  () => accountService.getBalance(accountId),
);
```

---

## 7. Pub/Sub กับ TypeScript

### 7.1 Redis Pub/Sub

```typescript
// src/cache/pubsub.ts
import Redis from 'ioredis';

export interface CacheInvalidationEvent {
  type: 'INVALIDATE' | 'REFRESH';
  pattern: string;
  reason: string;
  timestamp: string;
}

export class RedisPubSub {
  private publisher: Redis;
  private subscriber: Redis;
  private handlers = new Map<string, Set<(message: string) => void>>();

  constructor(redisOptions?: any) {
    // Publisher และ Subscriber ต้องเป็น connection แยกกัน
    this.publisher = new Redis(redisOptions);
    this.subscriber = new Redis(redisOptions);

    this.subscriber.on('message', (channel: string, message: string) => {
      const channelHandlers = this.handlers.get(channel);
      channelHandlers?.forEach(handler => handler(message));
    });

    this.subscriber.on('pmessage', (pattern: string, channel: string, message: string) => {
      const patternHandlers = this.handlers.get(pattern);
      patternHandlers?.forEach(handler => handler(message));
    });
  }

  async publish(channel: string, message: string | object): Promise<number> {
    const payload = typeof message === 'string' ? message : JSON.stringify(message);
    return this.publisher.publish(channel, payload);
  }

  async subscribe(channel: string, handler: (message: string) => void): Promise<void> {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, new Set());
      await this.subscriber.subscribe(channel);
    }
    this.handlers.get(channel)!.add(handler);
  }

  async psubscribe(pattern: string, handler: (message: string) => void): Promise<void> {
    if (!this.handlers.has(pattern)) {
      this.handlers.set(pattern, new Set());
      await this.subscriber.psubscribe(pattern);
    }
    this.handlers.get(pattern)!.add(handler);
  }

  async unsubscribe(channel: string): Promise<void> {
    this.handlers.delete(channel);
    await this.subscriber.unsubscribe(channel);
  }

  async disconnect(): Promise<void> {
    await Promise.all([
      this.publisher.quit(),
      this.subscriber.quit(),
    ]);
  }
}

// Cache Invalidation ผ่าน Pub/Sub
export class DistributedCacheInvalidator {
  private pubsub: RedisPubSub;
  private localCache: Map<string, unknown> = new Map();

  constructor(redisOptions?: any) {
    this.pubsub = new RedisPubSub(redisOptions);
    this.setupInvalidationListener();
  }

  private setupInvalidationListener(): void {
    this.pubsub.subscribe(
      'cache:invalidation',
      (message) => {
        const event = JSON.parse(message) as CacheInvalidationEvent;
        this.handleInvalidation(event);
      }
    );
  }

  private handleInvalidation(event: CacheInvalidationEvent): void {
    if (event.type === 'INVALIDATE') {
      // ลบ keys ที่ตรงกับ pattern
      for (const key of this.localCache.keys()) {
        if (this.matchPattern(key, event.pattern)) {
          this.localCache.delete(key);
          console.log(`Local cache invalidated: ${key} (reason: ${event.reason})`);
        }
      }
    }
  }

  async broadcastInvalidation(pattern: string, reason: string): Promise<void> {
    await this.pubsub.publish('cache:invalidation', {
      type: 'INVALIDATE',
      pattern,
      reason,
      timestamp: new Date().toISOString(),
    } as CacheInvalidationEvent);
  }

  private matchPattern(key: string, pattern: string): boolean {
    const regexPattern = pattern.replace('*', '.*').replace('?', '.');
    return new RegExp(`^${regexPattern}$`).test(key);
  }
}
```

---

## 8. In-Memory Caching ด้วย node-cache

### 8.1 node-cache Setup

```typescript
// src/cache/memory.cache.ts
import NodeCache from 'node-cache';

export class InMemoryCache<T> {
  private cache: NodeCache;

  constructor(options: {
    stdTTL?: number;
    maxKeys?: number;
    checkperiod?: number;
  } = {}) {
    this.cache = new NodeCache({
      stdTTL: options.stdTTL ?? 300,    // default TTL 5 นาที
      maxKeys: options.maxKeys ?? 1000,  // สูงสุด 1000 keys
      checkperiod: options.checkperiod ?? 60, // ตรวจสอบ expired ทุก 60 วินาที
      useClones: true,                   // clone objects เพื่อ immutability
    });

    this.cache.on('expired', (key: string, value: T) => {
      console.log(`In-memory cache expired: ${key}`);
    });
  }

  set(key: string, value: T, ttlSeconds?: number): boolean {
    return ttlSeconds !== undefined
      ? this.cache.set(key, value, ttlSeconds)
      : this.cache.set(key, value);
  }

  get(key: string): T | undefined {
    return this.cache.get<T>(key);
  }

  delete(key: string): number {
    return this.cache.del(key);
  }

  has(key: string): boolean {
    return this.cache.has(key);
  }

  getStats(): {
    keys: number;
    hits: number;
    misses: number;
    hitRate: number;
  } {
    const stats = this.cache.getStats();
    const total = stats.hits + stats.misses;
    return {
      keys: stats.keys,
      hits: stats.hits,
      misses: stats.misses,
      hitRate: total > 0 ? stats.hits / total : 0,
    };
  }
}

// Two-level cache: L1 (memory) + L2 (Redis)
export class TwoLevelCache<T> {
  private l1: InMemoryCache<T>;
  private l2: StringCache;

  constructor(redis: Redis) {
    this.l1 = new InMemoryCache({ stdTTL: 60, maxKeys: 500 }); // L1: 1 นาที
    this.l2 = new StringCache(redis);
  }

  async get(
    key: string,
    fetchFn: () => Promise<T>,
    l2TTL = 300
  ): Promise<T> {
    // ลอง L1 ก่อน
    const l1Value = this.l1.get(key);
    if (l1Value !== undefined) {
      metrics.l1Hit(key);
      return l1Value;
    }

    // ลอง L2
    const l2Value = await this.l2.getJSON<T>(key);
    if (l2Value !== null) {
      metrics.l2Hit(key);
      this.l1.set(key, l2Value); // Populate L1
      return l2Value;
    }

    // Cache miss: fetch จาก source
    metrics.cacheMiss(key);
    const data = await fetchFn();
    
    // บันทึกทั้งสองชั้น
    this.l1.set(key, data);
    await this.l2.setJSON(key, data, l2TTL);
    
    return data;
  }

  invalidate(key: string): void {
    this.l1.delete(key);
    // L2 จะ expire เอง หรือเรียก invalidate เพิ่มเติม
  }
}
```

---

## 9. HTTP Cache Headers

### 9.1 Cache-Control Middleware

```typescript
// src/middleware/http-cache.middleware.ts
import { Request, Response, NextFunction } from 'express';

export interface CacheOptions {
  maxAge?: number;          // วินาที
  sMaxAge?: number;         // CDN cache
  noStore?: boolean;        // ห้าม cache เลย
  noCache?: boolean;        // ต้อง revalidate
  private?: boolean;        // cache เฉพาะ browser
  public?: boolean;         // cache ได้ทุกที่
  immutable?: boolean;      // ไม่เปลี่ยนแน่นอน
  mustRevalidate?: boolean;
  etag?: boolean;
}

export function httpCache(options: CacheOptions) {
  return (req: Request, res: Response, next: NextFunction) => {
    const directives: string[] = [];

    if (options.noStore) {
      res.setHeader('Cache-Control', 'no-store');
      return next();
    }

    if (options.private) directives.push('private');
    if (options.public) directives.push('public');
    if (options.noCache) directives.push('no-cache');
    if (options.maxAge !== undefined) directives.push(`max-age=${options.maxAge}`);
    if (options.sMaxAge !== undefined) directives.push(`s-maxage=${options.sMaxAge}`);
    if (options.immutable) directives.push('immutable');
    if (options.mustRevalidate) directives.push('must-revalidate');

    res.setHeader('Cache-Control', directives.join(', '));

    if (options.etag !== false) {
      // ETag middleware สำหรับ conditional requests
      res.on('finish', () => {
        if (!res.getHeader('ETag')) {
          const body = (res as any).body;
          if (body) {
            const hash = require('crypto')
              .createHash('md5')
              .update(JSON.stringify(body))
              .digest('hex');
            res.setHeader('ETag', `"${hash}"`);
          }
        }
      });
    }

    next();
  };
}

// ใช้งาน
import express from 'express';
const router = express.Router();

// ข้อมูลที่เปลี่ยนแปลงบ่อย: no-cache
router.get('/balance', httpCache({ noCache: true, private: true }), getBalance);

// ข้อมูล static: cache นาน
router.get('/exchange-rates', httpCache({ 
  public: true, 
  maxAge: 300,        // browser cache 5 นาที
  sMaxAge: 600,       // CDN cache 10 นาที
}), getExchangeRates);

// ข้อมูลที่ไม่ควร cache เลย
router.post('/transfer', httpCache({ noStore: true }), transfer);
```

---

## 10. Query Result Caching

### 10.1 Database Query Cache

```typescript
// src/cache/query.cache.ts
export class QueryCache {
  constructor(
    private readonly redis: Redis,
    private readonly defaultTTL = 300
  ) {}

  // Cache decorator สำหรับ repository methods
  cacheQuery<TArgs extends unknown[], TResult>(
    fn: (...args: TArgs) => Promise<TResult>,
    keyBuilder: (...args: TArgs) => string,
    ttlSeconds?: number
  ) {
    return async (...args: TArgs): Promise<TResult> => {
      const cacheKey = keyBuilder(...args);
      const cached = await this.redis.get(cacheKey);
      
      if (cached) {
        return JSON.parse(cached) as TResult;
      }
      
      const result = await fn(...args);
      await this.redis.setex(
        cacheKey,
        ttlSeconds ?? this.defaultTTL,
        JSON.stringify(result)
      );
      
      return result;
    };
  }

  // Cache ด้วย tag สำหรับ group invalidation
  async setWithTags(
    key: string,
    value: unknown,
    tags: string[],
    ttlSeconds = this.defaultTTL
  ): Promise<void> {
    const pipeline = this.redis.pipeline();
    
    // บันทึก value
    pipeline.setex(key, ttlSeconds, JSON.stringify(value));
    
    // บันทึก key ไว้ใน tag sets
    for (const tag of tags) {
      pipeline.sadd(`tag:${tag}`, key);
      pipeline.expire(`tag:${tag}`, ttlSeconds + 60);
    }
    
    await pipeline.exec();
  }

  // Invalidate ทุก key ที่มี tag นี้
  async invalidateByTag(tag: string): Promise<void> {
    const keys = await this.redis.smembers(`tag:${tag}`);
    if (keys.length > 0) {
      const pipeline = this.redis.pipeline();
      keys.forEach(key => pipeline.del(key));
      pipeline.del(`tag:${tag}`);
      await pipeline.exec();
      console.log(`Invalidated ${keys.length} keys with tag: ${tag}`);
    }
  }
}

// ใช้งาน tag-based invalidation
const queryCache = new QueryCache(redisClient.getClient());

// Cache account data พร้อม tag
await queryCache.setWithTags(
  `account:${accountId}`,
  accountData,
  [`user:${userId}`, `accounts`],
  300
);

// เมื่อ user update ข้อมูล invalidate ทุก account ของ user นั้น
await queryCache.invalidateByTag(`user:${userId}`);
```

---

## 11. Session Caching

### 11.1 Session Store

```typescript
// src/cache/session.cache.ts
export interface Session {
  id: string;
  userId: string;
  roles: string[];
  permissions: string[];
  deviceInfo: {
    userAgent: string;
    ip: string;
  };
  createdAt: string;
  lastActivityAt: string;
  expiresAt: string;
}

export class SessionCache {
  private readonly SESSION_PREFIX = 'session:';
  private readonly SESSION_TTL = 86400; // 24 ชั่วโมง

  constructor(private readonly redis: Redis) {}

  async create(session: Omit<Session, 'id'>): Promise<Session> {
    const id = `sess-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    const fullSession: Session = { id, ...session };
    
    const key = `${this.SESSION_PREFIX}${id}`;
    await this.redis.setex(key, this.SESSION_TTL, JSON.stringify(fullSession));
    
    // เพิ่ม session ใน user's session set
    await this.redis.sadd(`user:sessions:${session.userId}`, id);
    await this.redis.expire(`user:sessions:${session.userId}`, this.SESSION_TTL);
    
    return fullSession;
  }

  async get(sessionId: string): Promise<Session | null> {
    const raw = await this.redis.get(`${this.SESSION_PREFIX}${sessionId}`);
    if (!raw) return null;
    
    const session = JSON.parse(raw) as Session;
    
    // อัพเดท last activity
    session.lastActivityAt = new Date().toISOString();
    await this.update(sessionId, session);
    
    return session;
  }

  async update(sessionId: string, data: Partial<Session>): Promise<void> {
    const existing = await this.get(sessionId);
    if (!existing) return;
    
    const updated = { ...existing, ...data };
    await this.redis.setex(
      `${this.SESSION_PREFIX}${sessionId}`,
      this.SESSION_TTL,
      JSON.stringify(updated)
    );
  }

  async delete(sessionId: string): Promise<void> {
    const session = await this.get(sessionId);
    if (session) {
      await this.redis.srem(`user:sessions:${session.userId}`, sessionId);
    }
    await this.redis.del(`${this.SESSION_PREFIX}${sessionId}`);
  }

  // Logout จากทุก devices
  async deleteAllUserSessions(userId: string): Promise<void> {
    const sessionIds = await this.redis.smembers(`user:sessions:${userId}`);
    
    const pipeline = this.redis.pipeline();
    sessionIds.forEach(id => {
      pipeline.del(`${this.SESSION_PREFIX}${id}`);
    });
    pipeline.del(`user:sessions:${userId}`);
    
    await pipeline.exec();
    console.log(`ลบ ${sessionIds.length} sessions ของ user: ${userId}`);
  }

  async getUserSessions(userId: string): Promise<Session[]> {
    const sessionIds = await this.redis.smembers(`user:sessions:${userId}`);
    
    const sessions = await Promise.all(
      sessionIds.map(id => this.get(id))
    );
    
    return sessions.filter((s): s is Session => s !== null);
  }
}
```

---

## 12. Cache Monitoring และ Statistics

### 12.1 Cache Stats Dashboard

```typescript
// src/cache/cache.monitor.ts
export class CacheMonitor {
  constructor(private readonly redis: Redis) {}

  async getRedisInfo(): Promise<{
    usedMemory: string;
    maxMemory: string;
    hitRate: number;
    totalKeys: number;
    uptime: string;
  }> {
    const info = await this.redis.info('stats');
    const memoryInfo = await this.redis.info('memory');
    const keyspaceInfo = await this.redis.info('keyspace');
    
    // Parse Redis INFO output
    const parseInfo = (str: string): Record<string, string> => {
      return Object.fromEntries(
        str.split('\r\n')
          .filter(line => line.includes(':'))
          .map(line => line.split(':') as [string, string])
      );
    };

    const stats = parseInfo(info);
    const memory = parseInfo(memoryInfo);

    const hits = parseInt(stats['keyspace_hits'] ?? '0');
    const misses = parseInt(stats['keyspace_misses'] ?? '0');
    const total = hits + misses;

    return {
      usedMemory: memory['used_memory_human'] ?? 'N/A',
      maxMemory: memory['maxmemory_human'] ?? 'N/A',
      hitRate: total > 0 ? hits / total : 0,
      totalKeys: await this.redis.dbsize(),
      uptime: stats['uptime_in_seconds'] ?? '0',
    };
  }

  async getKeysByPattern(pattern: string): Promise<string[]> {
    return this.redis.keys(pattern);
  }

  async getKeyInfo(key: string): Promise<{
    type: string;
    ttl: number;
    size: number;
  }> {
    const [type, ttl, size] = await Promise.all([
      this.redis.type(key),
      this.redis.ttl(key),
      this.redis.memory('USAGE', key) as Promise<number>,
    ]);

    return {
      type,
      ttl,
      size: size ?? 0,
    };
  }
}

// Placeholder metrics
const metrics = {
  cacheHit: (key: string) => {},
  cacheMiss: (key: string) => {},
  l1Hit: (key: string) => {},
  l2Hit: (key: string) => {},
};
```

---

## สรุปบทที่ 89

ในบทนี้เราได้เรียนรู้:

1. **ทำไม Caching ถึงสำคัญ** - ผลกระทบต่อ performance และ scalability
2. **Redis กับ ioredis** - การเชื่อมต่อและกำหนดค่า
3. **Redis Data Structures** - String, Hash, List, Set, Sorted Set
4. **Cache Patterns**:
   - Cache-Aside: ดูก่อน ถ้าไม่มีค่อย fetch
   - Write-Through: เขียน DB และ cache พร้อมกัน
   - Write-Behind: เขียน cache ก่อน DB ทีหลัง
   - Read-Through: cache โปร่งใส
5. **Cache Stampede** - ป้องกันด้วย Mutex Lock
6. **TTL Management** - กำหนด TTL อย่างชาญฉลาดพร้อม jitter
7. **Pub/Sub** - Distributed cache invalidation
8. **In-Memory Cache** - node-cache สำหรับ L1 cache
9. **HTTP Cache Headers** - Cache-Control, ETag
10. **Query Cache** - Cache database queries พร้อม tag-based invalidation
11. **Session Cache** - จัดการ user sessions
12. **Two-Level Cache** - L1 memory + L2 Redis
