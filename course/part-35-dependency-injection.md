# Part 35: Dependency Injection ใน TypeScript

## บทนำ

Dependency Injection (DI) คือ design pattern ที่ช่วยลดการผูกมัด (coupling) ระหว่าง components โดยแทนที่ class จะสร้าง dependencies ของตัวเอง จะรับ dependencies จากภายนอกแทน

### ปัญหาที่ DI แก้ไข

```typescript
// โค้ดที่ไม่ใช้ DI (Tightly Coupled)
class OrderService {
  private db: MySQLDatabase;
  private emailer: SendGridEmailer;
  private cache: RedisCache;

  constructor() {
    // สร้าง dependencies ด้วยตัวเอง
    this.db = new MySQLDatabase("localhost:3306", "mydb");
    this.emailer = new SendGridEmailer("api_key_here");
    this.cache = new RedisCache("localhost:6379");
  }
  
  // ทดสอบยาก เพราะพึ่งพา real database และ services
  async createOrder(items: any[]): Promise<void> {
    await this.db.insert("orders", items);
    await this.emailer.send("customer@example.com", "คำสั่งซื้อสำเร็จ");
  }
}
```

### ข้อเสียของ Tight Coupling

1. ทดสอบยาก (ต้องมี real MySQL, Redis, SendGrid)
2. เปลี่ยน implementation ยาก
3. Reuse ยาก ในบริบทต่างๆ
4. Parallel development ยาก

---

## DI Concept และ Benefits

### ประเภทของ Dependency Injection

1. **Constructor Injection** - ส่ง dependencies ผ่าน constructor
2. **Property Injection** - ส่ง dependencies ผ่าน properties
3. **Method Injection** - ส่ง dependencies ผ่าน method parameters

---

## 1. Constructor Injection

Constructor Injection เป็น pattern ที่แนะนำมากที่สุด เพราะ dependencies ชัดเจนและ required

```typescript
// Interface ของ dependencies
interface IDatabase {
  query<T>(sql: string, params?: any[]): Promise<T[]>;
  execute(sql: string, params?: any[]): Promise<{ affectedRows: number }>;
  transaction<T>(operations: (db: IDatabase) => Promise<T>): Promise<T>;
}

interface IEmailService {
  send(to: string, subject: string, body: string): Promise<void>;
  sendTemplate(to: string, templateId: string, data: any): Promise<void>;
}

interface ILogger {
  info(message: string, meta?: any): void;
  error(message: string, error?: Error): void;
  warn(message: string, meta?: any): void;
}

interface ICacheService {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttlSeconds?: number): Promise<void>;
  delete(key: string): Promise<void>;
}

// Service ที่ใช้ Constructor Injection
class UserService {
  // Constructor รับ dependencies ทั้งหมด
  constructor(
    private readonly db: IDatabase,
    private readonly emailService: IEmailService,
    private readonly logger: ILogger,
    private readonly cache: ICacheService
  ) {}

  async createUser(data: {
    name: string;
    email: string;
    password: string;
  }): Promise<{ id: string; name: string; email: string }> {
    this.logger.info("กำลังสร้างผู้ใช้ใหม่", { email: data.email });

    // ตรวจสอบซ้ำ
    const existing = await this.db.query<any>(
      "SELECT id FROM users WHERE email = ?",
      [data.email]
    );
    
    if (existing.length > 0) {
      throw new Error("อีเมลนี้ถูกใช้งานแล้ว");
    }

    // บันทึกลง database
    const result = await this.db.execute(
      "INSERT INTO users (name, email, password) VALUES (?, ?, ?)",
      [data.name, data.email, data.password]
    );

    const userId = `user-${Date.now()}`;

    // ล้าง cache
    await this.cache.delete(`users:list`);

    // ส่ง welcome email
    await this.emailService.send(
      data.email,
      "ยินดีต้อนรับ!",
      `สวัสดี ${data.name} ขอบคุณที่สมัครสมาชิก`
    );

    this.logger.info("สร้างผู้ใช้สำเร็จ", { userId });

    return { id: userId, name: data.name, email: data.email };
  }

  async getUser(id: string): Promise<any> {
    // ดูจาก cache ก่อน
    const cached = await this.cache.get<any>(`user:${id}`);
    if (cached) {
      this.logger.info("ดึงข้อมูลจาก cache", { userId: id });
      return cached;
    }

    const users = await this.db.query<any>(
      "SELECT * FROM users WHERE id = ?",
      [id]
    );

    if (users.length === 0) {
      return null;
    }

    const user = users[0];
    
    // เก็บใน cache 5 นาที
    await this.cache.set(`user:${id}`, user, 300);

    return user;
  }
}
```

### ตัวอย่าง Mock สำหรับทดสอบ

```typescript
// Mock implementations สำหรับ testing

class MockDatabase implements IDatabase {
  private data: Map<string, any[]> = new Map();
  public queryCalls: string[] = [];
  public executeCalls: string[] = [];

  async query<T>(sql: string, params?: any[]): Promise<T[]> {
    this.queryCalls.push(sql);
    return [];
  }

  async execute(sql: string, params?: any[]): Promise<{ affectedRows: number }> {
    this.executeCalls.push(sql);
    return { affectedRows: 1 };
  }

  async transaction<T>(operations: (db: IDatabase) => Promise<T>): Promise<T> {
    return operations(this);
  }

  // Helper สำหรับ test
  mockQueryResult<T>(result: T[]): void {
    this.query = async () => result as any;
  }
}

class MockEmailService implements IEmailService {
  public sentEmails: Array<{ to: string; subject: string; body: string }> = [];
  public sentTemplates: Array<{ to: string; templateId: string; data: any }> = [];

  async send(to: string, subject: string, body: string): Promise<void> {
    this.sentEmails.push({ to, subject, body });
  }

  async sendTemplate(to: string, templateId: string, data: any): Promise<void> {
    this.sentTemplates.push({ to, templateId, data });
  }
}

class MockLogger implements ILogger {
  public logs: Array<{ level: string; message: string; meta?: any }> = [];

  info(message: string, meta?: any): void {
    this.logs.push({ level: "info", message, meta });
  }

  error(message: string, error?: Error): void {
    this.logs.push({ level: "error", message, meta: error });
  }

  warn(message: string, meta?: any): void {
    this.logs.push({ level: "warn", message, meta });
  }
}

class MockCacheService implements ICacheService {
  private store: Map<string, { value: any; expiry?: number }> = new Map();

  async get<T>(key: string): Promise<T | null> {
    const item = this.store.get(key);
    if (!item) return null;
    if (item.expiry && Date.now() > item.expiry) {
      this.store.delete(key);
      return null;
    }
    return item.value as T;
  }

  async set<T>(key: string, value: T, ttlSeconds?: number): Promise<void> {
    this.store.set(key, {
      value,
      expiry: ttlSeconds ? Date.now() + ttlSeconds * 1000 : undefined,
    });
  }

  async delete(key: string): Promise<void> {
    this.store.delete(key);
  }
}

// Unit test ที่ง่ายมากเพราะใช้ DI
async function testUserService() {
  const mockDb = new MockDatabase();
  const mockEmail = new MockEmailService();
  const mockLogger = new MockLogger();
  const mockCache = new MockCacheService();

  const userService = new UserService(mockDb, mockEmail, mockLogger, mockCache);

  // Test 1: สร้าง user สำเร็จ
  const user = await userService.createUser({
    name: "สมชาย",
    email: "somchai@test.com",
    password: "hashed_password",
  });

  console.log("✅ สร้าง user สำเร็จ:", user);
  console.log("✅ ส่ง email:", mockEmail.sentEmails.length, "ฉบับ");
  console.log("✅ Log entries:", mockLogger.logs.length, "รายการ");
}

testUserService();
```

---

## 2. Property Injection

Property Injection ใช้เมื่อ dependency เป็น optional

```typescript
// Property Injection
class ReportGenerator {
  // Optional dependencies ผ่าน properties
  private _logger: ILogger | null = null;
  private _cache: ICacheService | null = null;

  // Required dependency ผ่าน constructor
  constructor(private readonly db: IDatabase) {}

  // Setters สำหรับ optional dependencies
  set logger(logger: ILogger) {
    this._logger = logger;
  }

  set cache(cache: ICacheService) {
    this._cache = cache;
  }

  async generateSalesReport(fromDate: Date, toDate: Date): Promise<any[]> {
    // ใช้ logger ถ้ามี
    this._logger?.info("กำลังสร้างรายงาน", { fromDate, toDate });

    // ตรวจสอบ cache ก่อน (ถ้ามี)
    const cacheKey = `report:sales:${fromDate.getTime()}:${toDate.getTime()}`;
    if (this._cache) {
      const cached = await this._cache.get<any[]>(cacheKey);
      if (cached) {
        this._logger?.info("ใช้ข้อมูลจาก cache");
        return cached;
      }
    }

    const data = await this.db.query<any>(
      "SELECT * FROM sales WHERE date BETWEEN ? AND ?",
      [fromDate, toDate]
    );

    // บันทึก cache ถ้ามี
    if (this._cache) {
      await this._cache.set(cacheKey, data, 3600);
    }

    return data;
  }
}

// การใช้งาน
const db = new MockDatabase();
const generator = new ReportGenerator(db);

// เพิ่ม optional dependencies
generator.logger = new MockLogger();
generator.cache = new MockCacheService();

generator.generateSalesReport(new Date("2026-01-01"), new Date("2026-12-31"));
```

---

## 3. Method Injection

Method Injection ส่ง dependency เป็น parameter ของ method

```typescript
// Method Injection
class DataProcessor {
  // ส่ง strategy เป็น parameter ของ method
  processData<T, R>(
    data: T[],
    transformer: (item: T) => R,
    validator?: (item: T) => boolean
  ): R[] {
    const filtered = validator ? data.filter(validator) : data;
    return filtered.map(transformer);
  }

  async fetchAndProcess<T, R>(
    fetchFn: () => Promise<T[]>,    // inject fetch logic
    transformer: (item: T) => R,    // inject transform logic
    onError?: (error: Error) => void // inject error handler
  ): Promise<R[]> {
    try {
      const data = await fetchFn();
      return this.processData(data, transformer);
    } catch (error) {
      if (onError) {
        onError(error as Error);
        return [];
      }
      throw error;
    }
  }
}

// การใช้งาน
const processor = new DataProcessor();

interface RawUser { id: number; first_name: string; last_name: string; }
interface FormattedUser { id: number; fullName: string; }

const result = processor.processData<RawUser, FormattedUser>(
  [
    { id: 1, first_name: "สมชาย", last_name: "ใจดี" },
    { id: 2, first_name: "สมหญิง", last_name: "มีสุข" },
  ],
  (user) => ({ id: user.id, fullName: `${user.first_name} ${user.last_name}` }),
  (user) => user.id > 0  // validator
);

console.log(result);
```

---

## 4. DI Container (Manual Implementation)

### สร้าง Simple DI Container

```typescript
// Simple DI Container Implementation

type Constructor<T = any> = new (...args: any[]) => T;
type Factory<T = any> = (...args: any[]) => T;

type Lifetime = "singleton" | "transient" | "scoped";

interface Binding<T> {
  lifetime: Lifetime;
  factory: Factory<T>;
  instance?: T;
}

class Container {
  private bindings: Map<string | symbol, Binding<any>> = new Map();
  private scopedInstances: Map<string | symbol, any> = new Map();

  // Register by interface name
  bind<T>(
    token: string | symbol,
    factory: Factory<T>,
    lifetime: Lifetime = "transient"
  ): this {
    this.bindings.set(token, { lifetime, factory });
    return this;
  }

  // Register singleton
  singleton<T>(token: string | symbol, factory: Factory<T>): this {
    return this.bind(token, factory, "singleton");
  }

  // Register transient (new instance each time)
  transient<T>(token: string | symbol, factory: Factory<T>): this {
    return this.bind(token, factory, "transient");
  }

  // Register scoped (same instance within scope)
  scoped<T>(token: string | symbol, factory: Factory<T>): this {
    return this.bind(token, factory, "scoped");
  }

  // Resolve dependency
  resolve<T>(token: string | symbol): T {
    const binding = this.bindings.get(token);
    
    if (!binding) {
      throw new Error(`ไม่พบ binding สำหรับ token: ${String(token)}`);
    }

    switch (binding.lifetime) {
      case "singleton":
        if (!binding.instance) {
          binding.instance = binding.factory(this);
        }
        return binding.instance as T;

      case "scoped":
        if (!this.scopedInstances.has(token)) {
          this.scopedInstances.set(token, binding.factory(this));
        }
        return this.scopedInstances.get(token) as T;

      case "transient":
      default:
        return binding.factory(this) as T;
    }
  }

  // Create a new scope
  createScope(): Container {
    const scope = new Container();
    scope.bindings = this.bindings;
    scope.scopedInstances = new Map(); // Fresh scope instances
    return scope;
  }

  // Clear scoped instances
  clearScope(): void {
    this.scopedInstances.clear();
  }

  has(token: string | symbol): boolean {
    return this.bindings.has(token);
  }
}

// Tokens (type-safe keys)
const TOKENS = {
  Database: Symbol("IDatabase"),
  EmailService: Symbol("IEmailService"),
  Logger: Symbol("ILogger"),
  Cache: Symbol("ICacheService"),
  UserRepository: Symbol("IUserRepository"),
  UserService: Symbol("UserService"),
  ProductService: Symbol("ProductService"),
} as const;

// Concrete implementations
class MySQLDatabase implements IDatabase {
  constructor(private connectionString: string) {
    console.log(`[MySQL] เชื่อมต่อ: ${connectionString}`);
  }

  async query<T>(sql: string, params?: any[]): Promise<T[]> {
    console.log(`[MySQL] Query: ${sql}`);
    return [];
  }

  async execute(sql: string, params?: any[]): Promise<{ affectedRows: number }> {
    console.log(`[MySQL] Execute: ${sql}`);
    return { affectedRows: 1 };
  }

  async transaction<T>(operations: (db: IDatabase) => Promise<T>): Promise<T> {
    console.log("[MySQL] Begin Transaction");
    try {
      const result = await operations(this);
      console.log("[MySQL] Commit");
      return result;
    } catch (error) {
      console.log("[MySQL] Rollback");
      throw error;
    }
  }
}

class ProductionEmailService implements IEmailService {
  constructor(private apiKey: string) {}

  async send(to: string, subject: string, body: string): Promise<void> {
    console.log(`[Email] ส่งไปที่ ${to}: ${subject}`);
  }

  async sendTemplate(to: string, templateId: string, data: any): Promise<void> {
    console.log(`[Email] ส่ง template ${templateId} ไปที่ ${to}`);
  }
}

class ConsoleLogger implements ILogger {
  info(message: string, meta?: any): void {
    console.log(`ℹ️ [INFO] ${message}`, meta || "");
  }

  error(message: string, error?: Error): void {
    console.error(`❌ [ERROR] ${message}`, error?.message || "");
  }

  warn(message: string, meta?: any): void {
    console.warn(`⚠️ [WARN] ${message}`, meta || "");
  }
}

class RedisCache implements ICacheService {
  private store: Map<string, any> = new Map();

  constructor(private redisUrl: string) {
    console.log(`[Redis] เชื่อมต่อ: ${redisUrl}`);
  }

  async get<T>(key: string): Promise<T | null> {
    return this.store.get(key) || null;
  }

  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    this.store.set(key, value);
  }

  async delete(key: string): Promise<void> {
    this.store.delete(key);
  }
}

// สร้างและ configure container
function createContainer(config: {
  dbConnection: string;
  emailApiKey: string;
  redisUrl: string;
}): Container {
  const container = new Container();

  // Register infrastructure
  container.singleton(TOKENS.Database, () =>
    new MySQLDatabase(config.dbConnection)
  );

  container.singleton(TOKENS.EmailService, () =>
    new ProductionEmailService(config.emailApiKey)
  );

  container.singleton(TOKENS.Logger, () =>
    new ConsoleLogger()
  );

  container.singleton(TOKENS.Cache, () =>
    new RedisCache(config.redisUrl)
  );

  // Register services
  container.transient(TOKENS.UserService, (c) =>
    new UserService(
      c.resolve<IDatabase>(TOKENS.Database),
      c.resolve<IEmailService>(TOKENS.EmailService),
      c.resolve<ILogger>(TOKENS.Logger),
      c.resolve<ICacheService>(TOKENS.Cache)
    )
  );

  return container;
}

// การใช้งาน
const container = createContainer({
  dbConnection: "mysql://localhost:3306/mydb",
  emailApiKey: "sendgrid_api_key",
  redisUrl: "redis://localhost:6379",
});

const userService1 = container.resolve<UserService>(TOKENS.UserService);
const userService2 = container.resolve<UserService>(TOKENS.UserService);

// transient = สร้างใหม่ทุกครั้ง
console.log("UserService instances เหมือนกัน:", userService1 === userService2); // false

const logger1 = container.resolve<ILogger>(TOKENS.Logger);
const logger2 = container.resolve<ILogger>(TOKENS.Logger);

// singleton = instance เดียวกัน
console.log("Logger instances เหมือนกัน:", logger1 === logger2); // true
```

---

## 5. InversifyJS

InversifyJS คือ DI Container ที่ powerful ที่สุดสำหรับ TypeScript

### การติดตั้ง InversifyJS

```bash
npm install inversify reflect-metadata
npm install --save-dev @types/node
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "lib": ["es2015", "dom"],
    "module": "commonjs",
    "moduleResolution": "node",
    "target": "es5",
    "strict": true
  }
}
```

### Basic Inversify Setup

```typescript
// inversify.config.ts
import "reflect-metadata";
import { Container, injectable, inject, interfaces } from "inversify";

// Define tokens
const INVERSIFY_TYPES = {
  IDatabase: Symbol.for("IDatabase"),
  IEmailService: Symbol.for("IEmailService"),
  ILogger: Symbol.for("ILogger"),
  ICacheService: Symbol.for("ICacheService"),
  IUserRepository: Symbol.for("IUserRepository"),
  UserService: Symbol.for("UserService"),
  OrderService: Symbol.for("OrderService"),
};

// Apply decorators to classes

@injectable()
class InversifyMySQLDatabase implements IDatabase {
  private connectionString: string;

  constructor() {
    this.connectionString = process.env.DATABASE_URL || "mysql://localhost/mydb";
    console.log(`[MySQL] เชื่อมต่อด้วย InversifyJS: ${this.connectionString}`);
  }

  async query<T>(sql: string, params?: any[]): Promise<T[]> {
    console.log(`[MySQL] Query: ${sql}`);
    return [];
  }

  async execute(sql: string, params?: any[]): Promise<{ affectedRows: number }> {
    return { affectedRows: 1 };
  }

  async transaction<T>(operations: (db: IDatabase) => Promise<T>): Promise<T> {
    return operations(this);
  }
}

@injectable()
class InversifyEmailService implements IEmailService {
  async send(to: string, subject: string, body: string): Promise<void> {
    console.log(`[Email] ส่งถึง ${to}: ${subject}`);
  }

  async sendTemplate(to: string, templateId: string, data: any): Promise<void> {
    console.log(`[Email] Template ${templateId} ถึง ${to}`);
  }
}

@injectable()
class InversifyLogger implements ILogger {
  info(message: string, meta?: any): void {
    console.log(`ℹ️ ${message}`, meta || "");
  }

  error(message: string, error?: Error): void {
    console.error(`❌ ${message}`, error?.message || "");
  }

  warn(message: string, meta?: any): void {
    console.warn(`⚠️ ${message}`, meta || "");
  }
}

@injectable()
class InversifyUserService {
  constructor(
    @inject(INVERSIFY_TYPES.IDatabase) private db: IDatabase,
    @inject(INVERSIFY_TYPES.IEmailService) private emailService: IEmailService,
    @inject(INVERSIFY_TYPES.ILogger) private logger: ILogger
  ) {
    this.logger.info("UserService initialized ด้วย InversifyJS");
  }

  async createUser(data: { name: string; email: string }): Promise<void> {
    this.logger.info("สร้างผู้ใช้", { name: data.name });
    await this.db.execute("INSERT INTO users VALUES (?, ?)", [data.name, data.email]);
    await this.emailService.send(data.email, "ยินดีต้อนรับ!", `สวัสดี ${data.name}`);
  }
}

// Configure Container
const inversifyContainer = new Container();

inversifyContainer
  .bind<IDatabase>(INVERSIFY_TYPES.IDatabase)
  .to(InversifyMySQLDatabase)
  .inSingletonScope();

inversifyContainer
  .bind<IEmailService>(INVERSIFY_TYPES.IEmailService)
  .to(InversifyEmailService)
  .inSingletonScope();

inversifyContainer
  .bind<ILogger>(INVERSIFY_TYPES.ILogger)
  .to(InversifyLogger)
  .inSingletonScope();

inversifyContainer
  .bind<InversifyUserService>(INVERSIFY_TYPES.UserService)
  .to(InversifyUserService)
  .inTransientScope();

// ใช้งาน
const userSvc = inversifyContainer.get<InversifyUserService>(INVERSIFY_TYPES.UserService);
userSvc.createUser({ name: "สมชาย", email: "somchai@example.com" });
```

### Inversify Modules

```typescript
// ใช้ ContainerModule เพื่อจัดระเบียบ bindings

import { ContainerModule } from "inversify";

// Infrastructure Module
const infrastructureModule = new ContainerModule((bind) => {
  bind<IDatabase>(INVERSIFY_TYPES.IDatabase)
    .to(InversifyMySQLDatabase)
    .inSingletonScope();

  bind<IEmailService>(INVERSIFY_TYPES.IEmailService)
    .to(InversifyEmailService)
    .inSingletonScope();

  bind<ILogger>(INVERSIFY_TYPES.ILogger)
    .to(InversifyLogger)
    .inSingletonScope();
});

// Application Module
const applicationModule = new ContainerModule((bind) => {
  bind<InversifyUserService>(INVERSIFY_TYPES.UserService)
    .to(InversifyUserService)
    .inTransientScope();
});

// สร้าง container จาก modules
const modularContainer = new Container();
modularContainer.load(infrastructureModule, applicationModule);

const svc = modularContainer.get<InversifyUserService>(INVERSIFY_TYPES.UserService);
```

### Factory Injection ใน Inversify

```typescript
// Factory binding

@injectable()
class DatabaseConnectionFactory {
  create(connectionString: string): IDatabase {
    if (connectionString.startsWith("mysql://")) {
      return new InversifyMySQLDatabase();
    }
    throw new Error(`ไม่รองรับ connection string: ${connectionString}`);
  }
}

// Register factory
inversifyContainer
  .bind<interfaces.Factory<IDatabase>>("Factory<IDatabase>")
  .toFactory<IDatabase>((context) => {
    return (connectionString: string) => {
      const factory = context.container.get<DatabaseConnectionFactory>(
        "DatabaseConnectionFactory"
      );
      return factory.create(connectionString);
    };
  });
```

---

## 6. TSyringe

TSyringe คือ lightweight DI container ของ Microsoft

### การติดตั้ง TSyringe

```bash
npm install tsyringe reflect-metadata
```

### TSyringe Basic Usage

```typescript
// tsyringe.config.ts
import "reflect-metadata";
import { container, injectable, inject, singleton } from "tsyringe";

// Injectable classes
@singleton()
class TSyringeMySQLDatabase implements IDatabase {
  async query<T>(sql: string, params?: any[]): Promise<T[]> {
    console.log(`[TSyringe MySQL] ${sql}`);
    return [];
  }

  async execute(sql: string, params?: any[]): Promise<{ affectedRows: number }> {
    return { affectedRows: 1 };
  }

  async transaction<T>(ops: (db: IDatabase) => Promise<T>): Promise<T> {
    return ops(this);
  }
}

@singleton()
class TSyringeEmailService implements IEmailService {
  async send(to: string, subject: string, body: string): Promise<void> {
    console.log(`[TSyringe Email] ${to}: ${subject}`);
  }

  async sendTemplate(to: string, templateId: string, data: any): Promise<void> {
    console.log(`[TSyringe Email Template] ${templateId} to ${to}`);
  }
}

@injectable()
class TSyringeUserService {
  constructor(
    @inject("IDatabase") private db: IDatabase,
    @inject("IEmailService") private emailService: IEmailService
  ) {}

  async getUsers(): Promise<any[]> {
    return this.db.query("SELECT * FROM users");
  }
}

// Register
container.register<IDatabase>("IDatabase", {
  useClass: TSyringeMySQLDatabase,
});

container.register<IEmailService>("IEmailService", {
  useClass: TSyringeEmailService,
});

// Resolve
const userSvcFromTSyringe = container.resolve(TSyringeUserService);
userSvcFromTSyringe.getUsers();
```

---

## 7. Advanced DI Patterns

### Decorator-based Middleware

```typescript
// ใช้ DI กับ middleware decorators

function Logged(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  
  descriptor.value = async function (...args: any[]) {
    const logger = (this as any).logger;
    logger?.info(`เรียกใช้ ${target.constructor.name}.${propertyKey}`);
    
    const start = Date.now();
    try {
      const result = await originalMethod.apply(this, args);
      logger?.info(`${propertyKey} สำเร็จ (${Date.now() - start}ms)`);
      return result;
    } catch (error) {
      logger?.error(`${propertyKey} ล้มเหลว`, error as Error);
      throw error;
    }
  };
  
  return descriptor;
}

function Cached(ttlSeconds: number = 300) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const cache = (this as any).cache;
      if (!cache) return originalMethod.apply(this, args);
      
      const cacheKey = `${target.constructor.name}:${propertyKey}:${JSON.stringify(args)}`;
      
      const cached = await cache.get(cacheKey);
      if (cached !== null) {
        console.log(`[Cache HIT] ${cacheKey}`);
        return cached;
      }
      
      const result = await originalMethod.apply(this, args);
      await cache.set(cacheKey, result, ttlSeconds);
      console.log(`[Cache SET] ${cacheKey}`);
      
      return result;
    };
    
    return descriptor;
  };
}

// Service ที่ใช้ decorators
class ProductService {
  constructor(
    private readonly db: IDatabase,
    private readonly logger: ILogger,
    private readonly cache: ICacheService
  ) {}

  @Logged
  @Cached(600) // cache 10 นาที
  async getProducts(category?: string): Promise<any[]> {
    const sql = category
      ? "SELECT * FROM products WHERE category = ?"
      : "SELECT * FROM products";
    const params = category ? [category] : [];
    return this.db.query(sql, params);
  }

  @Logged
  async createProduct(data: { name: string; price: number; category: string }): Promise<any> {
    const result = await this.db.execute(
      "INSERT INTO products (name, price, category) VALUES (?, ?, ?)",
      [data.name, data.price, data.category]
    );
    
    // ล้าง cache
    await this.cache.delete(`ProductService:getProducts:`);
    
    return { id: `prod-${Date.now()}`, ...data };
  }
}
```

### Circular Dependency Resolution

```typescript
// แก้ปัญหา Circular Dependency

// ปัญหา: A ต้องการ B, B ต้องการ A
// แก้ไขโดยใช้ lazy injection หรือ Mediator

// Pattern 1: ใช้ Provider
type Provider<T> = () => T;

class ServiceA {
  private _serviceB: ServiceB | null = null;

  // รับ Provider แทน ServiceB โดยตรง
  constructor(private serviceBProvider: Provider<ServiceB>) {}

  private get serviceB(): ServiceB {
    if (!this._serviceB) {
      this._serviceB = this.serviceBProvider();
    }
    return this._serviceB;
  }

  doSomething(): string {
    return `A + ${this.serviceB.doSomethingElse()}`;
  }
}

class ServiceB {
  constructor(private serviceA: ServiceA) {}

  doSomethingElse(): string {
    return "B";
  }
}

// Container handles circular dependency
class CircularContainer {
  private instances: Map<string, any> = new Map();

  resolveServiceA(): ServiceA {
    if (!this.instances.has("A")) {
      const serviceA = new ServiceA(() => this.resolveServiceB());
      this.instances.set("A", serviceA);
    }
    return this.instances.get("A");
  }

  resolveServiceB(): ServiceB {
    if (!this.instances.has("B")) {
      const serviceB = new ServiceB(this.resolveServiceA());
      this.instances.set("B", serviceB);
    }
    return this.instances.get("B");
  }
}

const circularContainer = new CircularContainer();
const serviceA = circularContainer.resolveServiceA();
console.log(serviceA.doSomething()); // "A + B"
```

---

## 8. DI ใน Testing

### Test Container Configuration

```typescript
// test/container.ts - Test container setup

function createTestContainer(overrides: Partial<{
  database: IDatabase;
  emailService: IEmailService;
  logger: ILogger;
  cache: ICacheService;
}> = {}): Container {
  const testContainer = new Container();

  // ใช้ mock implementations เป็น default
  testContainer.singleton(
    TOKENS.Database,
    () => overrides.database || new MockDatabase()
  );

  testContainer.singleton(
    TOKENS.EmailService,
    () => overrides.emailService || new MockEmailService()
  );

  testContainer.singleton(
    TOKENS.Logger,
    () => overrides.logger || new MockLogger()
  );

  testContainer.singleton(
    TOKENS.Cache,
    () => overrides.cache || new MockCacheService()
  );

  testContainer.transient(
    TOKENS.UserService,
    (c) => new UserService(
      c.resolve<IDatabase>(TOKENS.Database),
      c.resolve<IEmailService>(TOKENS.EmailService),
      c.resolve<ILogger>(TOKENS.Logger),
      c.resolve<ICacheService>(TOKENS.Cache)
    )
  );

  return testContainer;
}

// ตัวอย่าง test suite
describe("UserService Integration Tests", () => {
  let testContainer: Container;
  let mockDb: MockDatabase;
  let mockEmail: MockEmailService;

  beforeEach(() => {
    mockDb = new MockDatabase();
    mockEmail = new MockEmailService();

    testContainer = createTestContainer({
      database: mockDb,
      emailService: mockEmail,
    });
  });

  it("สร้าง user และส่ง welcome email", async () => {
    const userService = testContainer.resolve<UserService>(TOKENS.UserService);

    await userService.createUser({
      name: "สมชาย",
      email: "test@example.com",
      password: "password",
    });

    // ตรวจสอบว่าส่ง email แล้ว
    const emailsSent = mockEmail.sentEmails;
    expect(emailsSent.length).toBe(1);
    expect(emailsSent[0].to).toBe("test@example.com");
  });

  it("บันทึกข้อมูลลง database", async () => {
    const userService = testContainer.resolve<UserService>(TOKENS.UserService);

    await userService.createUser({
      name: "สมหญิง",
      email: "test2@example.com",
      password: "password",
    });

    expect(mockDb.executeCalls.length).toBeGreaterThan(0);
    expect(mockDb.executeCalls[0]).toContain("INSERT");
  });
});
```

---

## 9. Real-world DI Example: Express.js App

```typescript
// Complete Express.js application with DI

// Domain
interface IProduct {
  id: string;
  name: string;
  price: number;
  stock: number;
}

interface IProductRepository {
  findAll(): Promise<IProduct[]>;
  findById(id: string): Promise<IProduct | null>;
  save(product: IProduct): Promise<void>;
  update(id: string, data: Partial<IProduct>): Promise<void>;
}

// Application
class ProductService {
  constructor(
    private readonly productRepo: IProductRepository,
    private readonly logger: ILogger
  ) {}

  async getAllProducts(): Promise<IProduct[]> {
    this.logger.info("ดึงสินค้าทั้งหมด");
    return this.productRepo.findAll();
  }

  async getProductById(id: string): Promise<IProduct | null> {
    this.logger.info(`ดึงสินค้า ID: ${id}`);
    return this.productRepo.findById(id);
  }

  async createProduct(data: Omit<IProduct, "id">): Promise<IProduct> {
    const product: IProduct = {
      id: `prod-${Date.now()}`,
      ...data,
    };
    
    await this.productRepo.save(product);
    this.logger.info(`สร้างสินค้า: ${product.name}`);
    
    return product;
  }

  async updateStock(id: string, quantity: number): Promise<void> {
    const product = await this.productRepo.findById(id);
    if (!product) {
      throw new Error(`ไม่พบสินค้า: ${id}`);
    }
    
    const newStock = product.stock + quantity;
    if (newStock < 0) {
      throw new Error("สินค้าไม่เพียงพอ");
    }
    
    await this.productRepo.update(id, { stock: newStock });
    this.logger.info(`อัพเดต stock สินค้า ${id}: ${newStock}`);
  }
}

// Infrastructure
class InMemoryProductRepository implements IProductRepository {
  private products: Map<string, IProduct> = new Map([
    ["1", { id: "1", name: "MacBook Pro", price: 89000, stock: 10 }],
    ["2", { id: "2", name: "iPhone 15", price: 45000, stock: 50 }],
  ]);

  async findAll(): Promise<IProduct[]> {
    return [...this.products.values()];
  }

  async findById(id: string): Promise<IProduct | null> {
    return this.products.get(id) || null;
  }

  async save(product: IProduct): Promise<void> {
    this.products.set(product.id, product);
  }

  async update(id: string, data: Partial<IProduct>): Promise<void> {
    const existing = this.products.get(id);
    if (existing) {
      this.products.set(id, { ...existing, ...data });
    }
  }
}

// Controller
class ProductController {
  constructor(private productService: ProductService) {}

  async getAll(req: any, res: any): Promise<void> {
    try {
      const products = await this.productService.getAllProducts();
      res.status(200).json({ success: true, data: products });
    } catch (error) {
      res.status(500).json({ success: false, error: (error as Error).message });
    }
  }

  async getById(req: any, res: any): Promise<void> {
    try {
      const product = await this.productService.getProductById(req.params.id);
      if (!product) {
        res.status(404).json({ success: false, error: "ไม่พบสินค้า" });
        return;
      }
      res.status(200).json({ success: true, data: product });
    } catch (error) {
      res.status(500).json({ success: false, error: (error as Error).message });
    }
  }

  async create(req: any, res: any): Promise<void> {
    try {
      const product = await this.productService.createProduct(req.body);
      res.status(201).json({ success: true, data: product });
    } catch (error) {
      res.status(400).json({ success: false, error: (error as Error).message });
    }
  }
}

// Composition Root
function buildApp() {
  // Infrastructure
  const productRepo = new InMemoryProductRepository();
  const logger = new ConsoleLogger();

  // Application
  const productService = new ProductService(productRepo, logger);

  // Controllers
  const productController = new ProductController(productService);

  return { productController };
}

// Demo
async function runApp() {
  const { productController } = buildApp();

  const mockRes = {
    statusCode: 200,
    body: null,
    status(code: number) { this.statusCode = code; return this; },
    json(data: any) { this.body = data; console.log(`Response ${this.statusCode}:`, JSON.stringify(data, null, 2)); },
  };

  await productController.getAll({}, mockRes);
  await productController.getById({ params: { id: "1" } }, mockRes);
  await productController.create(
    { body: { name: "AirPods Pro", price: 9500, stock: 30 } },
    mockRes
  );
}

runApp();
```

---

## 10. DI Best Practices

### Do's และ Don'ts

```typescript
// ✅ DO: ใช้ interfaces แทน concrete classes

interface IUserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

class GoodUserService {
  constructor(
    private userRepo: IUserRepository // ✅ รับ interface
  ) {}
}

// ❌ DON'T: รับ concrete class โดยตรง
class BadUserService {
  constructor(
    private userRepo: MySQLUserRepository // ❌ ผูกกับ MySQL
  ) {}
}

// ✅ DO: Composition Root ที่เดียว
// ❌ DON'T: สร้าง Container หลายที่

// ✅ DO: ทดสอบด้วย mock dependencies
async function testableExample() {
  const mockRepo: IUserRepository = {
    findById: async () => null,
    save: async () => {},
  };
  
  const service = new GoodUserService(mockRepo);
  // ทดสอบได้ง่าย!
}

// ✅ DO: ใช้ factory functions สำหรับ complex configuration
function createProductionContainer(): Container {
  const c = new Container();
  
  c.singleton(TOKENS.Database, () =>
    new MySQLDatabase(process.env.DATABASE_URL!)
  );
  
  c.singleton(TOKENS.Cache, () =>
    new RedisCache(process.env.REDIS_URL!)
  );
  
  return c;
}

function createTestContainer(): Container {
  const c = new Container();
  
  c.singleton(TOKENS.Database, () => new MockDatabase());
  c.singleton(TOKENS.Cache, () => new MockCacheService());
  
  return c;
}
```

### Service Locator vs DI

```typescript
// ❌ Anti-pattern: Service Locator
class BadService {
  doWork(): void {
    // ดึง dependency จาก global location
    const db = GlobalServiceLocator.get("IDatabase");
    db.query("SELECT 1");
  }
}

// ✅ Good: Dependency Injection
class GoodService {
  constructor(private db: IDatabase) {} // ประกาศ dependencies ชัดเจน
  
  doWork(): void {
    this.db.query("SELECT 1");
  }
}
```

---

## สรุป

### เปรียบเทียบ DI Libraries

| Feature | Manual DI | InversifyJS | TSyringe |
|---------|-----------|-------------|---------|
| Type Safety | ✅ | ✅✅ | ✅✅ |
| Decorators | ❌ | ✅ | ✅ |
| Scope Support | ✅ | ✅✅ | ✅ |
| Module System | ❌ | ✅ | ❌ |
| Bundle Size | 0 KB | ~20 KB | ~5 KB |
| Learning Curve | ต่ำ | สูง | กลาง |
| Use Case | โปรเจกต์เล็ก | โปรเจกต์ใหญ่ | โปรเจกต์กลาง |

### เมื่อไหรควรใช้ DI

1. **ควรใช้เมื่อ:**
   - โปรเจกต์มีขนาดปานกลาง-ใหญ่
   - ต้องการ testability สูง
   - มีหลาย implementation (prod/test/mock)
   - ทีมใหญ่ที่ต้องการ separation of concerns

2. **อาจไม่จำเป็นเมื่อ:**
   - Scripts ขนาดเล็ก
   - Prototype ที่ทดสอบด่วน
   - Functions ล้วนๆ ที่ไม่มี state

## แบบฝึกหัด

1. สร้าง DI Container ของตัวเองที่รองรับ child containers
2. ใช้ InversifyJS กับ Express.js application จริง
3. สร้าง test doubles ด้วย DI สำหรับ HTTP client
4. Implement Decorator-based caching และ logging ด้วย DI
5. แก้ปัญหา circular dependency ด้วย Provider pattern
6. สร้าง feature flags system ด้วย DI
7. Migrate legacy code ที่ใช้ Service Locator ไปเป็น DI
8. สร้าง multi-tenant application ด้วย scoped DI containers
