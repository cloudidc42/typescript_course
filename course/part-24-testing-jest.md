# ตอนที่ 24: การทดสอบด้วย Jest และ TypeScript

## บทนำ

การทดสอบโค้ด (Testing) เป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์ที่มีคุณภาพ Jest เป็น testing framework ที่นิยมใช้กับ JavaScript และ TypeScript เนื่องจากมีความสามารถครบครันและใช้งานง่าย ในบทนี้เราจะเรียนรู้วิธีการตั้งค่าและใช้งาน Jest กับ TypeScript อย่างครบถ้วน

---

## 24.1 การติดตั้งและตั้งค่า Jest กับ TypeScript

### การติดตั้ง packages ที่จำเป็น

```bash
# สร้าง project ใหม่
mkdir typescript-jest-demo
cd typescript-jest-demo
npm init -y

# ติดตั้ง TypeScript และ Jest
npm install --save-dev typescript jest ts-jest @types/jest

# ติดตั้ง ts-node สำหรับ development
npm install --save-dev ts-node @types/node
```

### การตั้งค่า tsconfig.json

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
    "allowSyntheticDefaultImports": true,
    "moduleResolution": "node",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

### การตั้งค่า jest.config.ts

```typescript
// jest.config.ts
import type { Config } from '@jest/types';

const config: Config.InitialOptions = {
  // ใช้ ts-jest เป็น transformer
  preset: 'ts-jest',
  
  // environment สำหรับรัน tests
  testEnvironment: 'node',
  
  // pattern สำหรับหาไฟล์ test
  testMatch: [
    '**/__tests__/**/*.ts',
    '**/*.test.ts',
    '**/*.spec.ts'
  ],
  
  // เส้นทางสำหรับ module resolution
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1'
  },
  
  // การตั้งค่า coverage
  collectCoverage: false, // เปิดเมื่อต้องการ
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'clover', 'html'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
    '!src/**/*.spec.ts',
    '!src/index.ts'
  ],
  
  // coverage thresholds
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  
  // setup files
  setupFilesAfterFramework: [],
  
  // verbose output
  verbose: true,
  
  // transform options
  transform: {
    '^.+\\.tsx?$': [
      'ts-jest',
      {
        tsconfig: {
          strict: true
        }
      }
    ]
  }
};

export default config;
```

### การตั้งค่า package.json scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --watchAll=false",
    "test:verbose": "jest --verbose"
  }
}
```

---

## 24.2 @jest/types และการตั้งค่า ts-jest

### ความเข้าใจเกี่ยวกับ @jest/types

```typescript
// types/jest-extended.d.ts
import { Config } from '@jest/types';

// การใช้ Config types
type JestConfig = Config.InitialOptions;
type ProjectConfig = Config.ProjectConfig;
type GlobalConfig = Config.GlobalConfig;

// ตัวอย่างการสร้าง config พร้อม types
const createConfig = (): JestConfig => ({
  preset: 'ts-jest',
  testEnvironment: 'node',
  globals: {
    'ts-jest': {
      diagnostics: {
        warnOnly: true
      }
    }
  }
});

export { createConfig };
```

### การตั้งค่า ts-jest แบบละเอียด

```typescript
// jest.config.ts - การตั้งค่าแบบ advanced
import type { Config } from '@jest/types';
import { pathsToModuleNameMapper } from 'ts-jest';
import { compilerOptions } from './tsconfig.json';

const config: Config.InitialOptions = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  
  // ใช้ paths จาก tsconfig
  moduleNameMapper: pathsToModuleNameMapper(compilerOptions.paths || {}, {
    prefix: '<rootDir>/'
  }),
  
  // ts-jest configuration
  globals: {
    'ts-jest': {
      // ใช้ tsconfig เฉพาะสำหรับ tests
      tsconfig: {
        strict: true,
        esModuleInterop: true,
        allowJs: true,
        checkJs: true
      },
      
      // แสดง diagnostics เป็น warnings แทน errors
      diagnostics: {
        warnOnly: false,
        ignoreCodes: [151001]
      },
      
      // isolate modules สำหรับ performance
      isolatedModules: false
    }
  }
};

export default config;
```

---

## 24.3 Unit Testing Functions

### การสร้าง utility functions

```typescript
// src/utils/math.ts
export const add = (a: number, b: number): number => a + b;

export const subtract = (a: number, b: number): number => a - b;

export const multiply = (a: number, b: number): number => a * b;

export const divide = (a: number, b: number): number => {
  if (b === 0) {
    throw new Error('ไม่สามารถหารด้วยศูนย์ได้');
  }
  return a / b;
};

export const factorial = (n: number): number => {
  if (n < 0) {
    throw new Error('factorial ไม่รองรับจำนวนลบ');
  }
  if (n === 0 || n === 1) return 1;
  return n * factorial(n - 1);
};

export const isPrime = (n: number): boolean => {
  if (n < 2) return false;
  for (let i = 2; i <= Math.sqrt(n); i++) {
    if (n % i === 0) return false;
  }
  return true;
};

export const fibonacci = (n: number): number[] => {
  if (n <= 0) return [];
  if (n === 1) return [0];
  const result = [0, 1];
  for (let i = 2; i < n; i++) {
    result.push(result[i - 1] + result[i - 2]);
  }
  return result;
};
```

### Unit tests สำหรับ math functions

```typescript
// src/utils/__tests__/math.test.ts
import { add, subtract, multiply, divide, factorial, isPrime, fibonacci } from '../math';

describe('Math Utilities', () => {
  // === add function ===
  describe('add()', () => {
    it('ควรบวกเลขสองตัวได้ถูกต้อง', () => {
      expect(add(2, 3)).toBe(5);
    });

    it('ควรบวกกับจำนวนลบได้', () => {
      expect(add(-1, 5)).toBe(4);
      expect(add(-3, -2)).toBe(-5);
    });

    it('ควรบวกกับศูนย์ได้', () => {
      expect(add(0, 0)).toBe(0);
      expect(add(5, 0)).toBe(5);
    });

    it('ควรบวกทศนิยมได้', () => {
      expect(add(0.1, 0.2)).toBeCloseTo(0.3);
    });
  });

  // === subtract function ===
  describe('subtract()', () => {
    it('ควรลบเลขสองตัวได้ถูกต้อง', () => {
      expect(subtract(5, 3)).toBe(2);
    });

    it('ควรให้ผลลบได้', () => {
      expect(subtract(3, 5)).toBe(-2);
    });
  });

  // === multiply function ===
  describe('multiply()', () => {
    it('ควรคูณเลขสองตัวได้ถูกต้อง', () => {
      expect(multiply(3, 4)).toBe(12);
    });

    it('คูณด้วยศูนย์ควรได้ศูนย์', () => {
      expect(multiply(5, 0)).toBe(0);
    });
  });

  // === divide function ===
  describe('divide()', () => {
    it('ควรหารเลขสองตัวได้ถูกต้อง', () => {
      expect(divide(10, 2)).toBe(5);
    });

    it('ควร throw error เมื่อหารด้วยศูนย์', () => {
      expect(() => divide(10, 0)).toThrow('ไม่สามารถหารด้วยศูนย์ได้');
    });

    it('ควร throw Error instance เมื่อหารด้วยศูนย์', () => {
      expect(() => divide(10, 0)).toThrowError(Error);
    });
  });

  // === factorial function ===
  describe('factorial()', () => {
    it('factorial ของ 0 ควรเป็น 1', () => {
      expect(factorial(0)).toBe(1);
    });

    it('factorial ของ 1 ควรเป็น 1', () => {
      expect(factorial(1)).toBe(1);
    });

    it('factorial ของ 5 ควรเป็น 120', () => {
      expect(factorial(5)).toBe(120);
    });

    it('ควร throw error กับจำนวนลบ', () => {
      expect(() => factorial(-1)).toThrow('factorial ไม่รองรับจำนวนลบ');
    });
  });

  // === isPrime function ===
  describe('isPrime()', () => {
    it.each([2, 3, 5, 7, 11, 13, 17, 19, 23])(
      '%d ควรเป็นจำนวนเฉพาะ',
      (prime) => {
        expect(isPrime(prime)).toBe(true);
      }
    );

    it.each([0, 1, 4, 6, 8, 9, 10, 15])(
      '%d ไม่ควรเป็นจำนวนเฉพาะ',
      (notPrime) => {
        expect(isPrime(notPrime)).toBe(false);
      }
    );
  });

  // === fibonacci function ===
  describe('fibonacci()', () => {
    it('n=0 ควรคืน array ว่าง', () => {
      expect(fibonacci(0)).toEqual([]);
    });

    it('n=1 ควรคืน [0]', () => {
      expect(fibonacci(1)).toEqual([0]);
    });

    it('n=8 ควรคืน fibonacci sequence ที่ถูกต้อง', () => {
      expect(fibonacci(8)).toEqual([0, 1, 1, 2, 3, 5, 8, 13]);
    });

    it('ควรมีความยาวตามที่กำหนด', () => {
      const result = fibonacci(10);
      expect(result).toHaveLength(10);
    });
  });
});
```

### String utility functions

```typescript
// src/utils/string.ts
export const capitalize = (str: string): string => {
  if (!str) return str;
  return str.charAt(0).toUpperCase() + str.slice(1).toLowerCase();
};

export const truncate = (str: string, maxLength: number, suffix = '...'): string => {
  if (str.length <= maxLength) return str;
  return str.slice(0, maxLength - suffix.length) + suffix;
};

export const isPalindrome = (str: string): boolean => {
  const cleaned = str.toLowerCase().replace(/[^a-z0-9]/g, '');
  return cleaned === cleaned.split('').reverse().join('');
};

export const countWords = (str: string): number => {
  if (!str.trim()) return 0;
  return str.trim().split(/\s+/).length;
};

export const slugify = (str: string): string => {
  return str
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
};
```

```typescript
// src/utils/__tests__/string.test.ts
import { capitalize, truncate, isPalindrome, countWords, slugify } from '../string';

describe('String Utilities', () => {
  describe('capitalize()', () => {
    it('ควร capitalize ตัวอักษรแรก', () => {
      expect(capitalize('hello')).toBe('Hello');
    });

    it('ควรทำให้ตัวอักษรที่เหลือเป็นตัวพิมพ์เล็ก', () => {
      expect(capitalize('hELLO WORLD')).toBe('Hello world');
    });

    it('ควรคืนค่าเดิมเมื่อ string ว่าง', () => {
      expect(capitalize('')).toBe('');
    });
  });

  describe('truncate()', () => {
    it('ควรตัด string ที่ยาวเกินไป', () => {
      expect(truncate('Hello World', 8)).toBe('Hello...');
    });

    it('ไม่ควรตัด string ที่สั้นกว่า maxLength', () => {
      expect(truncate('Hi', 10)).toBe('Hi');
    });

    it('ควรใช้ custom suffix', () => {
      expect(truncate('Hello World', 8, ' [...]')).toBe('He [...]');
    });
  });

  describe('isPalindrome()', () => {
    it.each(['racecar', 'level', 'A man a plan a canal Panama', 'Was it a car or a cat I saw'])(
      '"%s" ควรเป็น palindrome',
      (palindrome) => {
        expect(isPalindrome(palindrome)).toBe(true);
      }
    );

    it.each(['hello', 'world', 'typescript'])(
      '"%s" ไม่ควรเป็น palindrome',
      (notPalindrome) => {
        expect(isPalindrome(notPalindrome)).toBe(false);
      }
    );
  });

  describe('countWords()', () => {
    it('ควรนับคำได้ถูกต้อง', () => {
      expect(countWords('Hello World')).toBe(2);
      expect(countWords('one two three four')).toBe(4);
    });

    it('string ว่างควรมี 0 คำ', () => {
      expect(countWords('')).toBe(0);
      expect(countWords('   ')).toBe(0);
    });
  });

  describe('slugify()', () => {
    it('ควรแปลง string เป็น slug', () => {
      expect(slugify('Hello World')).toBe('hello-world');
      expect(slugify('TypeScript Course 2024')).toBe('typescript-course-2024');
    });

    it('ควรลบ special characters', () => {
      expect(slugify('Hello! World?')).toBe('hello-world');
    });
  });
});
```

---

## 24.4 Testing Classes

### สร้าง class ที่จะทดสอบ

```typescript
// src/models/BankAccount.ts
export class InsufficientFundsError extends Error {
  constructor(
    public readonly amount: number,
    public readonly balance: number
  ) {
    super(`ยอดเงินไม่เพียงพอ: ต้องการ ${amount} แต่มี ${balance}`);
    this.name = 'InsufficientFundsError';
  }
}

export interface Transaction {
  id: string;
  type: 'deposit' | 'withdrawal' | 'transfer';
  amount: number;
  timestamp: Date;
  description?: string;
}

export class BankAccount {
  private _balance: number;
  private _transactions: Transaction[] = [];
  private _accountNumber: string;

  constructor(
    accountNumber: string,
    initialBalance: number = 0
  ) {
    if (initialBalance < 0) {
      throw new Error('ยอดเงินเริ่มต้นต้องไม่ติดลบ');
    }
    this._accountNumber = accountNumber;
    this._balance = initialBalance;
  }

  get balance(): number {
    return this._balance;
  }

  get accountNumber(): string {
    return this._accountNumber;
  }

  get transactions(): ReadonlyArray<Transaction> {
    return this._transactions;
  }

  deposit(amount: number, description?: string): void {
    if (amount <= 0) {
      throw new Error('จำนวนเงินฝากต้องมากกว่าศูนย์');
    }
    this._balance += amount;
    this._transactions.push({
      id: this.generateId(),
      type: 'deposit',
      amount,
      timestamp: new Date(),
      description
    });
  }

  withdraw(amount: number, description?: string): void {
    if (amount <= 0) {
      throw new Error('จำนวนเงินถอนต้องมากกว่าศูนย์');
    }
    if (amount > this._balance) {
      throw new InsufficientFundsError(amount, this._balance);
    }
    this._balance -= amount;
    this._transactions.push({
      id: this.generateId(),
      type: 'withdrawal',
      amount,
      timestamp: new Date(),
      description
    });
  }

  transfer(amount: number, targetAccount: BankAccount): void {
    this.withdraw(amount, `โอนไปยัง ${targetAccount.accountNumber}`);
    targetAccount.deposit(amount, `รับโอนจาก ${this._accountNumber}`);
  }

  getStatement(): string {
    const lines = [
      `บัญชีเลขที่: ${this._accountNumber}`,
      `ยอดคงเหลือ: ${this._balance} บาท`,
      '--- ประวัติธุรกรรม ---',
      ...this._transactions.map(t =>
        `${t.timestamp.toLocaleDateString('th-TH')} | ${t.type} | ${t.amount} บาท${t.description ? ` | ${t.description}` : ''}`
      )
    ];
    return lines.join('\n');
  }

  private generateId(): string {
    return `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### Tests สำหรับ BankAccount class

```typescript
// src/models/__tests__/BankAccount.test.ts
import { BankAccount, InsufficientFundsError } from '../BankAccount';

describe('BankAccount', () => {
  let account: BankAccount;

  // สร้าง account ใหม่ก่อนแต่ละ test
  beforeEach(() => {
    account = new BankAccount('ACC-001', 1000);
  });

  describe('constructor', () => {
    it('ควรสร้าง account ด้วยยอดเงินเริ่มต้นถูกต้อง', () => {
      const acc = new BankAccount('ACC-002', 500);
      expect(acc.balance).toBe(500);
      expect(acc.accountNumber).toBe('ACC-002');
    });

    it('ควรสร้าง account ด้วยยอดเงิน 0 เมื่อไม่ระบุ', () => {
      const acc = new BankAccount('ACC-003');
      expect(acc.balance).toBe(0);
    });

    it('ควร throw error เมื่อยอดเงินเริ่มต้นติดลบ', () => {
      expect(() => new BankAccount('ACC-004', -100)).toThrow(
        'ยอดเงินเริ่มต้นต้องไม่ติดลบ'
      );
    });
  });

  describe('deposit()', () => {
    it('ควรเพิ่มยอดเงินได้ถูกต้อง', () => {
      account.deposit(500);
      expect(account.balance).toBe(1500);
    });

    it('ควรบันทึก transaction', () => {
      account.deposit(500, 'เงินเดือน');
      expect(account.transactions).toHaveLength(1);
      expect(account.transactions[0].type).toBe('deposit');
      expect(account.transactions[0].amount).toBe(500);
      expect(account.transactions[0].description).toBe('เงินเดือน');
    });

    it('ควร throw error เมื่อฝากจำนวนน้อยกว่าหรือเท่ากับ 0', () => {
      expect(() => account.deposit(0)).toThrow('จำนวนเงินฝากต้องมากกว่าศูนย์');
      expect(() => account.deposit(-100)).toThrow('จำนวนเงินฝากต้องมากกว่าศูนย์');
    });
  });

  describe('withdraw()', () => {
    it('ควรลดยอดเงินได้ถูกต้อง', () => {
      account.withdraw(300);
      expect(account.balance).toBe(700);
    });

    it('ควร throw InsufficientFundsError เมื่อยอดเงินไม่พอ', () => {
      expect(() => account.withdraw(1500)).toThrow(InsufficientFundsError);
    });

    it('InsufficientFundsError ควรมีข้อมูลถูกต้อง', () => {
      try {
        account.withdraw(1500);
        fail('ควร throw error');
      } catch (error) {
        if (error instanceof InsufficientFundsError) {
          expect(error.amount).toBe(1500);
          expect(error.balance).toBe(1000);
          expect(error.name).toBe('InsufficientFundsError');
        }
      }
    });
  });

  describe('transfer()', () => {
    let targetAccount: BankAccount;

    beforeEach(() => {
      targetAccount = new BankAccount('ACC-005', 500);
    });

    it('ควรโอนเงินระหว่าง accounts ได้ถูกต้อง', () => {
      account.transfer(300, targetAccount);
      expect(account.balance).toBe(700);
      expect(targetAccount.balance).toBe(800);
    });

    it('ควรบันทึก transactions ทั้งสอง accounts', () => {
      account.transfer(300, targetAccount);
      expect(account.transactions).toHaveLength(1);
      expect(targetAccount.transactions).toHaveLength(1);
      expect(account.transactions[0].type).toBe('withdrawal');
      expect(targetAccount.transactions[0].type).toBe('deposit');
    });

    it('ควร throw error เมื่อยอดเงินต้นทางไม่พอ', () => {
      expect(() => account.transfer(2000, targetAccount)).toThrow(
        InsufficientFundsError
      );
    });
  });

  describe('getStatement()', () => {
    it('ควรแสดง statement ที่ถูกต้อง', () => {
      account.deposit(500, 'เงินเดือน');
      account.withdraw(200, 'ค่าอาหาร');

      const statement = account.getStatement();
      expect(statement).toContain('ACC-001');
      expect(statement).toContain('1300');
      expect(statement).toContain('เงินเดือน');
      expect(statement).toContain('ค่าอาหาร');
    });
  });
});
```

---

## 24.5 Mocking with TypeScript

### การ mock modules

```typescript
// src/services/emailService.ts
export interface EmailOptions {
  to: string;
  subject: string;
  body: string;
  from?: string;
}

export interface EmailResult {
  success: boolean;
  messageId: string;
  timestamp: Date;
}

export class EmailService {
  private apiKey: string;

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  async sendEmail(options: EmailOptions): Promise<EmailResult> {
    // การส่งอีเมลจริงๆ (จะถูก mock ใน tests)
    const response = await fetch('https://api.emailservice.com/send', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(options)
    });

    if (!response.ok) {
      throw new Error(`ส่งอีเมลล้มเหลว: ${response.statusText}`);
    }

    return {
      success: true,
      messageId: `MSG-${Date.now()}`,
      timestamp: new Date()
    };
  }
}
```

```typescript
// src/services/userService.ts
import { EmailService, EmailOptions } from './emailService';

export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}

export class UserService {
  constructor(private emailService: EmailService) {}

  async registerUser(name: string, email: string): Promise<User> {
    const user: User = {
      id: `USER-${Date.now()}`,
      name,
      email,
      createdAt: new Date()
    };

    // ส่งอีเมลยืนยัน
    await this.emailService.sendEmail({
      to: email,
      subject: 'ยินดีต้อนรับ!',
      body: `สวัสดี ${name}, บัญชีของคุณถูกสร้างแล้ว`
    });

    return user;
  }
}
```

```typescript
// src/services/__tests__/userService.test.ts
import { UserService } from '../userService';
import { EmailService } from '../emailService';

// Mock EmailService module
jest.mock('../emailService');

describe('UserService', () => {
  let userService: UserService;
  let mockEmailService: jest.Mocked<EmailService>;

  beforeEach(() => {
    // สร้าง mock instance
    mockEmailService = new EmailService('fake-key') as jest.Mocked<EmailService>;
    
    // กำหนดการทำงานของ mock
    mockEmailService.sendEmail = jest.fn().mockResolvedValue({
      success: true,
      messageId: 'MOCK-MSG-001',
      timestamp: new Date()
    });

    userService = new UserService(mockEmailService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('ควรสร้าง user ใหม่ได้', async () => {
    const user = await userService.registerUser('สมชาย', 'somchai@example.com');
    
    expect(user).toMatchObject({
      name: 'สมชาย',
      email: 'somchai@example.com'
    });
    expect(user.id).toBeDefined();
    expect(user.createdAt).toBeInstanceOf(Date);
  });

  it('ควรส่งอีเมลยืนยันหลังสมัครสมาชิก', async () => {
    await userService.registerUser('สมชาย', 'somchai@example.com');
    
    expect(mockEmailService.sendEmail).toHaveBeenCalledTimes(1);
    expect(mockEmailService.sendEmail).toHaveBeenCalledWith({
      to: 'somchai@example.com',
      subject: 'ยินดีต้อนรับ!',
      body: 'สวัสดี สมชาย, บัญชีของคุณถูกสร้างแล้ว'
    });
  });

  it('ควร throw error เมื่อส่งอีเมลล้มเหลว', async () => {
    mockEmailService.sendEmail = jest.fn().mockRejectedValue(
      new Error('ส่งอีเมลล้มเหลว')
    );

    await expect(
      userService.registerUser('สมชาย', 'somchai@example.com')
    ).rejects.toThrow('ส่งอีเมลล้มเหลว');
  });
});
```

---

## 24.6 Mock Types

### การสร้าง typed mocks

```typescript
// src/__tests__/typedMocks.test.ts
import { jest } from '@jest/globals';

// Interface ที่จะ mock
interface DatabaseService {
  findById(id: string): Promise<{ id: string; name: string } | null>;
  findAll(): Promise<Array<{ id: string; name: string }>>;
  save(data: { name: string }): Promise<{ id: string; name: string }>;
  delete(id: string): Promise<boolean>;
}

// สร้าง typed mock
const createMockDatabase = (): jest.Mocked<DatabaseService> => ({
  findById: jest.fn(),
  findAll: jest.fn(),
  save: jest.fn(),
  delete: jest.fn()
});

describe('Typed Mocks', () => {
  let db: jest.Mocked<DatabaseService>;

  beforeEach(() => {
    db = createMockDatabase();
  });

  it('ควร mock findById ได้', async () => {
    const mockUser = { id: '1', name: 'สมชาย' };
    db.findById.mockResolvedValue(mockUser);

    const result = await db.findById('1');
    expect(result).toEqual(mockUser);
    expect(db.findById).toHaveBeenCalledWith('1');
  });

  it('ควร mock findAll ได้', async () => {
    const mockUsers = [
      { id: '1', name: 'สมชาย' },
      { id: '2', name: 'สมหญิง' }
    ];
    db.findAll.mockResolvedValue(mockUsers);

    const results = await db.findAll();
    expect(results).toHaveLength(2);
    expect(results[0].name).toBe('สมชาย');
  });

  it('ควร mock save ได้', async () => {
    const newUser = { id: '3', name: 'ใหม่' };
    db.save.mockResolvedValue(newUser);

    const result = await db.save({ name: 'ใหม่' });
    expect(result.id).toBe('3');
  });

  it('ควรตรวจสอบ call count ได้', async () => {
    db.findAll.mockResolvedValue([]);

    await db.findAll();
    await db.findAll();
    await db.findAll();

    expect(db.findAll).toHaveBeenCalledTimes(3);
  });
});
```

### การใช้ jest.spyOn กับ TypeScript

```typescript
// src/services/logger.ts
export class Logger {
  private context: string;

  constructor(context: string) {
    this.context = context;
  }

  info(message: string): void {
    console.log(`[${this.context}] INFO: ${message}`);
  }

  error(message: string, error?: Error): void {
    console.error(`[${this.context}] ERROR: ${message}`, error);
  }

  warn(message: string): void {
    console.warn(`[${this.context}] WARN: ${message}`);
  }
}
```

```typescript
// src/services/__tests__/logger.test.ts
import { Logger } from '../logger';

describe('Logger with spyOn', () => {
  let logger: Logger;
  let consoleSpy: jest.SpyInstance;

  beforeEach(() => {
    logger = new Logger('TEST');
  });

  afterEach(() => {
    jest.restoreAllMocks();
  });

  it('ควร log ด้วย console.log สำหรับ info', () => {
    consoleSpy = jest.spyOn(console, 'log').mockImplementation(() => {});
    
    logger.info('ทดสอบ message');
    
    expect(consoleSpy).toHaveBeenCalledTimes(1);
    expect(consoleSpy).toHaveBeenCalledWith('[TEST] INFO: ทดสอบ message');
  });

  it('ควร log ด้วย console.error สำหรับ error', () => {
    consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});
    const testError = new Error('Test error');
    
    logger.error('เกิดข้อผิดพลาด', testError);
    
    expect(consoleSpy).toHaveBeenCalledWith('[TEST] ERROR: เกิดข้อผิดพลาด', testError);
  });

  it('ควร spy method บน class instance ได้', () => {
    const infoSpy = jest.spyOn(logger, 'info');
    
    logger.info('test message');
    
    expect(infoSpy).toHaveBeenCalledWith('test message');
    infoSpy.mockRestore();
  });
});
```

---

## 24.7 Testing Async Code

### การทดสอบ Promise และ async/await

```typescript
// src/services/apiService.ts
export interface Post {
  id: number;
  userId: number;
  title: string;
  body: string;
}

export interface ApiError {
  message: string;
  statusCode: number;
}

export class ApiService {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async getPost(id: number): Promise<Post> {
    const response = await fetch(`${this.baseUrl}/posts/${id}`);
    
    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status}`);
    }
    
    return response.json() as Promise<Post>;
  }

  async getPosts(userId?: number): Promise<Post[]> {
    const url = userId
      ? `${this.baseUrl}/posts?userId=${userId}`
      : `${this.baseUrl}/posts`;
    
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status}`);
    }
    
    return response.json() as Promise<Post[]>;
  }

  async createPost(data: Omit<Post, 'id'>): Promise<Post> {
    const response = await fetch(`${this.baseUrl}/posts`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });

    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status}`);
    }

    return response.json() as Promise<Post>;
  }
}
```

```typescript
// src/services/__tests__/apiService.test.ts
import { ApiService, Post } from '../apiService';

// Mock fetch globally
global.fetch = jest.fn();

const mockFetch = fetch as jest.MockedFunction<typeof fetch>;

describe('ApiService', () => {
  let apiService: ApiService;
  const BASE_URL = 'https://api.example.com';

  beforeEach(() => {
    apiService = new ApiService(BASE_URL);
    jest.clearAllMocks();
  });

  const createMockResponse = (data: unknown, ok = true, status = 200) => ({
    ok,
    status,
    json: jest.fn().mockResolvedValue(data),
    statusText: ok ? 'OK' : 'Error'
  } as unknown as Response);

  describe('getPost()', () => {
    it('ควรดึงข้อมูล post ได้ด้วย async/await', async () => {
      const mockPost: Post = {
        id: 1,
        userId: 1,
        title: 'Test Post',
        body: 'Test Body'
      };
      mockFetch.mockResolvedValue(createMockResponse(mockPost));

      const post = await apiService.getPost(1);

      expect(post).toEqual(mockPost);
      expect(fetch).toHaveBeenCalledWith(`${BASE_URL}/posts/1`);
    });

    it('ควร throw error เมื่อ HTTP error', async () => {
      mockFetch.mockResolvedValue(createMockResponse({}, false, 404));

      await expect(apiService.getPost(999)).rejects.toThrow('HTTP Error: 404');
    });

    it('ควรดึงข้อมูลได้ด้วย Promise .then()', () => {
      const mockPost: Post = { id: 1, userId: 1, title: 'Test', body: 'Body' };
      mockFetch.mockResolvedValue(createMockResponse(mockPost));

      return apiService.getPost(1).then(post => {
        expect(post).toEqual(mockPost);
      });
    });
  });

  describe('getPosts()', () => {
    it('ควรดึง posts ทั้งหมดได้', async () => {
      const mockPosts: Post[] = [
        { id: 1, userId: 1, title: 'Post 1', body: 'Body 1' },
        { id: 2, userId: 1, title: 'Post 2', body: 'Body 2' }
      ];
      mockFetch.mockResolvedValue(createMockResponse(mockPosts));

      const posts = await apiService.getPosts();

      expect(posts).toHaveLength(2);
      expect(fetch).toHaveBeenCalledWith(`${BASE_URL}/posts`);
    });

    it('ควรกรอง posts ตาม userId', async () => {
      const mockPosts: Post[] = [
        { id: 1, userId: 2, title: 'Post', body: 'Body' }
      ];
      mockFetch.mockResolvedValue(createMockResponse(mockPosts));

      await apiService.getPosts(2);

      expect(fetch).toHaveBeenCalledWith(`${BASE_URL}/posts?userId=2`);
    });
  });

  describe('createPost()', () => {
    it('ควรสร้าง post ใหม่ได้', async () => {
      const newPostData = { userId: 1, title: 'New Post', body: 'New Body' };
      const createdPost: Post = { id: 101, ...newPostData };
      mockFetch.mockResolvedValue(createMockResponse(createdPost, true, 201));

      const post = await apiService.createPost(newPostData);

      expect(post.id).toBe(101);
      expect(fetch).toHaveBeenCalledWith(
        `${BASE_URL}/posts`,
        expect.objectContaining({
          method: 'POST',
          body: JSON.stringify(newPostData)
        })
      );
    });
  });
});
```

### การทดสอบ setTimeout และ setInterval

```typescript
// src/utils/timer.ts
export const delay = (ms: number): Promise<void> => {
  return new Promise(resolve => setTimeout(resolve, ms));
};

export class Debouncer {
  private timer: ReturnType<typeof setTimeout> | null = null;

  debounce<T extends (...args: Parameters<T>) => void>(
    fn: T,
    wait: number
  ): (...args: Parameters<T>) => void {
    return (...args: Parameters<T>) => {
      if (this.timer !== null) {
        clearTimeout(this.timer);
      }
      this.timer = setTimeout(() => {
        fn(...args);
        this.timer = null;
      }, wait);
    };
  }

  cancel(): void {
    if (this.timer !== null) {
      clearTimeout(this.timer);
      this.timer = null;
    }
  }
}
```

```typescript
// src/utils/__tests__/timer.test.ts
import { delay, Debouncer } from '../timer';

describe('Timer Utilities', () => {
  describe('delay()', () => {
    it('ควรรอเวลาที่กำหนด', async () => {
      jest.useFakeTimers();
      
      let resolved = false;
      const promise = delay(1000).then(() => { resolved = true; });
      
      expect(resolved).toBe(false);
      
      jest.advanceTimersByTime(1000);
      await promise;
      
      expect(resolved).toBe(true);
      
      jest.useRealTimers();
    });
  });

  describe('Debouncer', () => {
    beforeEach(() => {
      jest.useFakeTimers();
    });

    afterEach(() => {
      jest.useRealTimers();
    });

    it('ควร debounce function call', () => {
      const debouncer = new Debouncer();
      const mockFn = jest.fn();
      const debouncedFn = debouncer.debounce(mockFn, 500);

      // เรียกหลายครั้งในระยะเวลาสั้น
      debouncedFn('first');
      debouncedFn('second');
      debouncedFn('third');

      // ยังไม่ควรถูกเรียก
      expect(mockFn).not.toHaveBeenCalled();

      // เลื่อนเวลาไป 500ms
      jest.advanceTimersByTime(500);

      // ควรถูกเรียกเพียงครั้งเดียวกับ argument ล่าสุด
      expect(mockFn).toHaveBeenCalledTimes(1);
      expect(mockFn).toHaveBeenCalledWith('third');
    });

    it('ควร cancel debounced call', () => {
      const debouncer = new Debouncer();
      const mockFn = jest.fn();
      const debouncedFn = debouncer.debounce(mockFn, 500);

      debouncedFn('test');
      debouncer.cancel();

      jest.advanceTimersByTime(500);

      expect(mockFn).not.toHaveBeenCalled();
    });
  });
});
```

---

## 24.8 Testing with Spies

```typescript
// src/services/orderService.ts
export interface Order {
  id: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
}

export class OrderService {
  private orders: Map<string, Order> = new Map();

  createOrder(items: Order['items']): Order {
    const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    const order: Order = {
      id: `ORD-${Date.now()}`,
      items,
      total,
      status: 'pending'
    };
    this.orders.set(order.id, order);
    this.notifyCustomer(order);
    return order;
  }

  confirmOrder(orderId: string): Order {
    const order = this.orders.get(orderId);
    if (!order) throw new Error('ไม่พบ order');
    order.status = 'confirmed';
    this.logActivity('confirm', orderId);
    return order;
  }

  protected notifyCustomer(order: Order): void {
    // ส่งการแจ้งเตือนลูกค้า
    console.log(`แจ้งเตือนลูกค้า: order ${order.id} ถูกสร้างแล้ว`);
  }

  protected logActivity(action: string, orderId: string): void {
    console.log(`Activity: ${action} - ${orderId}`);
  }
}
```

```typescript
// src/services/__tests__/orderService.test.ts
import { OrderService, Order } from '../orderService';

describe('OrderService with Spies', () => {
  let orderService: OrderService;

  beforeEach(() => {
    orderService = new OrderService();
  });

  afterEach(() => {
    jest.restoreAllMocks();
  });

  it('ควรสร้าง order ได้', () => {
    const notifySpy = jest.spyOn(orderService as any, 'notifyCustomer');
    
    const items = [{ productId: 'P001', quantity: 2, price: 100 }];
    const order = orderService.createOrder(items);

    expect(order.status).toBe('pending');
    expect(order.total).toBe(200);
    expect(notifySpy).toHaveBeenCalledWith(expect.objectContaining({
      status: 'pending',
      total: 200
    }));
  });

  it('ควร spy บน console.log', () => {
    const consoleSpy = jest.spyOn(console, 'log').mockImplementation(() => {});
    
    const items = [{ productId: 'P001', quantity: 1, price: 50 }];
    orderService.createOrder(items);

    expect(consoleSpy).toHaveBeenCalled();
    expect(consoleSpy.mock.calls[0][0]).toContain('ถูกสร้างแล้ว');
  });

  it('ควร confirm order ได้', () => {
    const logSpy = jest.spyOn(orderService as any, 'logActivity');
    jest.spyOn(console, 'log').mockImplementation(() => {});

    const items = [{ productId: 'P001', quantity: 1, price: 100 }];
    const order = orderService.createOrder(items);
    const confirmed = orderService.confirmOrder(order.id);

    expect(confirmed.status).toBe('confirmed');
    expect(logSpy).toHaveBeenCalledWith('confirm', order.id);
  });

  it('ควร spy กับ return value', () => {
    jest.spyOn(console, 'log').mockImplementation(() => {});
    
    // Mock return value
    const mockOrder: Order = {
      id: 'MOCK-ORD',
      items: [],
      total: 999,
      status: 'pending'
    };
    
    const createSpy = jest.spyOn(orderService, 'createOrder').mockReturnValue(mockOrder);
    
    const items = [{ productId: 'P001', quantity: 1, price: 100 }];
    const result = orderService.createOrder(items);

    expect(result.id).toBe('MOCK-ORD');
    expect(result.total).toBe(999);
    expect(createSpy).toHaveBeenCalledTimes(1);
  });
});
```

---

## 24.9 Integration Tests

```typescript
// src/app.ts - Express application
import express, { Request, Response, NextFunction } from 'express';

export interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
}

// In-memory store สำหรับ demo
const products: Map<string, Product> = new Map([
  ['P001', { id: 'P001', name: 'สินค้า A', price: 100, stock: 50 }],
  ['P002', { id: 'P002', name: 'สินค้า B', price: 250, stock: 30 }]
]);

export const createApp = () => {
  const app = express();
  app.use(express.json());

  // GET /products
  app.get('/products', (_req: Request, res: Response) => {
    res.json(Array.from(products.values()));
  });

  // GET /products/:id
  app.get('/products/:id', (req: Request, res: Response) => {
    const product = products.get(req.params.id);
    if (!product) {
      return res.status(404).json({ error: 'ไม่พบสินค้า' });
    }
    res.json(product);
  });

  // POST /products
  app.post('/products', (req: Request, res: Response) => {
    const { name, price, stock } = req.body as Partial<Product>;
    
    if (!name || price === undefined || stock === undefined) {
      return res.status(400).json({ error: 'ข้อมูลไม่ครบถ้วน' });
    }

    const id = `P${String(products.size + 1).padStart(3, '0')}`;
    const product: Product = { id, name, price, stock };
    products.set(id, product);
    
    res.status(201).json(product);
  });

  // PUT /products/:id
  app.put('/products/:id', (req: Request, res: Response) => {
    const product = products.get(req.params.id);
    if (!product) {
      return res.status(404).json({ error: 'ไม่พบสินค้า' });
    }

    const updated = { ...product, ...req.body as Partial<Product>, id: product.id };
    products.set(product.id, updated);
    res.json(updated);
  });

  // DELETE /products/:id
  app.delete('/products/:id', (req: Request, res: Response) => {
    if (!products.has(req.params.id)) {
      return res.status(404).json({ error: 'ไม่พบสินค้า' });
    }
    products.delete(req.params.id);
    res.status(204).send();
  });

  // Error handler
  app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
    res.status(500).json({ error: err.message });
  });

  return app;
};
```

```typescript
// src/__tests__/integration/products.test.ts
import request from 'supertest';
import { createApp } from '../../app';

// ต้องติดตั้ง supertest: npm install --save-dev supertest @types/supertest

const app = createApp();

describe('Products API Integration Tests', () => {
  describe('GET /products', () => {
    it('ควรคืน list ของ products', async () => {
      const response = await request(app)
        .get('/products')
        .expect('Content-Type', /json/)
        .expect(200);

      expect(Array.isArray(response.body)).toBe(true);
      expect(response.body.length).toBeGreaterThan(0);
    });
  });

  describe('GET /products/:id', () => {
    it('ควรคืน product ที่มีอยู่', async () => {
      const response = await request(app)
        .get('/products/P001')
        .expect(200);

      expect(response.body).toMatchObject({
        id: 'P001',
        name: 'สินค้า A'
      });
    });

    it('ควรคืน 404 เมื่อไม่พบ product', async () => {
      const response = await request(app)
        .get('/products/INVALID')
        .expect(404);

      expect(response.body.error).toBe('ไม่พบสินค้า');
    });
  });

  describe('POST /products', () => {
    it('ควรสร้าง product ใหม่ได้', async () => {
      const newProduct = {
        name: 'สินค้าใหม่',
        price: 500,
        stock: 100
      };

      const response = await request(app)
        .post('/products')
        .send(newProduct)
        .expect(201);

      expect(response.body).toMatchObject(newProduct);
      expect(response.body.id).toBeDefined();
    });

    it('ควรคืน 400 เมื่อข้อมูลไม่ครบ', async () => {
      const response = await request(app)
        .post('/products')
        .send({ name: 'สินค้าไม่ครบ' })
        .expect(400);

      expect(response.body.error).toBe('ข้อมูลไม่ครบถ้วน');
    });
  });

  describe('PUT /products/:id', () => {
    it('ควรอัปเดต product ได้', async () => {
      const response = await request(app)
        .put('/products/P001')
        .send({ price: 150 })
        .expect(200);

      expect(response.body.price).toBe(150);
      expect(response.body.id).toBe('P001');
    });
  });

  describe('DELETE /products/:id', () => {
    it('ควรลบ product ได้', async () => {
      // สร้าง product ก่อน
      const createResponse = await request(app)
        .post('/products')
        .send({ name: 'จะลบ', price: 100, stock: 5 });

      const id = createResponse.body.id;

      await request(app)
        .delete(`/products/${id}`)
        .expect(204);

      // ตรวจสอบว่าถูกลบแล้ว
      await request(app)
        .get(`/products/${id}`)
        .expect(404);
    });
  });
});
```

---

## 24.10 Test Utilities Typing

```typescript
// src/test-utils/index.ts
import { jest } from '@jest/globals';

// Generic test factory
export const createMockFn = <T extends (...args: any[]) => any>(): jest.MockedFunction<T> => {
  return jest.fn() as jest.MockedFunction<T>;
};

// Builder pattern สำหรับ test data
export class ProductBuilder {
  private product = {
    id: 'TEST-001',
    name: 'Test Product',
    price: 100,
    stock: 10
  };

  withId(id: string): this {
    this.product.id = id;
    return this;
  }

  withName(name: string): this {
    this.product.name = name;
    return this;
  }

  withPrice(price: number): this {
    this.product.price = price;
    return this;
  }

  withStock(stock: number): this {
    this.product.stock = stock;
    return this;
  }

  build() {
    return { ...this.product };
  }
}

// Type-safe mock factory
export const createTypedMock = <T extends object>(
  partial: Partial<jest.Mocked<T>> = {}
): jest.Mocked<T> => {
  return partial as jest.Mocked<T>;
};

// Async test helper
export const waitFor = (
  condition: () => boolean,
  timeout = 5000,
  interval = 100
): Promise<void> => {
  return new Promise((resolve, reject) => {
    const startTime = Date.now();
    const check = () => {
      if (condition()) {
        resolve();
      } else if (Date.now() - startTime > timeout) {
        reject(new Error(`Condition not met within ${timeout}ms`));
      } else {
        setTimeout(check, interval);
      }
    };
    check();
  });
};
```

---

## 24.11 Snapshot Testing

```typescript
// src/utils/formatter.ts
export interface UserProfile {
  id: string;
  username: string;
  email: string;
  role: 'admin' | 'user' | 'moderator';
  createdAt: Date;
  preferences: {
    theme: 'light' | 'dark';
    language: string;
    notifications: boolean;
  };
}

export const formatUserProfile = (user: UserProfile): object => {
  return {
    id: user.id,
    username: user.username,
    email: user.email.toLowerCase(),
    role: user.role,
    joinDate: user.createdAt.toISOString().split('T')[0],
    settings: {
      theme: user.preferences.theme,
      lang: user.preferences.language,
      notifs: user.preferences.notifications
    }
  };
};

export const generateHtmlCard = (user: UserProfile): string => {
  return `
    <div class="user-card" data-role="${user.role}">
      <h2>${user.username}</h2>
      <p>${user.email}</p>
      <span class="badge badge-${user.role}">${user.role}</span>
    </div>
  `.trim();
};
```

```typescript
// src/utils/__tests__/formatter.test.ts
import { formatUserProfile, generateHtmlCard, UserProfile } from '../formatter';

const createTestUser = (overrides: Partial<UserProfile> = {}): UserProfile => ({
  id: 'USER-001',
  username: 'testuser',
  email: 'Test@Example.com',
  role: 'user',
  createdAt: new Date('2024-01-15T00:00:00.000Z'),
  preferences: {
    theme: 'light',
    language: 'th',
    notifications: true
  },
  ...overrides
});

describe('Snapshot Testing', () => {
  it('formatUserProfile ควรตรงกับ snapshot', () => {
    const user = createTestUser();
    const formatted = formatUserProfile(user);
    expect(formatted).toMatchSnapshot();
  });

  it('formatUserProfile สำหรับ admin ควรตรงกับ snapshot', () => {
    const admin = createTestUser({ role: 'admin', username: 'adminuser' });
    const formatted = formatUserProfile(admin);
    expect(formatted).toMatchSnapshot();
  });

  it('generateHtmlCard ควรตรงกับ inline snapshot', () => {
    const user = createTestUser();
    const html = generateHtmlCard(user);
    expect(html).toMatchInlineSnapshot(`
      "<div class="user-card" data-role="user">
            <h2>testuser</h2>
            <p>Test@Example.com</p>
            <span class="badge badge-user">user</span>
          </div>"
    `);
  });

  it('ควรอัปเดต snapshot เมื่อโครงสร้างเปลี่ยน', () => {
    const user = createTestUser({
      preferences: {
        theme: 'dark',
        language: 'en',
        notifications: false
      }
    });
    const formatted = formatUserProfile(user);
    // รัน jest --updateSnapshot เพื่ออัปเดต
    expect(formatted).toMatchSnapshot();
  });
});
```

---

## 24.12 Coverage Reports

```bash
# รัน tests พร้อม coverage
npm run test:coverage

# รัน coverage เฉพาะบาง files
npx jest --coverage --collectCoverageFrom="src/utils/**/*.ts"

# เปิด coverage report ใน browser (หลังรัน)
open coverage/lcov-report/index.html
```

### การตั้งค่า coverage thresholds

```typescript
// jest.config.ts - coverage configuration
import type { Config } from '@jest/types';

const config: Config.InitialOptions = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  
  collectCoverage: true,
  coverageDirectory: 'coverage',
  
  coverageReporters: [
    'text',           // แสดงใน terminal
    'text-summary',   // สรุปใน terminal
    'lcov',           // สำหรับ CI tools
    'html',           // HTML report
    'json',           // JSON data
    'clover'          // XML format
  ],
  
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
    '!src/**/*.spec.ts',
    '!src/index.ts',
    '!src/types/**',
    '!src/migrations/**'
  ],
  
  // กำหนด minimum coverage
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 85,
      lines: 85,
      statements: 85
    },
    // กำหนด threshold เฉพาะ files
    './src/utils/math.ts': {
      branches: 100,
      functions: 100,
      lines: 100,
      statements: 100
    }
  }
};

export default config;
```

---

## 24.13 Complete Test Suite for Express API

```typescript
// src/routes/auth.ts
import express, { Router, Request, Response } from 'express';
import jwt from 'jsonwebtoken';

export interface LoginRequest {
  email: string;
  password: string;
}

export interface AuthResponse {
  token: string;
  user: {
    id: string;
    email: string;
    name: string;
  };
}

// Users store (mock)
const users = new Map([
  ['user@example.com', {
    id: 'U001',
    email: 'user@example.com',
    password: 'hashed_password',
    name: 'Test User'
  }]
]);

const JWT_SECRET = process.env.JWT_SECRET || 'test-secret';

export const authRouter = (): Router => {
  const router = express.Router();

  router.post('/login', async (req: Request<{}, AuthResponse, LoginRequest>, res: Response) => {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({ error: 'กรุณาระบุ email และ password' });
    }

    const user = users.get(email);
    if (!user || user.password !== `hashed_${password}`) {
      return res.status(401).json({ error: 'email หรือ password ไม่ถูกต้อง' });
    }

    const token = jwt.sign(
      { userId: user.id, email: user.email },
      JWT_SECRET,
      { expiresIn: '1h' }
    );

    res.json({
      token,
      user: { id: user.id, email: user.email, name: user.name }
    });
  });

  router.post('/logout', (_req: Request, res: Response) => {
    res.json({ message: 'ออกจากระบบสำเร็จ' });
  });

  return router;
};
```

```typescript
// src/__tests__/integration/auth.test.ts
import request from 'supertest';
import express from 'express';
import jwt from 'jsonwebtoken';
import { authRouter } from '../../routes/auth';

const createTestApp = () => {
  const app = express();
  app.use(express.json());
  app.use('/auth', authRouter());
  return app;
};

describe('Auth API', () => {
  const app = createTestApp();

  describe('POST /auth/login', () => {
    it('ควร login ได้ด้วย credentials ที่ถูกต้อง', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({ email: 'user@example.com', password: 'password' })
        .expect(200);

      expect(response.body).toHaveProperty('token');
      expect(response.body).toHaveProperty('user');
      expect(response.body.user.email).toBe('user@example.com');

      // ตรวจสอบ JWT token
      const decoded = jwt.verify(
        response.body.token,
        'test-secret'
      ) as { userId: string; email: string };
      expect(decoded.email).toBe('user@example.com');
    });

    it('ควร return 401 เมื่อ credentials ไม่ถูกต้อง', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({ email: 'user@example.com', password: 'wrong' })
        .expect(401);

      expect(response.body.error).toContain('ไม่ถูกต้อง');
    });

    it('ควร return 400 เมื่อ email ว่าง', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({ email: '', password: 'password' })
        .expect(400);

      expect(response.body.error).toContain('กรุณาระบุ');
    });

    it('ควร return 400 เมื่อไม่มี body', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({})
        .expect(400);
    });
  });

  describe('POST /auth/logout', () => {
    it('ควร logout ได้สำเร็จ', async () => {
      const response = await request(app)
        .post('/auth/logout')
        .expect(200);

      expect(response.body.message).toBe('ออกจากระบบสำเร็จ');
    });
  });
});
```

---

## 24.14 Testing Patterns

### Pattern: AAA (Arrange-Act-Assert)

```typescript
// src/__tests__/patterns/aaa.test.ts
import { BankAccount } from '../../models/BankAccount';

describe('AAA Pattern Examples', () => {
  it('ควรฝากเงินได้ถูกต้อง', () => {
    // === ARRANGE ===
    // เตรียมข้อมูลและ objects ที่จำเป็น
    const account = new BankAccount('ACC-001', 1000);
    const depositAmount = 500;

    // === ACT ===
    // ดำเนินการที่ต้องการทดสอบ
    account.deposit(depositAmount);

    // === ASSERT ===
    // ตรวจสอบผลลัพธ์
    expect(account.balance).toBe(1500);
  });
});
```

### Pattern: Given-When-Then (BDD Style)

```typescript
// src/__tests__/patterns/bdd.test.ts
describe('BDD Style Tests', () => {
  describe('Given: ผู้ใช้มีบัญชีที่มียอดเงิน 1000 บาท', () => {
    let account: any;

    beforeEach(() => {
      const { BankAccount } = require('../../models/BankAccount');
      account = new BankAccount('ACC-BDD', 1000);
    });

    describe('When: ผู้ใช้ฝากเงิน 500 บาท', () => {
      beforeEach(() => {
        account.deposit(500);
      });

      it('Then: ยอดเงินควรเป็น 1500 บาท', () => {
        expect(account.balance).toBe(1500);
      });

      it('Then: ควรมี transaction 1 รายการ', () => {
        expect(account.transactions).toHaveLength(1);
      });
    });

    describe('When: ผู้ใช้ถอนเงิน 2000 บาท', () => {
      it('Then: ควร throw InsufficientFundsError', () => {
        const { InsufficientFundsError } = require('../../models/BankAccount');
        expect(() => account.withdraw(2000)).toThrow(InsufficientFundsError);
      });
    });
  });
});
```

### Pattern: Test Doubles

```typescript
// src/__tests__/patterns/testDoubles.test.ts

// Dummy - ส่งผ่านเป็น argument แต่ไม่ถูกใช้งาน
const dummyLogger = {
  log: () => {},
  error: () => {},
  warn: () => {}
};

// Stub - ให้ค่าที่กำหนดไว้ล่วงหน้า
const stubEmailService = {
  sendEmail: jest.fn().mockResolvedValue({ success: true, messageId: 'STUB-001' })
};

// Fake - มีการ implement จริงแต่ง่ายกว่าของจริง
class FakeDatabase {
  private data: Map<string, any> = new Map();

  async save(id: string, data: any): Promise<void> {
    this.data.set(id, data);
  }

  async findById(id: string): Promise<any | null> {
    return this.data.get(id) || null;
  }

  async findAll(): Promise<any[]> {
    return Array.from(this.data.values());
  }

  clear(): void {
    this.data.clear();
  }
}

// Mock - ตรวจสอบว่าถูกเรียกด้วย arguments ที่ถูกต้อง
const mockNotificationService = {
  send: jest.fn(),
  sendBatch: jest.fn()
};

describe('Test Doubles Pattern', () => {
  const fakeDb = new FakeDatabase();

  beforeEach(() => {
    fakeDb.clear();
    jest.clearAllMocks();
  });

  it('Stub - ควรให้ค่าที่กำหนด', async () => {
    const result = await stubEmailService.sendEmail({
      to: 'test@example.com',
      subject: 'Test',
      body: 'Body'
    });
    expect(result.success).toBe(true);
  });

  it('Fake - ควรทำงานเหมือน real database', async () => {
    await fakeDb.save('1', { name: 'สมชาย' });
    const found = await fakeDb.findById('1');
    expect(found.name).toBe('สมชาย');
  });

  it('Mock - ควรตรวจสอบการเรียก method', async () => {
    mockNotificationService.send({ to: 'user@test.com', message: 'Hello' });
    
    expect(mockNotificationService.send).toHaveBeenCalledWith({
      to: 'user@test.com',
      message: 'Hello'
    });
  });
});
```

---

## 24.15 สรุป Jest Configuration สมบูรณ์

```typescript
// jest.config.ts - complete configuration
import type { Config } from '@jest/types';

const config: Config.InitialOptions = {
  // Preset
  preset: 'ts-jest',
  testEnvironment: 'node',
  
  // Test file patterns
  testMatch: [
    '<rootDir>/src/**/__tests__/**/*.ts',
    '<rootDir>/src/**/*.test.ts',
    '<rootDir>/src/**/*.spec.ts'
  ],
  
  // Transform
  transform: {
    '^.+\\.tsx?$': ['ts-jest', {
      tsconfig: {
        strict: true,
        esModuleInterop: true
      }
    }]
  },
  
  // Module name mapping
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1'
  },
  
  // Setup files
  setupFilesAfterFramework: ['<rootDir>/src/test-utils/setup.ts'],
  
  // Coverage
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts'
  ],
  coverageThreshold: {
    global: { lines: 80, functions: 80, branches: 80, statements: 80 }
  },
  
  // Timeouts
  testTimeout: 10000,
  
  // Reporter
  reporters: [
    'default',
    ['jest-junit', {
      outputDirectory: 'test-results',
      outputName: 'junit.xml'
    }]
  ],
  
  // Globals
  globals: {
    'ts-jest': {
      diagnostics: false,
      isolatedModules: true
    }
  },
  
  verbose: true,
  clearMocks: true,
  restoreMocks: true
};

export default config;
```

---

## สรุปบทที่ 24

ในบทนี้เราได้เรียนรู้:
- การตั้งค่า Jest กับ TypeScript ด้วย ts-jest
- การใช้ @jest/types สำหรับ type safety
- Unit testing functions, classes, และ async code
- การ mock modules และ dependencies
- การใช้ spies สำหรับตรวจสอบ side effects
- Integration testing กับ Express API
- Snapshot testing
- Coverage reports และ thresholds
- Testing patterns: AAA, BDD, Test Doubles

**ขั้นตอนต่อไป**: ในบทที่ 25 เราจะเรียนรู้เกี่ยวกับการตั้งค่า React กับ TypeScript
