# ตอนที่ 38: NestJS Services & Providers

## แนวคิด Provider (Provider Concept)

Provider คือหัวใจสำคัญของ NestJS Dependency Injection system ทุกอย่างที่สามารถ inject ได้คือ provider รวมถึง services, repositories, factories, helpers และอื่นๆ

```typescript
// Provider คือ class ที่มี @Injectable() decorator
import { Injectable } from '@nestjs/common';

@Injectable()
export class UserService {
  // ...
}

// Provider ถูกลงทะเบียนใน module
@Module({
  providers: [UserService],
})
export class UserModule {}

// Provider สามารถ inject เข้า constructor ได้
@Injectable()
export class OrderService {
  constructor(private userService: UserService) {}
}
```

---

## @Injectable Decorator

`@Injectable()` เป็น decorator ที่บอก NestJS ว่า class นี้สามารถ inject ได้ผ่าน Dependency Injection system

```typescript
import { Injectable, Scope } from '@nestjs/common';

// Singleton (default) - สร้างครั้งเดียวต่อ application lifecycle
@Injectable()
export class SingletonService {
  private count = 0;

  increment() {
    return ++this.count;
  }
}

// Request-scoped - สร้างใหม่ทุก HTTP request
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {
  private requestId: string;

  constructor() {
    this.requestId = Math.random().toString(36).substr(2, 9);
  }

  getRequestId() {
    return this.requestId;
  }
}

// Transient - สร้างใหม่ทุกครั้งที่ inject
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {
  private instanceId: string;

  constructor() {
    this.instanceId = Math.random().toString(36).substr(2, 9);
  }

  getInstanceId() {
    return this.instanceId;
  }
}
```

---

## การสร้าง Service (Service Creation)

### Service พื้นฐาน
```typescript
// src/users/users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
  BadRequestException,
} from '@nestjs/common';
import * as bcrypt from 'bcrypt';
import { v4 as uuidv4 } from 'uuid';

interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  role: string;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

interface CreateUserData {
  name: string;
  email: string;
  password: string;
  role?: string;
}

@Injectable()
export class UsersService {
  private users: User[] = [];

  async findAll(): Promise<Omit<User, 'password'>[]> {
    return this.users.map(({ password, ...user }) => user);
  }

  async findById(id: string): Promise<User> {
    const user = this.users.find(u => u.id === id);
    if (!user) {
      throw new NotFoundException(`ไม่พบผู้ใช้ ID: ${id}`);
    }
    return user;
  }

  async findByEmail(email: string): Promise<User | undefined> {
    return this.users.find(u => u.email === email);
  }

  async create(data: CreateUserData): Promise<Omit<User, 'password'>> {
    const existing = await this.findByEmail(data.email);
    if (existing) {
      throw new ConflictException('Email นี้ถูกใช้แล้ว');
    }

    const hashedPassword = await bcrypt.hash(data.password, 10);
    const user: User = {
      id: uuidv4(),
      name: data.name,
      email: data.email.toLowerCase(),
      password: hashedPassword,
      role: data.role || 'user',
      isActive: true,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.users.push(user);
    const { password, ...result } = user;
    return result;
  }

  async update(id: string, data: Partial<CreateUserData>): Promise<Omit<User, 'password'>> {
    const user = await this.findById(id);
    const index = this.users.findIndex(u => u.id === id);

    if (data.password) {
      data.password = await bcrypt.hash(data.password, 10);
    }

    if (data.email && data.email !== user.email) {
      const existing = await this.findByEmail(data.email);
      if (existing) {
        throw new ConflictException('Email นี้ถูกใช้แล้ว');
      }
    }

    this.users[index] = {
      ...user,
      ...data,
      updatedAt: new Date(),
    };

    const { password, ...result } = this.users[index];
    return result;
  }

  async delete(id: string): Promise<void> {
    await this.findById(id); // ตรวจสอบว่ามีอยู่
    this.users = this.users.filter(u => u.id !== id);
  }

  async validatePassword(email: string, password: string): Promise<User | null> {
    const user = await this.findByEmail(email);
    if (!user) return null;

    const isMatch = await bcrypt.compare(password, user.password);
    return isMatch ? user : null;
  }

  async deactivate(id: string): Promise<void> {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) {
      throw new NotFoundException(`ไม่พบผู้ใช้ ID: ${id}`);
    }
    this.users[index].isActive = false;
  }
}
```

---

## Repository Injection (การ Inject Repository)

```typescript
// src/products/product.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, FindManyOptions, FindOneOptions } from 'typeorm';
import { Product } from './entities/product.entity';

@Injectable()
export class ProductRepository {
  constructor(
    @InjectRepository(Product)
    private readonly repository: Repository<Product>,
  ) {}

  async findAll(options?: FindManyOptions<Product>): Promise<Product[]> {
    return this.repository.find(options);
  }

  async findById(id: number): Promise<Product | null> {
    return this.repository.findOne({ where: { id } });
  }

  async findBySlug(slug: string): Promise<Product | null> {
    return this.repository.findOne({ where: { slug } });
  }

  async create(data: Partial<Product>): Promise<Product> {
    const product = this.repository.create(data);
    return this.repository.save(product);
  }

  async update(id: number, data: Partial<Product>): Promise<Product> {
    await this.repository.update(id, data);
    return this.findById(id);
  }

  async delete(id: number): Promise<void> {
    await this.repository.delete(id);
  }

  async count(options?: FindManyOptions<Product>): Promise<number> {
    return this.repository.count(options);
  }

  async findWithPagination(
    page: number,
    limit: number,
    options?: FindManyOptions<Product>,
  ): Promise<{ items: Product[]; total: number }> {
    const [items, total] = await this.repository.findAndCount({
      ...options,
      skip: (page - 1) * limit,
      take: limit,
    });
    return { items, total };
  }
}

// src/products/products.service.ts (ใช้ ProductRepository)
@Injectable()
export class ProductsService {
  constructor(private productRepository: ProductRepository) {}

  async findAll(page: number, limit: number) {
    const { items, total } = await this.productRepository.findWithPagination(
      page,
      limit,
      { order: { createdAt: 'DESC' } },
    );
    return {
      data: items,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }
}
```

---

## Custom Providers (Provider แบบ Custom)

### Value Providers
```typescript
// ใช้ useValue เพื่อ provide ค่าคงที่
@Module({
  providers: [
    // provide constant value
    {
      provide: 'APP_CONFIG',
      useValue: {
        appName: 'MyApp',
        version: '1.0.0',
        environment: process.env.NODE_ENV || 'development',
      },
    },
    // provide third-party library
    {
      provide: 'HTTP_CLIENT',
      useValue: axios.create({
        baseURL: 'https://api.example.com',
        timeout: 5000,
      }),
    },
    // provide connection options
    {
      provide: 'DATABASE_OPTIONS',
      useValue: {
        host: 'localhost',
        port: 5432,
        database: 'mydb',
      },
    },
  ],
})
export class AppModule {}

// การ inject ค่าเหล่านี้
@Injectable()
export class AppService {
  constructor(
    @Inject('APP_CONFIG') private config: any,
    @Inject('HTTP_CLIENT') private httpClient: any,
    @Inject('DATABASE_OPTIONS') private dbOptions: any,
  ) {
    console.log(`กำลังรันใน ${this.config.environment}`);
  }
}
```

### Factory Providers
```typescript
// ใช้ useFactory เพื่อสร้าง provider แบบ dynamic
import { Module, FactoryProvider } from '@nestjs/common';
import { ConfigService } from './config.service';

const databaseProvider: FactoryProvider = {
  provide: 'DATABASE_CONNECTION',
  useFactory: async (configService: ConfigService) => {
    const dbConfig = configService.get('database');
    // สร้าง connection จริง
    const connection = {
      host: dbConfig.host,
      port: dbConfig.port,
      database: dbConfig.name,
      connected: true,
    };
    console.log(`เชื่อมต่อ database ที่ ${dbConfig.host}`);
    return connection;
  },
  inject: [ConfigService],  // inject dependencies เข้า factory
};

const cacheProvider: FactoryProvider = {
  provide: 'CACHE_MANAGER',
  useFactory: async (configService: ConfigService) => {
    const redisConfig = configService.get('redis');
    return {
      host: redisConfig.host,
      port: redisConfig.port,
      connected: true,
    };
  },
  inject: [ConfigService],
};

@Module({
  providers: [
    ConfigService,
    databaseProvider,
    cacheProvider,
  ],
})
export class DatabaseModule {}

// การใช้งาน factory provider ที่ซับซ้อนขึ้น
@Module({
  providers: [
    {
      provide: 'PAYMENT_SERVICE',
      useFactory: (configService: ConfigService, loggerService: LoggerService) => {
        const apiKey = configService.get('PAYMENT_API_KEY');
        const environment = configService.get('NODE_ENV');
        
        return {
          apiKey,
          baseUrl: environment === 'production'
            ? 'https://api.payment.com'
            : 'https://sandbox.payment.com',
          logger: loggerService,
          charge: async (amount: number, currency: string) => {
            loggerService.log(`ชาร์จ ${amount} ${currency}`);
            return { success: true, transactionId: uuidv4() };
          },
        };
      },
      inject: [ConfigService, LoggerService],
    },
  ],
})
export class PaymentModule {}
```

### Class Providers (useClass)
```typescript
// ใช้ useClass เพื่อ override default implementation
interface IEmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

@Injectable()
class RealEmailService implements IEmailService {
  async sendEmail(to: string, subject: string, body: string) {
    // ส่ง email จริงผ่าน SMTP
    console.log(`ส่ง email จริงไปยัง ${to}`);
  }
}

@Injectable()
class MockEmailService implements IEmailService {
  async sendEmail(to: string, subject: string, body: string) {
    // Mock สำหรับ testing
    console.log(`[MOCK] ส่ง email ไปยัง ${to}: ${subject}`);
  }
}

const emailProvider = {
  provide: 'EMAIL_SERVICE',
  useClass: process.env.NODE_ENV === 'test' ? MockEmailService : RealEmailService,
};

@Module({
  providers: [emailProvider],
  exports: [emailProvider],
})
export class EmailModule {}
```

---

## Alias Providers (useExisting)
```typescript
// ใช้ useExisting เพื่อสร้าง alias สำหรับ provider
@Injectable()
class LoggerService {
  log(message: string) {
    console.log(`[LOG] ${message}`);
  }
  
  error(message: string) {
    console.error(`[ERROR] ${message}`);
  }
}

@Module({
  providers: [
    LoggerService,
    // สร้าง alias - 'Logger' จะ point ไปยัง LoggerService instance เดียวกัน
    {
      provide: 'Logger',
      useExisting: LoggerService,
    },
    // สร้าง alias อีกตัว
    {
      provide: 'AppLogger',
      useExisting: LoggerService,
    },
  ],
})
export class LoggerModule {}

// ทั้ง 3 จะได้ instance เดียวกัน (Singleton)
@Injectable()
export class MyService {
  constructor(
    private logger: LoggerService,         // ได้ instance เดียวกัน
    @Inject('Logger') private log: LoggerService,     // ได้ instance เดียวกัน
    @Inject('AppLogger') private appLog: LoggerService, // ได้ instance เดียวกัน
  ) {}
}
```

---

## Circular Dependencies (Dependencies แบบวนเวียน)

Circular dependencies เกิดขึ้นเมื่อ Service A ต้องการ Service B และ Service B ก็ต้องการ Service A

```typescript
// ปัญหา: Circular dependency
@Injectable()
export class UserService {
  constructor(private orderService: OrderService) {} // ต้องการ OrderService
}

@Injectable()
export class OrderService {
  constructor(private userService: UserService) {} // ต้องการ UserService
}

// วิธีแก้ไข 1: ใช้ forwardRef
import { Injectable, forwardRef, Inject } from '@nestjs/common';

@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => OrderService))
    private orderService: OrderService,
  ) {}

  async getUserWithOrders(userId: string) {
    const orders = await this.orderService.findByUserId(userId);
    return { userId, orders };
  }
}

@Injectable()
export class OrderService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private userService: UserService,
  ) {}

  async getOrderWithUser(orderId: string) {
    const user = await this.userService.findById('user-id');
    return { orderId, user };
  }
  
  async findByUserId(userId: string) {
    return []; // mock
  }
}

// module ก็ต้องใช้ forwardRef
@Module({
  imports: [forwardRef(() => OrdersModule)],
  providers: [UserService],
  exports: [UserService],
})
export class UsersModule {}

@Module({
  imports: [forwardRef(() => UsersModule)],
  providers: [OrderService],
  exports: [OrderService],
})
export class OrdersModule {}

// วิธีแก้ไข 2: แยก shared logic ออกมา (แนะนำ)
@Injectable()
export class UserOrderHelperService {
  // logic ที่ใช้ร่วมกัน
  async computeUserStats(userId: string, orders: any[]) {
    return {
      userId,
      totalOrders: orders.length,
      totalAmount: orders.reduce((sum, o) => sum + o.amount, 0),
    };
  }
}

@Injectable()
export class UserService {
  constructor(private helperService: UserOrderHelperService) {}
}

@Injectable()
export class OrderService {
  constructor(private helperService: UserOrderHelperService) {}
}
```

---

## Provider Scopes (ขอบเขตของ Provider)

### DEFAULT Scope (Singleton)
```typescript
import { Injectable, Scope } from '@nestjs/common';

// Singleton - default, สร้างครั้งเดียว share ทั้ง app
@Injectable()  // เทียบเท่ากับ @Injectable({ scope: Scope.DEFAULT })
export class AppConfigService {
  private config: Record<string, any> = {};

  set(key: string, value: any) {
    this.config[key] = value;
  }

  get(key: string) {
    return this.config[key];
  }
}

// ตรวจสอบ Singleton behavior
@Injectable()
export class CounterService {
  private count = 0;

  increment() {
    return ++this.count;
  }

  getCount() {
    return this.count;
  }
}

// เมื่อ inject CounterService ในหลายที่
// ทุกที่จะได้ instance เดียวกัน
@Injectable()
export class ServiceA {
  constructor(private counter: CounterService) {
    this.counter.increment(); // count = 1
  }
}

@Injectable()
export class ServiceB {
  constructor(private counter: CounterService) {
    this.counter.increment(); // count = 2 (instance เดียวกัน)
  }
}
```

### REQUEST Scope
```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

// สร้างใหม่ทุก HTTP request
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  private data: Map<string, any> = new Map();

  constructor(@Inject(REQUEST) private request: Request) {}

  get requestId(): string {
    return (this.request.headers['x-request-id'] as string) || 
           Math.random().toString(36).substr(2, 9);
  }

  get userId(): string | undefined {
    return (this.request as any).user?.id;
  }

  set(key: string, value: any) {
    this.data.set(key, value);
  }

  get(key: string) {
    return this.data.get(key);
  }
}

// Request-scoped service ที่ใช้ request data
@Injectable({ scope: Scope.REQUEST })
export class AuditService {
  constructor(
    @Inject(REQUEST) private request: Request,
    private logService: LogService,
  ) {}

  async logAction(action: string, resource: string, resourceId: string) {
    const userId = (this.request as any).user?.id;
    const ip = this.request.ip;
    const userAgent = this.request.headers['user-agent'];

    await this.logService.create({
      userId,
      action,
      resource,
      resourceId,
      ip,
      userAgent,
      timestamp: new Date(),
    });
  }
}
```

### TRANSIENT Scope
```typescript
// สร้างใหม่ทุกครั้งที่ inject
@Injectable({ scope: Scope.TRANSIENT })
export class UniqueIdService {
  private id: string;

  constructor() {
    this.id = uuidv4();
    console.log(`สร้าง UniqueIdService instance ใหม่: ${this.id}`);
  }

  getId(): string {
    return this.id;
  }
}

// ทุก injection จะได้ instance ใหม่
@Injectable()
export class ServiceX {
  constructor(private uniqueId: UniqueIdService) {
    console.log(`ServiceX ได้ id: ${uniqueId.getId()}`);
  }
}

@Injectable()
export class ServiceY {
  constructor(private uniqueId: UniqueIdService) {
    console.log(`ServiceY ได้ id: ${uniqueId.getId()}`); // id ต่างจาก ServiceX
  }
}
```

---

## Unit Testing Services (การทดสอบ Services)

### Setup สำหรับ Testing
```typescript
// src/users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { NotFoundException, ConflictException } from '@nestjs/common';

describe('UsersService', () => {
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [UsersService],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findAll', () => {
    it('should return empty array initially', async () => {
      const users = await service.findAll();
      expect(users).toEqual([]);
    });
  });

  describe('create', () => {
    it('should create a user successfully', async () => {
      const userData = {
        name: 'สมชาย ใจดี',
        email: 'somchai@example.com',
        password: 'password123',
      };

      const user = await service.create(userData);
      
      expect(user).toBeDefined();
      expect(user.name).toBe(userData.name);
      expect(user.email).toBe(userData.email);
      expect((user as any).password).toBeUndefined(); // ไม่ควร return password
    });

    it('should throw ConflictException for duplicate email', async () => {
      const userData = {
        name: 'สมชาย',
        email: 'test@example.com',
        password: 'password123',
      };

      await service.create(userData);
      await expect(service.create(userData)).rejects.toThrow(ConflictException);
    });
  });

  describe('findById', () => {
    it('should throw NotFoundException for non-existent user', async () => {
      await expect(service.findById('non-existent-id')).rejects.toThrow(
        NotFoundException,
      );
    });

    it('should return user by id', async () => {
      const created = await service.create({
        name: 'Test User',
        email: 'test2@example.com',
        password: 'password123',
      });

      const found = await service.findById((created as any).id);
      expect(found).toBeDefined();
      expect(found.email).toBe('test2@example.com');
    });
  });
});
```

### Testing with Mocks
```typescript
// src/orders/orders.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { OrdersService } from './orders.service';
import { UsersService } from '../users/users.service';
import { EmailService } from '../email/email.service';
import { NotFoundException } from '@nestjs/common';

// Mock factories
const mockUsersService = {
  findById: jest.fn(),
  findAll: jest.fn(),
};

const mockEmailService = {
  sendEmail: jest.fn(),
  sendOrderConfirmation: jest.fn(),
};

describe('OrdersService', () => {
  let ordersService: OrdersService;
  let usersService: jest.Mocked<UsersService>;
  let emailService: jest.Mocked<EmailService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrdersService,
        {
          provide: UsersService,
          useValue: mockUsersService,
        },
        {
          provide: EmailService,
          useValue: mockEmailService,
        },
      ],
    }).compile();

    ordersService = module.get<OrdersService>(OrdersService);
    usersService = module.get(UsersService);
    emailService = module.get(EmailService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('createOrder', () => {
    it('should create order and send confirmation email', async () => {
      const mockUser = {
        id: 'user-1',
        name: 'สมชาย',
        email: 'somchai@example.com',
      };

      mockUsersService.findById.mockResolvedValue(mockUser);
      mockEmailService.sendOrderConfirmation.mockResolvedValue(undefined);

      const orderData = {
        userId: 'user-1',
        items: [{ productId: 'p1', quantity: 2, price: 100 }],
      };

      const order = await ordersService.createOrder(orderData);

      expect(order).toBeDefined();
      expect(order.userId).toBe('user-1');
      expect(mockUsersService.findById).toHaveBeenCalledWith('user-1');
      expect(mockEmailService.sendOrderConfirmation).toHaveBeenCalledWith(
        mockUser.email,
        expect.any(Object),
      );
    });

    it('should throw NotFoundException when user not found', async () => {
      mockUsersService.findById.mockRejectedValue(
        new NotFoundException('ไม่พบผู้ใช้'),
      );

      await expect(
        ordersService.createOrder({ userId: 'not-found', items: [] }),
      ).rejects.toThrow(NotFoundException);
    });
  });
});
```

### Testing with Repository Mock
```typescript
// src/products/products.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ProductsService } from './products.service';
import { Product } from './entities/product.entity';

type MockRepository<T = any> = Partial<Record<keyof Repository<T>, jest.Mock>>;

const createMockRepository = <T = any>(): MockRepository<T> => ({
  find: jest.fn(),
  findOne: jest.fn(),
  findAndCount: jest.fn(),
  create: jest.fn(),
  save: jest.fn(),
  update: jest.fn(),
  delete: jest.fn(),
  count: jest.fn(),
});

describe('ProductsService', () => {
  let service: ProductsService;
  let productRepository: MockRepository<Product>;

  beforeEach(async () => {
    productRepository = createMockRepository<Product>();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ProductsService,
        {
          provide: getRepositoryToken(Product),
          useValue: productRepository,
        },
      ],
    }).compile();

    service = module.get<ProductsService>(ProductsService);
  });

  it('should find all products with pagination', async () => {
    const mockProducts = [
      { id: 1, name: 'สินค้า A', price: 100 },
      { id: 2, name: 'สินค้า B', price: 200 },
    ];

    productRepository.findAndCount.mockResolvedValue([mockProducts, 2]);

    const result = await service.findAll(1, 10);

    expect(result.data).toEqual(mockProducts);
    expect(result.total).toBe(2);
    expect(productRepository.findAndCount).toHaveBeenCalledWith({
      skip: 0,
      take: 10,
    });
  });

  it('should create a product', async () => {
    const createDto = { name: 'สินค้าใหม่', price: 150, stock: 10 };
    const savedProduct = { id: 1, ...createDto, createdAt: new Date() };

    productRepository.create.mockReturnValue(createDto);
    productRepository.save.mockResolvedValue(savedProduct);

    const result = await service.create(createDto as any);

    expect(result).toEqual(savedProduct);
    expect(productRepository.create).toHaveBeenCalledWith(createDto);
    expect(productRepository.save).toHaveBeenCalled();
  });
});
```

---

## Advanced Service Patterns (รูปแบบ Service ขั้นสูง)

### Service with Events
```typescript
// src/orders/orders.service.ts
import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class OrdersService {
  constructor(private eventEmitter: EventEmitter2) {}

  async createOrder(data: any) {
    // สร้าง order
    const order = { id: uuidv4(), ...data, createdAt: new Date() };
    
    // emit event
    this.eventEmitter.emit('order.created', {
      order,
      userId: data.userId,
    });

    return order;
  }

  async updateOrderStatus(id: string, status: string) {
    const order = { id, status, updatedAt: new Date() };
    
    // emit event ตาม status
    if (status === 'shipped') {
      this.eventEmitter.emit('order.shipped', { order });
    } else if (status === 'delivered') {
      this.eventEmitter.emit('order.delivered', { order });
    }

    return order;
  }
}

// Event handler
import { OnEvent } from '@nestjs/event-emitter';

@Injectable()
export class OrderEventsHandler {
  constructor(private emailService: EmailService) {}

  @OnEvent('order.created')
  async handleOrderCreated(payload: { order: any; userId: string }) {
    console.log(`คำสั่งซื้อใหม่: ${payload.order.id}`);
    await this.emailService.sendOrderConfirmation(payload.userId, payload.order);
  }

  @OnEvent('order.shipped')
  async handleOrderShipped(payload: { order: any }) {
    console.log(`จัดส่งแล้ว: ${payload.order.id}`);
    // ส่ง SMS หรือ notification
  }

  @OnEvent('order.delivered')
  async handleOrderDelivered(payload: { order: any }) {
    console.log(`ส่งถึงแล้ว: ${payload.order.id}`);
    // อัปเดต metrics
  }
}
```

### Caching Service Pattern
```typescript
// src/cache/cache.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CacheService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async get<T>(key: string): Promise<T | null> {
    return this.cacheManager.get<T>(key);
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    await this.cacheManager.set(key, value, ttl);
  }

  async del(key: string): Promise<void> {
    await this.cacheManager.del(key);
  }

  async getOrSet<T>(
    key: string,
    factory: () => Promise<T>,
    ttl?: number,
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached !== null && cached !== undefined) {
      return cached;
    }

    const value = await factory();
    await this.set(key, value, ttl);
    return value;
  }
}

// การใช้งาน CacheService
@Injectable()
export class ProductsService {
  constructor(
    private cacheService: CacheService,
    private productRepository: ProductRepository,
  ) {}

  async findById(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;
    
    return this.cacheService.getOrSet(
      cacheKey,
      () => this.productRepository.findById(id),
      300, // 5 นาที
    );
  }

  async update(id: number, data: any): Promise<Product> {
    const updated = await this.productRepository.update(id, data);
    
    // invalidate cache
    await this.cacheService.del(`product:${id}`);
    
    return updated;
  }
}
```

### Transaction Service Pattern
```typescript
// src/orders/order-transaction.service.ts
import { Injectable } from '@nestjs/common';
import { InjectEntityManager } from '@nestjs/typeorm';
import { EntityManager } from 'typeorm';

@Injectable()
export class OrderTransactionService {
  constructor(
    @InjectEntityManager()
    private entityManager: EntityManager,
  ) {}

  async createOrderWithItems(data: {
    userId: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
  }) {
    return this.entityManager.transaction(async (manager) => {
      // 1. สร้าง order
      const order = manager.create('Order', {
        userId: data.userId,
        totalAmount: data.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
        status: 'pending',
      });
      const savedOrder = await manager.save(order);

      // 2. สร้าง order items
      const orderItems = data.items.map(item => 
        manager.create('OrderItem', {
          orderId: savedOrder.id,
          productId: item.productId,
          quantity: item.quantity,
          price: item.price,
        })
      );
      await manager.save(orderItems);

      // 3. อัปเดต stock
      for (const item of data.items) {
        await manager.decrement(
          'Product' as any,
          { id: item.productId },
          'stock',
          item.quantity,
        );
      }

      return { order: savedOrder, items: orderItems };
    });
  }
}
```

---

## Service Injection Patterns (รูปแบบการ Inject Services)

### Constructor Injection (แนะนำ)
```typescript
@Injectable()
export class OrdersService {
  // Constructor injection - type-safe และ testable
  constructor(
    private readonly usersService: UsersService,
    private readonly productsService: ProductsService,
    private readonly emailService: EmailService,
    @InjectRepository(Order) private readonly orderRepository: Repository<Order>,
    @Inject('PAYMENT_SERVICE') private readonly paymentService: any,
  ) {}
}
```

### Property Injection
```typescript
@Injectable()
export class OrdersService {
  // Property injection - ใช้เมื่อ constructor injection ไม่สะดวก
  @Inject(UsersService)
  private usersService: UsersService;

  @Inject(EmailService)
  private emailService: EmailService;
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **Provider Concept**: แนวคิด provider และ Dependency Injection
2. **@Injectable Decorator**: การใช้ injectable decorator
3. **Service Creation**: การสร้าง services พื้นฐานและขั้นสูง
4. **Repository Injection**: การ inject repositories
5. **Custom Providers**: Value, Factory, Class providers
6. **Alias Providers**: useExisting สำหรับสร้าง aliases
7. **Circular Dependencies**: การแก้ไข circular dependencies ด้วย forwardRef
8. **Provider Scopes**: DEFAULT, REQUEST, TRANSIENT
9. **Unit Testing**: การทดสอบ services ด้วย mocks
10. **Advanced Patterns**: Events, Caching, Transactions

ในบทต่อไปเราจะเรียนรู้เรื่อง Database Integration และ Authentication
