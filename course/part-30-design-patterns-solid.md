# ตอนที่ 30: SOLID Principles กับ TypeScript

## บทนำ

SOLID เป็นหลักการออกแบบซอฟต์แวร์ 5 ข้อที่ Robert C. Martin (Uncle Bob) นำเสนอ ซึ่งช่วยให้โค้ดมีความยืดหยุ่น บำรุงรักษาง่าย และขยายได้ดี TypeScript ด้วย type system ที่แข็งแกร่ง ช่วยให้เราสามารถนำ SOLID principles ไปใช้งานได้อย่างมีประสิทธิภาพ

---

## 1. Single Responsibility Principle (SRP)

### หลักการ
"A class should have one, and only one, reason to change."
คลาสหนึ่งๆ ควรมีหน้าที่รับผิดชอบเพียงหน้าที่เดียว และมีเหตุผลเปลี่ยนแปลงเพียงเหตุผลเดียว

### 1.1 ตัวอย่างที่ละเมิด SRP

```typescript
// ❌ BAD - UserManager ทำหลายสิ่ง: จัดการข้อมูล, ส่งอีเมล, บันทึก log, สร้าง report
class UserManager {
  private users: User[] = [];

  // หน้าที่ที่ 1: จัดการข้อมูลผู้ใช้
  addUser(user: User): void {
    this.users.push(user);
    // บันทึก log
    const logEntry = `${new Date().toISOString()} - User added: ${user.email}`;
    fs.appendFileSync('app.log', logEntry + '\n');
    // ส่งอีเมลยินดีต้อนรับ
    const emailBody = `
      สวัสดี ${user.name},
      ยินดีต้อนรับสู่ระบบของเรา!
    `;
    nodemailer.sendMail({
      to: user.email,
      subject: 'ยินดีต้อนรับ',
      text: emailBody,
    });
  }

  // หน้าที่ที่ 2: สร้าง report
  generateUserReport(): string {
    const lines = this.users.map(u =>
      `${u.id},${u.name},${u.email},${u.role}`
    );
    return ['id,name,email,role', ...lines].join('\n');
  }

  // หน้าที่ที่ 3: validate
  validateUser(user: User): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(user.email) && user.name.length > 0;
  }
}
```

### 1.2 การแก้ไขให้ถูกต้องตาม SRP

```typescript
// ✅ GOOD - แยก responsibilities ออกเป็นคลาสต่างๆ

// 1. User Repository - รับผิดชอบการจัดเก็บข้อมูล
interface UserRepository {
  findAll(): User[];
  findById(id: string): User | undefined;
  save(user: User): void;
  delete(id: string): void;
}

class InMemoryUserRepository implements UserRepository {
  private users: Map<string, User> = new Map();

  findAll(): User[] {
    return Array.from(this.users.values());
  }

  findById(id: string): User | undefined {
    return this.users.get(id);
  }

  save(user: User): void {
    this.users.set(user.id, user);
  }

  delete(id: string): void {
    this.users.delete(id);
  }
}

// 2. User Validator - รับผิดชอบการ validate
interface ValidationResult {
  isValid: boolean;
  errors: string[];
}

class UserValidator {
  validate(user: Partial<User>): ValidationResult {
    const errors: string[] = [];
    
    if (!user.name || user.name.trim().length === 0) {
      errors.push('ชื่อต้องไม่ว่าง');
    }
    
    if (!user.email) {
      errors.push('อีเมลต้องไม่ว่าง');
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(user.email)) {
      errors.push('รูปแบบอีเมลไม่ถูกต้อง');
    }
    
    if (!user.role) {
      errors.push('ต้องระบุบทบาท');
    }
    
    return { isValid: errors.length === 0, errors };
  }
}

// 3. Email Service - รับผิดชอบการส่งอีเมล
interface EmailMessage {
  to: string;
  subject: string;
  body: string;
}

interface EmailService {
  send(message: EmailMessage): Promise<void>;
}

class WelcomeEmailService {
  constructor(private emailService: EmailService) {}

  async sendWelcomeEmail(user: User): Promise<void> {
    await this.emailService.send({
      to: user.email,
      subject: `ยินดีต้อนรับสู่ระบบ, ${user.name}!`,
      body: `
        สวัสดีคุณ ${user.name},
        
        บัญชีของคุณถูกสร้างเรียบร้อยแล้ว
        อีเมล: ${user.email}
        บทบาท: ${user.role}
        
        ขอบคุณที่เข้าร่วมกับเรา!
      `,
    });
  }
}

// 4. Logger - รับผิดชอบการบันทึก log
interface Logger {
  info(message: string, meta?: Record<string, unknown>): void;
  error(message: string, error?: Error, meta?: Record<string, unknown>): void;
  warn(message: string, meta?: Record<string, unknown>): void;
}

class ConsoleLogger implements Logger {
  info(message: string, meta?: Record<string, unknown>): void {
    console.log(`[INFO] ${new Date().toISOString()} - ${message}`, meta ?? '');
  }

  error(message: string, error?: Error, meta?: Record<string, unknown>): void {
    console.error(`[ERROR] ${new Date().toISOString()} - ${message}`, error, meta ?? '');
  }

  warn(message: string, meta?: Record<string, unknown>): void {
    console.warn(`[WARN] ${new Date().toISOString()} - ${message}`, meta ?? '');
  }
}

// 5. Report Generator - รับผิดชอบการสร้าง report
interface ReportFormat {
  generate(users: User[]): string;
}

class CsvUserReport implements ReportFormat {
  generate(users: User[]): string {
    const header = 'id,name,email,role,createdAt';
    const rows = users.map(u =>
      `${u.id},"${u.name}","${u.email}",${u.role},${u.createdAt}`
    );
    return [header, ...rows].join('\n');
  }
}

class JsonUserReport implements ReportFormat {
  generate(users: User[]): string {
    return JSON.stringify(
      users.map(u => ({
        id: u.id,
        name: u.name,
        email: u.email,
        role: u.role,
        createdAt: u.createdAt,
      })),
      null,
      2
    );
  }
}

// 6. User Service - orchestrates ทุกอย่าง
class UserService {
  constructor(
    private userRepository: UserRepository,
    private userValidator: UserValidator,
    private welcomeEmailService: WelcomeEmailService,
    private logger: Logger
  ) {}

  async createUser(userData: Omit<User, 'id' | 'createdAt'>): Promise<User> {
    // Validate
    const validation = this.userValidator.validate(userData);
    if (!validation.isValid) {
      throw new Error(`Validation failed: ${validation.errors.join(', ')}`);
    }

    // Create user
    const user: User = {
      ...userData,
      id: crypto.randomUUID(),
      createdAt: new Date(),
    };

    // Save
    this.userRepository.save(user);
    this.logger.info('User created', { userId: user.id, email: user.email });

    // Send welcome email (fire and forget)
    this.welcomeEmailService.sendWelcomeEmail(user).catch(error => {
      this.logger.error('Failed to send welcome email', error, { userId: user.id });
    });

    return user;
  }

  getUser(id: string): User {
    const user = this.userRepository.findById(id);
    if (!user) throw new Error(`ไม่พบผู้ใช้ ID: ${id}`);
    return user;
  }

  getAllUsers(): User[] {
    return this.userRepository.findAll();
  }
}
```

---

## 2. Open/Closed Principle (OCP)

### หลักการ
"Software entities should be open for extension, but closed for modification."
ซอฟต์แวร์ควรสามารถขยายได้โดยไม่ต้องแก้ไขโค้ดที่มีอยู่แล้ว

### 2.1 ตัวอย่างที่ละเมิด OCP

```typescript
// ❌ BAD - ต้องแก้ไข PaymentProcessor ทุกครั้งที่เพิ่ม payment method ใหม่
class PaymentProcessor {
  processPayment(
    amount: number,
    currency: string,
    method: 'credit_card' | 'paypal' | 'bitcoin' | 'bank_transfer'
  ): PaymentResult {
    switch (method) {
      case 'credit_card':
        // Logic สำหรับ credit card
        console.log(`ชำระด้วยบัตรเครดิต: ${amount} ${currency}`);
        return { success: true, transactionId: 'CC-001', fee: amount * 0.03 };
      
      case 'paypal':
        // Logic สำหรับ PayPal
        console.log(`ชำระด้วย PayPal: ${amount} ${currency}`);
        return { success: true, transactionId: 'PP-001', fee: amount * 0.029 };
      
      case 'bitcoin':
        // Logic สำหรับ Bitcoin
        console.log(`ชำระด้วย Bitcoin: ${amount} ${currency}`);
        return { success: true, transactionId: 'BTC-001', fee: amount * 0.01 };
      
      case 'bank_transfer':
        // Logic สำหรับ Bank Transfer
        console.log(`โอนเงินผ่านธนาคาร: ${amount} ${currency}`);
        return { success: true, transactionId: 'BT-001', fee: 25 };
      
      // ถ้าต้องการเพิ่ม PromptPay ต้องแก้ไขคลาสนี้ ❌
      default:
        throw new Error(`ไม่รองรับวิธีการชำระเงิน: ${method}`);
    }
  }
}
```

### 2.2 การแก้ไขให้ถูกต้องตาม OCP

```typescript
// ✅ GOOD - ใช้ polymorphism แทน switch

interface PaymentResult {
  success: boolean;
  transactionId: string;
  fee: number;
  message?: string;
}

interface PaymentDetails {
  amount: number;
  currency: string;
  description?: string;
}

// Abstract payment strategy
abstract class PaymentMethod {
  abstract readonly name: string;
  abstract readonly feeRate: number;
  
  abstract process(details: PaymentDetails): Promise<PaymentResult>;
  
  calculateFee(amount: number): number {
    return amount * this.feeRate;
  }
  
  protected generateTransactionId(): string {
    return `${this.name.toUpperCase()}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Credit Card
class CreditCardPayment extends PaymentMethod {
  readonly name = 'credit_card';
  readonly feeRate = 0.03;

  constructor(
    private cardNumber: string,
    private expiryDate: string,
    private cvv: string
  ) {
    super();
  }

  async process(details: PaymentDetails): Promise<PaymentResult> {
    // ตรวจสอบ card validity
    if (!this.validateCard()) {
      return { success: false, transactionId: '', fee: 0, message: 'บัตรไม่ถูกต้อง' };
    }
    
    const fee = this.calculateFee(details.amount);
    
    // Call credit card API...
    console.log(`ชำระด้วยบัตรเครดิต: ${details.amount} ${details.currency}`);
    
    return {
      success: true,
      transactionId: this.generateTransactionId(),
      fee,
      message: `ชำระสำเร็จด้วยบัตรเครดิต`,
    };
  }

  private validateCard(): boolean {
    // Luhn algorithm check
    return this.cardNumber.length === 16;
  }
}

// PayPal
class PayPalPayment extends PaymentMethod {
  readonly name = 'paypal';
  readonly feeRate = 0.029;

  constructor(private paypalEmail: string) {
    super();
  }

  async process(details: PaymentDetails): Promise<PaymentResult> {
    console.log(`ชำระด้วย PayPal: ${details.amount} ${details.currency}`);
    
    return {
      success: true,
      transactionId: this.generateTransactionId(),
      fee: this.calculateFee(details.amount),
      message: `ชำระสำเร็จผ่าน PayPal`,
    };
  }
}

// PromptPay - เพิ่มใหม่โดยไม่ต้องแก้ไขคลาสเดิม ✅
class PromptPayPayment extends PaymentMethod {
  readonly name = 'promptpay';
  readonly feeRate = 0;

  constructor(private phoneNumber: string) {
    super();
  }

  async process(details: PaymentDetails): Promise<PaymentResult> {
    // Generate QR code
    const qrData = this.generateQRCode(details.amount);
    console.log(`ชำระด้วย PromptPay: ${details.amount} THB, QR: ${qrData}`);
    
    // Wait for payment confirmation...
    return {
      success: true,
      transactionId: this.generateTransactionId(),
      fee: 0, // PromptPay ฟรี!
      message: `ชำระสำเร็จผ่าน PromptPay`,
    };
  }

  private generateQRCode(amount: number): string {
    return `QR_${this.phoneNumber}_${amount}`;
  }
}

// Cryptocurrency - เพิ่มใหม่ง่ายมาก ✅
class CryptoPayment extends PaymentMethod {
  readonly name: string;
  readonly feeRate: number;

  constructor(
    private readonly cryptoType: 'BTC' | 'ETH' | 'USDT',
    private walletAddress: string
  ) {
    super();
    this.name = cryptoType.toLowerCase();
    this.feeRate = cryptoType === 'USDT' ? 0.001 : 0.01;
  }

  async process(details: PaymentDetails): Promise<PaymentResult> {
    console.log(`ชำระด้วย ${this.cryptoType}: ${details.amount} ${details.currency}`);
    
    return {
      success: true,
      transactionId: this.generateTransactionId(),
      fee: this.calculateFee(details.amount),
      message: `ชำระสำเร็จด้วย ${this.cryptoType}`,
    };
  }
}

// Payment Processor - ปิดสำหรับการแก้ไข เปิดสำหรับการขยาย
class PaymentProcessor {
  private methods: Map<string, PaymentMethod> = new Map();

  registerPaymentMethod(method: PaymentMethod): void {
    this.methods.set(method.name, method);
  }

  async processPayment(
    methodName: string,
    details: PaymentDetails
  ): Promise<PaymentResult> {
    const method = this.methods.get(methodName);
    if (!method) {
      throw new Error(`ไม่รองรับวิธีการชำระเงิน: ${methodName}`);
    }
    return method.process(details);
  }

  getAvailableMethods(): string[] {
    return Array.from(this.methods.keys());
  }
}

// การใช้งาน
const processor = new PaymentProcessor();
processor.registerPaymentMethod(new CreditCardPayment('4111111111111111', '12/25', '123'));
processor.registerPaymentMethod(new PayPalPayment('user@example.com'));
processor.registerPaymentMethod(new PromptPayPayment('0812345678'));
processor.registerPaymentMethod(new CryptoPayment('BTC', '1A2B3C4D'));

// เพิ่ม method ใหม่ได้เลยโดยไม่ต้องแก้ PaymentProcessor
```

### 2.3 OCP กับ Report Generation

```typescript
// ✅ OCP กับระบบ report
interface ReportData {
  title: string;
  generatedAt: Date;
  data: Record<string, unknown>[];
}

abstract class ReportGenerator {
  abstract readonly format: string;
  abstract readonly mimeType: string;
  
  abstract generate(reportData: ReportData): Buffer | string;
  
  getFileName(title: string): string {
    const sanitized = title.toLowerCase().replace(/\s+/g, '-');
    return `${sanitized}-${Date.now()}.${this.format}`;
  }
}

class CsvReportGenerator extends ReportGenerator {
  readonly format = 'csv';
  readonly mimeType = 'text/csv';

  generate(reportData: ReportData): string {
    if (reportData.data.length === 0) return '';
    
    const headers = Object.keys(reportData.data[0]).join(',');
    const rows = reportData.data.map(row =>
      Object.values(row).map(v => `"${v}"`).join(',')
    );
    
    return [headers, ...rows].join('\n');
  }
}

class JsonReportGenerator extends ReportGenerator {
  readonly format = 'json';
  readonly mimeType = 'application/json';

  generate(reportData: ReportData): string {
    return JSON.stringify({
      title: reportData.title,
      generatedAt: reportData.generatedAt.toISOString(),
      count: reportData.data.length,
      data: reportData.data,
    }, null, 2);
  }
}

class PdfReportGenerator extends ReportGenerator {
  readonly format = 'pdf';
  readonly mimeType = 'application/pdf';

  generate(reportData: ReportData): Buffer {
    // สร้าง PDF (ตัวอย่างเท่านั้น)
    const content = `PDF: ${reportData.title} - ${reportData.data.length} records`;
    return Buffer.from(content);
  }
}

// Report Service ที่ปิดสำหรับการแก้ไข
class ReportService {
  private generators: Map<string, ReportGenerator> = new Map();

  registerGenerator(generator: ReportGenerator): void {
    this.generators.set(generator.format, generator);
  }

  export(data: ReportData, format: string): { content: Buffer | string; fileName: string; mimeType: string } {
    const generator = this.generators.get(format);
    if (!generator) {
      throw new Error(`ไม่รองรับ format: ${format}`);
    }
    
    return {
      content: generator.generate(data),
      fileName: generator.getFileName(data.title),
      mimeType: generator.mimeType,
    };
  }
}
```

---

## 3. Liskov Substitution Principle (LSP)

### หลักการ
"Objects of a subtype should be substitutable for objects of the supertype."
objects ของ subtype ควรใช้แทน objects ของ supertype ได้โดยไม่ทำให้โปรแกรมผิดพลาด

### 3.1 ตัวอย่างที่ละเมิด LSP

```typescript
// ❌ BAD - Square ละเมิด LSP เพราะ Square ไม่สามารถแทน Rectangle ได้อย่างถูกต้อง
class Rectangle {
  protected width: number = 0;
  protected height: number = 0;

  setWidth(width: number): void { this.width = width; }
  setHeight(height: number): void { this.height = height; }
  getArea(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  setWidth(width: number): void {
    this.width = width;
    this.height = width; // ละเมิด! แก้ both dimensions
  }

  setHeight(height: number): void {
    this.height = height;
    this.width = height; // ละเมิด! แก้ both dimensions
  }
}

// Function ที่ assume behaviors ของ Rectangle
function processShape(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(10);
  
  // สำหรับ Rectangle: 5 * 10 = 50 ✓
  // สำหรับ Square: 10 * 10 = 100 ✗ (ผิดคาดหมาย!)
  console.log(`Area: ${rect.getArea()}`); 
}

const rect = new Rectangle();
processShape(rect); // 50 ✓

const sq = new Square();
processShape(sq); // 100 ✗ LSP violation!
```

### 3.2 การแก้ไขให้ถูกต้องตาม LSP

```typescript
// ✅ GOOD - ใช้ interface แทน inheritance ที่ไม่เหมาะสม
interface Shape {
  getArea(): number;
  getPerimeter(): number;
  describe(): string;
}

class Rectangle implements Shape {
  constructor(
    private readonly width: number,
    private readonly height: number
  ) {}

  getArea(): number { return this.width * this.height; }
  getPerimeter(): number { return 2 * (this.width + this.height); }
  describe(): string { return `สี่เหลี่ยมผืนผ้า ${this.width}x${this.height}`; }
}

class Square implements Shape {
  constructor(private readonly side: number) {}

  getArea(): number { return this.side * this.side; }
  getPerimeter(): number { return 4 * this.side; }
  describe(): string { return `สี่เหลี่ยมจัตุรัส ${this.side}x${this.side}`; }
}

class Circle implements Shape {
  constructor(private readonly radius: number) {}

  getArea(): number { return Math.PI * this.radius ** 2; }
  getPerimeter(): number { return 2 * Math.PI * this.radius; }
  describe(): string { return `วงกลม รัศมี ${this.radius}`; }
}

// ใช้งานได้กับ Shape ทุกชนิด
function printShapeInfo(shape: Shape): void {
  console.log(`รูปร่าง: ${shape.describe()}`);
  console.log(`พื้นที่: ${shape.getArea().toFixed(2)}`);
  console.log(`เส้นรอบรูป: ${shape.getPerimeter().toFixed(2)}`);
}

const shapes: Shape[] = [
  new Rectangle(5, 10),
  new Square(7),
  new Circle(3),
];

shapes.forEach(printShapeInfo); // ทำงานถูกต้องทุกตัว ✓
```

### 3.3 LSP กับ Collection Hierarchy

```typescript
// ✅ LSP-compliant collection hierarchy
interface ReadableCollection<T> {
  get(index: number): T | undefined;
  find(predicate: (item: T) => boolean): T | undefined;
  filter(predicate: (item: T) => boolean): T[];
  map<U>(transform: (item: T) => U): U[];
  reduce<U>(callback: (acc: U, item: T) => U, initial: U): U;
  forEach(callback: (item: T) => void): void;
  readonly size: number;
  [Symbol.iterator](): Iterator<T>;
}

interface WritableCollection<T> extends ReadableCollection<T> {
  add(item: T): void;
  remove(item: T): boolean;
  clear(): void;
}

class TypedArray<T> implements WritableCollection<T> {
  private items: T[] = [];

  get(index: number): T | undefined { return this.items[index]; }
  find(predicate: (item: T) => boolean): T | undefined { return this.items.find(predicate); }
  filter(predicate: (item: T) => boolean): T[] { return this.items.filter(predicate); }
  map<U>(transform: (item: T) => U): U[] { return this.items.map(transform); }
  reduce<U>(callback: (acc: U, item: T) => U, initial: U): U { return this.items.reduce(callback, initial); }
  forEach(callback: (item: T) => void): void { this.items.forEach(callback); }
  get size(): number { return this.items.length; }
  [Symbol.iterator](): Iterator<T> { return this.items[Symbol.iterator](); }

  add(item: T): void { this.items.push(item); }
  remove(item: T): boolean {
    const index = this.items.indexOf(item);
    if (index > -1) { this.items.splice(index, 1); return true; }
    return false;
  }
  clear(): void { this.items = []; }
}

// ImmutableCollection ยังคง LSP เพราะ ReadableCollection ไม่มี write operations
class ImmutableCollection<T> implements ReadableCollection<T> {
  constructor(private readonly items: T[]) {}

  get(index: number): T | undefined { return this.items[index]; }
  find(predicate: (item: T) => boolean): T | undefined { return this.items.find(predicate); }
  filter(predicate: (item: T) => boolean): T[] { return this.items.filter(predicate); }
  map<U>(transform: (item: T) => U): U[] { return this.items.map(transform); }
  reduce<U>(callback: (acc: U, item: T) => U, initial: U): U { return this.items.reduce(callback, initial); }
  forEach(callback: (item: T) => void): void { this.items.forEach(callback); }
  get size(): number { return this.items.length; }
  [Symbol.iterator](): Iterator<T> { return this.items[Symbol.iterator](); }

  // เพิ่ม method สำหรับ immutable operations
  with(item: T): ImmutableCollection<T> { return new ImmutableCollection([...this.items, item]); }
  without(item: T): ImmutableCollection<T> { return new ImmutableCollection(this.items.filter(i => i !== item)); }
}

// Function สามารถทำงานกับ ReadableCollection ได้ทั้งคู่
function sumNumbers(collection: ReadableCollection<number>): number {
  return collection.reduce((sum, n) => sum + n, 0);
}

const mutableList = new TypedArray<number>();
mutableList.add(1); mutableList.add(2); mutableList.add(3);

const immutableList = new ImmutableCollection<number>([1, 2, 3]);

console.log(sumNumbers(mutableList));   // 6 ✓
console.log(sumNumbers(immutableList)); // 6 ✓
```

---

## 4. Interface Segregation Principle (ISP)

### หลักการ
"Clients should not be forced to depend on interfaces they do not use."
ไม่ควรบังคับให้ client ต้องขึ้นอยู่กับ interface ที่ไม่ได้ใช้

### 4.1 ตัวอย่างที่ละเมิด ISP

```typescript
// ❌ BAD - Interface ใหญ่เกินไป ทุกคนต้อง implement ทุกอย่าง
interface Animal {
  // ทุกสัตว์ต้องทำทุกอย่าง แม้จะไม่สมเหตุสมผล
  eat(): void;
  sleep(): void;
  fly(): void;    // ปลาบิน? ❌
  swim(): void;   // นกบิน swim ได้? ❌
  run(): void;
  bark(): void;   // ทุกสัตว์เห่า? ❌
  meow(): void;
  layEggs(): void;
  giveBirth(): void; // นกออกลูก? ❌
}

// Dog ต้อง implement fly() ที่ไม่สมเหตุสมผล
class Dog implements Animal {
  eat(): void { console.log('กินอาหาร'); }
  sleep(): void { console.log('นอนหลับ'); }
  fly(): void { throw new Error('สุนัขบินไม่ได้!'); } // ❌
  swim(): void { console.log('ว่ายน้ำ'); }
  run(): void { console.log('วิ่ง'); }
  bark(): void { console.log('เห่า'); }
  meow(): void { throw new Error('สุนัขไม่ร้องเหมียว!'); } // ❌
  layEggs(): void { throw new Error('สุนัขไม่ออกไข่!'); } // ❌
  giveBirth(): void { console.log('ออกลูก'); }
}
```

### 4.2 การแก้ไขให้ถูกต้องตาม ISP

```typescript
// ✅ GOOD - แยก interfaces ตามความสามารถ

// Base interfaces - ความสามารถพื้นฐาน
interface Eatable {
  eat(food: string): void;
}

interface Sleepable {
  sleep(hours: number): void;
}

// Specialized interfaces
interface Flyable {
  fly(altitude: number): void;
  land(): void;
  readonly maxAltitude: number;
}

interface Swimmable {
  swim(distance: number): void;
  dive(depth: number): void;
  readonly maxDepth: number;
}

interface Walkable {
  walk(distance: number): void;
  run(speed: number): void;
}

interface Vocalizable {
  makeSound(): void;
}

interface EggLayer {
  layEggs(count: number): void;
}

interface LiveBearing {
  giveBirth(): void;
}

// Animals ที่ implement เฉพาะ interfaces ที่เกี่ยวข้อง
class Dog implements Eatable, Sleepable, Walkable, Vocalizable, Swimmable, LiveBearing {
  readonly maxDepth = 2; // เมตร

  eat(food: string): void { console.log(`สุนัขกิน: ${food}`); }
  sleep(hours: number): void { console.log(`สุนัขนอน ${hours} ชั่วโมง`); }
  walk(distance: number): void { console.log(`สุนัขเดิน ${distance} เมตร`); }
  run(speed: number): void { console.log(`สุนัขวิ่ง ${speed} กม./ชม.`); }
  makeSound(): void { console.log('โฮ่ง โฮ่ง!'); }
  swim(distance: number): void { console.log(`สุนัขว่ายน้ำ ${distance} เมตร`); }
  dive(depth: number): void {
    if (depth > this.maxDepth) throw new Error(`ดำได้ไม่เกิน ${this.maxDepth} เมตร`);
    console.log(`สุนัขดำน้ำ ${depth} เมตร`);
  }
  giveBirth(): void { console.log('สุนัขออกลูก'); }
}

class Eagle implements Eatable, Sleepable, Walkable, Flyable, Vocalizable, EggLayer {
  readonly maxAltitude = 3000; // เมตร

  eat(food: string): void { console.log(`นกอินทรีกิน: ${food}`); }
  sleep(hours: number): void { console.log(`นกอินทรีนอน ${hours} ชั่วโมง`); }
  walk(distance: number): void { console.log(`นกอินทรีเดิน ${distance} เมตร`); }
  run(speed: number): void { console.log(`นกอินทรีวิ่ง ${speed} กม./ชม.`); }
  fly(altitude: number): void {
    if (altitude > this.maxAltitude) throw new Error('บินสูงเกินไป!');
    console.log(`นกอินทรีบิน ${altitude} เมตร`);
  }
  land(): void { console.log('นกอินทรีลงจอด'); }
  makeSound(): void { console.log('แก็ก แก็ก!'); }
  layEggs(count: number): void { console.log(`นกอินทรีออกไข่ ${count} ฟอง`); }
}

class Shark implements Eatable, Sleepable, Swimmable, Vocalizable, LiveBearing {
  readonly maxDepth = 374; // เมตร

  eat(food: string): void { console.log(`ฉลามกิน: ${food}`); }
  sleep(hours: number): void { console.log(`ฉลามพัก ${hours} ชั่วโมง`); }
  swim(distance: number): void { console.log(`ฉลามว่ายน้ำ ${distance} เมตร`); }
  dive(depth: number): void {
    if (depth > this.maxDepth) throw new Error('ดำลึกเกินไป');
    console.log(`ฉลามดำน้ำ ${depth} เมตร`);
  }
  makeSound(): void { console.log('...'); } // ฉลามไม่มีเสียง
  giveBirth(): void { console.log('ฉลามออกลูก'); }
}

// Functions ที่ type-safe เฉพาะ capability ที่ต้องการ
function makeAnimalFly(animal: Flyable, altitude: number): void {
  animal.fly(altitude);
}

function feedAnimal(animal: Eatable, food: string): void {
  animal.eat(food);
}

function makeAnimalSwim(animal: Swimmable, distance: number): void {
  animal.swim(distance);
}

// TypeScript จะ error ถ้าใช้ผิด
makeAnimalFly(new Eagle(), 100);    // ✓
// makeAnimalFly(new Dog(), 100);   // ✗ TypeScript error! Dog ไม่ implement Flyable
feedAnimal(new Shark(), 'ปลา');    // ✓
makeAnimalSwim(new Dog(), 50);     // ✓
```

---

## 5. Dependency Inversion Principle (DIP)

### หลักการ
"High-level modules should not depend on low-level modules. Both should depend on abstractions."
โมดูลระดับสูงไม่ควรขึ้นอยู่กับโมดูลระดับต่ำ ทั้งคู่ควรขึ้นอยู่กับ abstractions

### 5.1 ตัวอย่างที่ละเมิด DIP

```typescript
// ❌ BAD - OrderService ขึ้นอยู่กับ concrete classes โดยตรง
class MySQLDatabase {
  save(table: string, data: Record<string, unknown>): void {
    console.log(`บันทึกใน MySQL: ${table}`, data);
  }
  
  find(table: string, id: string): Record<string, unknown> | null {
    console.log(`ค้นหาใน MySQL: ${table}/${id}`);
    return null;
  }
}

class GmailEmailService {
  sendEmail(to: string, subject: string, body: string): void {
    console.log(`ส่งอีเมลผ่าน Gmail ถึง ${to}: ${subject}`);
  }
}

class StripePaymentGateway {
  charge(amount: number, currency: string, cardToken: string): { success: boolean; transactionId: string } {
    console.log(`เรียกเก็บเงินผ่าน Stripe: ${amount} ${currency}`);
    return { success: true, transactionId: 'stripe_' + Date.now() };
  }
}

// ❌ OrderService ขึ้นอยู่กับ concrete classes
class OrderService {
  private db = new MySQLDatabase();           // concrete dependency ❌
  private emailService = new GmailEmailService();  // concrete dependency ❌
  private paymentGateway = new StripePaymentGateway(); // concrete dependency ❌

  async placeOrder(order: Order): Promise<void> {
    // ใช้ Stripe โดยตรง - ถ้าอยากเปลี่ยนเป็น Omise ต้องแก้ OrderService ❌
    const result = this.paymentGateway.charge(order.total, 'THB', order.cardToken!);
    
    if (result.success) {
      // บันทึกใน MySQL โดยตรง ❌
      this.db.save('orders', { ...order, transactionId: result.transactionId });
      
      // ส่ง Gmail โดยตรง ❌
      this.emailService.sendEmail(
        order.customerEmail,
        'ยืนยันคำสั่งซื้อ',
        `คำสั่งซื้อของคุณได้รับการยืนยันแล้ว`
      );
    }
  }
}
```

### 5.2 การแก้ไขให้ถูกต้องตาม DIP

```typescript
// ✅ GOOD - กำหนด abstractions ก่อน

// Abstractions (interfaces)
interface Database {
  save<T extends Record<string, unknown>>(collection: string, data: T): Promise<string>;
  findById<T>(collection: string, id: string): Promise<T | null>;
  findAll<T>(collection: string, filters?: Record<string, unknown>): Promise<T[]>;
  update<T>(collection: string, id: string, data: Partial<T>): Promise<T | null>;
  delete(collection: string, id: string): Promise<boolean>;
}

interface EmailNotificationService {
  sendOrderConfirmation(email: string, order: Order): Promise<void>;
  sendShippingNotification(email: string, orderId: string, trackingNumber: string): Promise<void>;
  sendPaymentFailure(email: string, orderId: string, reason: string): Promise<void>;
}

interface PaymentGateway {
  processPayment(amount: number, currency: string, paymentMethod: PaymentMethodData): Promise<PaymentGatewayResult>;
  refund(transactionId: string, amount?: number): Promise<RefundResult>;
  getTransaction(transactionId: string): Promise<TransactionInfo>;
}

interface InventoryService {
  checkAvailability(productId: string, quantity: number): Promise<boolean>;
  reserve(productId: string, quantity: number, orderId: string): Promise<ReservationResult>;
  release(productId: string, quantity: number, orderId: string): Promise<void>;
}

// ✅ High-level module ขึ้นอยู่กับ abstractions เท่านั้น
class OrderService {
  constructor(
    private readonly database: Database,
    private readonly emailService: EmailNotificationService,
    private readonly paymentGateway: PaymentGateway,
    private readonly inventoryService: InventoryService,
    private readonly logger: Logger
  ) {}

  async placeOrder(orderData: CreateOrderRequest): Promise<Order> {
    this.logger.info('เริ่มสร้างคำสั่งซื้อ', { customerId: orderData.customerId });

    // ตรวจสอบสินค้าในคลัง
    for (const item of orderData.items) {
      const available = await this.inventoryService.checkAvailability(
        item.productId,
        item.quantity
      );
      if (!available) {
        throw new InsufficientInventoryError(item.productId, item.quantity);
      }
    }

    // จองสินค้า
    const orderId = crypto.randomUUID();
    for (const item of orderData.items) {
      await this.inventoryService.reserve(item.productId, item.quantity, orderId);
    }

    // ชำระเงิน
    const paymentResult = await this.paymentGateway.processPayment(
      orderData.totalAmount,
      orderData.currency,
      orderData.paymentMethod
    );

    if (!paymentResult.success) {
      // คืนสินค้าที่จองไว้
      for (const item of orderData.items) {
        await this.inventoryService.release(item.productId, item.quantity, orderId);
      }
      throw new PaymentFailedException(paymentResult.errorMessage ?? 'Payment failed');
    }

    // บันทึกคำสั่งซื้อ
    const order: Order = {
      id: orderId,
      customerId: orderData.customerId,
      items: orderData.items,
      totalAmount: orderData.totalAmount,
      currency: orderData.currency,
      transactionId: paymentResult.transactionId,
      status: 'confirmed',
      createdAt: new Date(),
    };

    await this.database.save<Order>('orders', order);

    // ส่งอีเมลยืนยัน
    await this.emailService.sendOrderConfirmation(orderData.customerEmail, order);

    this.logger.info('สร้างคำสั่งซื้อสำเร็จ', { orderId, transactionId: paymentResult.transactionId });

    return order;
  }
}

// Low-level implementations (concrete classes)
class PostgreSQLDatabase implements Database {
  async save<T extends Record<string, unknown>>(collection: string, data: T): Promise<string> {
    console.log(`PostgreSQL: บันทึกใน ${collection}`);
    return data.id as string;
  }

  async findById<T>(collection: string, id: string): Promise<T | null> {
    console.log(`PostgreSQL: ค้นหาใน ${collection}/${id}`);
    return null;
  }

  async findAll<T>(collection: string, filters?: Record<string, unknown>): Promise<T[]> {
    console.log(`PostgreSQL: ค้นหาทั้งหมดใน ${collection}`, filters);
    return [];
  }

  async update<T>(collection: string, id: string, data: Partial<T>): Promise<T | null> {
    console.log(`PostgreSQL: อัปเดต ${collection}/${id}`);
    return null;
  }

  async delete(collection: string, id: string): Promise<boolean> {
    console.log(`PostgreSQL: ลบ ${collection}/${id}`);
    return true;
  }
}

// สามารถเปลี่ยน implementation ได้โดยไม่ต้องแก้ OrderService
class MongoDatabase implements Database {
  async save<T extends Record<string, unknown>>(collection: string, data: T): Promise<string> {
    console.log(`MongoDB: บันทึกใน ${collection}`);
    return data._id as string;
  }
  // ... implement other methods
}

class OmisePaymentGateway implements PaymentGateway {
  async processPayment(amount: number, currency: string, paymentMethod: PaymentMethodData): Promise<PaymentGatewayResult> {
    console.log(`Omise: เรียกเก็บ ${amount} ${currency}`);
    return { success: true, transactionId: `omise_${Date.now()}` };
  }
  // ... implement other methods
}

// Composition Root - จัดการ dependencies ตรงนี้ที่เดียว
function createOrderService(): OrderService {
  const db = new PostgreSQLDatabase();
  const emailService = new SendgridEmailService();
  const paymentGateway = new OmisePaymentGateway();
  const inventoryService = new WarehouseInventoryService();
  const logger = new ConsoleLogger();

  return new OrderService(db, emailService, paymentGateway, inventoryService, logger);
}
```

---

## 6. SOLID ในโลกความเป็นจริง

### 6.1 ตัวอย่าง E-commerce System

```typescript
// ระบบคิดราคา Product ที่ใช้ SOLID ครบทุกข้อ

// ISP: แยก interfaces ตาม role
interface Discountable {
  getDiscountPercentage(): number;
  isDiscountActive(): boolean;
}

interface Taxable {
  getTaxRate(): number;
  getTaxCategory(): string;
}

interface Shippable {
  getWeight(): number; // กก.
  getDimensions(): { width: number; height: number; depth: number }; // ซม.
  requiresSpecialHandling(): boolean;
}

// SRP: แต่ละคลาสมีหน้าที่เดียว
class PriceCalculator {
  calculate(basePrice: number, quantity: number): number {
    return basePrice * quantity;
  }
}

class DiscountCalculator {
  apply(price: number, discountable: Discountable): number {
    if (!discountable.isDiscountActive()) return price;
    return price * (1 - discountable.getDiscountPercentage() / 100);
  }
}

class TaxCalculator {
  calculate(price: number, taxable: Taxable): number {
    return price * taxable.getTaxRate();
  }
}

// OCP: PricingStrategy เปิดสำหรับการขยาย
abstract class PricingStrategy {
  abstract apply(basePrice: number, quantity: number): number;
  abstract describe(): string;
}

class StandardPricing extends PricingStrategy {
  apply(basePrice: number, quantity: number): number { return basePrice * quantity; }
  describe(): string { return 'ราคามาตรฐาน'; }
}

class BulkDiscountPricing extends PricingStrategy {
  constructor(
    private readonly minQuantity: number,
    private readonly discountPercent: number
  ) { super(); }

  apply(basePrice: number, quantity: number): number {
    if (quantity >= this.minQuantity) {
      return basePrice * quantity * (1 - this.discountPercent / 100);
    }
    return basePrice * quantity;
  }

  describe(): string {
    return `ลด ${this.discountPercent}% เมื่อซื้อ ${this.minQuantity}+ ชิ้น`;
  }
}

class TieredPricing extends PricingStrategy {
  constructor(private readonly tiers: Array<{ min: number; max: number; price: number }>) {
    super();
  }

  apply(basePrice: number, quantity: number): number {
    const tier = this.tiers.find(t => quantity >= t.min && quantity <= t.max);
    return tier ? tier.price * quantity : basePrice * quantity;
  }

  describe(): string { return 'ราคาแบบขั้นบันได'; }
}

// DIP: ProductPricingService ขึ้นอยู่กับ abstractions
class ProductPricingService {
  constructor(
    private readonly pricingStrategy: PricingStrategy,
    private readonly discountCalculator: DiscountCalculator,
    private readonly taxCalculator: TaxCalculator,
    private readonly logger: Logger
  ) {}

  calculateFinalPrice(
    basePrice: number,
    quantity: number,
    product: Discountable & Taxable
  ): PricingBreakdown {
    const subtotal = this.pricingStrategy.apply(basePrice, quantity);
    const afterDiscount = this.discountCalculator.apply(subtotal, product);
    const taxAmount = this.taxCalculator.calculate(afterDiscount, product);
    const total = afterDiscount + taxAmount;

    this.logger.info('คำนวณราคาสำเร็จ', { basePrice, quantity, total });

    return {
      subtotal,
      discountAmount: subtotal - afterDiscount,
      taxAmount,
      total,
      breakdown: {
        pricingStrategy: this.pricingStrategy.describe(),
        discountApplied: product.isDiscountActive() ? `${product.getDiscountPercentage()}%` : 'ไม่มี',
        taxCategory: product.getTaxCategory(),
        taxRate: `${product.getTaxRate() * 100}%`,
      },
    };
  }
}

interface PricingBreakdown {
  subtotal: number;
  discountAmount: number;
  taxAmount: number;
  total: number;
  breakdown: {
    pricingStrategy: string;
    discountApplied: string;
    taxCategory: string;
    taxRate: string;
  };
}
```

---

## 7. สรุป SOLID Principles

### 7.1 Checklist สำหรับ Code Review

```typescript
/*
 * SOLID Code Review Checklist:
 *
 * ✅ SRP - Single Responsibility Principle
 *   - คลาสนี้มีหน้าที่รับผิดชอบเดียวหรือไม่?
 *   - มีเหตุผลในการเปลี่ยนแปลงเพียงเหตุผลเดียวหรือไม่?
 *   - ถ้าคลาสทำหลายอย่าง ควรแยกออกได้หรือไม่?
 *
 * ✅ OCP - Open/Closed Principle
 *   - สามารถเพิ่ม behavior ใหม่ได้โดยไม่แก้ไขโค้ดเดิมหรือไม่?
 *   - ใช้ abstraction/interface/inheritance ที่เหมาะสมหรือไม่?
 *   - มี switch/if-else ที่ตรวจสอบ type แล้วทำสิ่งต่างๆ หรือไม่? (อาจละเมิด OCP)
 *
 * ✅ LSP - Liskov Substitution Principle
 *   - subtype สามารถแทน supertype ได้โดยไม่ทำให้โปรแกรมผิดหรือไม่?
 *   - method ใน subtype โยน exceptions ที่ supertype ไม่ได้โยนหรือไม่?
 *   - ข้อตกลง (preconditions, postconditions) ถูกรักษาไว้หรือไม่?
 *
 * ✅ ISP - Interface Segregation Principle
 *   - interface มีขนาดเล็กและเฉพาะเจาะจงหรือไม่?
 *   - มี client ที่ต้อง implement method ที่ไม่ใช้หรือไม่?
 *   - ควรแยก interface ออกเป็นส่วนเล็กๆ หรือไม่?
 *
 * ✅ DIP - Dependency Inversion Principle
 *   - high-level modules ขึ้นอยู่กับ abstractions ไม่ใช่ concrete classes?
 *   - มีการ inject dependencies แทนการสร้างใน constructor หรือไม่?
 *   - สามารถ mock dependencies ในการ test ได้ง่ายหรือไม่?
 */

// ตัวอย่าง Unit Test ที่ง่ายขึ้นเมื่อทำตาม DIP
class MockDatabase implements Database {
  private savedItems: Record<string, unknown>[] = [];
  
  async save<T extends Record<string, unknown>>(collection: string, data: T): Promise<string> {
    this.savedItems.push({ collection, ...data });
    return data.id as string;
  }
  
  async findById<T>(collection: string, id: string): Promise<T | null> {
    return this.savedItems.find(i => i.id === id) as T ?? null;
  }
  
  async findAll<T>(collection: string): Promise<T[]> {
    return this.savedItems.filter(i => i.collection === collection) as T[];
  }
  
  async update<T>(_collection: string, id: string, data: Partial<T>): Promise<T | null> {
    const index = this.savedItems.findIndex(i => i.id === id);
    if (index > -1) {
      this.savedItems[index] = { ...this.savedItems[index], ...data };
      return this.savedItems[index] as T;
    }
    return null;
  }
  
  async delete(_collection: string, id: string): Promise<boolean> {
    const index = this.savedItems.findIndex(i => i.id === id);
    if (index > -1) {
      this.savedItems.splice(index, 1);
      return true;
    }
    return false;
  }
  
  getSavedItems() { return this.savedItems; }
}

// Test ที่ clean และ isolated
describe('OrderService', () => {
  let orderService: OrderService;
  let mockDb: MockDatabase;
  let mockEmail: jest.Mocked<EmailNotificationService>;
  let mockPayment: jest.Mocked<PaymentGateway>;
  let mockInventory: jest.Mocked<InventoryService>;

  beforeEach(() => {
    mockDb = new MockDatabase();
    mockEmail = { 
      sendOrderConfirmation: jest.fn().mockResolvedValue(undefined),
      sendShippingNotification: jest.fn().mockResolvedValue(undefined),
      sendPaymentFailure: jest.fn().mockResolvedValue(undefined),
    };
    mockPayment = {
      processPayment: jest.fn().mockResolvedValue({ 
        success: true, 
        transactionId: 'test_123' 
      }),
      refund: jest.fn(),
      getTransaction: jest.fn(),
    };
    mockInventory = {
      checkAvailability: jest.fn().mockResolvedValue(true),
      reserve: jest.fn().mockResolvedValue({ success: true }),
      release: jest.fn().mockResolvedValue(undefined),
    };

    orderService = new OrderService(
      mockDb, mockEmail, mockPayment, mockInventory, new ConsoleLogger()
    );
  });

  it('ควรสร้างคำสั่งซื้อสำเร็จ', async () => {
    const result = await orderService.placeOrder({
      customerId: 'user-1',
      customerEmail: 'test@example.com',
      items: [{ productId: 'prod-1', quantity: 2, price: 100 }],
      totalAmount: 200,
      currency: 'THB',
      paymentMethod: { type: 'card', token: 'tok_test' },
    });

    expect(result.status).toBe('confirmed');
    expect(result.transactionId).toBe('test_123');
    expect(mockEmail.sendOrderConfirmation).toHaveBeenCalledOnce();
  });
});
```

---

SOLID Principles เป็นแนวทางที่ช่วยให้โค้ด TypeScript ของเรามีคุณภาพสูงขึ้น บำรุงรักษาง่ายขึ้น และทดสอบได้ง่ายขึ้น การนำไปใช้ไม่ได้หมายความว่าต้องทำทุกข้อในทุกส่วนของโค้ด แต่ควรใช้เมื่อมันเหมาะสมและช่วยแก้ปัญหาจริง

---

*จบบทที่ 30 - SOLID Principles กับ TypeScript*
