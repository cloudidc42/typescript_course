# ส่วนที่ 61: Advanced Testing Patterns ใน TypeScript

## บทนำ

การทดสอบซอฟต์แวร์เป็นส่วนสำคัญของการพัฒนาแอปพลิเคชันที่มีคุณภาพ ในบทนี้เราจะเรียนรู้รูปแบบการทดสอบขั้นสูงที่ช่วยให้เราเขียนโค้ดที่น่าเชื่อถือและบำรุงรักษาได้ง่ายขึ้น

---

## 1. Test Doubles: Mocks, Stubs, Spies และ Fakes

Test doubles คือวัตถุจำลองที่ใช้แทนส่วนประกอบจริงในการทดสอบ มีหลายประเภทดังนี้:

### 1.1 Stubs - การสร้างค่าคืนที่กำหนดไว้ล่วงหน้า

```typescript
// interface ของ service จริง
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

// Stub สำหรับการทดสอบ
class UserRepositoryStub implements UserRepository {
  private users: Map<string, User> = new Map();

  // กำหนดข้อมูลสำหรับทดสอบ
  setUser(user: User): void {
    this.users.set(user.id, user);
  }

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async save(user: User): Promise<void> {
    this.users.set(user.id, user);
  }

  async delete(id: string): Promise<void> {
    this.users.delete(id);
  }
}

// Service ที่ต้องการทดสอบ
class UserService {
  constructor(private readonly userRepo: UserRepository) {}

  async getUserById(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) {
      throw new Error(`User with id ${id} not found`);
    }
    return user;
  }

  async promoteToAdmin(id: string): Promise<void> {
    const user = await this.getUserById(id);
    user.role = 'admin';
    await this.userRepo.save(user);
  }
}

// การใช้งาน Stub ในการทดสอบ
describe('UserService', () => {
  let userService: UserService;
  let userRepoStub: UserRepositoryStub;

  beforeEach(() => {
    userRepoStub = new UserRepositoryStub();
    userService = new UserService(userRepoStub);
  });

  test('getUserById ควรคืนผู้ใช้เมื่อพบ', async () => {
    const mockUser: User = {
      id: '1',
      name: 'สมชาย ใจดี',
      email: 'somchai@example.com',
      role: 'user'
    };
    userRepoStub.setUser(mockUser);

    const result = await userService.getUserById('1');
    expect(result).toEqual(mockUser);
  });

  test('getUserById ควรโยน error เมื่อไม่พบผู้ใช้', async () => {
    await expect(userService.getUserById('999')).rejects.toThrow(
      'User with id 999 not found'
    );
  });
});
```

### 1.2 Mocks - การตรวจสอบการเรียกใช้งาน

```typescript
import { jest } from '@jest/globals';

// สร้าง Mock ด้วย Jest
describe('UserService with Mocks', () => {
  test('promoteToAdmin ควรบันทึกผู้ใช้หลังจาก promote', async () => {
    // สร้าง mock functions
    const mockFindById = jest.fn<() => Promise<User | null>>();
    const mockSave = jest.fn<() => Promise<void>>();

    const mockRepo: UserRepository = {
      findById: mockFindById,
      save: mockSave,
      delete: jest.fn()
    };

    const user: User = {
      id: '1',
      name: 'สมชาย ใจดี',
      email: 'somchai@example.com',
      role: 'user'
    };

    mockFindById.mockResolvedValue(user);
    mockSave.mockResolvedValue(undefined);

    const service = new UserService(mockRepo);
    await service.promoteToAdmin('1');

    // ตรวจสอบว่า save ถูกเรียกพร้อมข้อมูลที่ถูกต้อง
    expect(mockSave).toHaveBeenCalledTimes(1);
    expect(mockSave).toHaveBeenCalledWith({
      ...user,
      role: 'admin'
    });
  });

  test('promoteToAdmin ควรเรียก findById ก่อน save', async () => {
    const callOrder: string[] = [];
    const mockFindById = jest.fn(async () => {
      callOrder.push('findById');
      return {
        id: '1',
        name: 'test',
        email: 'test@example.com',
        role: 'user' as const
      };
    });
    const mockSave = jest.fn(async () => {
      callOrder.push('save');
    });

    const mockRepo: UserRepository = {
      findById: mockFindById,
      save: mockSave,
      delete: jest.fn()
    };

    const service = new UserService(mockRepo);
    await service.promoteToAdmin('1');

    expect(callOrder).toEqual(['findById', 'save']);
  });
});
```

### 1.3 Spies - การสอดแนมการทำงาน

```typescript
class EmailService {
  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    console.log(`Sending welcome email to ${email}`);
    // โค้ดส่ง email จริง
  }

  async sendPasswordResetEmail(email: string, token: string): Promise<void> {
    console.log(`Sending password reset to ${email}`);
    // โค้ดส่ง email จริง
  }
}

class RegistrationService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService
  ) {}

  async registerUser(
    name: string,
    email: string,
    password: string
  ): Promise<User> {
    const user: User = {
      id: Math.random().toString(36),
      name,
      email,
      role: 'user'
    };

    await this.userRepo.save(user);
    await this.emailService.sendWelcomeEmail(email, name);

    return user;
  }
}

describe('RegistrationService', () => {
  test('ควรส่ง welcome email หลังจาก register สำเร็จ', async () => {
    const userRepo = new UserRepositoryStub();
    const emailService = new EmailService();

    // สร้าง spy บน method จริง
    const sendWelcomeEmailSpy = jest
      .spyOn(emailService, 'sendWelcomeEmail')
      .mockResolvedValue(undefined);

    const service = new RegistrationService(userRepo, emailService);
    await service.registerUser('สมหญิง', 'somying@example.com', 'password123');

    expect(sendWelcomeEmailSpy).toHaveBeenCalledWith(
      'somying@example.com',
      'สมหญิง'
    );
  });
});
```

### 1.4 Fakes - การสร้างการใช้งานจำลองที่ทำงานจริง

```typescript
// Fake ที่ทำงานได้จริงแต่ใช้ in-memory storage
class InMemoryUserRepository implements UserRepository {
  private storage: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.storage.get(id) ?? null;
  }

  async save(user: User): Promise<void> {
    this.storage.set(user.id, user);
  }

  async delete(id: string): Promise<void> {
    this.storage.delete(id);
  }

  // helper methods สำหรับการทดสอบ
  clear(): void {
    this.storage.clear();
  }

  size(): number {
    return this.storage.size;
  }

  getAllUsers(): User[] {
    return Array.from(this.storage.values());
  }
}

describe('UserService with InMemory Fake', () => {
  let userService: UserService;
  let fakeRepo: InMemoryUserRepository;

  beforeEach(() => {
    fakeRepo = new InMemoryUserRepository();
    userService = new UserService(fakeRepo);
  });

  afterEach(() => {
    fakeRepo.clear();
  });

  test('ควรจัดการ multiple users ได้', async () => {
    const users: User[] = [
      { id: '1', name: 'ผู้ใช้ที่ 1', email: 'user1@test.com', role: 'user' },
      { id: '2', name: 'ผู้ใช้ที่ 2', email: 'user2@test.com', role: 'user' },
      { id: '3', name: 'ผู้ใช้ที่ 3', email: 'user3@test.com', role: 'admin' }
    ];

    for (const user of users) {
      await fakeRepo.save(user);
    }

    expect(fakeRepo.size()).toBe(3);
    const found = await userService.getUserById('2');
    expect(found.name).toBe('ผู้ใช้ที่ 2');
  });
});
```

---

## 2. Property-Based Testing ด้วย fast-check

Property-based testing คือการทดสอบโดยสร้างข้อมูลทดสอบแบบสุ่มและตรวจสอบ properties ที่ควรเป็นจริงเสมอ

### 2.1 ติดตั้งและตั้งค่า fast-check

```bash
npm install --save-dev fast-check
```

```typescript
import * as fc from 'fast-check';

// ฟังก์ชันที่ต้องการทดสอบ
function sortNumbers(nums: number[]): number[] {
  return [...nums].sort((a, b) => a - b);
}

function reverseString(str: string): string {
  return str.split('').reverse().join('');
}

// Property-based tests
describe('Property-based tests', () => {
  test('sortNumbers: ผลลัพธ์ควรมีความยาวเท่าเดิม', () => {
    fc.assert(
      fc.property(fc.array(fc.integer()), (nums) => {
        const sorted = sortNumbers(nums);
        return sorted.length === nums.length;
      })
    );
  });

  test('sortNumbers: ทุก element ควรอยู่ในลำดับที่เพิ่มขึ้น', () => {
    fc.assert(
      fc.property(fc.array(fc.integer()), (nums) => {
        const sorted = sortNumbers(nums);
        for (let i = 0; i < sorted.length - 1; i++) {
          if (sorted[i] > sorted[i + 1]) return false;
        }
        return true;
      })
    );
  });

  test('reverseString: reverse สองครั้งควรได้ string เดิม', () => {
    fc.assert(
      fc.property(fc.string(), (str) => {
        return reverseString(reverseString(str)) === str;
      })
    );
  });

  test('reverseString: ความยาวไม่ควรเปลี่ยน', () => {
    fc.assert(
      fc.property(fc.string(), (str) => {
        return reverseString(str).length === str.length;
      })
    );
  });
});
```

### 2.2 Custom Arbitraries

```typescript
import * as fc from 'fast-check';

interface Product {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartItem {
  product: Product;
  quantity: number;
}

// Custom arbitrary สำหรับ Product
const productArbitrary = fc.record<Product>({
  id: fc.uuid(),
  name: fc.string({ minLength: 1, maxLength: 100 }),
  price: fc.float({ min: 0.01, max: 10000 }),
  quantity: fc.integer({ min: 0, max: 1000 })
});

// ฟังก์ชันคำนวณราคารวม
function calculateTotal(items: CartItem[]): number {
  return items.reduce(
    (total, item) => total + item.product.price * item.quantity,
    0
  );
}

describe('Cart calculations', () => {
  test('calculateTotal: ตะกร้าว่างควรมีราคา 0', () => {
    expect(calculateTotal([])).toBe(0);
  });

  test('calculateTotal: ราคารวมต้องไม่ติดลบ', () => {
    const cartItemArbitrary = fc.record<CartItem>({
      product: productArbitrary,
      quantity: fc.integer({ min: 1, max: 100 })
    });

    fc.assert(
      fc.property(fc.array(cartItemArbitrary), (items) => {
        return calculateTotal(items) >= 0;
      })
    );
  });

  test('calculateTotal: การเพิ่มสินค้าต้องเพิ่มราคา', () => {
    fc.assert(
      fc.property(
        fc.array(
          fc.record<CartItem>({
            product: productArbitrary,
            quantity: fc.integer({ min: 1, max: 100 })
          })
        ),
        fc.record<CartItem>({
          product: productArbitrary,
          quantity: fc.integer({ min: 1, max: 100 })
        }),
        (existingItems, newItem) => {
          const totalBefore = calculateTotal(existingItems);
          const totalAfter = calculateTotal([...existingItems, newItem]);
          return totalAfter >= totalBefore;
        }
      )
    );
  });
});
```

### 2.3 Stateful Property-Based Testing

```typescript
import * as fc from 'fast-check';

class BankAccount {
  private balance: number = 0;
  private transactions: number[] = [];

  deposit(amount: number): void {
    if (amount <= 0) throw new Error('Amount must be positive');
    this.balance += amount;
    this.transactions.push(amount);
  }

  withdraw(amount: number): void {
    if (amount <= 0) throw new Error('Amount must be positive');
    if (amount > this.balance) throw new Error('Insufficient funds');
    this.balance -= amount;
    this.transactions.push(-amount);
  }

  getBalance(): number {
    return this.balance;
  }

  getTransactionCount(): number {
    return this.transactions.length;
  }
}

// Model-based testing
class BankAccountModel {
  balance: number = 0;

  deposit(amount: number): void {
    this.balance += amount;
  }

  withdraw(amount: number): void {
    this.balance -= amount;
  }
}

describe('BankAccount stateful property tests', () => {
  test('ยอดเงินควรสอดคล้องกับการฝากและถอน', () => {
    fc.assert(
      fc.property(
        fc.array(
          fc.oneof(
            fc.record({ type: fc.constant('deposit'), amount: fc.float({ min: 0.01, max: 10000 }) }),
            fc.record({ type: fc.constant('withdraw'), amount: fc.float({ min: 0.01, max: 1000 }) })
          )
        ),
        (operations) => {
          const account = new BankAccount();
          let expectedBalance = 0;

          for (const op of operations) {
            if (op.type === 'deposit') {
              account.deposit(op.amount);
              expectedBalance += op.amount;
            } else {
              try {
                account.withdraw(op.amount);
                expectedBalance -= op.amount;
              } catch {
                // ถ้าเงินไม่พอ ยอดเงินไม่เปลี่ยน
              }
            }
          }

          return Math.abs(account.getBalance() - expectedBalance) < 0.001;
        }
      )
    );
  });
});
```

---

## 3. Contract Testing

Contract testing ช่วยให้มั่นใจว่า services สื่อสารกันได้อย่างถูกต้องตาม contract ที่ตกลงกัน

### 3.1 Consumer Contract Testing

```typescript
import Pact from '@pact-foundation/pact';
import { like, eachLike } from '@pact-foundation/pact/src/dsl/matchers';

const provider = new Pact({
  consumer: 'UserConsumer',
  provider: 'UserProvider',
  port: 3000,
  log: './logs/pact.log',
  dir: './pacts',
  logLevel: 'warn'
});

// API Client ที่ต้องการทดสอบ
class UserApiClient {
  constructor(private readonly baseUrl: string) {}

  async getUser(id: string): Promise<User> {
    const response = await fetch(`${this.baseUrl}/users/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  }

  async listUsers(): Promise<User[]> {
    const response = await fetch(`${this.baseUrl}/users`);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  }
}

describe('User API Consumer Contract Tests', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());

  describe('GET /users/:id', () => {
    test('ควรได้รับข้อมูลผู้ใช้', async () => {
      await provider.addInteraction({
        state: 'มีผู้ใช้ที่มี id = 1',
        uponReceiving: 'คำขอข้อมูลผู้ใช้ที่มี id = 1',
        withRequest: {
          method: 'GET',
          path: '/users/1'
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            id: like('1'),
            name: like('สมชาย ใจดี'),
            email: like('somchai@example.com'),
            role: like('user')
          }
        }
      });

      const client = new UserApiClient('http://localhost:3000');
      const user = await client.getUser('1');

      expect(user.id).toBeDefined();
      expect(user.name).toBeDefined();
      expect(user.email).toBeDefined();
    });
  });
});
```

### 3.2 Provider Contract Verification

```typescript
import { Verifier } from '@pact-foundation/pact';

describe('Provider Contract Verification', () => {
  test('ตรวจสอบว่า provider เป็นไปตาม contract', async () => {
    const opts = {
      providerBaseUrl: 'http://localhost:3001',
      pactUrls: ['./pacts/UserConsumer-UserProvider.json'],
      provider: 'UserProvider',
      stateHandlers: {
        'มีผู้ใช้ที่มี id = 1': async () => {
          // setup test data
          console.log('Setting up user with id 1');
        }
      }
    };

    await new Verifier(opts).verifyProvider();
  });
});
```

---

## 4. Mutation Testing

Mutation testing ช่วยวัดคุณภาพของ test suite โดยการแก้ไขโค้ดเล็กน้อยและตรวจสอบว่า tests สามารถจับข้อผิดพลาดได้

### 4.1 ตั้งค่า Stryker

```bash
npx stryker init
```

```json
// stryker.config.json
{
  "$schema": "./node_modules/@stryker-mutator/core/schema/stryker-schema.json",
  "packageManager": "npm",
  "reporters": ["html", "clear-text", "progress"],
  "testRunner": "jest",
  "coverageAnalysis": "perTest",
  "mutate": [
    "src/**/*.ts",
    "!src/**/*.spec.ts",
    "!src/**/*.test.ts"
  ]
}
```

### 4.2 ตัวอย่างที่ดีของ Mutation Testing

```typescript
// โค้ดที่มี mutation ที่น่าสนใจ
class PasswordValidator {
  validate(password: string): boolean {
    // Mutation 1: เปลี่ยน >= เป็น >
    if (password.length < 8) return false;
    
    // Mutation 2: เปลี่ยน || เป็น &&
    const hasUppercase = /[A-Z]/.test(password);
    const hasLowercase = /[a-z]/.test(password);
    const hasDigit = /\d/.test(password);
    const hasSpecial = /[!@#$%^&*]/.test(password);

    return hasUppercase && hasLowercase && hasDigit && hasSpecial;
  }
}

// Tests ที่ครอบคลุมทุก mutation
describe('PasswordValidator', () => {
  let validator: PasswordValidator;

  beforeEach(() => {
    validator = new PasswordValidator();
  });

  // Test ที่ตรวจสอบ boundary condition
  test('รหัสผ่าน 7 ตัวอักษรไม่ผ่าน', () => {
    expect(validator.validate('Abc1!xy')).toBe(false); // 7 chars
  });

  test('รหัสผ่าน 8 ตัวอักษรที่ถูกต้องผ่าน', () => {
    expect(validator.validate('Abc1!xyz')).toBe(true); // 8 chars
  });

  // Tests สำหรับแต่ละ condition
  test('ไม่มีตัวพิมพ์ใหญ่ไม่ผ่าน', () => {
    expect(validator.validate('abc1!xyz')).toBe(false);
  });

  test('ไม่มีตัวพิมพ์เล็กไม่ผ่าน', () => {
    expect(validator.validate('ABC1!XYZ')).toBe(false);
  });

  test('ไม่มีตัวเลขไม่ผ่าน', () => {
    expect(validator.validate('Abcd!xyz')).toBe(false);
  });

  test('ไม่มีอักขระพิเศษไม่ผ่าน', () => {
    expect(validator.validate('Abcd1xyz')).toBe(false);
  });

  test('ครบทุกเงื่อนไขผ่าน', () => {
    expect(validator.validate('Abcd1!ef')).toBe(true);
  });
});
```

---

## 5. Test Containers สำหรับ Integration Tests

Testcontainers ช่วยให้เราเขียน integration tests ที่ใช้ database หรือ services จริงใน Docker

### 5.1 ติดตั้ง Testcontainers

```bash
npm install --save-dev testcontainers
```

### 5.2 Integration Test กับ PostgreSQL

```typescript
import {
  PostgreSqlContainer,
  StartedPostgreSqlContainer
} from '@testcontainers/postgresql';
import { Pool } from 'pg';

interface UserRecord {
  id: string;
  name: string;
  email: string;
  created_at: Date;
}

class UserDatabase {
  constructor(private readonly pool: Pool) {}

  async createTable(): Promise<void> {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
      )
    `);
  }

  async insertUser(name: string, email: string): Promise<UserRecord> {
    const result = await this.pool.query<UserRecord>(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [name, email]
    );
    return result.rows[0];
  }

  async findByEmail(email: string): Promise<UserRecord | null> {
    const result = await this.pool.query<UserRecord>(
      'SELECT * FROM users WHERE email = $1',
      [email]
    );
    return result.rows[0] ?? null;
  }

  async count(): Promise<number> {
    const result = await this.pool.query<{ count: string }>(
      'SELECT COUNT(*) as count FROM users'
    );
    return parseInt(result.rows[0].count);
  }
}

describe('UserDatabase Integration Tests', () => {
  let container: StartedPostgreSqlContainer;
  let pool: Pool;
  let db: UserDatabase;

  beforeAll(async () => {
    // เริ่ม PostgreSQL container
    container = await new PostgreSqlContainer('postgres:15')
      .withDatabase('testdb')
      .withUsername('testuser')
      .withPassword('testpass')
      .start();

    pool = new Pool({
      host: container.getHost(),
      port: container.getPort(),
      database: container.getDatabase(),
      user: container.getUsername(),
      password: container.getPassword()
    });

    db = new UserDatabase(pool);
    await db.createTable();
  }, 60000);

  afterAll(async () => {
    await pool.end();
    await container.stop();
  });

  beforeEach(async () => {
    await pool.query('DELETE FROM users');
  });

  test('ควรเพิ่มผู้ใช้ได้', async () => {
    const user = await db.insertUser('สมชาย ใจดี', 'somchai@test.com');
    
    expect(user.id).toBeDefined();
    expect(user.name).toBe('สมชาย ใจดี');
    expect(user.email).toBe('somchai@test.com');
    expect(user.created_at).toBeDefined();
  });

  test('ควรค้นหาผู้ใช้ตาม email ได้', async () => {
    await db.insertUser('ทดสอบ ระบบ', 'test@example.com');
    
    const found = await db.findByEmail('test@example.com');
    expect(found).not.toBeNull();
    expect(found?.name).toBe('ทดสอบ ระบบ');
  });

  test('ควรนับจำนวนผู้ใช้ได้', async () => {
    await db.insertUser('ผู้ใช้ 1', 'user1@test.com');
    await db.insertUser('ผู้ใช้ 2', 'user2@test.com');
    await db.insertUser('ผู้ใช้ 3', 'user3@test.com');

    const count = await db.count();
    expect(count).toBe(3);
  });
}, 120000);
```

### 5.3 Integration Test กับ Redis

```typescript
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import Redis from 'ioredis';

class CacheService {
  constructor(private readonly redis: Redis) {}

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.redis.setex(key, ttlSeconds, value);
    } else {
      await this.redis.set(key, value);
    }
  }

  async get(key: string): Promise<string | null> {
    return this.redis.get(key);
  }

  async delete(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.redis.exists(key);
    return result === 1;
  }
}

describe('CacheService Integration Tests', () => {
  let container: StartedRedisContainer;
  let redis: Redis;
  let cache: CacheService;

  beforeAll(async () => {
    container = await new RedisContainer('redis:7').start();
    redis = new Redis({
      host: container.getHost(),
      port: container.getPort()
    });
    cache = new CacheService(redis);
  }, 60000);

  afterAll(async () => {
    await redis.quit();
    await container.stop();
  });

  beforeEach(async () => {
    await redis.flushall();
  });

  test('ควรบันทึกและดึงข้อมูลได้', async () => {
    await cache.set('key1', 'value1');
    const result = await cache.get('key1');
    expect(result).toBe('value1');
  });

  test('ควร expire หลังจาก TTL', async () => {
    await cache.set('key-ttl', 'temporary', 1);
    
    const before = await cache.get('key-ttl');
    expect(before).toBe('temporary');
    
    await new Promise(resolve => setTimeout(resolve, 1100));
    
    const after = await cache.get('key-ttl');
    expect(after).toBeNull();
  });
}, 120000);
```

---

## 6. Snapshot Testing Strategies

### 6.1 Inline Snapshots

```typescript
describe('Snapshot tests', () => {
  interface Config {
    database: { host: string; port: number; name: string };
    cache: { ttl: number; maxSize: number };
    features: { enableNewUI: boolean; enableBeta: boolean };
  }

  function getDefaultConfig(): Config {
    return {
      database: {
        host: 'localhost',
        port: 5432,
        name: 'myapp'
      },
      cache: {
        ttl: 3600,
        maxSize: 1000
      },
      features: {
        enableNewUI: false,
        enableBeta: false
      }
    };
  }

  test('config ควรมีค่า default ที่ถูกต้อง', () => {
    const config = getDefaultConfig();
    expect(config).toMatchInlineSnapshot(`
      {
        "cache": {
          "maxSize": 1000,
          "ttl": 3600,
        },
        "database": {
          "host": "localhost",
          "name": "myapp",
          "port": 5432,
        },
        "features": {
          "enableBeta": false,
          "enableNewUI": false,
        },
      }
    `);
  });
});
```

### 6.2 Custom Snapshot Serializers

```typescript
expect.addSnapshotSerializer({
  test: (val) => val && typeof val === 'object' && 'email' in val,
  print: (val: any) => {
    const sanitized = { ...val };
    if (sanitized.email) {
      sanitized.email = '***@***.***';
    }
    if (sanitized.password) {
      sanitized.password = '***';
    }
    return JSON.stringify(sanitized, null, 2);
  }
});

describe('Custom Snapshot Serializer', () => {
  test('ควร sanitize ข้อมูลส่วนตัว', () => {
    const userWithSensitiveData = {
      id: '123',
      name: 'สมชาย ใจดี',
      email: 'somchai@private.com',
      password: 'secret123'
    };

    // snapshot จะแสดง email และ password ที่ถูก mask
    expect(userWithSensitiveData).toMatchSnapshot();
  });
});
```

---

## 7. Testing Hooks และ Custom Hooks

### 7.1 Setup สำหรับ Testing Hooks

```bash
npm install --save-dev @testing-library/react @testing-library/react-hooks react-test-renderer
```

### 7.2 Testing Custom React Hooks

```typescript
import { renderHook, act } from '@testing-library/react';

// Custom hook ที่ต้องการทดสอบ
function useCounter(initialValue: number = 0) {
  const [count, setCount] = React.useState(initialValue);

  const increment = React.useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  const decrement = React.useCallback(() => {
    setCount(prev => prev - 1);
  }, []);

  const reset = React.useCallback(() => {
    setCount(initialValue);
  }, [initialValue]);

  return { count, increment, decrement, reset };
}

describe('useCounter hook', () => {
  test('ควรเริ่มต้นที่ 0', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  test('ควรเริ่มต้นที่ค่าที่กำหนด', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  test('increment ควรเพิ่มค่า 1', () => {
    const { result } = renderHook(() => useCounter());
    act(() => {
      result.current.increment();
    });
    expect(result.current.count).toBe(1);
  });

  test('decrement ควรลดค่า 1', () => {
    const { result } = renderHook(() => useCounter(5));
    act(() => {
      result.current.decrement();
    });
    expect(result.current.count).toBe(4);
  });

  test('reset ควรกลับไปค่าเริ่มต้น', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.increment();
      result.current.increment();
    });
    expect(result.current.count).toBe(7);
    
    act(() => {
      result.current.reset();
    });
    expect(result.current.count).toBe(5);
  });
});
```

### 7.3 Testing Hooks กับ Context

```typescript
import React, { createContext, useContext, useState, ReactNode } from 'react';
import { renderHook, act } from '@testing-library/react';

interface ThemeContextValue {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme ต้องใช้ภายใน ThemeProvider');
  }
  return context;
}

describe('useTheme hook', () => {
  const wrapper = ({ children }: { children: ReactNode }) => (
    <ThemeProvider>{children}</ThemeProvider>
  );

  test('ควรเริ่มต้นด้วย light theme', () => {
    const { result } = renderHook(() => useTheme(), { wrapper });
    expect(result.current.theme).toBe('light');
  });

  test('toggleTheme ควรเปลี่ยนจาก light เป็น dark', () => {
    const { result } = renderHook(() => useTheme(), { wrapper });
    
    act(() => {
      result.current.toggleTheme();
    });
    
    expect(result.current.theme).toBe('dark');
  });

  test('ควรโยน error เมื่อใช้นอก Provider', () => {
    expect(() => {
      renderHook(() => useTheme());
    }).toThrow('useTheme ต้องใช้ภายใน ThemeProvider');
  });
});
```

---

## 8. Testing Redux และ Zustand

### 8.1 Testing Redux Toolkit

```typescript
import { configureStore } from '@reduxjs/toolkit';
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

// User slice
interface UsersState {
  users: User[];
  loading: boolean;
  error: string | null;
}

const usersSlice = createSlice({
  name: 'users',
  initialState: {
    users: [],
    loading: false,
    error: null
  } as UsersState,
  reducers: {
    setLoading: (state, action: PayloadAction<boolean>) => {
      state.loading = action.payload;
    },
    setUsers: (state, action: PayloadAction<User[]>) => {
      state.users = action.payload;
      state.loading = false;
    },
    setError: (state, action: PayloadAction<string>) => {
      state.error = action.payload;
      state.loading = false;
    },
    addUser: (state, action: PayloadAction<User>) => {
      state.users.push(action.payload);
    },
    removeUser: (state, action: PayloadAction<string>) => {
      state.users = state.users.filter(u => u.id !== action.payload);
    }
  }
});

export const { setLoading, setUsers, setError, addUser, removeUser } = usersSlice.actions;

function createTestStore() {
  return configureStore({
    reducer: {
      users: usersSlice.reducer
    }
  });
}

describe('Users Redux Slice', () => {
  test('ค่าเริ่มต้นควรถูกต้อง', () => {
    const store = createTestStore();
    const state = store.getState().users;
    
    expect(state.users).toEqual([]);
    expect(state.loading).toBe(false);
    expect(state.error).toBeNull();
  });

  test('setLoading ควรอัปเดต loading state', () => {
    const store = createTestStore();
    store.dispatch(setLoading(true));
    
    expect(store.getState().users.loading).toBe(true);
  });

  test('addUser ควรเพิ่มผู้ใช้', () => {
    const store = createTestStore();
    const user: User = {
      id: '1',
      name: 'สมชาย',
      email: 'somchai@test.com',
      role: 'user'
    };
    
    store.dispatch(addUser(user));
    
    expect(store.getState().users.users).toHaveLength(1);
    expect(store.getState().users.users[0]).toEqual(user);
  });

  test('removeUser ควรลบผู้ใช้', () => {
    const store = createTestStore();
    const users: User[] = [
      { id: '1', name: 'ผู้ใช้ 1', email: 'u1@test.com', role: 'user' },
      { id: '2', name: 'ผู้ใช้ 2', email: 'u2@test.com', role: 'user' }
    ];
    
    store.dispatch(setUsers(users));
    store.dispatch(removeUser('1'));
    
    const remainingUsers = store.getState().users.users;
    expect(remainingUsers).toHaveLength(1);
    expect(remainingUsers[0].id).toBe('2');
  });
});
```

### 8.2 Testing Zustand Store

```typescript
import { create } from 'zustand';
import { act, renderHook } from '@testing-library/react';

interface CounterStore {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  incrementBy: (amount: number) => void;
}

const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  incrementBy: (amount) => set((state) => ({ count: state.count + amount }))
}));

describe('CounterStore (Zustand)', () => {
  beforeEach(() => {
    // reset store ก่อนแต่ละ test
    useCounterStore.setState({ count: 0 });
  });

  test('ค่าเริ่มต้นควรเป็น 0', () => {
    const { result } = renderHook(() => useCounterStore());
    expect(result.current.count).toBe(0);
  });

  test('increment ควรเพิ่มค่า', () => {
    const { result } = renderHook(() => useCounterStore());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  test('incrementBy ควรเพิ่มค่าตามที่กำหนด', () => {
    const { result } = renderHook(() => useCounterStore());
    
    act(() => {
      result.current.incrementBy(5);
    });
    
    expect(result.current.count).toBe(5);
  });

  test('สามารถ dispatch หลาย actions ต่อเนื่องได้', () => {
    const { result } = renderHook(() => useCounterStore());
    
    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.increment();
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(2);
  });
});
```

---

## 9. Testing Async Operations

### 9.1 Testing Promises และ Async/Await

```typescript
async function fetchUserData(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  
  if (!response.ok) {
    if (response.status === 404) {
      throw new Error('User not found');
    }
    throw new Error(`API Error: ${response.status}`);
  }
  
  return response.json();
}

// Testing async functions
describe('fetchUserData', () => {
  beforeEach(() => {
    global.fetch = jest.fn();
  });

  afterEach(() => {
    jest.resetAllMocks();
  });

  test('ควร return user เมื่อ success', async () => {
    const mockUser: User = {
      id: '1',
      name: 'สมชาย',
      email: 'somchai@test.com',
      role: 'user'
    };

    (global.fetch as jest.Mock).mockResolvedValue({
      ok: true,
      json: async () => mockUser
    });

    const result = await fetchUserData('1');
    expect(result).toEqual(mockUser);
  });

  test('ควรโยน error เมื่อ 404', async () => {
    (global.fetch as jest.Mock).mockResolvedValue({
      ok: false,
      status: 404
    });

    await expect(fetchUserData('999')).rejects.toThrow('User not found');
  });

  test('ควรโยน error เมื่อ server error', async () => {
    (global.fetch as jest.Mock).mockResolvedValue({
      ok: false,
      status: 500
    });

    await expect(fetchUserData('1')).rejects.toThrow('API Error: 500');
  });
});
```

### 9.2 Testing Observable/Stream

```typescript
import { from, of, throwError, timer } from 'rxjs';
import { mergeMap, retry, catchError } from 'rxjs/operators';
import { TestScheduler } from 'rxjs/testing';

describe('RxJS Testing', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  test('ควรทดสอบ observable ด้วย marble testing', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c|', { a: 1, b: 2, c: 3 });
      const expected = '--a--b--c|';

      expectObservable(source$).toBe(expected, { a: 1, b: 2, c: 3 });
    });
  });
});
```

---

## 10. Performance Testing

### 10.1 Benchmark Testing

```typescript
import Benchmark from 'benchmark';

// ฟังก์ชันที่ต้องการ benchmark
function sortWithBuiltIn(arr: number[]): number[] {
  return [...arr].sort((a, b) => a - b);
}

function sortWithQuickSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;
  
  const pivot = arr[Math.floor(arr.length / 2)];
  const left = arr.filter(x => x < pivot);
  const middle = arr.filter(x => x === pivot);
  const right = arr.filter(x => x > pivot);
  
  return [...sortWithQuickSort(left), ...middle, ...sortWithQuickSort(right)];
}

const suite = new Benchmark.Suite();
const testArray = Array.from({ length: 10000 }, () => Math.random() * 10000);

suite
  .add('Built-in sort', () => {
    sortWithBuiltIn(testArray);
  })
  .add('Quick sort', () => {
    sortWithQuickSort(testArray);
  })
  .on('cycle', (event: Benchmark.Event) => {
    console.log(String(event.target));
  })
  .on('complete', function(this: Benchmark.Suite) {
    console.log(`เร็วที่สุด: ${this.filter('fastest').map('name')}`);
  })
  .run({ async: true });
```

### 10.2 Memory Usage Testing

```typescript
import { performance } from 'perf_hooks';

function measureMemory<T>(fn: () => T): { result: T; memoryUsed: number } {
  const startMemory = process.memoryUsage().heapUsed;
  const result = fn();
  const endMemory = process.memoryUsage().heapUsed;
  
  return {
    result,
    memoryUsed: endMemory - startMemory
  };
}

describe('Memory performance tests', () => {
  test('การสร้าง large array ไม่ควรใช้ memory มากเกิน 50MB', () => {
    const { memoryUsed } = measureMemory(() => {
      return Array.from({ length: 100000 }, (_, i) => ({
        id: i,
        name: `item-${i}`,
        value: Math.random()
      }));
    });

    const memoryInMB = memoryUsed / 1024 / 1024;
    expect(memoryInMB).toBeLessThan(50);
  });
});
```

---

## 11. Visual Regression Testing

### 11.1 ตั้งค่า Storybook กับ Chromatic

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@chromatic-com/storybook'
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {}
  }
};

export default config;
```

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  component: Button,
  title: 'Components/Button'
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    variant: 'primary',
    label: 'คลิกที่นี่'
  }
};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    label: 'ยกเลิก'
  }
};

export const Disabled: Story = {
  args: {
    variant: 'primary',
    label: 'ปิดใช้งาน',
    disabled: true
  }
};
```

---

## 12. Testing Patterns สรุป

### 12.1 Test Helper Utilities

```typescript
// test-utils.ts
export function createMockDate(dateString: string): Date {
  return new Date(dateString);
}

export function waitFor(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

export function createMockUser(overrides?: Partial<User>): User {
  return {
    id: 'test-id-' + Math.random().toString(36).substr(2, 9),
    name: 'ผู้ใช้ทดสอบ',
    email: `test${Date.now()}@example.com`,
    role: 'user',
    ...overrides
  };
}

export function createMockUsers(count: number): User[] {
  return Array.from({ length: count }, (_, index) =>
    createMockUser({
      id: `user-${index + 1}`,
      name: `ผู้ใช้ที่ ${index + 1}`,
      email: `user${index + 1}@example.com`
    })
  );
}
```

### 12.2 Custom Jest Matchers

```typescript
// custom-matchers.ts
import { expect } from '@jest/globals';

expect.extend({
  toBeValidEmail(received: string) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    const pass = emailRegex.test(received);
    
    return {
      pass,
      message: () =>
        pass
          ? `ไม่ควรเป็น email ที่ถูกต้อง: ${received}`
          : `${received} ไม่ใช่ email ที่ถูกต้อง`
    };
  },

  toBeWithinRange(received: number, floor: number, ceiling: number) {
    const pass = received >= floor && received <= ceiling;
    
    return {
      pass,
      message: () =>
        pass
          ? `ไม่ควรอยู่ในช่วง ${floor} - ${ceiling} แต่ได้ ${received}`
          : `${received} ไม่อยู่ในช่วง ${floor} - ${ceiling}`
    };
  }
});

// ประกาศ types สำหรับ custom matchers
declare global {
  namespace jest {
    interface Matchers<R> {
      toBeValidEmail(): R;
      toBeWithinRange(floor: number, ceiling: number): R;
    }
  }
}

// การใช้งาน custom matchers
describe('Custom Matchers', () => {
  test('email validation', () => {
    expect('user@example.com').toBeValidEmail();
    expect('invalid-email').not.toBeValidEmail();
  });

  test('range validation', () => {
    expect(5).toBeWithinRange(1, 10);
    expect(15).not.toBeWithinRange(1, 10);
  });
});
```

---

## สรุปบทที่ 61

ในบทนี้เราได้เรียนรู้:

1. **Test Doubles** - Stubs, Mocks, Spies, Fakes และวิธีใช้งานแต่ละประเภท
2. **Property-based Testing** - การทดสอบด้วย fast-check และ custom arbitraries
3. **Contract Testing** - การทดสอบ contract ระหว่าง services
4. **Mutation Testing** - การวัดคุณภาพ test suite ด้วย Stryker
5. **Test Containers** - การทดสอบ integration กับ database จริง
6. **Snapshot Testing** - inline snapshots และ custom serializers
7. **Visual Regression Testing** - การทดสอบ UI ด้วย Storybook
8. **Testing Hooks** - การทดสอบ React hooks
9. **Testing Redux/Zustand** - การทดสอบ state management
10. **Testing Async Operations** - การทดสอบ promises และ observables
11. **Performance Testing** - การทดสอบประสิทธิภาพ

---

## แบบฝึกหัด

1. สร้าง test suite สำหรับ TodoList ที่ใช้ Zustand
2. เขียน property-based tests สำหรับฟังก์ชัน string manipulation
3. สร้าง integration test ที่ใช้ MySQL testcontainer
4. เพิ่ม custom matchers สำหรับ Thai text validation
5. ตั้งค่า visual regression testing ด้วย Playwright

---

*ต่อไป: ส่วนที่ 62 - TDD with TypeScript*
