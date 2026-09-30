# Part 86: Real-world FinTech Application ด้วย TypeScript

## บทนำ

ในบทนี้เราจะสร้างแอปพลิเคชัน FinTech ที่ครบครันด้วย TypeScript โดยครอบคลุมการจัดการบัญชี การโอนเงิน ประวัติธุรกรรม และระบบบันทึกบัญชีแบบ Double-Entry Bookkeeping

---

## 1. สถาปัตยกรรม FinTech Application

### 1.1 โครงสร้างโปรเจกต์

```
fintech-app/
├── src/
│   ├── domain/
│   │   ├── money.ts
│   │   ├── account.ts
│   │   ├── transaction.ts
│   │   └── ledger.ts
│   ├── services/
│   │   ├── account.service.ts
│   │   ├── transfer.service.ts
│   │   ├── transaction.service.ts
│   │   └── audit.service.ts
│   ├── repositories/
│   │   ├── account.repository.ts
│   │   └── transaction.repository.ts
│   ├── middleware/
│   │   ├── rate-limiter.ts
│   │   └── auth.ts
│   └── api/
│       └── routes.ts
├── package.json
└── tsconfig.json
```

### 1.2 package.json

```json
{
  "name": "fintech-app",
  "version": "1.0.0",
  "dependencies": {
    "decimal.js": "^10.4.3",
    "uuid": "^9.0.0",
    "express": "^4.18.2",
    "winston": "^3.11.0",
    "rate-limiter-flexible": "^4.0.0",
    "joi": "^17.11.0"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "@types/express": "^4.17.21",
    "@types/uuid": "^9.0.7",
    "ts-node": "^10.9.2"
  }
}
```

### 1.3 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "exactOptionalPropertyTypes": true
  }
}
```

---

## 2. Money Type - การจัดการตัวเลขทางการเงิน

### 2.1 ปัญหา Floating-Point

```typescript
// ปัญหาที่เกิดขึ้นกับ floating-point
console.log(0.1 + 0.2); // 0.30000000000000004 ไม่ใช่ 0.3!
console.log(0.1 + 0.2 === 0.3); // false

// ตัวอย่างปัญหาในระบบการเงิน
const price1 = 10.1;
const price2 = 20.2;
const total = price1 + price2;
console.log(total); // 30.299999999999997 ผิด!
```

### 2.2 ใช้ BigInt สำหรับ Money

```typescript
// src/domain/money.ts
export type Currency = 'THB' | 'USD' | 'EUR' | 'JPY' | 'GBP';

export interface MoneyJSON {
  amount: string;
  currency: Currency;
}

export class Money {
  // เก็บในหน่วยที่เล็กที่สุด (สตางค์สำหรับบาท, cents สำหรับ USD)
  private readonly _amount: bigint;
  private readonly _currency: Currency;

  // ตัวคูณสำหรับแต่ละสกุลเงิน (จำนวนหน่วยย่อยต่อ 1 หน่วยหลัก)
  private static readonly CURRENCY_DECIMALS: Record<Currency, number> = {
    THB: 2, // 1 บาท = 100 สตางค์
    USD: 2, // 1 dollar = 100 cents
    EUR: 2, // 1 euro = 100 cents
    JPY: 0, // เยนไม่มีทศนิยม
    GBP: 2, // 1 pound = 100 pence
  };

  constructor(amount: bigint, currency: Currency) {
    this._amount = amount;
    this._currency = currency;
  }

  // สร้างจากตัวเลขทศนิยม เช่น Money.fromDecimal(10.50, 'THB')
  static fromDecimal(amount: number | string, currency: Currency): Money {
    const decimals = Money.CURRENCY_DECIMALS[currency];
    const multiplier = BigInt(10 ** decimals);
    
    // แปลงเป็น string เพื่อหลีกเลี่ยงปัญหา floating-point
    const amountStr = typeof amount === 'number' 
      ? amount.toFixed(decimals) 
      : amount;
    
    const [whole, fraction = ''] = amountStr.split('.');
    const paddedFraction = fraction.padEnd(decimals, '0').slice(0, decimals);
    
    const bigIntAmount = BigInt(whole) * multiplier + BigInt(paddedFraction || '0');
    return new Money(bigIntAmount, currency);
  }

  // สร้างจาก string เช่น "1050" สตางค์ = 10.50 บาท
  static fromMinorUnit(amount: bigint | string | number, currency: Currency): Money {
    return new Money(BigInt(amount), currency);
  }

  // สร้าง Zero money
  static zero(currency: Currency): Money {
    return new Money(0n, currency);
  }

  get amount(): bigint {
    return this._amount;
  }

  get currency(): Currency {
    return this._currency;
  }

  // แปลงเป็น string แสดงผล
  toDecimalString(): string {
    const decimals = Money.CURRENCY_DECIMALS[this._currency];
    if (decimals === 0) {
      return this._amount.toString();
    }
    
    const multiplier = BigInt(10 ** decimals);
    const isNegative = this._amount < 0n;
    const absoluteAmount = isNegative ? -this._amount : this._amount;
    
    const whole = absoluteAmount / multiplier;
    const fraction = absoluteAmount % multiplier;
    
    const fractionStr = fraction.toString().padStart(decimals, '0');
    const result = `${whole}.${fractionStr}`;
    
    return isNegative ? `-${result}` : result;
  }

  // บวก
  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  // ลบ
  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount - other._amount, this._currency);
  }

  // คูณ (ใช้สำหรับดอกเบี้ยหรือ fee)
  multiply(factor: number): Money {
    // ใช้ทศนิยม 10 ตำแหน่งเพื่อความแม่นยำ
    const factorStr = factor.toFixed(10);
    const [whole, fraction] = factorStr.split('.');
    
    const precision = 10;
    const numerator = BigInt(whole) * BigInt(10 ** precision) + BigInt(fraction);
    const denominator = BigInt(10 ** precision);
    
    const result = (this._amount * numerator) / denominator;
    return new Money(result, this._currency);
  }

  // หาร (ใช้สำหรับแบ่งเงิน)
  divide(divisor: number): Money {
    const precision = 10;
    const multiplier = BigInt(10 ** precision);
    const result = (this._amount * multiplier) / BigInt(Math.round(divisor * 10 ** precision));
    return new Money(result, this._currency);
  }

  // เปรียบเทียบ
  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  greaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  greaterThanOrEqual(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount >= other._amount;
  }

  lessThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount < other._amount;
  }

  isZero(): boolean {
    return this._amount === 0n;
  }

  isNegative(): boolean {
    return this._amount < 0n;
  }

  isPositive(): boolean {
    return this._amount > 0n;
  }

  // แปลงเป็น JSON
  toJSON(): MoneyJSON {
    return {
      amount: this._amount.toString(),
      currency: this._currency,
    };
  }

  // สร้างจาก JSON
  static fromJSON(json: MoneyJSON): Money {
    return new Money(BigInt(json.amount), json.currency);
  }

  // แสดงผลสวย
  format(locale = 'th-TH'): string {
    const decimalValue = parseFloat(this.toDecimalString());
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: this._currency,
    }).format(decimalValue);
  }

  toString(): string {
    return `${this.toDecimalString()} ${this._currency}`;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(
        `ไม่สามารถดำเนินการระหว่างสกุลเงินต่างกัน: ${this._currency} vs ${other._currency}`
      );
    }
  }
}
```

### 2.3 ทดสอบ Money Class

```typescript
// ทดสอบการใช้งาน Money
const price1 = Money.fromDecimal(10.1, 'THB');
const price2 = Money.fromDecimal(20.2, 'THB');
const total = price1.add(price2);

console.log(total.toString()); // "30.30 THB" ถูกต้อง!
console.log(total.format()); // "฿30.30"

// ทดสอบ multiply
const salary = Money.fromDecimal(50000, 'THB');
const tax = salary.multiply(0.15); // ภาษี 15%
console.log(tax.toString()); // "7500.00 THB"

// ทดสอบ BigInt
const bigAmount = Money.fromDecimal('999999999999.99', 'THB');
console.log(bigAmount.format()); // ✓ ไม่มีปัญหา precision
```

### 2.4 ใช้ Decimal.js แทน BigInt

```typescript
import Decimal from 'decimal.js';

// กำหนดค่า Decimal.js
Decimal.set({ 
  precision: 20, 
  rounding: Decimal.ROUND_HALF_UP,
  toExpPos: 20,
  toExpNeg: -20
});

export class DecimalMoney {
  private readonly _amount: Decimal;
  private readonly _currency: Currency;

  constructor(amount: Decimal | string | number, currency: Currency) {
    this._amount = new Decimal(amount);
    this._currency = currency;
  }

  add(other: DecimalMoney): DecimalMoney {
    this.assertSameCurrency(other);
    return new DecimalMoney(this._amount.plus(other._amount), this._currency);
  }

  subtract(other: DecimalMoney): DecimalMoney {
    this.assertSameCurrency(other);
    return new DecimalMoney(this._amount.minus(other._amount), this._currency);
  }

  multiply(factor: number | string): DecimalMoney {
    return new DecimalMoney(this._amount.times(factor), this._currency);
  }

  divide(divisor: number | string): DecimalMoney {
    return new DecimalMoney(this._amount.dividedBy(divisor), this._currency);
  }

  // ปัดเศษตามกฎการเงิน
  round(decimalPlaces = 2): DecimalMoney {
    return new DecimalMoney(
      this._amount.toDecimalPlaces(decimalPlaces, Decimal.ROUND_HALF_UP),
      this._currency
    );
  }

  toNumber(): number {
    return this._amount.toNumber();
  }

  toString(): string {
    return `${this._amount.toFixed(2)} ${this._currency}`;
  }

  private assertSameCurrency(other: DecimalMoney): void {
    if (this._currency !== other._currency) {
      throw new Error(`สกุลเงินไม่ตรงกัน: ${this._currency} vs ${other._currency}`);
    }
  }
}
```

---

## 3. Account Management

### 3.1 Account Types และ Interfaces

```typescript
// src/domain/account.ts
import { Money } from './money';
import { v4 as uuidv4 } from 'uuid';

export type AccountType = 'CHECKING' | 'SAVINGS' | 'INVESTMENT' | 'CREDIT';
export type AccountStatus = 'ACTIVE' | 'FROZEN' | 'CLOSED' | 'PENDING_VERIFICATION';

export interface AccountLimits {
  dailyTransferLimit: Money;
  singleTransferLimit: Money;
  monthlyWithdrawalLimit: Money;
  minimumBalance: Money;
}

export interface AccountMetadata {
  openedAt: Date;
  lastTransactionAt?: Date;
  closedAt?: Date;
  freezeReason?: string;
}

export interface Account {
  id: string;
  accountNumber: string;
  ownerId: string;
  type: AccountType;
  status: AccountStatus;
  balance: Money;
  availableBalance: Money; // balance - pending holds
  pendingHolds: Money;
  limits: AccountLimits;
  metadata: AccountMetadata;
  currency: Currency;
}
```

### 3.2 Account Factory

```typescript
import { Currency } from './money';

export class AccountFactory {
  static createChecking(ownerId: string, currency: Currency = 'THB'): Account {
    const now = new Date();
    return {
      id: uuidv4(),
      accountNumber: AccountFactory.generateAccountNumber('001'),
      ownerId,
      type: 'CHECKING',
      status: 'PENDING_VERIFICATION',
      balance: Money.zero(currency),
      availableBalance: Money.zero(currency),
      pendingHolds: Money.zero(currency),
      limits: {
        dailyTransferLimit: Money.fromDecimal(500000, currency),    // 500,000 บาท/วัน
        singleTransferLimit: Money.fromDecimal(200000, currency),   // 200,000 บาท/ครั้ง
        monthlyWithdrawalLimit: Money.fromDecimal(2000000, currency), // 2,000,000 บาท/เดือน
        minimumBalance: Money.fromDecimal(500, currency),           // ขั้นต่ำ 500 บาท
      },
      metadata: { openedAt: now },
      currency,
    };
  }

  static createSavings(ownerId: string, currency: Currency = 'THB'): Account {
    const now = new Date();
    return {
      id: uuidv4(),
      accountNumber: AccountFactory.generateAccountNumber('002'),
      ownerId,
      type: 'SAVINGS',
      status: 'PENDING_VERIFICATION',
      balance: Money.zero(currency),
      availableBalance: Money.zero(currency),
      pendingHolds: Money.zero(currency),
      limits: {
        dailyTransferLimit: Money.fromDecimal(300000, currency),
        singleTransferLimit: Money.fromDecimal(100000, currency),
        monthlyWithdrawalLimit: Money.fromDecimal(1000000, currency),
        minimumBalance: Money.fromDecimal(1000, currency),
      },
      metadata: { openedAt: now },
      currency,
    };
  }

  static createInvestment(ownerId: string, currency: Currency = 'THB'): Account {
    const now = new Date();
    return {
      id: uuidv4(),
      accountNumber: AccountFactory.generateAccountNumber('003'),
      ownerId,
      type: 'INVESTMENT',
      status: 'PENDING_VERIFICATION',
      balance: Money.zero(currency),
      availableBalance: Money.zero(currency),
      pendingHolds: Money.zero(currency),
      limits: {
        dailyTransferLimit: Money.fromDecimal(5000000, currency),
        singleTransferLimit: Money.fromDecimal(5000000, currency),
        monthlyWithdrawalLimit: Money.fromDecimal(50000000, currency),
        minimumBalance: Money.fromDecimal(10000, currency),
      },
      metadata: { openedAt: now },
      currency,
    };
  }

  private static generateAccountNumber(prefix: string): string {
    const timestamp = Date.now().toString().slice(-8);
    const random = Math.floor(Math.random() * 10000).toString().padStart(4, '0');
    return `${prefix}-${timestamp}-${random}`;
  }
}
```

### 3.3 Account Service

```typescript
// src/services/account.service.ts
import { Account, AccountType, AccountStatus } from '../domain/account';
import { Money, Currency } from '../domain/money';
import { AccountRepository } from '../repositories/account.repository';
import { AuditService } from './audit.service';

export class InsufficientFundsError extends Error {
  constructor(
    public readonly accountId: string,
    public readonly required: Money,
    public readonly available: Money
  ) {
    super(`เงินในบัญชีไม่เพียงพอ: ต้องการ ${required} มีอยู่ ${available}`);
    this.name = 'InsufficientFundsError';
  }
}

export class AccountFrozenError extends Error {
  constructor(public readonly accountId: string, reason?: string) {
    super(`บัญชีถูกอายัด${reason ? `: ${reason}` : ''}`);
    this.name = 'AccountFrozenError';
  }
}

export class LimitExceededError extends Error {
  constructor(
    public readonly limitType: string,
    public readonly amount: Money,
    public readonly limit: Money
  ) {
    super(`เกินวงเงิน${limitType}: ${amount} > ${limit}`);
    this.name = 'LimitExceededError';
  }
}

export class AccountService {
  constructor(
    private readonly accountRepo: AccountRepository,
    private readonly auditService: AuditService
  ) {}

  async getAccount(accountId: string): Promise<Account> {
    const account = await this.accountRepo.findById(accountId);
    if (!account) {
      throw new Error(`ไม่พบบัญชี: ${accountId}`);
    }
    return account;
  }

  async validateTransfer(
    fromAccountId: string,
    amount: Money
  ): Promise<void> {
    const account = await this.getAccount(fromAccountId);

    // ตรวจสอบสถานะบัญชี
    if (account.status === 'FROZEN') {
      throw new AccountFrozenError(fromAccountId, account.metadata.freezeReason);
    }
    if (account.status === 'CLOSED') {
      throw new Error('บัญชีถูกปิดแล้ว');
    }

    // ตรวจสอบยอดเงิน
    const balanceAfterTransfer = account.availableBalance.subtract(amount);
    if (balanceAfterTransfer.lessThan(account.limits.minimumBalance)) {
      throw new InsufficientFundsError(
        fromAccountId,
        amount,
        account.availableBalance
      );
    }

    // ตรวจสอบวงเงินต่อครั้ง
    if (amount.greaterThan(account.limits.singleTransferLimit)) {
      throw new LimitExceededError('ต่อครั้ง', amount, account.limits.singleTransferLimit);
    }

    // ตรวจสอบวงเงินต่อวัน
    const dailyTotal = await this.getDailyTransferTotal(fromAccountId);
    const projectedDaily = dailyTotal.add(amount);
    if (projectedDaily.greaterThan(account.limits.dailyTransferLimit)) {
      throw new LimitExceededError('ต่อวัน', projectedDaily, account.limits.dailyTransferLimit);
    }
  }

  async freezeAccount(accountId: string, reason: string, operatorId: string): Promise<void> {
    const account = await this.getAccount(accountId);
    
    await this.accountRepo.updateStatus(accountId, 'FROZEN', reason);
    
    await this.auditService.log({
      action: 'ACCOUNT_FROZEN',
      accountId,
      operatorId,
      details: { reason },
      timestamp: new Date(),
    });
  }

  async unfreezeAccount(accountId: string, operatorId: string): Promise<void> {
    const account = await this.getAccount(accountId);
    
    if (account.status !== 'FROZEN') {
      throw new Error('บัญชีไม่ได้ถูกอายัด');
    }
    
    await this.accountRepo.updateStatus(accountId, 'ACTIVE');
    
    await this.auditService.log({
      action: 'ACCOUNT_UNFROZEN',
      accountId,
      operatorId,
      timestamp: new Date(),
    });
  }

  async updateBalance(
    accountId: string,
    newBalance: Money,
    reason: string
  ): Promise<void> {
    await this.accountRepo.updateBalance(accountId, newBalance);
  }

  private async getDailyTransferTotal(accountId: string): Promise<Money> {
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    
    const transactions = await this.accountRepo.getTransactionsByDate(
      accountId,
      today,
      new Date()
    );
    
    const account = await this.getAccount(accountId);
    
    return transactions
      .filter(t => t.type === 'DEBIT')
      .reduce(
        (sum, t) => sum.add(t.amount),
        Money.zero(account.currency)
      );
  }
}
```

---

## 4. Transaction Types และ Validation

### 4.1 Transaction Domain

```typescript
// src/domain/transaction.ts
import { Money } from './money';
import { v4 as uuidv4 } from 'uuid';

export type TransactionType = 
  | 'DEPOSIT'
  | 'WITHDRAWAL'
  | 'TRANSFER_IN'
  | 'TRANSFER_OUT'
  | 'FEE'
  | 'INTEREST'
  | 'REVERSAL'
  | 'ADJUSTMENT';

export type TransactionStatus = 
  | 'PENDING'
  | 'PROCESSING'
  | 'COMPLETED'
  | 'FAILED'
  | 'REVERSED';

export interface TransactionParty {
  accountId: string;
  accountNumber: string;
  accountHolder: string;
  bankCode?: string; // สำหรับ cross-bank
}

export interface Transaction {
  id: string;
  referenceId: string; // เลขอ้างอิงที่แสดงให้ลูกค้า
  type: TransactionType;
  status: TransactionStatus;
  amount: Money;
  fee?: Money;
  from?: TransactionParty;
  to?: TransactionParty;
  description: string;
  note?: string;
  createdAt: Date;
  processedAt?: Date;
  completedAt?: Date;
  metadata: Record<string, unknown>;
  reversalOf?: string; // transaction id ที่ถูก reverse
}

export interface TransactionFilter {
  accountId: string;
  types?: TransactionType[];
  status?: TransactionStatus[];
  fromDate?: Date;
  toDate?: Date;
  minAmount?: Money;
  maxAmount?: Money;
  page?: number;
  pageSize?: number;
}

export interface PaginatedTransactions {
  transactions: Transaction[];
  total: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}
```

### 4.2 Transaction Validator

```typescript
import Joi from 'joi';

export interface TransferRequest {
  fromAccountId: string;
  toAccountId: string;
  amount: string; // decimal string
  currency: Currency;
  description: string;
  note?: string;
}

const transferSchema = Joi.object({
  fromAccountId: Joi.string().uuid().required().messages({
    'string.uuid': 'รหัสบัญชีต้นทางไม่ถูกต้อง',
    'any.required': 'กรุณาระบุบัญชีต้นทาง',
  }),
  toAccountId: Joi.string().uuid().required().messages({
    'string.uuid': 'รหัสบัญชีปลายทางไม่ถูกต้อง',
    'any.required': 'กรุณาระบุบัญชีปลายทาง',
  }),
  amount: Joi.string()
    .pattern(/^\d+(\.\d{1,2})?$/)
    .required()
    .messages({
      'string.pattern.base': 'จำนวนเงินไม่ถูกต้อง (ทศนิยมไม่เกิน 2 ตำแหน่ง)',
    }),
  currency: Joi.string().valid('THB', 'USD', 'EUR', 'JPY', 'GBP').required(),
  description: Joi.string().min(1).max(255).required(),
  note: Joi.string().max(500).optional(),
}).custom((value, helpers) => {
  if (value.fromAccountId === value.toAccountId) {
    return helpers.error('any.invalid', { message: 'บัญชีต้นทางและปลายทางต้องไม่ใช่บัญชีเดียวกัน' });
  }
  return value;
});

export function validateTransferRequest(data: unknown): TransferRequest {
  const { error, value } = transferSchema.validate(data, { abortEarly: false });
  if (error) {
    const messages = error.details.map(d => d.message).join(', ');
    throw new Error(`ข้อมูลไม่ถูกต้อง: ${messages}`);
  }
  return value as TransferRequest;
}
```

---

## 5. Transfer Operations

### 5.1 Same-Bank Transfer

```typescript
// src/services/transfer.service.ts
import { v4 as uuidv4 } from 'uuid';
import { Money } from '../domain/money';
import { Transaction, TransactionType } from '../domain/transaction';
import { AccountService } from './account.service';
import { LedgerService } from './ledger.service';
import { AuditService } from './audit.service';
import { TransactionRepository } from '../repositories/transaction.repository';

export interface TransferResult {
  transactionId: string;
  referenceId: string;
  status: 'SUCCESS' | 'FAILED';
  fromBalance: Money;
  toBalance: Money;
  fee: Money;
  completedAt: Date;
}

export class TransferService {
  constructor(
    private readonly accountService: AccountService,
    private readonly ledgerService: LedgerService,
    private readonly transactionRepo: TransactionRepository,
    private readonly auditService: AuditService
  ) {}

  async transferSameBank(
    fromAccountId: string,
    toAccountId: string,
    amount: Money,
    description: string,
    initiatedBy: string
  ): Promise<TransferResult> {
    // 1. Validate
    await this.accountService.validateTransfer(fromAccountId, amount);

    const fromAccount = await this.accountService.getAccount(fromAccountId);
    const toAccount = await this.accountService.getAccount(toAccountId);

    if (fromAccount.currency !== toAccount.currency) {
      throw new Error('ไม่รองรับการโอนระหว่างสกุลเงินต่างกัน (ใช้ FX Transfer แทน)');
    }

    // 2. คำนวณค่าธรรมเนียม
    const fee = this.calculateTransferFee(amount, 'SAME_BANK');

    // 3. สร้าง transaction records
    const transactionId = uuidv4();
    const referenceId = this.generateReferenceId();

    const debitTransaction: Transaction = {
      id: uuidv4(),
      referenceId,
      type: 'TRANSFER_OUT',
      status: 'PENDING',
      amount,
      fee,
      from: {
        accountId: fromAccountId,
        accountNumber: fromAccount.accountNumber,
        accountHolder: fromAccount.ownerId,
      },
      to: {
        accountId: toAccountId,
        accountNumber: toAccount.accountNumber,
        accountHolder: toAccount.ownerId,
      },
      description,
      createdAt: new Date(),
      metadata: { transactionId, initiatedBy },
    };

    const creditTransaction: Transaction = {
      id: uuidv4(),
      referenceId,
      type: 'TRANSFER_IN',
      status: 'PENDING',
      amount,
      from: {
        accountId: fromAccountId,
        accountNumber: fromAccount.accountNumber,
        accountHolder: fromAccount.ownerId,
      },
      to: {
        accountId: toAccountId,
        accountNumber: toAccount.accountNumber,
        accountHolder: toAccount.ownerId,
      },
      description,
      createdAt: new Date(),
      metadata: { transactionId, initiatedBy },
    };

    // 4. Execute transfer (atomic operation)
    try {
      await this.transactionRepo.beginTransaction();

      // บันทึก journal entries (double-entry bookkeeping)
      await this.ledgerService.recordTransfer({
        debitAccountId: fromAccountId,
        creditAccountId: toAccountId,
        amount,
        fee,
        referenceId,
      });

      // อัพเดท balance
      const totalDebit = amount.add(fee);
      const newFromBalance = fromAccount.balance.subtract(totalDebit);
      const newToBalance = toAccount.balance.add(amount);

      await this.accountService.updateBalance(fromAccountId, newFromBalance, 'TRANSFER_OUT');
      await this.accountService.updateBalance(toAccountId, newToBalance, 'TRANSFER_IN');

      // บันทึก transactions
      debitTransaction.status = 'COMPLETED';
      debitTransaction.completedAt = new Date();
      creditTransaction.status = 'COMPLETED';
      creditTransaction.completedAt = new Date();

      await this.transactionRepo.save(debitTransaction);
      await this.transactionRepo.save(creditTransaction);

      await this.transactionRepo.commit();

      // Audit log
      await this.auditService.log({
        action: 'TRANSFER_COMPLETED',
        accountId: fromAccountId,
        operatorId: initiatedBy,
        details: {
          toAccountId,
          amount: amount.toJSON(),
          fee: fee.toJSON(),
          referenceId,
        },
        timestamp: new Date(),
      });

      return {
        transactionId,
        referenceId,
        status: 'SUCCESS',
        fromBalance: newFromBalance,
        toBalance: newToBalance,
        fee,
        completedAt: new Date(),
      };
    } catch (error) {
      await this.transactionRepo.rollback();
      
      debitTransaction.status = 'FAILED';
      creditTransaction.status = 'FAILED';
      await this.transactionRepo.save(debitTransaction);
      await this.transactionRepo.save(creditTransaction);

      throw error;
    }
  }

  // คำนวณค่าธรรมเนียม
  private calculateTransferFee(amount: Money, type: 'SAME_BANK' | 'CROSS_BANK'): Money {
    const feeRules = {
      SAME_BANK: { 
        threshold: Money.fromDecimal(1000, 'THB'), 
        belowFee: Money.fromDecimal(0, 'THB'),    // ฟรีถ้าต่ำกว่า 1,000
        aboveFee: Money.fromDecimal(0, 'THB'),     // ฟรีทุกกรณี (same bank)
      },
      CROSS_BANK: { 
        threshold: Money.fromDecimal(1000, 'THB'),
        belowFee: Money.fromDecimal(25, 'THB'),    // 25 บาท
        aboveFee: Money.fromDecimal(25, 'THB'),    // 25 บาท
      },
    };

    const rule = feeRules[type];
    return amount.greaterThan(rule.threshold) ? rule.aboveFee : rule.belowFee;
  }

  private generateReferenceId(): string {
    const date = new Date();
    const dateStr = date.toISOString().slice(0, 10).replace(/-/g, '');
    const random = Math.random().toString(36).substring(2, 8).toUpperCase();
    return `TXN${dateStr}${random}`;
  }
}
```

### 5.2 Cross-Bank Transfer (PromptPay / BAHTNET)

```typescript
export interface CrossBankTransferOptions {
  destinationBankCode: string;
  destinationAccountNumber: string;
  destinationAccountName: string;
  promptPayId?: string; // เบอร์โทร หรือ เลขบัตรประชาชน
}

export class CrossBankTransferService extends TransferService {
  async transferCrossBank(
    fromAccountId: string,
    destination: CrossBankTransferOptions,
    amount: Money,
    description: string,
    initiatedBy: string
  ): Promise<TransferResult> {
    const fromAccount = await this.accountService.getAccount(fromAccountId);
    
    // ตรวจสอบวงเงิน cross-bank
    await this.accountService.validateTransfer(fromAccountId, amount);

    const fee = this.calculateCrossBankFee(amount);
    const referenceId = this.generateReferenceId();

    // ส่งผ่านระบบ BAHTNET หรือ ORFT
    const externalRef = await this.submitToPaymentNetwork({
      fromBank: 'OUR_BANK_CODE',
      fromAccount: fromAccount.accountNumber,
      toBank: destination.destinationBankCode,
      toAccount: destination.destinationAccountNumber,
      amount,
      reference: referenceId,
    });

    // บันทึก pending transaction
    const transaction: Transaction = {
      id: uuidv4(),
      referenceId,
      type: 'TRANSFER_OUT',
      status: 'PROCESSING', // cross-bank อาจต้องรอ
      amount,
      fee,
      from: {
        accountId: fromAccountId,
        accountNumber: fromAccount.accountNumber,
        accountHolder: fromAccount.ownerId,
      },
      to: {
        accountId: `EXTERNAL:${destination.destinationBankCode}:${destination.destinationAccountNumber}`,
        accountNumber: destination.destinationAccountNumber,
        accountHolder: destination.destinationAccountName,
        bankCode: destination.destinationBankCode,
      },
      description,
      createdAt: new Date(),
      metadata: { externalRef, initiatedBy, destinationBank: destination.destinationBankCode },
    };

    await this.transactionRepo.save(transaction);

    // Hold เงินทันที ขณะรอการยืนยัน
    const totalDebit = amount.add(fee);
    await this.accountService.addHold(fromAccountId, totalDebit, referenceId);

    return {
      transactionId: transaction.id,
      referenceId,
      status: 'SUCCESS',
      fromBalance: fromAccount.balance,
      toBalance: Money.zero(amount.currency),
      fee,
      completedAt: new Date(),
    };
  }

  private async submitToPaymentNetwork(params: {
    fromBank: string;
    fromAccount: string;
    toBank: string;
    toAccount: string;
    amount: Money;
    reference: string;
  }): Promise<string> {
    // TODO: เชื่อมต่อกับ BOT payment network API จริง
    return `EXT-${params.reference}`;
  }

  private calculateCrossBankFee(amount: Money): Money {
    // ค่าธรรมเนียม PromptPay: ฟรีถ้าไม่เกิน 5,000 บาท
    const freeLimit = Money.fromDecimal(5000, 'THB');
    if (amount.lessThan(freeLimit) || amount.equals(freeLimit)) {
      return Money.zero('THB');
    }
    return Money.fromDecimal(10, 'THB'); // 10 บาท ถ้าเกิน 5,000
  }
}
```

---

## 6. Transaction History with Filters

### 6.1 Transaction Query Builder

```typescript
// src/repositories/transaction.repository.ts
export class TransactionQueryBuilder {
  private conditions: string[] = [];
  private params: unknown[] = [];
  private paramIndex = 1;

  whereAccount(accountId: string): this {
    this.conditions.push(
      `(from_account_id = $${this.paramIndex} OR to_account_id = $${this.paramIndex})`
    );
    this.params.push(accountId);
    this.paramIndex++;
    return this;
  }

  whereTypes(types: TransactionType[]): this {
    if (types.length > 0) {
      const placeholders = types.map(() => `$${this.paramIndex++}`).join(', ');
      this.conditions.push(`type IN (${placeholders})`);
      this.params.push(...types);
    }
    return this;
  }

  whereDateRange(from?: Date, to?: Date): this {
    if (from) {
      this.conditions.push(`created_at >= $${this.paramIndex}`);
      this.params.push(from);
      this.paramIndex++;
    }
    if (to) {
      this.conditions.push(`created_at <= $${this.paramIndex}`);
      this.params.push(to);
      this.paramIndex++;
    }
    return this;
  }

  whereAmountRange(min?: Money, max?: Money): this {
    if (min) {
      this.conditions.push(`amount_minor >= $${this.paramIndex}`);
      this.params.push(min.amount.toString());
      this.paramIndex++;
    }
    if (max) {
      this.conditions.push(`amount_minor <= $${this.paramIndex}`);
      this.params.push(max.amount.toString());
      this.paramIndex++;
    }
    return this;
  }

  build(): { sql: string; params: unknown[] } {
    const whereClause = this.conditions.length > 0
      ? `WHERE ${this.conditions.join(' AND ')}`
      : '';
    
    return {
      sql: `SELECT * FROM transactions ${whereClause} ORDER BY created_at DESC`,
      params: this.params,
    };
  }
}
```

### 6.2 Transaction Service with Pagination

```typescript
// src/services/transaction.service.ts
export class TransactionService {
  constructor(private readonly transactionRepo: TransactionRepository) {}

  async getTransactionHistory(
    filter: TransactionFilter
  ): Promise<PaginatedTransactions> {
    const page = filter.page ?? 1;
    const pageSize = Math.min(filter.pageSize ?? 20, 100); // max 100 ต่อหน้า
    const offset = (page - 1) * pageSize;

    const builder = new TransactionQueryBuilder()
      .whereAccount(filter.accountId);

    if (filter.types?.length) {
      builder.whereTypes(filter.types);
    }

    if (filter.fromDate || filter.toDate) {
      builder.whereDateRange(filter.fromDate, filter.toDate);
    }

    if (filter.minAmount || filter.maxAmount) {
      builder.whereAmountRange(filter.minAmount, filter.maxAmount);
    }

    const { transactions, total } = await this.transactionRepo.findWithPagination(
      builder,
      offset,
      pageSize
    );

    return {
      transactions,
      total,
      page,
      pageSize,
      hasMore: offset + pageSize < total,
    };
  }

  // สรุปยอดรายเดือน
  async getMonthlySummary(
    accountId: string,
    year: number,
    month: number
  ): Promise<{
    totalIncome: Money;
    totalExpense: Money;
    netFlow: Money;
    transactionCount: number;
  }> {
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 0, 23, 59, 59);

    const transactions = await this.transactionRepo.findAll({
      accountId,
      fromDate: startDate,
      toDate: endDate,
      status: ['COMPLETED'],
    });

    const currency = transactions[0]?.amount.currency ?? 'THB';
    let totalIncome = Money.zero(currency as Currency);
    let totalExpense = Money.zero(currency as Currency);

    for (const tx of transactions) {
      if (tx.type === 'TRANSFER_IN' || tx.type === 'DEPOSIT' || tx.type === 'INTEREST') {
        totalIncome = totalIncome.add(tx.amount);
      } else if (tx.type === 'TRANSFER_OUT' || tx.type === 'WITHDRAWAL' || tx.type === 'FEE') {
        totalExpense = totalExpense.add(tx.amount);
      }
    }

    return {
      totalIncome,
      totalExpense,
      netFlow: totalIncome.subtract(totalExpense),
      transactionCount: transactions.length,
    };
  }

  // ค้นหาธุรกรรมที่น่าสงสัย (Anti-fraud)
  async detectSuspiciousTransactions(
    accountId: string,
    timeWindowHours = 24
  ): Promise<Transaction[]> {
    const fromDate = new Date(Date.now() - timeWindowHours * 60 * 60 * 1000);
    
    const transactions = await this.transactionRepo.findAll({
      accountId,
      fromDate,
      status: ['COMPLETED'],
    });

    const suspicious: Transaction[] = [];
    
    // กฎ 1: โอนหลายครั้งในช่วงเวลาสั้น (velocity check)
    const recentTxns = transactions.filter(
      t => t.type === 'TRANSFER_OUT' && 
      new Date(t.createdAt).getTime() > Date.now() - 60 * 60 * 1000 // 1 ชั่วโมง
    );
    if (recentTxns.length > 10) {
      suspicious.push(...recentTxns);
    }

    // กฎ 2: จำนวนเงินใกล้เคียง threshold (structuring detection)
    const largeThreshold = Money.fromDecimal(490000, 'THB');
    const limitThreshold = Money.fromDecimal(500000, 'THB');
    const nearThreshold = transactions.filter(
      t => t.amount.greaterThan(largeThreshold) && t.amount.lessThan(limitThreshold)
    );
    suspicious.push(...nearThreshold);

    // ลบ duplicates
    return [...new Map(suspicious.map(t => [t.id, t])).values()];
  }
}
```

---

## 7. Double-Entry Bookkeeping

### 7.1 Ledger Domain

```typescript
// src/domain/ledger.ts
export type EntryType = 'DEBIT' | 'CREDIT';
export type AccountingAccountType = 'ASSET' | 'LIABILITY' | 'EQUITY' | 'REVENUE' | 'EXPENSE';

export interface LedgerEntry {
  id: string;
  journalId: string;
  accountId: string;
  entryType: EntryType;
  amount: Money;
  description: string;
  createdAt: Date;
  metadata: Record<string, unknown>;
}

export interface JournalEntry {
  id: string;
  referenceId: string;
  entries: LedgerEntry[];
  description: string;
  createdAt: Date;
  createdBy: string;
  isBalanced: boolean; // debits === credits
}

// กฎ Double-Entry: debit === credit
export function validateJournalEntry(journal: JournalEntry): boolean {
  const currency = journal.entries[0]?.amount.currency;
  if (!currency) return false;

  let totalDebits = Money.zero(currency as Currency);
  let totalCredits = Money.zero(currency as Currency);

  for (const entry of journal.entries) {
    if (entry.entryType === 'DEBIT') {
      totalDebits = totalDebits.add(entry.amount);
    } else {
      totalCredits = totalCredits.add(entry.amount);
    }
  }

  return totalDebits.equals(totalCredits);
}
```

### 7.2 Ledger Service

```typescript
// src/services/ledger.service.ts
export class LedgerService {
  constructor(private readonly ledgerRepo: LedgerRepository) {}

  async recordTransfer(params: {
    debitAccountId: string;
    creditAccountId: string;
    amount: Money;
    fee: Money;
    referenceId: string;
  }): Promise<JournalEntry> {
    const journalId = uuidv4();
    const now = new Date();

    const entries: LedgerEntry[] = [
      // Debit ต้นทาง (ลดยอดทรัพย์สิน)
      {
        id: uuidv4(),
        journalId,
        accountId: params.debitAccountId,
        entryType: 'DEBIT',
        amount: params.amount,
        description: 'โอนออก',
        createdAt: now,
        metadata: { referenceId: params.referenceId },
      },
      // Credit ปลายทาง (เพิ่มยอดทรัพย์สิน)
      {
        id: uuidv4(),
        journalId,
        accountId: params.creditAccountId,
        entryType: 'CREDIT',
        amount: params.amount,
        description: 'รับโอน',
        createdAt: now,
        metadata: { referenceId: params.referenceId },
      },
    ];

    // บันทึกค่าธรรมเนียมถ้ามี
    if (!params.fee.isZero()) {
      entries.push(
        // Debit ค่าธรรมเนียมจากลูกค้า
        {
          id: uuidv4(),
          journalId,
          accountId: params.debitAccountId,
          entryType: 'DEBIT',
          amount: params.fee,
          description: 'ค่าธรรมเนียมการโอน',
          createdAt: now,
          metadata: {},
        },
        // Credit รายได้ค่าธรรมเนียมให้ธนาคาร
        {
          id: uuidv4(),
          journalId,
          accountId: 'BANK_FEE_REVENUE_ACCOUNT',
          entryType: 'CREDIT',
          amount: params.fee,
          description: 'รายได้ค่าธรรมเนียมการโอน',
          createdAt: now,
          metadata: {},
        }
      );
    }

    const journal: JournalEntry = {
      id: journalId,
      referenceId: params.referenceId,
      entries,
      description: `โอนเงิน ${params.amount}`,
      createdAt: now,
      createdBy: 'SYSTEM',
      isBalanced: validateJournalEntry({ id: journalId, referenceId: params.referenceId, entries, description: '', createdAt: now, createdBy: 'SYSTEM', isBalanced: false }),
    };

    if (!journal.isBalanced) {
      throw new Error('Journal entry ไม่สมดุล (debits != credits)');
    }

    await this.ledgerRepo.saveJournal(journal);
    return journal;
  }

  async getAccountBalance(accountId: string, asOf?: Date): Promise<Money> {
    const entries = await this.ledgerRepo.getEntriesByAccount(accountId, asOf);
    
    if (entries.length === 0) {
      return Money.zero('THB');
    }

    const currency = entries[0].amount.currency;
    let balance = Money.zero(currency);

    for (const entry of entries) {
      if (entry.entryType === 'CREDIT') {
        balance = balance.add(entry.amount);
      } else {
        balance = balance.subtract(entry.amount);
      }
    }

    return balance;
  }

  // Trial Balance - ตรวจสอบความสมดุลของบัญชีทั้งระบบ
  async getTrialBalance(asOf: Date): Promise<{
    totalDebits: Money;
    totalCredits: Money;
    isBalanced: boolean;
  }> {
    const allEntries = await this.ledgerRepo.getAllEntries(asOf);
    
    if (allEntries.length === 0) {
      const zero = Money.zero('THB');
      return { totalDebits: zero, totalCredits: zero, isBalanced: true };
    }

    const currency = allEntries[0].amount.currency;
    let totalDebits = Money.zero(currency);
    let totalCredits = Money.zero(currency);

    for (const entry of allEntries) {
      if (entry.entryType === 'DEBIT') {
        totalDebits = totalDebits.add(entry.amount);
      } else {
        totalCredits = totalCredits.add(entry.amount);
      }
    }

    return {
      totalDebits,
      totalCredits,
      isBalanced: totalDebits.equals(totalCredits),
    };
  }
}
```

---

## 8. Audit Logging

### 8.1 Audit Service

```typescript
// src/services/audit.service.ts
import winston from 'winston';

export type AuditAction =
  | 'ACCOUNT_CREATED'
  | 'ACCOUNT_FROZEN'
  | 'ACCOUNT_UNFROZEN'
  | 'ACCOUNT_CLOSED'
  | 'TRANSFER_INITIATED'
  | 'TRANSFER_COMPLETED'
  | 'TRANSFER_FAILED'
  | 'TRANSFER_REVERSED'
  | 'LOGIN_SUCCESS'
  | 'LOGIN_FAILED'
  | 'PASSWORD_CHANGED'
  | 'LIMIT_CHANGED'
  | 'SUSPICIOUS_ACTIVITY';

export interface AuditLog {
  id: string;
  action: AuditAction;
  accountId?: string;
  operatorId?: string;
  ipAddress?: string;
  userAgent?: string;
  details: Record<string, unknown>;
  timestamp: Date;
  sessionId?: string;
}

export class AuditService {
  private readonly logger: winston.Logger;

  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({ 
          filename: 'logs/audit.log',
          maxsize: 50 * 1024 * 1024, // 50MB
          maxFiles: 30, // เก็บ 30 ไฟล์
          tailable: true,
        }),
        new winston.transports.File({ 
          filename: 'logs/audit-error.log',
          level: 'error',
        }),
      ],
    });

    if (process.env.NODE_ENV !== 'production') {
      this.logger.add(new winston.transports.Console({
        format: winston.format.simple(),
      }));
    }
  }

  async log(auditLog: Omit<AuditLog, 'id'>): Promise<void> {
    const log: AuditLog = {
      id: uuidv4(),
      ...auditLog,
    };

    // บันทึกลง log file
    this.logger.info('AUDIT', log);

    // บันทึกลง database สำหรับ query
    await this.saveToDatabase(log);

    // ถ้าเป็น action สำคัญ ส่ง alert
    if (this.isHighRiskAction(auditLog.action)) {
      await this.sendAlert(log);
    }
  }

  async getAuditTrail(
    accountId: string,
    fromDate?: Date,
    toDate?: Date
  ): Promise<AuditLog[]> {
    return this.queryDatabase({
      accountId,
      fromDate,
      toDate,
    });
  }

  private isHighRiskAction(action: AuditAction): boolean {
    const highRiskActions: AuditAction[] = [
      'ACCOUNT_FROZEN',
      'TRANSFER_REVERSED',
      'SUSPICIOUS_ACTIVITY',
      'LIMIT_CHANGED',
    ];
    return highRiskActions.includes(action);
  }

  private async sendAlert(log: AuditLog): Promise<void> {
    // ส่ง notification ไปยัง security team
    console.warn(`🚨 HIGH RISK ACTION: ${log.action} on account ${log.accountId}`);
    // TODO: ส่ง email/Slack alert จริง
  }

  private async saveToDatabase(log: AuditLog): Promise<void> {
    // TODO: implement database save
  }

  private async queryDatabase(params: {
    accountId: string;
    fromDate?: Date;
    toDate?: Date;
  }): Promise<AuditLog[]> {
    // TODO: implement database query
    return [];
  }
}
```

---

## 9. Rate Limiting สำหรับ Transfers

### 9.1 Rate Limiter Implementation

```typescript
// src/middleware/rate-limiter.ts
import { RateLimiterRedis, RateLimiterMemory } from 'rate-limiter-flexible';
import type Redis from 'ioredis';

export interface RateLimitResult {
  allowed: boolean;
  remainingPoints: number;
  msBeforeNext: number;
  resetAt: Date;
}

export class TransferRateLimiter {
  private readonly perMinuteLimiter: RateLimiterMemory;
  private readonly perHourLimiter: RateLimiterMemory;
  private readonly perDayLimiter: RateLimiterMemory;

  constructor() {
    // จำกัด 10 ครั้ง/นาที
    this.perMinuteLimiter = new RateLimiterMemory({
      points: 10,
      duration: 60,
      blockDuration: 60,
    });

    // จำกัด 100 ครั้ง/ชั่วโมง
    this.perHourLimiter = new RateLimiterMemory({
      points: 100,
      duration: 3600,
      blockDuration: 3600,
    });

    // จำกัด 500 ครั้ง/วัน
    this.perDayLimiter = new RateLimiterMemory({
      points: 500,
      duration: 86400,
      blockDuration: 86400,
    });
  }

  async checkLimit(userId: string): Promise<RateLimitResult> {
    const key = `transfer:${userId}`;

    try {
      const [minuteRes, hourRes, dayRes] = await Promise.all([
        this.perMinuteLimiter.consume(key),
        this.perHourLimiter.consume(key),
        this.perDayLimiter.consume(key),
      ]);

      // หา limiter ที่ใกล้เต็มที่สุด
      const mostRestricted = [minuteRes, hourRes, dayRes]
        .sort((a, b) => a.remainingPoints - b.remainingPoints)[0];

      return {
        allowed: true,
        remainingPoints: mostRestricted.remainingPoints,
        msBeforeNext: mostRestricted.msBeforeNext,
        resetAt: new Date(Date.now() + mostRestricted.msBeforeNext),
      };
    } catch (error: unknown) {
      if (error && typeof error === 'object' && 'msBeforeNext' in error) {
        const rateLimiterError = error as { msBeforeNext: number };
        return {
          allowed: false,
          remainingPoints: 0,
          msBeforeNext: rateLimiterError.msBeforeNext,
          resetAt: new Date(Date.now() + rateLimiterError.msBeforeNext),
        };
      }
      throw error;
    }
  }

  async resetLimit(userId: string): Promise<void> {
    const key = `transfer:${userId}`;
    await Promise.all([
      this.perMinuteLimiter.delete(key),
      this.perHourLimiter.delete(key),
      this.perDayLimiter.delete(key),
    ]);
  }
}

// Middleware สำหรับ Express
export function createRateLimitMiddleware(limiter: TransferRateLimiter) {
  return async (req: any, res: any, next: any) => {
    const userId = req.user?.id;
    if (!userId) {
      return res.status(401).json({ error: 'ไม่ได้รับการยืนยันตัวตน' });
    }

    const result = await limiter.checkLimit(userId);
    
    res.setHeader('X-RateLimit-Remaining', result.remainingPoints);
    res.setHeader('X-RateLimit-Reset', result.resetAt.toISOString());

    if (!result.allowed) {
      const waitSeconds = Math.ceil(result.msBeforeNext / 1000);
      return res.status(429).json({
        error: `คุณทำรายการบ่อยเกินไป กรุณารอ ${waitSeconds} วินาที`,
        retryAfter: waitSeconds,
      });
    }

    next();
  };
}
```

---

## 10. Balance Calculations

### 10.1 Balance Calculator

```typescript
// src/domain/balance-calculator.ts
export interface BalanceSnapshot {
  accountId: string;
  balance: Money;
  availableBalance: Money;
  pendingHolds: Money;
  reservedBalance: Money; // ขั้นต่ำที่ต้องรักษา
  snapshotAt: Date;
}

export class BalanceCalculator {
  // คำนวณ available balance
  static calculateAvailable(
    balance: Money,
    pendingHolds: Money,
    minimumBalance: Money
  ): Money {
    const afterHolds = balance.subtract(pendingHolds);
    const afterMinimum = afterHolds.subtract(minimumBalance);
    
    // ถ้าติดลบ แสดงว่าใช้ได้ 0
    return afterMinimum.isNegative() ? Money.zero(balance.currency) : afterMinimum;
  }

  // คำนวณดอกเบี้ยเงินฝาก (compound interest)
  static calculateInterest(
    principal: Money,
    annualRate: number,
    compoundingFrequency: number, // ครั้ง/ปี (12 = รายเดือน)
    years: number
  ): Money {
    // สูตร: A = P(1 + r/n)^(nt)
    const r = annualRate / 100;
    const n = compoundingFrequency;
    const t = years;
    
    const factor = Math.pow(1 + r / n, n * t);
    const finalAmount = principal.multiply(factor);
    const interest = finalAmount.subtract(principal);
    
    return interest.round(2);
  }

  // คำนวณ average daily balance สำหรับดอกเบี้ย
  static calculateAverageDailyBalance(
    balanceHistory: Array<{ balance: Money; date: Date }>
  ): Money {
    if (balanceHistory.length === 0) return Money.zero('THB');

    const currency = balanceHistory[0].balance.currency;
    let totalBalance = Money.zero(currency);
    let totalDays = 0;

    for (let i = 0; i < balanceHistory.length - 1; i++) {
      const current = balanceHistory[i];
      const next = balanceHistory[i + 1];
      
      const days = Math.floor(
        (next.date.getTime() - current.date.getTime()) / (1000 * 60 * 60 * 24)
      );
      
      const weightedBalance = current.balance.multiply(days);
      totalBalance = totalBalance.add(weightedBalance);
      totalDays += days;
    }

    if (totalDays === 0) return balanceHistory[0].balance;
    return totalBalance.divide(totalDays);
  }
}
```

### 10.2 Interest Calculation Service

```typescript
export class InterestService {
  // คำนวณดอกเบี้ยออมทรัพย์รายเดือน
  async calculateMonthlyInterest(
    accountId: string,
    month: number,
    year: number
  ): Promise<Money> {
    const account = await this.accountRepo.findById(accountId);
    if (!account || account.type !== 'SAVINGS') {
      throw new Error('เฉพาะบัญชีออมทรัพย์เท่านั้นที่ได้รับดอกเบี้ย');
    }

    // ดึงประวัติ balance ทั้งเดือน
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 0);
    const balanceHistory = await this.getBalanceHistory(accountId, startDate, endDate);

    // คำนวณ average daily balance
    const avgBalance = BalanceCalculator.calculateAverageDailyBalance(balanceHistory);
    
    // อัตราดอกเบี้ย 1% ต่อปี (ตัวอย่าง)
    const annualRate = 0.01;
    const daysInMonth = endDate.getDate();
    const daysInYear = 365;
    
    // ดอกเบี้ย = (ADB × อัตรา × จำนวนวัน) / 365
    const interest = avgBalance
      .multiply(annualRate)
      .multiply(daysInMonth)
      .divide(daysInYear);
    
    return interest.round(2);
  }

  private async getBalanceHistory(
    accountId: string,
    from: Date,
    to: Date
  ): Promise<Array<{ balance: Money; date: Date }>> {
    // TODO: implement real query
    return [];
  }
}
```

---

## 11. API Routes

### 11.1 Express Routes

```typescript
// src/api/routes.ts
import express from 'express';
import { TransferService } from '../services/transfer.service';
import { AccountService } from '../services/account.service';
import { TransactionService } from '../services/transaction.service';
import { TransferRateLimiter, createRateLimitMiddleware } from '../middleware/rate-limiter';
import { validateTransferRequest } from '../domain/transaction';
import { Money } from '../domain/money';

const router = express.Router();
const rateLimiter = new TransferRateLimiter();

// GET /accounts/:id/balance
router.get('/accounts/:id/balance', async (req, res) => {
  try {
    const account = await accountService.getAccount(req.params.id);
    
    res.json({
      accountId: account.id,
      balance: account.balance.toJSON(),
      availableBalance: account.availableBalance.toJSON(),
      pendingHolds: account.pendingHolds.toJSON(),
      currency: account.currency,
      formattedBalance: account.balance.format(),
    });
  } catch (error) {
    res.status(404).json({ error: (error as Error).message });
  }
});

// POST /transfers
router.post(
  '/transfers',
  createRateLimitMiddleware(rateLimiter),
  async (req, res) => {
    try {
      const transferReq = validateTransferRequest(req.body);
      const amount = Money.fromDecimal(transferReq.amount, transferReq.currency);
      
      const result = await transferService.transferSameBank(
        transferReq.fromAccountId,
        transferReq.toAccountId,
        amount,
        transferReq.description,
        req.user?.id ?? 'UNKNOWN'
      );

      res.status(201).json({
        success: true,
        transactionId: result.transactionId,
        referenceId: result.referenceId,
        fromBalance: result.fromBalance.toJSON(),
        fee: result.fee.toJSON(),
        completedAt: result.completedAt,
      });
    } catch (error) {
      if ((error as Error).name === 'InsufficientFundsError') {
        return res.status(422).json({ error: (error as Error).message });
      }
      if ((error as Error).name === 'LimitExceededError') {
        return res.status(422).json({ error: (error as Error).message });
      }
      if ((error as Error).name === 'AccountFrozenError') {
        return res.status(403).json({ error: (error as Error).message });
      }
      res.status(500).json({ error: 'เกิดข้อผิดพลาดภายในระบบ' });
    }
  }
);

// GET /accounts/:id/transactions
router.get('/accounts/:id/transactions', async (req, res) => {
  try {
    const { page, pageSize, fromDate, toDate, types } = req.query;
    
    const filter: TransactionFilter = {
      accountId: req.params.id,
      page: page ? parseInt(page as string) : 1,
      pageSize: pageSize ? parseInt(pageSize as string) : 20,
      fromDate: fromDate ? new Date(fromDate as string) : undefined,
      toDate: toDate ? new Date(toDate as string) : undefined,
      types: types ? (types as string).split(',') as TransactionType[] : undefined,
    };

    const result = await transactionService.getTransactionHistory(filter);
    
    res.json({
      transactions: result.transactions.map(t => ({
        ...t,
        amount: t.amount.toJSON(),
        fee: t.fee?.toJSON(),
        formattedAmount: t.amount.format(),
      })),
      pagination: {
        total: result.total,
        page: result.page,
        pageSize: result.pageSize,
        hasMore: result.hasMore,
      },
    });
  } catch (error) {
    res.status(500).json({ error: (error as Error).message });
  }
});

export { router };
```

---

## 12. Complete Integration Test

### 12.1 End-to-End Test

```typescript
// tests/transfer.integration.test.ts
async function testCompleteTransferFlow() {
  console.log('=== ทดสอบ Flow การโอนเงินครบถ้วน ===\n');

  // สร้างบัญชีทดสอบ
  const alice = AccountFactory.createChecking('user-alice', 'THB');
  const bob = AccountFactory.createChecking('user-bob', 'THB');

  // เติมเงินในบัญชี Alice
  alice.status = 'ACTIVE';
  alice.balance = Money.fromDecimal(50000, 'THB');
  alice.availableBalance = Money.fromDecimal(49500, 'THB'); // หัก minimum balance

  bob.status = 'ACTIVE';
  bob.balance = Money.fromDecimal(10000, 'THB');
  bob.availableBalance = Money.fromDecimal(9500, 'THB');

  console.log('บัญชี Alice (ก่อน):', alice.balance.format());
  console.log('บัญชี Bob (ก่อน):', bob.balance.format());

  // โอนเงิน
  const transferAmount = Money.fromDecimal(15000, 'THB');
  
  const aliceNewBalance = alice.balance.subtract(transferAmount);
  const bobNewBalance = bob.balance.add(transferAmount);

  console.log('\n--- โอน 15,000 บาท จาก Alice ไป Bob ---');
  console.log('บัญชี Alice (หลัง):', aliceNewBalance.format());
  console.log('บัญชี Bob (หลัง):', bobNewBalance.format());

  // ทดสอบ Money precision
  const precision1 = Money.fromDecimal(0.1, 'THB');
  const precision2 = Money.fromDecimal(0.2, 'THB');
  const sum = precision1.add(precision2);
  console.log('\n=== ทดสอบ Precision ===');
  console.log(`0.1 + 0.2 = ${sum.toDecimalString()} (ต้องได้ 0.30)`);
  console.log(`ถูกต้อง: ${sum.toDecimalString() === '0.30'}`);

  // ทดสอบ compound interest
  const principal = Money.fromDecimal(100000, 'THB');
  const interest = BalanceCalculator.calculateInterest(principal, 1.5, 12, 1);
  console.log('\n=== ทดสอบ ดอกเบี้ยทบต้น ===');
  console.log(`เงินต้น: ${principal.format()}`);
  console.log(`ดอกเบี้ยปีที่ 1 (1.5% ต่อปี, ทบรายเดือน): ${interest.format()}`);
}

testCompleteTransferFlow().catch(console.error);
```

---

## 13. Error Handling and Recovery

### 13.1 Saga Pattern สำหรับ Distributed Transactions

```typescript
// ใช้ Saga pattern สำหรับ cross-service transactions
interface SagaStep<T> {
  name: string;
  execute: () => Promise<T>;
  compensate: () => Promise<void>;
}

class TransferSaga {
  private executedSteps: Array<{ name: string; compensate: () => Promise<void> }> = [];

  async execute(steps: SagaStep<unknown>[]): Promise<void> {
    for (const step of steps) {
      try {
        await step.execute();
        this.executedSteps.push({ 
          name: step.name, 
          compensate: step.compensate 
        });
      } catch (error) {
        console.error(`Saga step "${step.name}" failed:`, error);
        await this.compensate();
        throw error;
      }
    }
  }

  private async compensate(): Promise<void> {
    // ย้อนกลับทุก step ที่ execute ไปแล้ว (ย้อนกลับ)
    const stepsToCompensate = [...this.executedSteps].reverse();
    
    for (const step of stepsToCompensate) {
      try {
        console.log(`Compensating: ${step.name}`);
        await step.compensate();
      } catch (error) {
        console.error(`Compensation failed for ${step.name}:`, error);
        // บันทึก failed compensation สำหรับ manual intervention
      }
    }
  }
}

// ใช้งาน Saga
async function executeTransferWithSaga(
  fromAccountId: string,
  toAccountId: string,
  amount: Money
): Promise<void> {
  const saga = new TransferSaga();
  
  await saga.execute([
    {
      name: 'validate-balance',
      execute: async () => {
        await accountService.validateTransfer(fromAccountId, amount);
      },
      compensate: async () => {}, // ไม่ต้อง compensate
    },
    {
      name: 'debit-from-account',
      execute: async () => {
        await accountService.debitAccount(fromAccountId, amount);
      },
      compensate: async () => {
        // คืนเงินให้บัญชีต้นทาง
        await accountService.creditAccount(fromAccountId, amount);
      },
    },
    {
      name: 'credit-to-account',
      execute: async () => {
        await accountService.creditAccount(toAccountId, amount);
      },
      compensate: async () => {
        await accountService.debitAccount(toAccountId, amount);
      },
    },
    {
      name: 'record-ledger',
      execute: async () => {
        await ledgerService.recordTransfer({
          debitAccountId: fromAccountId,
          creditAccountId: toAccountId,
          amount,
          fee: Money.zero(amount.currency),
          referenceId: uuidv4(),
        });
      },
      compensate: async () => {
        // บันทึก reversal entry
      },
    },
  ]);
}
```

---

## 14. Monitoring และ Metrics

### 14.1 Transaction Metrics

```typescript
export class TransactionMetrics {
  private successCount = 0;
  private failureCount = 0;
  private totalAmount = Money.zero('THB');
  private responseTimes: number[] = [];

  recordSuccess(amount: Money, responseTimeMs: number): void {
    this.successCount++;
    this.totalAmount = this.totalAmount.add(amount);
    this.responseTimes.push(responseTimeMs);
  }

  recordFailure(): void {
    this.failureCount++;
  }

  getStats(): {
    successRate: number;
    totalTransactions: number;
    totalAmount: Money;
    avgResponseTime: number;
    p95ResponseTime: number;
    p99ResponseTime: number;
  } {
    const total = this.successCount + this.failureCount;
    const sorted = [...this.responseTimes].sort((a, b) => a - b);
    
    return {
      successRate: total > 0 ? this.successCount / total : 0,
      totalTransactions: total,
      totalAmount: this.totalAmount,
      avgResponseTime: sorted.length > 0 
        ? sorted.reduce((a, b) => a + b, 0) / sorted.length 
        : 0,
      p95ResponseTime: sorted[Math.floor(sorted.length * 0.95)] ?? 0,
      p99ResponseTime: sorted[Math.floor(sorted.length * 0.99)] ?? 0,
    };
  }
}
```

---

## สรุปบทที่ 86

ในบทนี้เราได้เรียนรู้:

1. **Money Type** - การหลีกเลี่ยงปัญหา floating-point ด้วย BigInt และ Decimal.js
2. **Account Management** - การจัดการบัญชีประเภทต่างๆ พร้อม limits
3. **Transaction Validation** - การตรวจสอบก่อนทำธุรกรรม
4. **Transfer Operations** - การโอนเงินทั้ง same-bank และ cross-bank
5. **Transaction History** - การค้นหาและ filter ประวัติธุรกรรม
6. **Double-Entry Bookkeeping** - หลักการบัญชีสองฝ่าย
7. **Audit Logging** - การบันทึกทุกการกระทำเพื่อตรวจสอบ
8. **Rate Limiting** - การป้องกันการทำธุรกรรมบ่อยเกินไป
9. **Balance Calculations** - การคำนวณยอดเงินและดอกเบี้ย
10. **Saga Pattern** - การจัดการ distributed transactions
