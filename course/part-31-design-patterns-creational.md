# ตอนที่ 31: Creational Design Patterns กับ TypeScript

## บทนำ

Creational Design Patterns คือรูปแบบการออกแบบที่เกี่ยวข้องกับกระบวนการสร้าง objects โดยมีเป้าหมายให้ระบบเป็นอิสระจาก "วิธีการสร้าง" objects เหล่านั้น TypeScript ด้วย type system ช่วยให้เราสามารถนำ patterns เหล่านี้ไปใช้ได้อย่างปลอดภัยและชัดเจน

**Creational Patterns หลัก 5 ตัว:**
1. **Singleton** - มั่นใจว่ามี instance เดียว
2. **Factory Method** - สร้าง objects โดย subclasses
3. **Abstract Factory** - สร้าง family ของ objects
4. **Builder** - สร้าง objects ทีละขั้นตอน
5. **Prototype** - clone objects ที่มีอยู่แล้ว

---

## 1. Singleton Pattern

### Intent (เจตนา)
รับประกันว่า class มี instance เพียง instance เดียว และจัดให้มี global access point สำหรับ instance นั้น

### เมื่อใดควรใช้
- เมื่อต้องการ object เดียวที่แชร์ทั้งระบบ (เช่น config, logger, connection pool)
- เมื่อต้องการควบคุมการเข้าถึง shared resource
- เมื่อ global state จำเป็นและต้องการ centralized control

### เมื่อใดไม่ควรใช้
- เมื่อ class ต้องการ testability สูง (Singleton ทำ unit test ยาก)
- เมื่อ instance อาจต้องมีหลายตัวในอนาคต
- เมื่อเป็น anti-pattern ที่ซ่อน dependencies

### 1.1 Singleton พื้นฐาน

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection | null = null;
  private connectionCount = 0;
  private isConnected = false;

  // Private constructor ป้องกันการสร้างด้วย new
  private constructor(
    private readonly host: string,
    private readonly port: number,
    private readonly database: string
  ) {}

  // Static method เพื่อเข้าถึง instance
  static getInstance(
    host: string = 'localhost',
    port: number = 5432,
    database: string = 'mydb'
  ): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection(host, port, database);
    }
    return DatabaseConnection.instance;
  }

  async connect(): Promise<void> {
    if (!this.isConnected) {
      console.log(`เชื่อมต่อกับ ${this.host}:${this.port}/${this.database}`);
      this.isConnected = true;
    }
    this.connectionCount++;
  }

  async disconnect(): Promise<void> {
    this.connectionCount--;
    if (this.connectionCount <= 0) {
      this.isConnected = false;
      console.log('ตัดการเชื่อมต่อ');
    }
  }

  async query<T>(sql: string, params: unknown[] = []): Promise<T[]> {
    if (!this.isConnected) throw new Error('ยังไม่ได้เชื่อมต่อ');
    console.log(`Query: ${sql}`, params);
    return [] as T[];
  }

  getStatus(): { connected: boolean; connections: number } {
    return { connected: this.isConnected, connections: this.connectionCount };
  }

  // สำหรับ testing - reset instance
  static resetInstance(): void {
    DatabaseConnection.instance = null;
  }
}

// การใช้งาน
const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();
console.log(db1 === db2); // true - เป็น instance เดียวกัน
```

### 1.2 Thread-safe Singleton (สำหรับ Node.js)

```typescript
class ConfigurationManager {
  private static instance: ConfigurationManager | null = null;
  private static creatingInstance = false;
  private config: Map<string, unknown> = new Map();

  private constructor() {
    this.loadDefaultConfig();
  }

  static getInstance(): ConfigurationManager {
    if (!ConfigurationManager.instance) {
      if (ConfigurationManager.creatingInstance) {
        throw new Error('Circular dependency detected in ConfigurationManager');
      }
      ConfigurationManager.creatingInstance = true;
      ConfigurationManager.instance = new ConfigurationManager();
      ConfigurationManager.creatingInstance = false;
    }
    return ConfigurationManager.instance;
  }

  private loadDefaultConfig(): void {
    this.config.set('app.name', 'MyApp');
    this.config.set('app.version', '1.0.0');
    this.config.set('api.timeout', 30000);
    this.config.set('cache.ttl', 300);
    this.config.set('log.level', 'info');
  }

  get<T>(key: string): T | undefined {
    return this.config.get(key) as T | undefined;
  }

  set<T>(key: string, value: T): void {
    this.config.set(key, value);
  }

  getOrDefault<T>(key: string, defaultValue: T): T {
    return (this.config.get(key) as T) ?? defaultValue;
  }

  loadFromEnvironment(): void {
    // โหลด config จาก environment variables
    Object.entries(process.env).forEach(([key, value]) => {
      if (value !== undefined) {
        this.config.set(`env.${key}`, value);
      }
    });
  }

  toObject(): Record<string, unknown> {
    return Object.fromEntries(this.config);
  }
}

// การใช้งาน
const config = ConfigurationManager.getInstance();
config.set('database.url', 'postgres://localhost/mydb');
console.log(config.get<string>('database.url')); // 'postgres://localhost/mydb'

// ทุกที่ที่เรียก getInstance() จะได้ instance เดียวกัน
const config2 = ConfigurationManager.getInstance();
console.log(config2.get<string>('database.url')); // 'postgres://localhost/mydb'
```

### 1.3 Singleton กับ Lazy Initialization

```typescript
// Logger Singleton แบบ lazy
class Logger {
  private static _instance: Logger;
  private logLevel: 'debug' | 'info' | 'warn' | 'error' = 'info';
  private logHistory: Array<{ level: string; message: string; timestamp: Date }> = [];

  private constructor() {}

  static get instance(): Logger {
    if (!Logger._instance) {
      Logger._instance = new Logger();
    }
    return Logger._instance;
  }

  setLevel(level: typeof this.logLevel): void {
    this.logLevel = level;
  }

  private shouldLog(level: typeof this.logLevel): boolean {
    const levels = ['debug', 'info', 'warn', 'error'];
    return levels.indexOf(level) >= levels.indexOf(this.logLevel);
  }

  private log(level: typeof this.logLevel, message: string, meta?: unknown): void {
    if (!this.shouldLog(level)) return;
    
    const entry = { level, message, timestamp: new Date() };
    this.logHistory.push(entry);
    
    const prefix = `[${level.toUpperCase()}] ${entry.timestamp.toISOString()}`;
    const output = meta ? `${prefix} - ${message}` : `${prefix} - ${message}`;
    
    switch (level) {
      case 'error': console.error(output, meta ?? ''); break;
      case 'warn': console.warn(output, meta ?? ''); break;
      default: console.log(output, meta ?? '');
    }
  }

  debug(message: string, meta?: unknown): void { this.log('debug', message, meta); }
  info(message: string, meta?: unknown): void { this.log('info', message, meta); }
  warn(message: string, meta?: unknown): void { this.log('warn', message, meta); }
  error(message: string, error?: Error, meta?: unknown): void {
    this.log('error', `${message}${error ? ': ' + error.message : ''}`, meta);
  }

  getHistory(count?: number): typeof this.logHistory {
    return count ? this.logHistory.slice(-count) : this.logHistory;
  }
}

// การใช้งาน
Logger.instance.setLevel('debug');
Logger.instance.info('แอปพลิเคชันเริ่มทำงาน');
Logger.instance.debug('โหลด config สำเร็จ', { config: { port: 3000 } });
Logger.instance.warn('Cache miss', { key: 'user:123' });
```

---

## 2. Factory Method Pattern

### Intent
กำหนด interface สำหรับสร้าง object แต่ให้ subclasses ตัดสินใจว่าจะสร้าง class ไหน

### เมื่อใดควรใช้
- เมื่อไม่รู้ล่วงหน้าว่าต้องสร้าง object ประเภทไหน
- เมื่อต้องการให้ subclasses ควบคุมการสร้าง objects
- เมื่อต้องการ encapsulate การสร้าง objects ที่ซับซ้อน

### 2.1 Factory Method พื้นฐาน

```typescript
// Product interface
interface Notification {
  readonly type: string;
  send(recipient: string, message: string): Promise<void>;
  schedule(recipient: string, message: string, scheduledAt: Date): Promise<void>;
  getDeliveryStatus(messageId: string): Promise<DeliveryStatus>;
}

type DeliveryStatus = 'pending' | 'sent' | 'delivered' | 'failed';

// Concrete Products
class EmailNotification implements Notification {
  readonly type = 'email';

  async send(recipient: string, message: string): Promise<void> {
    console.log(`📧 ส่งอีเมลถึง ${recipient}: ${message}`);
    // Call email API...
  }

  async schedule(recipient: string, message: string, scheduledAt: Date): Promise<void> {
    console.log(`📅 กำหนดส่งอีเมลถึง ${recipient} เวลา ${scheduledAt.toLocaleString('th-TH')}`);
  }

  async getDeliveryStatus(messageId: string): Promise<DeliveryStatus> {
    return 'delivered';
  }
}

class SMSNotification implements Notification {
  readonly type = 'sms';

  async send(recipient: string, message: string): Promise<void> {
    // ตัด message ถ้ายาวเกิน 160 ตัวอักษร
    const smsMessage = message.length > 160 ? message.substring(0, 157) + '...' : message;
    console.log(`📱 ส่ง SMS ถึง ${recipient}: ${smsMessage}`);
  }

  async schedule(recipient: string, message: string, scheduledAt: Date): Promise<void> {
    console.log(`📅 กำหนดส่ง SMS ถึง ${recipient} เวลา ${scheduledAt.toLocaleString('th-TH')}`);
  }

  async getDeliveryStatus(messageId: string): Promise<DeliveryStatus> {
    return 'sent';
  }
}

class PushNotification implements Notification {
  readonly type = 'push';
  
  constructor(private readonly appId: string) {}

  async send(recipient: string, message: string): Promise<void> {
    console.log(`🔔 ส่ง Push Notification ถึง ${recipient} ผ่าน App ${this.appId}: ${message}`);
  }

  async schedule(recipient: string, message: string, scheduledAt: Date): Promise<void> {
    console.log(`📅 กำหนดส่ง Push ถึง ${recipient} เวลา ${scheduledAt.toLocaleString('th-TH')}`);
  }

  async getDeliveryStatus(messageId: string): Promise<DeliveryStatus> {
    return 'pending';
  }
}

class LineNotification implements Notification {
  readonly type = 'line';

  async send(recipient: string, message: string): Promise<void> {
    console.log(`💚 ส่ง LINE message ถึง ${recipient}: ${message}`);
  }

  async schedule(recipient: string, message: string, scheduledAt: Date): Promise<void> {
    console.log(`📅 กำหนดส่ง LINE ถึง ${recipient} เวลา ${scheduledAt.toLocaleString('th-TH')}`);
  }

  async getDeliveryStatus(messageId: string): Promise<DeliveryStatus> {
    return 'delivered';
  }
}

// Creator (Factory Method)
abstract class NotificationFactory {
  abstract createNotification(): Notification;

  // Template method ที่ใช้ factory method
  async notify(recipient: string, message: string): Promise<void> {
    const notification = this.createNotification();
    await notification.send(recipient, message);
    console.log(`ส่งการแจ้งเตือนประเภท ${notification.type} สำเร็จ`);
  }

  async scheduledNotify(recipient: string, message: string, scheduledAt: Date): Promise<void> {
    const notification = this.createNotification();
    await notification.schedule(recipient, message, scheduledAt);
  }
}

// Concrete Creators
class EmailNotificationFactory extends NotificationFactory {
  createNotification(): Notification {
    return new EmailNotification();
  }
}

class SMSNotificationFactory extends NotificationFactory {
  createNotification(): Notification {
    return new SMSNotification();
  }
}

class PushNotificationFactory extends NotificationFactory {
  constructor(private readonly appId: string) {
    super();
  }

  createNotification(): Notification {
    return new PushNotification(this.appId);
  }
}

// การใช้งาน
async function sendNotifications() {
  const factories: NotificationFactory[] = [
    new EmailNotificationFactory(),
    new SMSNotificationFactory(),
    new PushNotificationFactory('my-app-id'),
  ];

  for (const factory of factories) {
    await factory.notify('user@example.com', 'คำสั่งซื้อของคุณได้รับการยืนยัน');
  }
}
```

### 2.2 Factory Method แบบ Registry

```typescript
// Registry pattern กับ Factory Method
type NotificationType = 'email' | 'sms' | 'push' | 'line';

interface NotificationConfig {
  email?: { from: string; replyTo?: string };
  sms?: { senderId: string };
  push?: { appId: string; apiKey: string };
  line?: { channelToken: string };
}

class NotificationRegistry {
  private static factories: Map<NotificationType, () => Notification> = new Map();

  static register(type: NotificationType, factory: () => Notification): void {
    NotificationRegistry.factories.set(type, factory);
  }

  static create(type: NotificationType): Notification {
    const factory = NotificationRegistry.factories.get(type);
    if (!factory) {
      throw new Error(`ไม่รู้จักประเภทการแจ้งเตือน: ${type}`);
    }
    return factory();
  }

  static getAvailableTypes(): NotificationType[] {
    return Array.from(NotificationRegistry.factories.keys());
  }
}

// ลงทะเบียน factories
NotificationRegistry.register('email', () => new EmailNotification());
NotificationRegistry.register('sms', () => new SMSNotification());
NotificationRegistry.register('push', () => new PushNotification('default-app'));

// การใช้งาน
const emailNotif = NotificationRegistry.create('email');
await emailNotif.send('user@example.com', 'ข้อความทดสอบ');
```

---

## 3. Abstract Factory Pattern

### Intent
จัดให้มี interface สำหรับสร้าง families of related objects โดยไม่ระบุ concrete classes

### เมื่อใดควรใช้
- เมื่อต้องการสร้างกลุ่มของ objects ที่เกี่ยวข้องกัน
- เมื่อต้องการรับประกันว่า objects ที่สร้างมา compatible กัน
- เมื่อต้องการ switch ระหว่าง families of objects

### 3.1 Abstract Factory สำหรับ UI Components

```typescript
// Abstract Products
interface Button {
  readonly label: string;
  onClick: () => void;
  render(): string;
  disable(): void;
  enable(): void;
}

interface InputField {
  readonly placeholder: string;
  value: string;
  render(): string;
  validate(): ValidationResult;
}

interface Modal {
  title: string;
  content: string;
  show(): void;
  hide(): void;
  render(): string;
}

interface ValidationResult {
  isValid: boolean;
  errors: string[];
}

// Concrete Products - Material UI Theme
class MaterialButton implements Button {
  private disabled = false;
  onClick: () => void = () => {};

  constructor(
    readonly label: string,
    private readonly variant: 'contained' | 'outlined' | 'text' = 'contained',
    private readonly color: 'primary' | 'secondary' | 'error' = 'primary'
  ) {}

  render(): string {
    const classes = `material-button ${this.variant} ${this.color} ${this.disabled ? 'disabled' : ''}`;
    return `<button class="${classes}" ${this.disabled ? 'disabled' : ''}>${this.label}</button>`;
  }

  disable(): void { this.disabled = true; }
  enable(): void { this.disabled = false; }
}

class MaterialInput implements InputField {
  value = '';

  constructor(readonly placeholder: string, private readonly type: 'text' | 'email' | 'password' = 'text') {}

  render(): string {
    return `
      <div class="material-input">
        <input type="${this.type}" placeholder="${this.placeholder}" value="${this.value}" />
        <label>${this.placeholder}</label>
      </div>
    `;
  }

  validate(): ValidationResult {
    const errors: string[] = [];
    if (!this.value.trim()) {
      errors.push(`${this.placeholder} ต้องไม่ว่าง`);
    }
    if (this.type === 'email' && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(this.value)) {
      errors.push('รูปแบบอีเมลไม่ถูกต้อง');
    }
    return { isValid: errors.length === 0, errors };
  }
}

class MaterialModal implements Modal {
  private isVisible = false;

  constructor(public title: string, public content: string) {}

  show(): void { this.isVisible = true; console.log(`แสดง Modal: ${this.title}`); }
  hide(): void { this.isVisible = false; console.log(`ซ่อน Modal: ${this.title}`); }

  render(): string {
    if (!this.isVisible) return '';
    return `
      <div class="material-modal">
        <div class="material-modal-header">${this.title}</div>
        <div class="material-modal-content">${this.content}</div>
      </div>
    `;
  }
}

// Concrete Products - Bootstrap Theme
class BootstrapButton implements Button {
  private disabled = false;
  onClick: () => void = () => {};

  constructor(
    readonly label: string,
    private readonly variant: 'primary' | 'secondary' | 'danger' | 'success' = 'primary',
    private readonly size: 'sm' | 'md' | 'lg' = 'md'
  ) {}

  render(): string {
    const sizeClass = this.size !== 'md' ? `btn-${this.size}` : '';
    return `<button class="btn btn-${this.variant} ${sizeClass}" ${this.disabled ? 'disabled' : ''}>${this.label}</button>`;
  }

  disable(): void { this.disabled = true; }
  enable(): void { this.disabled = false; }
}

class BootstrapInput implements InputField {
  value = '';

  constructor(readonly placeholder: string, private readonly type: 'text' | 'email' | 'password' = 'text') {}

  render(): string {
    return `
      <div class="mb-3">
        <label class="form-label">${this.placeholder}</label>
        <input type="${this.type}" class="form-control" placeholder="${this.placeholder}" value="${this.value}" />
      </div>
    `;
  }

  validate(): ValidationResult {
    const errors: string[] = [];
    if (!this.value.trim()) {
      errors.push(`${this.placeholder} จำเป็นต้องกรอก`);
    }
    return { isValid: errors.length === 0, errors };
  }
}

class BootstrapModal implements Modal {
  private isVisible = false;

  constructor(public title: string, public content: string) {}

  show(): void { this.isVisible = true; }
  hide(): void { this.isVisible = false; }

  render(): string {
    return `
      <div class="modal ${this.isVisible ? 'show' : ''}" style="display: ${this.isVisible ? 'block' : 'none'}">
        <div class="modal-dialog">
          <div class="modal-content">
            <div class="modal-header"><h5>${this.title}</h5></div>
            <div class="modal-body">${this.content}</div>
          </div>
        </div>
      </div>
    `;
  }
}

// Abstract Factory
interface UIComponentFactory {
  createButton(label: string): Button;
  createInput(placeholder: string, type?: 'text' | 'email' | 'password'): InputField;
  createModal(title: string, content: string): Modal;
  readonly themeName: string;
}

// Concrete Factories
class MaterialUIFactory implements UIComponentFactory {
  readonly themeName = 'Material UI';

  createButton(label: string): Button {
    return new MaterialButton(label);
  }

  createInput(placeholder: string, type: 'text' | 'email' | 'password' = 'text'): InputField {
    return new MaterialInput(placeholder, type);
  }

  createModal(title: string, content: string): Modal {
    return new MaterialModal(title, content);
  }
}

class BootstrapFactory implements UIComponentFactory {
  readonly themeName = 'Bootstrap';

  createButton(label: string): Button {
    return new BootstrapButton(label);
  }

  createInput(placeholder: string, type: 'text' | 'email' | 'password' = 'text'): InputField {
    return new BootstrapInput(placeholder, type);
  }

  createModal(title: string, content: string): Modal {
    return new BootstrapModal(title, content);
  }
}

// Client code ที่ไม่รู้ว่าใช้ theme ไหน
class LoginForm {
  private emailInput: InputField;
  private passwordInput: InputField;
  private submitButton: Button;
  private errorModal: Modal;

  constructor(private readonly factory: UIComponentFactory) {
    this.emailInput = factory.createInput('อีเมล', 'email');
    this.passwordInput = factory.createInput('รหัสผ่าน', 'password');
    this.submitButton = factory.createButton('เข้าสู่ระบบ');
    this.errorModal = factory.createModal('ข้อผิดพลาด', 'อีเมลหรือรหัสผ่านไม่ถูกต้อง');

    this.submitButton.onClick = () => this.handleSubmit();
  }

  private async handleSubmit(): Promise<void> {
    const emailValidation = this.emailInput.validate();
    const passwordValidation = this.passwordInput.validate();

    if (!emailValidation.isValid || !passwordValidation.isValid) {
      const allErrors = [...emailValidation.errors, ...passwordValidation.errors];
      this.errorModal.content = allErrors.join(', ');
      this.errorModal.show();
      return;
    }

    // API call...
    console.log('กำลังเข้าสู่ระบบ...');
  }

  render(): string {
    return `
      <form class="login-form" data-theme="${this.factory.themeName}">
        <h2>เข้าสู่ระบบ</h2>
        ${this.emailInput.render()}
        ${this.passwordInput.render()}
        ${this.submitButton.render()}
        ${this.errorModal.render()}
      </form>
    `;
  }
}

// สลับ theme ได้ง่าย
const materialForm = new LoginForm(new MaterialUIFactory());
const bootstrapForm = new LoginForm(new BootstrapFactory());

console.log(materialForm.render());
console.log(bootstrapForm.render());
```

---

## 4. Builder Pattern

### Intent
แยกการสร้าง object ที่ซับซ้อนออกจาก representation ของมัน เพื่อให้กระบวนการสร้างเดียวกันสร้าง representations ที่แตกต่างกันได้

### เมื่อใดควรใช้
- เมื่อ object มี parameters มากมายใน constructor
- เมื่อต้องการสร้าง object แบบ step-by-step
- เมื่อต้องการ object ในหลาย configurations

### 4.1 Builder สำหรับ Query Builder

```typescript
type ComparisonOperator = '=' | '!=' | '>' | '<' | '>=' | '<=' | 'LIKE' | 'IN' | 'NOT IN' | 'BETWEEN' | 'IS NULL' | 'IS NOT NULL';
type JoinType = 'INNER' | 'LEFT' | 'RIGHT' | 'FULL';
type OrderDirection = 'ASC' | 'DESC';

interface WhereCondition {
  column: string;
  operator: ComparisonOperator;
  value?: unknown;
  connector: 'AND' | 'OR';
}

interface JoinClause {
  type: JoinType;
  table: string;
  on: string;
}

interface OrderClause {
  column: string;
  direction: OrderDirection;
}

interface QueryConfig {
  table: string;
  columns: string[];
  conditions: WhereCondition[];
  joins: JoinClause[];
  orderBy: OrderClause[];
  groupBy: string[];
  having: string | null;
  limitValue: number | null;
  offsetValue: number | null;
  isDistinct: boolean;
}

class QueryBuilder {
  private config: QueryConfig = {
    table: '',
    columns: ['*'],
    conditions: [],
    joins: [],
    orderBy: [],
    groupBy: [],
    having: null,
    limitValue: null,
    offsetValue: null,
    isDistinct: false,
  };
  private params: unknown[] = [];

  from(table: string): this {
    this.config.table = table;
    return this;
  }

  select(...columns: string[]): this {
    this.config.columns = columns;
    return this;
  }

  distinct(): this {
    this.config.isDistinct = true;
    return this;
  }

  where(column: string, operator: ComparisonOperator, value?: unknown): this {
    this.config.conditions.push({ column, operator, value, connector: 'AND' });
    if (value !== undefined) this.params.push(value);
    return this;
  }

  orWhere(column: string, operator: ComparisonOperator, value?: unknown): this {
    this.config.conditions.push({ column, operator, value, connector: 'OR' });
    if (value !== undefined) this.params.push(value);
    return this;
  }

  whereIn(column: string, values: unknown[]): this {
    this.config.conditions.push({ column, operator: 'IN', value: values, connector: 'AND' });
    return this;
  }

  whereNull(column: string): this {
    this.config.conditions.push({ column, operator: 'IS NULL', connector: 'AND' });
    return this;
  }

  whereNotNull(column: string): this {
    this.config.conditions.push({ column, operator: 'IS NOT NULL', connector: 'AND' });
    return this;
  }

  join(table: string, on: string, type: JoinType = 'INNER'): this {
    this.config.joins.push({ type, table, on });
    return this;
  }

  leftJoin(table: string, on: string): this {
    return this.join(table, on, 'LEFT');
  }

  rightJoin(table: string, on: string): this {
    return this.join(table, on, 'RIGHT');
  }

  orderBy(column: string, direction: OrderDirection = 'ASC'): this {
    this.config.orderBy.push({ column, direction });
    return this;
  }

  groupBy(...columns: string[]): this {
    this.config.groupBy.push(...columns);
    return this;
  }

  having(condition: string): this {
    this.config.having = condition;
    return this;
  }

  limit(count: number): this {
    this.config.limitValue = count;
    return this;
  }

  offset(count: number): this {
    this.config.offsetValue = count;
    return this;
  }

  paginate(page: number, perPage: number = 10): this {
    return this.limit(perPage).offset((page - 1) * perPage);
  }

  build(): { sql: string; params: unknown[] } {
    if (!this.config.table) throw new Error('ต้องระบุ table ด้วย .from()');

    let sql = 'SELECT ';
    if (this.config.isDistinct) sql += 'DISTINCT ';
    sql += this.config.columns.join(', ');
    sql += ` FROM ${this.config.table}`;

    if (this.config.joins.length > 0) {
      sql += ' ' + this.config.joins.map(j => `${j.type} JOIN ${j.table} ON ${j.on}`).join(' ');
    }

    if (this.config.conditions.length > 0) {
      const conditions = this.config.conditions.map((c, i) => {
        const prefix = i === 0 ? 'WHERE' : c.connector;
        if (c.operator === 'IS NULL' || c.operator === 'IS NOT NULL') {
          return `${prefix} ${c.column} ${c.operator}`;
        }
        if (c.operator === 'IN' || c.operator === 'NOT IN') {
          const placeholders = (c.value as unknown[]).map(() => '?').join(', ');
          return `${prefix} ${c.column} ${c.operator} (${placeholders})`;
        }
        return `${prefix} ${c.column} ${c.operator} ?`;
      });
      sql += ' ' + conditions.join(' ');
    }

    if (this.config.groupBy.length > 0) {
      sql += ` GROUP BY ${this.config.groupBy.join(', ')}`;
    }

    if (this.config.having) {
      sql += ` HAVING ${this.config.having}`;
    }

    if (this.config.orderBy.length > 0) {
      sql += ' ORDER BY ' + this.config.orderBy.map(o => `${o.column} ${o.direction}`).join(', ');
    }

    if (this.config.limitValue !== null) sql += ` LIMIT ${this.config.limitValue}`;
    if (this.config.offsetValue !== null) sql += ` OFFSET ${this.config.offsetValue}`;

    return { sql, params: this.params };
  }

  toSQL(): string {
    const { sql, params } = this.build();
    return params.reduce((s, p) => s.replace('?', typeof p === 'string' ? `'${p}'` : String(p)), sql) as string;
  }
}

// การใช้งาน
const query = new QueryBuilder()
  .from('users')
  .select('users.id', 'users.name', 'users.email', 'orders.total')
  .leftJoin('orders', 'users.id = orders.user_id')
  .where('users.active', '=', true)
  .where('users.created_at', '>=', '2024-01-01')
  .whereNull('users.deleted_at')
  .groupBy('users.id', 'users.name', 'users.email')
  .having('COUNT(orders.id) > 0')
  .orderBy('users.name', 'ASC')
  .paginate(1, 20);

console.log(query.toSQL());
```

### 4.2 Builder สำหรับ HTTP Request

```typescript
interface RequestConfig {
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  headers: Record<string, string>;
  params: Record<string, string>;
  body?: unknown;
  timeout: number;
  retries: number;
  retryDelay: number;
  withCredentials: boolean;
  responseType: 'json' | 'text' | 'blob';
}

class HttpRequestBuilder {
  private config: RequestConfig = {
    url: '',
    method: 'GET',
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
    params: {},
    timeout: 30000,
    retries: 0,
    retryDelay: 1000,
    withCredentials: false,
    responseType: 'json',
  };

  url(url: string): this {
    this.config.url = url;
    return this;
  }

  method(method: RequestConfig['method']): this {
    this.config.method = method;
    return this;
  }

  get(): this { return this.method('GET'); }
  post(): this { return this.method('POST'); }
  put(): this { return this.method('PUT'); }
  patch(): this { return this.method('PATCH'); }
  delete(): this { return this.method('DELETE'); }

  header(key: string, value: string): this {
    this.config.headers[key] = value;
    return this;
  }

  headers(headers: Record<string, string>): this {
    this.config.headers = { ...this.config.headers, ...headers };
    return this;
  }

  authorization(token: string, type: 'Bearer' | 'Basic' | 'Token' = 'Bearer'): this {
    return this.header('Authorization', `${type} ${token}`);
  }

  param(key: string, value: string | number | boolean): this {
    this.config.params[key] = String(value);
    return this;
  }

  params(params: Record<string, string | number | boolean>): this {
    Object.entries(params).forEach(([k, v]) => this.param(k, v));
    return this;
  }

  body(data: unknown): this {
    this.config.body = data;
    return this;
  }

  timeout(ms: number): this {
    this.config.timeout = ms;
    return this;
  }

  retry(count: number, delay: number = 1000): this {
    this.config.retries = count;
    this.config.retryDelay = delay;
    return this;
  }

  withCredentials(): this {
    this.config.withCredentials = true;
    return this;
  }

  responseType(type: RequestConfig['responseType']): this {
    this.config.responseType = type;
    return this;
  }

  build(): RequestConfig {
    if (!this.config.url) throw new Error('ต้องระบุ URL');
    return { ...this.config };
  }

  async execute<T>(): Promise<T> {
    const config = this.build();
    const url = new URL(config.url);
    Object.entries(config.params).forEach(([k, v]) => url.searchParams.set(k, v));

    let lastError: Error | null = null;

    for (let attempt = 0; attempt <= config.retries; attempt++) {
      if (attempt > 0) {
        await new Promise(resolve => setTimeout(resolve, config.retryDelay));
      }

      try {
        const response = await fetch(url.toString(), {
          method: config.method,
          headers: config.headers,
          body: config.body ? JSON.stringify(config.body) : undefined,
          credentials: config.withCredentials ? 'include' : 'same-origin',
          signal: AbortSignal.timeout(config.timeout),
        });

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        if (config.responseType === 'json') return response.json() as Promise<T>;
        if (config.responseType === 'text') return response.text() as unknown as Promise<T>;
        return response.blob() as unknown as Promise<T>;
      } catch (error) {
        lastError = error as Error;
        if (attempt === config.retries) break;
      }
    }

    throw lastError ?? new Error('Request failed');
  }
}

// การใช้งาน
const request = new HttpRequestBuilder()
  .url('https://api.example.com/products')
  .get()
  .authorization('my-jwt-token')
  .param('page', 1)
  .param('limit', 20)
  .param('category', 'electronics')
  .timeout(5000)
  .retry(3, 1000);

const products = await request.execute<Product[]>();

// POST request
const createRequest = new HttpRequestBuilder()
  .url('https://api.example.com/products')
  .post()
  .authorization('my-jwt-token')
  .body({ name: 'สินค้าใหม่', price: 999 })
  .timeout(10000);

const newProduct = await createRequest.execute<Product>();
```

---

## 5. Prototype Pattern

### Intent
สร้าง objects ใหม่โดย clone objects ที่มีอยู่แล้ว แทนที่จะสร้างใหม่ตั้งแต่ต้น

### เมื่อใดควรใช้
- เมื่อการสร้าง object ใหม่มีต้นทุนสูง (เช่น มี initialization ซับซ้อน)
- เมื่อต้องการสร้าง objects ที่คล้ายกันโดยแก้ไขแค่บางส่วน
- เมื่อต้องการ cache objects ที่ใช้เวลานานในการสร้าง

### 5.1 Prototype Pattern สำหรับ Document

```typescript
interface Cloneable<T> {
  clone(): T;
  deepClone(): T;
}

interface DocumentElement {
  id: string;
  type: string;
  style: Partial<CSSStyleDeclaration>;
  children: DocumentElement[];
}

class DocumentTemplate implements Cloneable<DocumentTemplate> {
  private elements: DocumentElement[] = [];
  private metadata: {
    title: string;
    author: string;
    version: number;
    createdAt: Date;
    tags: string[];
  };

  constructor(title: string, author: string) {
    this.metadata = {
      title,
      author,
      version: 1,
      createdAt: new Date(),
      tags: [],
    };
  }

  addElement(element: DocumentElement): this {
    this.elements.push(element);
    return this;
  }

  setTitle(title: string): this {
    this.metadata.title = title;
    return this;
  }

  addTag(tag: string): this {
    this.metadata.tags.push(tag);
    return this;
  }

  // Shallow clone - แชร์ references กับ original
  clone(): DocumentTemplate {
    const cloned = new DocumentTemplate(this.metadata.title, this.metadata.author);
    cloned.elements = [...this.elements];
    cloned.metadata = { ...this.metadata, version: this.metadata.version + 1 };
    return cloned;
  }

  // Deep clone - คัดลอกทุกอย่างอย่างอิสระ
  deepClone(): DocumentTemplate {
    const cloned = new DocumentTemplate(this.metadata.title, this.metadata.author);
    cloned.elements = JSON.parse(JSON.stringify(this.elements));
    cloned.metadata = {
      ...this.metadata,
      createdAt: new Date(this.metadata.createdAt),
      tags: [...this.metadata.tags],
      version: this.metadata.version + 1,
    };
    return cloned;
  }

  getInfo(): typeof this.metadata & { elementCount: number } {
    return { ...this.metadata, elementCount: this.elements.length };
  }
}

// การใช้งาน
const baseTemplate = new DocumentTemplate('Template พื้นฐาน', 'ระบบ');
baseTemplate
  .addElement({ id: 'header', type: 'header', style: {}, children: [] })
  .addElement({ id: 'footer', type: 'footer', style: {}, children: [] })
  .addTag('template');

// Clone เพื่อสร้าง document ใหม่จาก template
const invoiceDoc = baseTemplate.deepClone();
invoiceDoc.setTitle('ใบแจ้งหนี้').addTag('invoice');

const reportDoc = baseTemplate.deepClone();
reportDoc.setTitle('รายงานประจำเดือน').addTag('report');

console.log(baseTemplate.getInfo()); // version: 1
console.log(invoiceDoc.getInfo());   // version: 2
console.log(reportDoc.getInfo());    // version: 2
```

### 5.2 Prototype Registry

```typescript
// Registry สำหรับเก็บ prototypes
class PrototypeRegistry<T extends Cloneable<T>> {
  private prototypes: Map<string, T> = new Map();

  register(name: string, prototype: T): void {
    this.prototypes.set(name, prototype);
  }

  unregister(name: string): void {
    this.prototypes.delete(name);
  }

  create(name: string): T {
    const prototype = this.prototypes.get(name);
    if (!prototype) {
      throw new Error(`ไม่พบ prototype: ${name}`);
    }
    return prototype.deepClone();
  }

  getNames(): string[] {
    return Array.from(this.prototypes.keys());
  }
}

// Game Character Prototype
class GameCharacter implements Cloneable<GameCharacter> {
  constructor(
    public name: string,
    public characterClass: 'warrior' | 'mage' | 'archer' | 'thief',
    public stats: {
      hp: number; maxHp: number;
      mp: number; maxMp: number;
      attack: number; defense: number;
      speed: number; luck: number;
    },
    public skills: string[],
    public equipment: { weapon?: string; armor?: string; accessory?: string }
  ) {}

  clone(): GameCharacter {
    return new GameCharacter(
      this.name,
      this.characterClass,
      { ...this.stats },
      [...this.skills],
      { ...this.equipment }
    );
  }

  deepClone(): GameCharacter {
    return new GameCharacter(
      this.name,
      this.characterClass,
      JSON.parse(JSON.stringify(this.stats)),
      [...this.skills],
      JSON.parse(JSON.stringify(this.equipment))
    );
  }

  levelUp(increases: Partial<GameCharacter['stats']>): void {
    Object.assign(this.stats, increases);
  }

  toString(): string {
    return `${this.name} (${this.characterClass}) HP:${this.stats.hp}/${this.stats.maxHp}`;
  }
}

// สร้าง prototype characters
const characterRegistry = new PrototypeRegistry<GameCharacter>();

// Warrior prototype
characterRegistry.register('warrior', new GameCharacter(
  'นักรบ', 'warrior',
  { hp: 200, maxHp: 200, mp: 50, maxMp: 50, attack: 80, defense: 70, speed: 60, luck: 40 },
  ['ฟันหนัก', 'ปิดกั้น', 'โจมตีแบบหมุน'],
  { weapon: 'ดาบเหล็ก', armor: 'เกราะหนัง' }
));

// Mage prototype
characterRegistry.register('mage', new GameCharacter(
  'นักเวทย์', 'mage',
  { hp: 100, maxHp: 100, mp: 200, maxMp: 200, attack: 40, defense: 30, speed: 70, luck: 60 },
  ['ไฟ', 'น้ำแข็ง', 'ฟ้าผ่า', 'ฟื้นฟู MP'],
  { weapon: 'ไม้เท้าเวทย์', armor: 'เสื้อคลุมนักเวทย์' }
));

// สร้าง characters จาก prototypes
const player1 = characterRegistry.create('warrior');
player1.name = 'อโนเดช';
player1.levelUp({ attack: 10, defense: 5, hp: 50, maxHp: 50 });

const player2 = characterRegistry.create('mage');
player2.name = 'มณีรัตน์';
player2.levelUp({ mp: 50, maxMp: 50, attack: 15 });

console.log(player1.toString()); // อโนเดช (warrior) HP:250/250
console.log(player2.toString()); // มณีรัตน์ (mage) HP:100/100
```

---

## 6. เปรียบเทียบ Creational Patterns

```typescript
/*
 * เปรียบเทียบ Creational Patterns:
 *
 * Singleton:
 * - จุดประสงค์: ควบคุมจำนวน instances
 * - ใช้เมื่อ: ต้องการ shared state หรือ resource ทั่วทั้ง app
 * - ตัวอย่าง: Logger, Config, DB Connection Pool
 *
 * Factory Method:
 * - จุดประสงค์: ซ่อน logic การสร้าง objects
 * - ใช้เมื่อ: ไม่รู้ล่วงหน้าว่าต้องสร้าง type ไหน
 * - ตัวอย่าง: Notification factories, Document parsers
 *
 * Abstract Factory:
 * - จุดประสงค์: สร้าง families of related objects
 * - ใช้เมื่อ: ต้องการ objects หลายตัวที่ compatible กัน
 * - ตัวอย่าง: UI Component libraries, Cross-platform widgets
 *
 * Builder:
 * - จุดประสงค์: สร้าง complex objects step-by-step
 * - ใช้เมื่อ: Object มี parameters มาก, หรือต้องการ fluent API
 * - ตัวอย่าง: Query builders, Request builders, Document builders
 *
 * Prototype:
 * - จุดประสงค์: Clone objects ที่มีอยู่
 * - ใช้เมื่อ: การสร้าง object มีต้นทุนสูง หรือต้องการ copy ที่คล้ายกัน
 * - ตัวอย่าง: Game characters, Document templates, Config presets
 */

// ตัวอย่างการใช้ทุก patterns ร่วมกัน
class ApplicationBootstrap {
  // Singleton - config และ logger
  private readonly config = ConfigurationManager.getInstance();
  private readonly logger = Logger.instance;

  // Abstract Factory - สร้าง UI components
  private readonly uiFactory: UIComponentFactory;

  // Builder - สร้าง HTTP requests
  private readonly httpBuilder = new HttpRequestBuilder();

  // Prototype Registry - สร้าง document templates
  private readonly docRegistry = new PrototypeRegistry<DocumentTemplate>();

  constructor(theme: 'material' | 'bootstrap' = 'material') {
    // Factory Method - เลือก UI factory ตาม theme
    this.uiFactory = theme === 'material'
      ? new MaterialUIFactory()
      : new BootstrapFactory();

    this.setupDocumentTemplates();
  }

  private setupDocumentTemplates(): void {
    const invoiceTemplate = new DocumentTemplate('ใบแจ้งหนี้', 'ระบบ');
    invoiceTemplate.addTag('invoice').addTag('financial');

    const reportTemplate = new DocumentTemplate('รายงาน', 'ระบบ');
    reportTemplate.addTag('report');

    this.docRegistry.register('invoice', invoiceTemplate);
    this.docRegistry.register('report', reportTemplate);
  }

  createLoginForm(): LoginForm {
    return new LoginForm(this.uiFactory);
  }

  createDocument(type: string, title: string): DocumentTemplate {
    const doc = this.docRegistry.create(type);
    doc.setTitle(title);
    return doc;
  }

  async callApi<T>(endpoint: string): Promise<T> {
    const token = this.config.get<string>('auth.token') ?? '';
    return this.httpBuilder
      .url(`${this.config.get<string>('api.baseUrl')}${endpoint}`)
      .get()
      .authorization(token)
      .timeout(this.config.getOrDefault('api.timeout', 30000))
      .execute<T>();
  }
}
```

---

## 7. สรุป

Creational Design Patterns แต่ละตัวมีจุดประสงค์ที่แตกต่างกัน:

| Pattern | เมื่อใช้ | ข้อดี |
|---------|---------|-------|
| **Singleton** | ต้องการ instance เดียว | Control shared resources |
| **Factory Method** | สร้าง objects แบบ flexible | Extensible, OCP-compliant |
| **Abstract Factory** | สร้าง related object families | Ensures compatibility |
| **Builder** | Object ซับซ้อน, parameters มาก | Readable, step-by-step |
| **Prototype** | Clone objects ที่สร้างยาก | Performance, flexibility |

TypeScript ช่วย Creational Patterns ด้วย:
1. **Interfaces** - กำหนด contracts ที่ชัดเจน
2. **Generics** - ทำให้ patterns reusable
3. **Private constructors** - บังคับ Singleton
4. **Method chaining types** - Builder ที่ type-safe
5. **Discriminated unions** - Factory ที่ปลอดภัย

---

*จบบทที่ 31 - Creational Design Patterns กับ TypeScript*
