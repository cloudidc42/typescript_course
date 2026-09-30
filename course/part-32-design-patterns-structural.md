# Part 32: Structural Design Patterns ใน TypeScript

## บทนำ

Structural Design Patterns คือรูปแบบการออกแบบที่มุ่งเน้นการจัดการความสัมพันธ์ระหว่าง class และ object เพื่อสร้างโครงสร้างที่ใหญ่ขึ้นและยืดหยุ่นมากขึ้น รูปแบบเหล่านี้ช่วยให้นักพัฒนาสามารถสร้างระบบที่ซับซ้อนได้จากส่วนประกอบขนาดเล็กๆ

ใน TypeScript เราสามารถนำ Structural Patterns ไปใช้ได้อย่างมีประสิทธิภาพเนื่องจาก TypeScript มี features ที่รองรับการเขียน OOP อย่างครบครัน ได้แก่ interfaces, generics, decorators และ access modifiers

## รูปแบบที่จะเรียนรู้ในบทนี้

1. **Adapter Pattern** - แปลง interface ของ class หนึ่งให้เข้ากับอีก interface
2. **Bridge Pattern** - แยก abstraction จาก implementation
3. **Composite Pattern** - จัดการ object ในโครงสร้างแบบต้นไม้
4. **Decorator Pattern** - เพิ่ม behavior ให้กับ object แบบ dynamic
5. **Facade Pattern** - สร้าง interface ที่ง่ายกว่าสำหรับ subsystem ที่ซับซ้อน
6. **Flyweight Pattern** - แชร์ state ของ object เพื่อลดการใช้หน่วยความจำ
7. **Proxy Pattern** - สร้าง placeholder แทน object จริง

---

## 1. Adapter Pattern (รูปแบบตัวแปลง)

### แนวคิด

Adapter Pattern ทำหน้าที่เป็นตัวแปลง (translator) ระหว่าง interface สองชุดที่ไม่เข้ากัน เปรียบเสมือนปลั๊กแปลงไฟที่ช่วยให้เราเสียบปลั๊กที่มีขนาดต่างกันเข้าด้วยกันได้

### เมื่อไหรควรใช้ Adapter Pattern

- เมื่อต้องการใช้ class ที่มีอยู่แล้วแต่ interface ไม่ตรงกับที่ต้องการ
- เมื่อต้องการสร้าง class ที่ reusable ซึ่งทำงานร่วมกับ class อื่นที่ interface ไม่เข้ากัน
- เมื่อต้องการ integrate library ของ third-party ที่มี interface แตกต่างจากที่ใช้อยู่

### โครงสร้าง Adapter Pattern

```typescript
// Target Interface - interface ที่ client ต้องการใช้
interface Target {
  request(): string;
}

// Adaptee - class ที่มีอยู่แล้วแต่ interface ไม่ตรงกัน
class Adaptee {
  specificRequest(): string {
    return "ผลลัพธ์จาก Adaptee";
  }
}

// Adapter - ทำหน้าที่แปลง interface
class Adapter implements Target {
  private adaptee: Adaptee;

  constructor(adaptee: Adaptee) {
    this.adaptee = adaptee;
  }

  request(): string {
    const result = this.adaptee.specificRequest();
    return `Adapter แปลงค่า: ${result}`;
  }
}

// Client code
function clientCode(target: Target) {
  console.log(target.request());
}

const adaptee = new Adaptee();
const adapter = new Adapter(adaptee);
clientCode(adapter);
// Output: Adapter แปลงค่า: ผลลัพธ์จาก Adaptee
```

### ตัวอย่างจริง: การแปลงข้อมูลจาก API ต่างๆ

```typescript
// ตัวอย่าง: ระบบการชำระเงินที่รองรับหลาย payment gateway

// Target interface ที่ระบบเราใช้
interface PaymentProcessor {
  processPayment(amount: number, currency: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<RefundResult>;
}

interface PaymentResult {
  success: boolean;
  transactionId: string;
  message: string;
}

interface RefundResult {
  success: boolean;
  refundId: string;
  message: string;
}

// Stripe API (ที่มี interface ต่างออกไป)
class StripePaymentGateway {
  async charge(
    amountInCents: number,
    currency: string,
    customerId: string
  ): Promise<{ id: string; status: string; error?: string }> {
    // จำลองการเรียก Stripe API
    console.log(`[Stripe] Charging ${amountInCents} cents in ${currency}`);
    return {
      id: `stripe_${Date.now()}`,
      status: "succeeded",
    };
  }

  async createRefund(
    chargeId: string,
    amountInCents: number
  ): Promise<{ id: string; status: string }> {
    console.log(`[Stripe] Refunding ${amountInCents} cents for charge ${chargeId}`);
    return {
      id: `re_${Date.now()}`,
      status: "succeeded",
    };
  }
}

// PayPal API (ที่มี interface อีกแบบหนึ่ง)
class PayPalPaymentGateway {
  async executePayment(
    value: string,
    currencyCode: string,
    payerId: string
  ): Promise<{ paymentId: string; state: string }> {
    console.log(`[PayPal] Executing payment of ${value} ${currencyCode}`);
    return {
      paymentId: `PAY-${Date.now()}`,
      state: "approved",
    };
  }

  async issueRefund(
    saleId: string,
    value: string,
    currencyCode: string
  ): Promise<{ id: string; state: string }> {
    console.log(`[PayPal] Issuing refund for sale ${saleId}`);
    return {
      id: `REF-${Date.now()}`,
      state: "completed",
    };
  }
}

// Stripe Adapter
class StripeAdapter implements PaymentProcessor {
  private stripeGateway: StripePaymentGateway;
  private customerId: string;

  constructor(stripeGateway: StripePaymentGateway, customerId: string) {
    this.stripeGateway = stripeGateway;
    this.customerId = customerId;
  }

  async processPayment(amount: number, currency: string): Promise<PaymentResult> {
    // แปลง amount จาก บาท เป็น สตางค์ (cents)
    const amountInCents = Math.round(amount * 100);
    
    const result = await this.stripeGateway.charge(
      amountInCents,
      currency,
      this.customerId
    );

    return {
      success: result.status === "succeeded",
      transactionId: result.id,
      message: result.error || "การชำระเงินสำเร็จ",
    };
  }

  async refund(transactionId: string, amount: number): Promise<RefundResult> {
    const amountInCents = Math.round(amount * 100);
    
    const result = await this.stripeGateway.createRefund(
      transactionId,
      amountInCents
    );

    return {
      success: result.status === "succeeded",
      refundId: result.id,
      message: "การคืนเงินสำเร็จ",
    };
  }
}

// PayPal Adapter
class PayPalAdapter implements PaymentProcessor {
  private paypalGateway: PayPalPaymentGateway;
  private payerId: string;

  constructor(paypalGateway: PayPalPaymentGateway, payerId: string) {
    this.paypalGateway = paypalGateway;
    this.payerId = payerId;
  }

  async processPayment(amount: number, currency: string): Promise<PaymentResult> {
    const result = await this.paypalGateway.executePayment(
      amount.toString(),
      currency.toUpperCase(),
      this.payerId
    );

    return {
      success: result.state === "approved",
      transactionId: result.paymentId,
      message: "การชำระเงินผ่าน PayPal สำเร็จ",
    };
  }

  async refund(transactionId: string, amount: number): Promise<RefundResult> {
    const result = await this.paypalGateway.issueRefund(
      transactionId,
      amount.toString(),
      "THB"
    );

    return {
      success: result.state === "completed",
      refundId: result.id,
      message: "การคืนเงินผ่าน PayPal สำเร็จ",
    };
  }
}

// OrderService ที่ใช้ PaymentProcessor interface
class OrderService {
  constructor(private paymentProcessor: PaymentProcessor) {}

  async checkout(amount: number, currency: string = "THB"): Promise<void> {
    console.log(`กำลังชำระเงิน ${amount} ${currency}...`);
    
    const result = await this.paymentProcessor.processPayment(amount, currency);
    
    if (result.success) {
      console.log(`✅ ชำระเงินสำเร็จ! Transaction ID: ${result.transactionId}`);
    } else {
      console.log(`❌ ชำระเงินล้มเหลว: ${result.message}`);
    }
  }

  async cancelOrder(transactionId: string, amount: number): Promise<void> {
    console.log(`กำลังคืนเงิน ${amount} บาท...`);
    
    const result = await this.paymentProcessor.refund(transactionId, amount);
    
    if (result.success) {
      console.log(`✅ คืนเงินสำเร็จ! Refund ID: ${result.refundId}`);
    }
  }
}

// การใช้งาน
async function main() {
  // ใช้ Stripe
  const stripeGateway = new StripePaymentGateway();
  const stripeAdapter = new StripeAdapter(stripeGateway, "cus_123");
  const orderServiceWithStripe = new OrderService(stripeAdapter);
  
  await orderServiceWithStripe.checkout(1500);
  
  // ใช้ PayPal
  const paypalGateway = new PayPalPaymentGateway();
  const paypalAdapter = new PayPalAdapter(paypalGateway, "payer_456");
  const orderServiceWithPayPal = new OrderService(paypalAdapter);
  
  await orderServiceWithPayPal.checkout(2000);
}

main();
```

### ตัวอย่างที่ 2: Database Adapter

```typescript
// ตัวอย่าง: รองรับฐานข้อมูลหลายชนิด

interface DatabaseAdapter {
  connect(): Promise<void>;
  query<T>(sql: string, params?: any[]): Promise<T[]>;
  disconnect(): Promise<void>;
}

// MySQL class (มีอยู่แล้ว)
class MySQLClient {
  private connection: any;

  async open(host: string, user: string, password: string, database: string): Promise<void> {
    console.log(`[MySQL] Connecting to ${host}/${database}`);
    this.connection = { connected: true };
  }

  async execute(query: string, values: any[]): Promise<{ rows: any[]; fields: any[] }> {
    console.log(`[MySQL] Executing: ${query}`);
    return { rows: [{ id: 1, name: "Test User" }], fields: [] };
  }

  async close(): Promise<void> {
    console.log("[MySQL] Connection closed");
    this.connection = null;
  }
}

// PostgreSQL class (มีอยู่แล้ว)
class PostgreSQLClient {
  private pool: any;

  async createPool(connectionString: string): Promise<void> {
    console.log(`[PostgreSQL] Creating pool with: ${connectionString}`);
    this.pool = { active: true };
  }

  async runQuery(text: string, values?: any[]): Promise<{ rows: any[] }> {
    console.log(`[PostgreSQL] Running: ${text}`);
    return { rows: [{ id: 1, name: "Test User" }] };
  }

  async endPool(): Promise<void> {
    console.log("[PostgreSQL] Pool ended");
    this.pool = null;
  }
}

// MySQL Adapter
class MySQLAdapter implements DatabaseAdapter {
  private client: MySQLClient;
  private config: { host: string; user: string; password: string; database: string };

  constructor(
    client: MySQLClient,
    config: { host: string; user: string; password: string; database: string }
  ) {
    this.client = client;
    this.config = config;
  }

  async connect(): Promise<void> {
    await this.client.open(
      this.config.host,
      this.config.user,
      this.config.password,
      this.config.database
    );
  }

  async query<T>(sql: string, params: any[] = []): Promise<T[]> {
    const result = await this.client.execute(sql, params);
    return result.rows as T[];
  }

  async disconnect(): Promise<void> {
    await this.client.close();
  }
}

// PostgreSQL Adapter
class PostgreSQLAdapter implements DatabaseAdapter {
  private client: PostgreSQLClient;
  private connectionString: string;

  constructor(client: PostgreSQLClient, connectionString: string) {
    this.client = client;
    this.connectionString = connectionString;
  }

  async connect(): Promise<void> {
    await this.client.createPool(this.connectionString);
  }

  async query<T>(sql: string, params: any[] = []): Promise<T[]> {
    const result = await this.client.runQuery(sql, params);
    return result.rows as T[];
  }

  async disconnect(): Promise<void> {
    await this.client.endPool();
  }
}

// UserRepository ที่ใช้ DatabaseAdapter
class UserRepository {
  constructor(private db: DatabaseAdapter) {}

  async findAll(): Promise<any[]> {
    await this.db.connect();
    const users = await this.db.query("SELECT * FROM users");
    await this.db.disconnect();
    return users;
  }

  async findById(id: number): Promise<any> {
    await this.db.connect();
    const users = await this.db.query("SELECT * FROM users WHERE id = ?", [id]);
    await this.db.disconnect();
    return users[0];
  }
}

// การใช้งาน
async function demoDatabase() {
  // ใช้ MySQL
  const mysqlClient = new MySQLClient();
  const mysqlAdapter = new MySQLAdapter(mysqlClient, {
    host: "localhost",
    user: "root",
    password: "password",
    database: "mydb",
  });
  
  const mysqlUserRepo = new UserRepository(mysqlAdapter);
  const mysqlUsers = await mysqlUserRepo.findAll();
  console.log("MySQL users:", mysqlUsers);

  // ใช้ PostgreSQL
  const pgClient = new PostgreSQLClient();
  const pgAdapter = new PostgreSQLAdapter(
    pgClient,
    "postgresql://user:pass@localhost/mydb"
  );
  
  const pgUserRepo = new UserRepository(pgAdapter);
  const pgUsers = await pgUserRepo.findAll();
  console.log("PostgreSQL users:", pgUsers);
}
```

### ตัวอย่างที่ 3: Logger Adapter

```typescript
// ตัวอย่าง: รองรับระบบ logging หลายชนิด

interface Logger {
  info(message: string, meta?: object): void;
  error(message: string, error?: Error): void;
  warn(message: string, meta?: object): void;
  debug(message: string, meta?: object): void;
}

// Winston Logger (third-party)
class WinstonLogger {
  log(level: string, message: string, metadata?: any): void {
    const timestamp = new Date().toISOString();
    console.log(`[Winston] ${timestamp} [${level.toUpperCase()}] ${message}`, metadata || "");
  }
}

// Pino Logger (third-party)
class PinoLogger {
  private level: string;
  
  constructor(level: string = "info") {
    this.level = level;
  }

  write(logLevel: string, msg: string, obj?: object): void {
    const time = Date.now();
    console.log(JSON.stringify({ level: logLevel, time, msg, ...obj }));
  }
}

// Winston Adapter
class WinstonAdapter implements Logger {
  constructor(private winston: WinstonLogger) {}

  info(message: string, meta?: object): void {
    this.winston.log("info", message, meta);
  }

  error(message: string, error?: Error): void {
    this.winston.log("error", message, { 
      errorMessage: error?.message,
      stack: error?.stack 
    });
  }

  warn(message: string, meta?: object): void {
    this.winston.log("warn", message, meta);
  }

  debug(message: string, meta?: object): void {
    this.winston.log("debug", message, meta);
  }
}

// Pino Adapter
class PinoAdapter implements Logger {
  constructor(private pino: PinoLogger) {}

  info(message: string, meta?: object): void {
    this.pino.write("info", message, meta);
  }

  error(message: string, error?: Error): void {
    this.pino.write("error", message, {
      err: { message: error?.message, stack: error?.stack },
    });
  }

  warn(message: string, meta?: object): void {
    this.pino.write("warn", message, meta);
  }

  debug(message: string, meta?: object): void {
    this.pino.write("debug", message, meta);
  }
}

// Application service ที่ใช้ Logger interface
class UserAuthService {
  constructor(private logger: Logger) {}

  async login(username: string, password: string): Promise<boolean> {
    this.logger.info("ผู้ใช้พยายาม login", { username });
    
    try {
      // จำลองการตรวจสอบ
      if (password === "correct") {
        this.logger.info("Login สำเร็จ", { username });
        return true;
      } else {
        this.logger.warn("Login ล้มเหลว - รหัสผ่านไม่ถูกต้อง", { username });
        return false;
      }
    } catch (err) {
      this.logger.error("เกิดข้อผิดพลาดขณะ login", err as Error);
      return false;
    }
  }
}

// การใช้งาน
const winstonLogger = new WinstonLogger();
const winstonAdapter = new WinstonAdapter(winstonLogger);
const authServiceWithWinston = new UserAuthService(winstonAdapter);
authServiceWithWinston.login("john", "correct");

const pinoLogger = new PinoLogger("debug");
const pinoAdapter = new PinoAdapter(pinoLogger);
const authServiceWithPino = new UserAuthService(pinoAdapter);
authServiceWithPino.login("jane", "wrong");
```

---

## 2. Bridge Pattern (รูปแบบสะพาน)

### แนวคิด

Bridge Pattern แยก abstraction ออกจาก implementation เพื่อให้ทั้งสองสามารถเปลี่ยนแปลงได้อิสระจากกัน รูปแบบนี้ช่วยหลีกเลี่ยง "explosion" ของ class เมื่อมีหลาย dimension ที่ต้องการ vary

### เมื่อไหรควรใช้ Bridge Pattern

- เมื่อต้องการหลีกเลี่ยงการผูกติดถาวรระหว่าง abstraction และ implementation
- เมื่อทั้ง abstraction และ implementation ต้องสามารถขยายได้ผ่าน subclassing
- เมื่อการเปลี่ยน implementation ไม่ควรส่งผลกระทบต่อ client code

### โครงสร้าง Bridge Pattern

```typescript
// Implementation Interface
interface Renderer {
  renderCircle(radius: number): void;
  renderSquare(side: number): void;
}

// Concrete Implementations
class SVGRenderer implements Renderer {
  renderCircle(radius: number): void {
    console.log(`<circle r="${radius}" />`);
  }

  renderSquare(side: number): void {
    console.log(`<rect width="${side}" height="${side}" />`);
  }
}

class CanvasRenderer implements Renderer {
  renderCircle(radius: number): void {
    console.log(`ctx.arc(0, 0, ${radius}, 0, 2 * Math.PI)`);
  }

  renderSquare(side: number): void {
    console.log(`ctx.fillRect(0, 0, ${side}, ${side})`);
  }
}

// Abstraction
abstract class Shape {
  constructor(protected renderer: Renderer) {}
  
  abstract draw(): void;
  abstract resize(factor: number): void;
}

// Refined Abstractions
class Circle extends Shape {
  private radius: number;

  constructor(renderer: Renderer, radius: number) {
    super(renderer);
    this.radius = radius;
  }

  draw(): void {
    this.renderer.renderCircle(this.radius);
  }

  resize(factor: number): void {
    this.radius *= factor;
  }
}

class Square extends Shape {
  private side: number;

  constructor(renderer: Renderer, side: number) {
    super(renderer);
    this.side = side;
  }

  draw(): void {
    this.renderer.renderSquare(this.side);
  }

  resize(factor: number): void {
    this.side *= factor;
  }
}

// การใช้งาน
const svgRenderer = new SVGRenderer();
const canvasRenderer = new CanvasRenderer();

const svgCircle = new Circle(svgRenderer, 50);
svgCircle.draw(); // <circle r="50" />

const canvasSquare = new Square(canvasRenderer, 100);
canvasSquare.draw(); // ctx.fillRect(0, 0, 100, 100)
```

### ตัวอย่างจริง: ระบบส่งข้อความ

```typescript
// ตัวอย่าง: ระบบ notification ที่รองรับหลาย channel และหลาย priority

// Implementation Interface
interface NotificationChannel {
  send(recipient: string, message: string): Promise<void>;
}

// Concrete Implementations
class EmailChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`📧 ส่ง Email ถึง ${recipient}: ${message}`);
    // จริงๆ จะเรียก Email API
  }
}

class SMSChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`📱 ส่ง SMS ถึง ${recipient}: ${message}`);
    // จริงๆ จะเรียก SMS API
  }
}

class PushNotificationChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`🔔 ส่ง Push Notification ถึง ${recipient}: ${message}`);
    // จริงๆ จะเรียก FCM/APNs
  }
}

class LineChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`💬 ส่ง LINE Message ถึง ${recipient}: ${message}`);
    // จริงๆ จะเรียก LINE Messaging API
  }
}

// Abstraction
abstract class Notification {
  constructor(
    protected channel: NotificationChannel,
    protected recipient: string
  ) {}

  abstract notify(data: any): Promise<void>;
}

// Refined Abstractions
class OrderNotification extends Notification {
  constructor(channel: NotificationChannel, recipient: string) {
    super(channel, recipient);
  }

  async notify(order: {
    orderId: string;
    status: string;
    totalAmount: number;
  }): Promise<void> {
    const message = 
      `คำสั่งซื้อ #${order.orderId} ` +
      `สถานะ: ${order.status} ` +
      `ยอดรวม: ${order.totalAmount} บาท`;
    
    await this.channel.send(this.recipient, message);
  }
}

class SecurityAlertNotification extends Notification {
  constructor(channel: NotificationChannel, recipient: string) {
    super(channel, recipient);
  }

  async notify(alert: {
    alertType: string;
    ipAddress: string;
    timestamp: Date;
  }): Promise<void> {
    const message = 
      `⚠️ การแจ้งเตือนความปลอดภัย: ${alert.alertType} ` +
      `จาก IP: ${alert.ipAddress} ` +
      `เวลา: ${alert.timestamp.toLocaleString("th-TH")}`;
    
    await this.channel.send(this.recipient, message);
  }
}

class PromotionNotification extends Notification {
  constructor(channel: NotificationChannel, recipient: string) {
    super(channel, recipient);
  }

  async notify(promotion: {
    title: string;
    discount: number;
    expiryDate: Date;
  }): Promise<void> {
    const message = 
      `🎉 ${promotion.title} ` +
      `ลด ${promotion.discount}% ` +
      `หมดอายุ: ${promotion.expiryDate.toLocaleDateString("th-TH")}`;
    
    await this.channel.send(this.recipient, message);
  }
}

// การใช้งาน
async function demoNotifications() {
  const emailChannel = new EmailChannel();
  const smsChannel = new SMSChannel();
  const pushChannel = new PushNotificationChannel();

  // Order notification ผ่าน Email
  const orderEmailNotif = new OrderNotification(emailChannel, "customer@example.com");
  await orderEmailNotif.notify({
    orderId: "ORD-001",
    status: "จัดส่งแล้ว",
    totalAmount: 1500,
  });

  // Order notification ผ่าน SMS
  const orderSMSNotif = new OrderNotification(smsChannel, "0812345678");
  await orderSMSNotif.notify({
    orderId: "ORD-001",
    status: "จัดส่งแล้ว",
    totalAmount: 1500,
  });

  // Security alert ผ่าน Push Notification
  const securityPushNotif = new SecurityAlertNotification(pushChannel, "user_123");
  await securityPushNotif.notify({
    alertType: "การเข้าสู่ระบบจากอุปกรณ์ใหม่",
    ipAddress: "192.168.1.100",
    timestamp: new Date(),
  });

  // Promotion ผ่าน LINE
  const lineChannel = new LineChannel();
  const promoLineNotif = new PromotionNotification(lineChannel, "U1234567890");
  await promoLineNotif.notify({
    title: "ลดราคาต้อนรับปีใหม่",
    discount: 30,
    expiryDate: new Date("2026-12-31"),
  });
}

demoNotifications();
```

### ตัวอย่างที่ 2: Device Driver Pattern

```typescript
// ตัวอย่าง: Bridge สำหรับ Device และ OS

// Implementation Interface
interface DeviceDriver {
  read(bytes: number): Buffer;
  write(data: Buffer): void;
  getStatus(): string;
}

// Concrete Implementations (OS-specific drivers)
class WindowsUSBDriver implements DeviceDriver {
  read(bytes: number): Buffer {
    console.log(`[Windows USB] อ่าน ${bytes} bytes`);
    return Buffer.alloc(bytes);
  }

  write(data: Buffer): void {
    console.log(`[Windows USB] เขียน ${data.length} bytes`);
  }

  getStatus(): string {
    return "Windows USB Ready";
  }
}

class LinuxUSBDriver implements DeviceDriver {
  read(bytes: number): Buffer {
    console.log(`[Linux USB] อ่าน ${bytes} bytes ผ่าน /dev/usb`);
    return Buffer.alloc(bytes);
  }

  write(data: Buffer): void {
    console.log(`[Linux USB] เขียน ${data.length} bytes ผ่าน /dev/usb`);
  }

  getStatus(): string {
    return "Linux USB Ready";
  }
}

// Abstraction
abstract class Device {
  constructor(protected driver: DeviceDriver) {}
  
  getDriverStatus(): string {
    return this.driver.getStatus();
  }

  abstract readData(size: number): Buffer;
  abstract writeData(data: Buffer): void;
}

// Refined Abstractions
class USBKeyboard extends Device {
  readData(size: number): Buffer {
    console.log("Keyboard: อ่านข้อมูล input");
    return this.driver.read(size);
  }

  writeData(data: Buffer): void {
    console.log("Keyboard: ส่งข้อมูล LED status");
    this.driver.write(data);
  }

  pressKey(key: string): void {
    console.log(`ผู้ใช้กดปุ่ม: ${key}`);
    const keyBuffer = Buffer.from(key);
    this.driver.write(keyBuffer);
  }
}

class USBStorage extends Device {
  readData(size: number): Buffer {
    console.log(`Storage: อ่านไฟล์ขนาด ${size} bytes`);
    return this.driver.read(size);
  }

  writeData(data: Buffer): void {
    console.log(`Storage: เขียนไฟล์ขนาด ${data.length} bytes`);
    this.driver.write(data);
  }

  saveFile(filename: string, content: string): void {
    console.log(`กำลังบันทึก ${filename}`);
    this.driver.write(Buffer.from(content));
  }
}

// การใช้งาน
const windowsDriver = new WindowsUSBDriver();
const linuxDriver = new LinuxUSBDriver();

const keyboardOnWindows = new USBKeyboard(windowsDriver);
keyboardOnWindows.pressKey("Enter");

const storageOnLinux = new USBStorage(linuxDriver);
storageOnLinux.saveFile("document.txt", "เนื้อหาของไฟล์");
```

---

## 3. Composite Pattern (รูปแบบผสม)

### แนวคิด

Composite Pattern ช่วยให้สามารถจัดการ object แบบ individual และ group ได้อย่างเหมือนกัน โดยสร้างโครงสร้างแบบต้นไม้ (tree structure) ซึ่งประกอบด้วย leaf node และ composite node

### เมื่อไหรควรใช้ Composite Pattern

- เมื่อต้องการแสดงลำดับชั้นของ "ส่วนทั้งหมด" ของ object
- เมื่อต้องการให้ client สามารถเพิกเฉย (ignore) ความแตกต่างระหว่าง compositions ของ object และ individual object

### โครงสร้าง Composite Pattern

```typescript
// Component Interface
interface FileSystemItem {
  getName(): string;
  getSize(): number;
  print(indent?: string): void;
}

// Leaf
class File implements FileSystemItem {
  constructor(
    private name: string,
    private size: number
  ) {}

  getName(): string {
    return this.name;
  }

  getSize(): number {
    return this.size;
  }

  print(indent: string = ""): void {
    console.log(`${indent}📄 ${this.name} (${this.size} bytes)`);
  }
}

// Composite
class Directory implements FileSystemItem {
  private children: FileSystemItem[] = [];

  constructor(private name: string) {}

  add(item: FileSystemItem): void {
    this.children.push(item);
  }

  remove(item: FileSystemItem): void {
    const index = this.children.indexOf(item);
    if (index !== -1) {
      this.children.splice(index, 1);
    }
  }

  getName(): string {
    return this.name;
  }

  getSize(): number {
    return this.children.reduce((total, item) => total + item.getSize(), 0);
  }

  print(indent: string = ""): void {
    console.log(`${indent}📁 ${this.name}/ (${this.getSize()} bytes)`);
    for (const child of this.children) {
      child.print(indent + "  ");
    }
  }
}

// การใช้งาน
const root = new Directory("root");
const src = new Directory("src");
const tests = new Directory("tests");

src.add(new File("index.ts", 1024));
src.add(new File("app.ts", 2048));

const components = new Directory("components");
components.add(new File("Button.tsx", 512));
components.add(new File("Modal.tsx", 768));
src.add(components);

tests.add(new File("app.test.ts", 1536));

root.add(src);
root.add(tests);
root.add(new File("package.json", 256));

root.print();
```

### ตัวอย่างจริง: ระบบ Menu

```typescript
// ตัวอย่าง: ระบบ menu แบบ multi-level

interface MenuItem {
  getTitle(): string;
  getUrl(): string | null;
  render(depth?: number): string;
  isActive(currentPath: string): boolean;
}

// Leaf: Menu Item ที่มี URL จริง
class PageMenuItem implements MenuItem {
  private active: boolean = false;

  constructor(
    private title: string,
    private url: string,
    private icon?: string
  ) {}

  getTitle(): string {
    return this.title;
  }

  getUrl(): string {
    return this.url;
  }

  isActive(currentPath: string): boolean {
    return currentPath === this.url || currentPath.startsWith(this.url + "/");
  }

  render(depth: number = 0): string {
    const indent = "  ".repeat(depth);
    const iconStr = this.icon ? `${this.icon} ` : "";
    const activeStr = this.isActive(this.url) ? " (active)" : "";
    return `${indent}<a href="${this.url}">${iconStr}${this.title}${activeStr}</a>`;
  }
}

// Composite: Menu Group ที่มี sub-items
class MenuGroup implements MenuItem {
  private children: MenuItem[] = [];

  constructor(
    private title: string,
    private icon?: string
  ) {}

  add(item: MenuItem): void {
    this.children.push(item);
  }

  getTitle(): string {
    return this.title;
  }

  getUrl(): null {
    return null;
  }

  isActive(currentPath: string): boolean {
    return this.children.some((child) => child.isActive(currentPath));
  }

  render(depth: number = 0): string {
    const indent = "  ".repeat(depth);
    const iconStr = this.icon ? `${this.icon} ` : "";
    const lines: string[] = [
      `${indent}<div class="menu-group">`,
      `${indent}  <span>${iconStr}${this.title}</span>`,
      `${indent}  <ul>`,
    ];

    for (const child of this.children) {
      lines.push(`${indent}    <li>${child.render(depth + 2)}</li>`);
    }

    lines.push(`${indent}  </ul>`);
    lines.push(`${indent}</div>`);

    return lines.join("\n");
  }
}

// Separator
class MenuSeparator implements MenuItem {
  getTitle(): string {
    return "---";
  }

  getUrl(): null {
    return null;
  }

  isActive(_currentPath: string): boolean {
    return false;
  }

  render(depth: number = 0): string {
    const indent = "  ".repeat(depth);
    return `${indent}<hr class="menu-separator" />`;
  }
}

// Navigation Builder
class NavigationBuilder {
  private items: MenuItem[] = [];

  addItem(item: MenuItem): this {
    this.items.push(item);
    return this;
  }

  render(): string {
    const lines = ["<nav>", "  <ul>"];

    for (const item of this.items) {
      lines.push(`    <li>${item.render(2)}</li>`);
    }

    lines.push("  </ul>", "</nav>");
    return lines.join("\n");
  }
}

// การสร้าง Navigation Menu
const nav = new NavigationBuilder();

nav
  .addItem(new PageMenuItem("หน้าแรก", "/", "🏠"))
  .addItem(
    (() => {
      const products = new MenuGroup("สินค้า", "🛍️");
      products.add(new PageMenuItem("สินค้าทั้งหมด", "/products"));
      products.add(new PageMenuItem("สินค้าใหม่", "/products/new"));
      products.add(new PageMenuItem("ลดราคา", "/products/sale"));
      
      const categories = new MenuGroup("หมวดหมู่");
      categories.add(new PageMenuItem("อิเล็กทรอนิกส์", "/products/electronics"));
      categories.add(new PageMenuItem("เสื้อผ้า", "/products/clothing"));
      products.add(categories);
      
      return products;
    })()
  )
  .addItem(new MenuSeparator())
  .addItem(new PageMenuItem("เกี่ยวกับเรา", "/about", "ℹ️"))
  .addItem(new PageMenuItem("ติดต่อ", "/contact", "📞"));

console.log(nav.render());
```

---

## 4. Decorator Pattern (รูปแบบตกแต่ง)

### แนวคิด

Decorator Pattern ช่วยเพิ่ม behavior ให้กับ object แต่ละตัวแบบ dynamic โดยไม่ส่งผลกระทบต่อ behavior ของ object อื่นจาก class เดียวกัน

### เมื่อไหรควรใช้ Decorator Pattern

- เมื่อต้องการเพิ่ม responsibilities ให้กับ object แบบ dynamic และ transparent
- เมื่อการขยายผ่าน subclassing ไม่เหมาะสม (เช่น อาจทำให้มี class จำนวนมากเกินไป)
- เมื่อต้องการสามารถถอด responsibilities ออกได้

### โครงสร้าง Decorator Pattern

```typescript
// Component Interface
interface Coffee {
  getDescription(): string;
  getCost(): number;
}

// Concrete Component
class SimpleCoffee implements Coffee {
  getDescription(): string {
    return "กาแฟดำ";
  }

  getCost(): number {
    return 40;
  }
}

// Base Decorator
abstract class CoffeeDecorator implements Coffee {
  constructor(protected coffee: Coffee) {}

  getDescription(): string {
    return this.coffee.getDescription();
  }

  getCost(): number {
    return this.coffee.getCost();
  }
}

// Concrete Decorators
class Milk extends CoffeeDecorator {
  getDescription(): string {
    return `${this.coffee.getDescription()}, นม`;
  }

  getCost(): number {
    return this.coffee.getCost() + 15;
  }
}

class Sugar extends CoffeeDecorator {
  getDescription(): string {
    return `${this.coffee.getDescription()}, น้ำตาล`;
  }

  getCost(): number {
    return this.coffee.getCost() + 5;
  }
}

class WhippedCream extends CoffeeDecorator {
  getDescription(): string {
    return `${this.coffee.getDescription()}, วิปครีม`;
  }

  getCost(): number {
    return this.coffee.getCost() + 25;
  }
}

class ExtraShot extends CoffeeDecorator {
  getDescription(): string {
    return `${this.coffee.getDescription()}, ช็อตพิเศษ`;
  }

  getCost(): number {
    return this.coffee.getCost() + 20;
  }
}

// การใช้งาน
let coffee: Coffee = new SimpleCoffee();
console.log(`${coffee.getDescription()} = ${coffee.getCost()} บาท`);

coffee = new Milk(coffee);
console.log(`${coffee.getDescription()} = ${coffee.getCost()} บาท`);

coffee = new Sugar(coffee);
console.log(`${coffee.getDescription()} = ${coffee.getCost()} บาท`);

coffee = new WhippedCream(new ExtraShot(coffee));
console.log(`${coffee.getDescription()} = ${coffee.getCost()} บาท`);
```

### ตัวอย่างจริง: HTTP Request Middleware

```typescript
// ตัวอย่าง: HTTP Request handler ที่มี middleware หลาย layer

interface RequestHandler {
  handle(request: HttpRequest): Promise<HttpResponse>;
}

interface HttpRequest {
  method: string;
  url: string;
  headers: Record<string, string>;
  body?: any;
  userId?: string;
  startTime?: number;
}

interface HttpResponse {
  status: number;
  body: any;
  headers: Record<string, string>;
}

// Concrete Handler
class ApiRequestHandler implements RequestHandler {
  async handle(request: HttpRequest): Promise<HttpResponse> {
    console.log(`[API] จัดการ ${request.method} ${request.url}`);
    
    // จำลองการประมวลผล request
    return {
      status: 200,
      body: { message: "สำเร็จ", data: { users: [] } },
      headers: { "Content-Type": "application/json" },
    };
  }
}

// Base Decorator
abstract class RequestHandlerDecorator implements RequestHandler {
  constructor(protected handler: RequestHandler) {}

  async handle(request: HttpRequest): Promise<HttpResponse> {
    return this.handler.handle(request);
  }
}

// Logging Decorator
class LoggingDecorator extends RequestHandlerDecorator {
  async handle(request: HttpRequest): Promise<HttpResponse> {
    const start = Date.now();
    console.log(`[LOG] ${request.method} ${request.url} - เริ่มประมวลผล`);
    
    const response = await super.handle(request);
    
    const duration = Date.now() - start;
    console.log(`[LOG] ${request.method} ${request.url} - สถานะ: ${response.status} (${duration}ms)`);
    
    return response;
  }
}

// Authentication Decorator
class AuthenticationDecorator extends RequestHandlerDecorator {
  async handle(request: HttpRequest): Promise<HttpResponse> {
    const authHeader = request.headers["Authorization"];
    
    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return {
        status: 401,
        body: { error: "ต้องมีการยืนยันตัวตน" },
        headers: { "Content-Type": "application/json" },
      };
    }

    const token = authHeader.substring(7);
    // จำลองการตรวจสอบ token
    if (token === "invalid") {
      return {
        status: 403,
        body: { error: "Token ไม่ถูกต้องหรือหมดอายุ" },
        headers: { "Content-Type": "application/json" },
      };
    }

    // เพิ่ม user info เข้า request
    request.userId = "user_123";
    
    return super.handle(request);
  }
}

// Rate Limiting Decorator
class RateLimitingDecorator extends RequestHandlerDecorator {
  private requests: Map<string, number[]> = new Map();
  private maxRequests: number;
  private windowMs: number;

  constructor(
    handler: RequestHandler,
    maxRequests: number = 100,
    windowMs: number = 60000
  ) {
    super(handler);
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
  }

  async handle(request: HttpRequest): Promise<HttpResponse> {
    const clientIp = request.headers["X-Forwarded-For"] || "unknown";
    const now = Date.now();
    
    // ดึงประวัติ request ของ IP นี้
    const clientRequests = this.requests.get(clientIp) || [];
    
    // กรองเอาเฉพาะ requests ที่อยู่ใน window
    const recentRequests = clientRequests.filter(
      (time) => now - time < this.windowMs
    );
    
    if (recentRequests.length >= this.maxRequests) {
      return {
        status: 429,
        body: { error: "คำร้องขอมากเกินไป กรุณาลองใหม่ภายหลัง" },
        headers: {
          "Content-Type": "application/json",
          "Retry-After": "60",
        },
      };
    }

    recentRequests.push(now);
    this.requests.set(clientIp, recentRequests);
    
    return super.handle(request);
  }
}

// Cache Decorator
class CacheDecorator extends RequestHandlerDecorator {
  private cache: Map<string, { response: HttpResponse; expiry: number }> = new Map();
  private ttlMs: number;

  constructor(handler: RequestHandler, ttlMs: number = 5000) {
    super(handler);
    this.ttlMs = ttlMs;
  }

  private getCacheKey(request: HttpRequest): string {
    return `${request.method}:${request.url}`;
  }

  async handle(request: HttpRequest): Promise<HttpResponse> {
    // Cache เฉพาะ GET requests
    if (request.method !== "GET") {
      return super.handle(request);
    }

    const cacheKey = this.getCacheKey(request);
    const cached = this.cache.get(cacheKey);
    
    if (cached && Date.now() < cached.expiry) {
      console.log(`[CACHE] Cache hit: ${cacheKey}`);
      return { ...cached.response, headers: { ...cached.response.headers, "X-Cache": "HIT" } };
    }

    const response = await super.handle(request);
    
    if (response.status === 200) {
      this.cache.set(cacheKey, {
        response,
        expiry: Date.now() + this.ttlMs,
      });
      console.log(`[CACHE] Cache stored: ${cacheKey}`);
    }

    return { ...response, headers: { ...response.headers, "X-Cache": "MISS" } };
  }
}

// การสร้าง Handler Pipeline
function createApiHandler(): RequestHandler {
  const baseHandler = new ApiRequestHandler();
  
  return new LoggingDecorator(
    new AuthenticationDecorator(
      new RateLimitingDecorator(
        new CacheDecorator(baseHandler, 10000),
        50,
        60000
      )
    )
  );
}

// การใช้งาน
async function demoDecorators() {
  const handler = createApiHandler();

  // Request ปกติ
  const response = await handler.handle({
    method: "GET",
    url: "/api/users",
    headers: {
      Authorization: "Bearer valid_token",
      "X-Forwarded-For": "192.168.1.1",
    },
  });
  console.log("Response:", response.status, response.body);

  // Request โดยไม่มี Auth
  const unauthorizedResponse = await handler.handle({
    method: "GET",
    url: "/api/users",
    headers: {
      "X-Forwarded-For": "192.168.1.2",
    },
  });
  console.log("Unauthorized:", unauthorizedResponse.status, unauthorizedResponse.body);
}

demoDecorators();
```

---

## 5. Facade Pattern (รูปแบบหน้ากาก)

### แนวคิด

Facade Pattern สร้าง interface ที่ง่ายกว่าสำหรับระบบหรือ library ที่ซับซ้อน โดยซ่อนความซับซ้อนของ subsystem ไว้เบื้องหลัง

### เมื่อไหรควรใช้ Facade Pattern

- เมื่อต้องการ interface ที่เรียบง่ายสำหรับ subsystem ที่ซับซ้อน
- เมื่อต้องการแยก subsystem ออกจาก client และ subsystem อื่นๆ
- เมื่อต้องการสร้าง layering สำหรับระบบ

### ตัวอย่างจริง: ระบบสั่งซื้อ E-commerce

```typescript
// Subsystems ต่างๆ

class InventorySystem {
  private inventory: Map<string, number> = new Map([
    ["PROD-001", 100],
    ["PROD-002", 50],
    ["PROD-003", 0],
  ]);

  checkAvailability(productId: string, quantity: number): boolean {
    const available = this.inventory.get(productId) || 0;
    return available >= quantity;
  }

  reserveItems(productId: string, quantity: number): string {
    const current = this.inventory.get(productId) || 0;
    this.inventory.set(productId, current - quantity);
    const reservationId = `RES-${Date.now()}`;
    console.log(`[Inventory] จองสินค้า ${productId} จำนวน ${quantity} ชิ้น (${reservationId})`);
    return reservationId;
  }

  releaseReservation(reservationId: string): void {
    console.log(`[Inventory] ยกเลิกการจอง ${reservationId}`);
  }
}

class PaymentSystem {
  async processPayment(
    amount: number,
    paymentMethod: string,
    paymentDetails: any
  ): Promise<{ success: boolean; transactionId: string }> {
    console.log(`[Payment] ประมวลผลการชำระเงิน ${amount} บาท ผ่าน ${paymentMethod}`);
    // จำลองการประมวลผล
    await new Promise((resolve) => setTimeout(resolve, 100));
    return {
      success: true,
      transactionId: `TXN-${Date.now()}`,
    };
  }

  async refundPayment(transactionId: string, amount: number): Promise<boolean> {
    console.log(`[Payment] คืนเงิน ${amount} บาท สำหรับ transaction ${transactionId}`);
    return true;
  }
}

class ShippingSystem {
  calculateShippingCost(
    origin: string,
    destination: string,
    weight: number
  ): number {
    const baseCost = 50;
    const weightCost = weight * 5;
    console.log(`[Shipping] คำนวณค่าจัดส่งจาก ${origin} ถึง ${destination}`);
    return baseCost + weightCost;
  }

  createShipment(
    orderId: string,
    address: string,
    items: any[]
  ): string {
    const trackingNumber = `TRK-${Date.now()}`;
    console.log(`[Shipping] สร้างการจัดส่ง ${trackingNumber} สำหรับ order ${orderId}`);
    return trackingNumber;
  }

  schedulePickup(trackingNumber: string, date: Date): void {
    console.log(`[Shipping] นัดรับสินค้า ${trackingNumber} วันที่ ${date.toLocaleDateString("th-TH")}`);
  }
}

class NotificationSystem {
  async sendOrderConfirmation(
    email: string,
    orderId: string,
    items: any[]
  ): Promise<void> {
    console.log(`[Notification] ส่งอีเมลยืนยันคำสั่งซื้อ ${orderId} ไปที่ ${email}`);
  }

  async sendShippingUpdate(
    email: string,
    trackingNumber: string
  ): Promise<void> {
    console.log(`[Notification] ส่งการอัพเดตการจัดส่ง ${trackingNumber} ไปที่ ${email}`);
  }

  async sendPaymentReceipt(
    email: string,
    amount: number,
    transactionId: string
  ): Promise<void> {
    console.log(`[Notification] ส่งใบเสร็จ ${amount} บาท (${transactionId}) ไปที่ ${email}`);
  }
}

class LoyaltySystem {
  addPoints(userId: string, amount: number): number {
    const points = Math.floor(amount / 10); // 1 point ต่อ 10 บาท
    console.log(`[Loyalty] เพิ่ม ${points} points ให้ user ${userId}`);
    return points;
  }

  redeemPoints(userId: string, points: number): number {
    const discount = points; // 1 point = 1 บาท
    console.log(`[Loyalty] ใช้ ${points} points (ลด ${discount} บาท) สำหรับ user ${userId}`);
    return discount;
  }
}

// Facade
class OrderFacade {
  private inventory: InventorySystem;
  private payment: PaymentSystem;
  private shipping: ShippingSystem;
  private notification: NotificationSystem;
  private loyalty: LoyaltySystem;

  constructor() {
    this.inventory = new InventorySystem();
    this.payment = new PaymentSystem();
    this.shipping = new ShippingSystem();
    this.notification = new NotificationSystem();
    this.loyalty = new LoyaltySystem();
  }

  async placeOrder(orderRequest: {
    userId: string;
    email: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
    shippingAddress: string;
    paymentMethod: string;
    paymentDetails: any;
    usePoints?: number;
  }): Promise<{ success: boolean; orderId: string; trackingNumber?: string }> {
    const orderId = `ORD-${Date.now()}`;
    const reservations: string[] = [];

    console.log(`\n=== เริ่มประมวลผลคำสั่งซื้อ ${orderId} ===\n`);

    try {
      // 1. ตรวจสอบและจองสินค้า
      for (const item of orderRequest.items) {
        if (!this.inventory.checkAvailability(item.productId, item.quantity)) {
          throw new Error(`สินค้า ${item.productId} มีไม่เพียงพอ`);
        }
        const reservationId = this.inventory.reserveItems(
          item.productId,
          item.quantity
        );
        reservations.push(reservationId);
      }

      // 2. คำนวณยอดรวม
      let totalAmount = orderRequest.items.reduce(
        (sum, item) => sum + item.price * item.quantity,
        0
      );

      // 3. หักคะแนน loyalty ถ้ามี
      if (orderRequest.usePoints && orderRequest.usePoints > 0) {
        const discount = this.loyalty.redeemPoints(
          orderRequest.userId,
          orderRequest.usePoints
        );
        totalAmount = Math.max(0, totalAmount - discount);
      }

      // 4. เพิ่มค่าจัดส่ง
      const shippingCost = this.shipping.calculateShippingCost(
        "กรุงเทพ",
        orderRequest.shippingAddress,
        1 // น้ำหนักสมมติ
      );
      totalAmount += shippingCost;

      // 5. ชำระเงิน
      const paymentResult = await this.payment.processPayment(
        totalAmount,
        orderRequest.paymentMethod,
        orderRequest.paymentDetails
      );

      if (!paymentResult.success) {
        throw new Error("การชำระเงินล้มเหลว");
      }

      // 6. สร้างการจัดส่ง
      const trackingNumber = this.shipping.createShipment(
        orderId,
        orderRequest.shippingAddress,
        orderRequest.items
      );

      // 7. เพิ่มคะแนน loyalty
      this.loyalty.addPoints(orderRequest.userId, totalAmount);

      // 8. ส่ง notifications
      await Promise.all([
        this.notification.sendOrderConfirmation(
          orderRequest.email,
          orderId,
          orderRequest.items
        ),
        this.notification.sendPaymentReceipt(
          orderRequest.email,
          totalAmount,
          paymentResult.transactionId
        ),
        this.notification.sendShippingUpdate(
          orderRequest.email,
          trackingNumber
        ),
      ]);

      console.log(`\n=== คำสั่งซื้อ ${orderId} สำเร็จ ===\n`);
      
      return { success: true, orderId, trackingNumber };
      
    } catch (error) {
      console.error(`เกิดข้อผิดพลาด: ${error}`);
      
      // Rollback: ยกเลิกการจองสินค้า
      for (const reservationId of reservations) {
        this.inventory.releaseReservation(reservationId);
      }
      
      return { success: false, orderId };
    }
  }
}

// การใช้งาน - Client code ง่ายมากเมื่อใช้ Facade
async function demoFacade() {
  const orderFacade = new OrderFacade();

  const result = await orderFacade.placeOrder({
    userId: "user_123",
    email: "customer@example.com",
    items: [
      { productId: "PROD-001", quantity: 2, price: 299 },
      { productId: "PROD-002", quantity: 1, price: 599 },
    ],
    shippingAddress: "เชียงใหม่",
    paymentMethod: "credit_card",
    paymentDetails: { cardNumber: "****1234" },
    usePoints: 50,
  });

  console.log("ผลลัพธ์:", result);
}

demoFacade();
```

---

## 6. Flyweight Pattern (รูปแบบน้ำหนักเบา)

### แนวคิด

Flyweight Pattern ลดการใช้หน่วยความจำโดยแชร์ state ที่ใช้ร่วมกัน (intrinsic state) ระหว่าง object หลายตัว ส่วน state ที่เปลี่ยนแปลงได้ (extrinsic state) จะถูกส่งผ่านเป็น parameter

### เมื่อไหรควรใช้ Flyweight Pattern

- เมื่อแอพลิเคชันต้องสร้าง object จำนวนมาก
- เมื่อ object กินหน่วยความจำมาก
- เมื่อ object มี state จำนวนมากที่ใช้ร่วมกันได้

### ตัวอย่างจริง: ระบบอนุภาคในเกม

```typescript
// Flyweight สำหรับ Particle ในเกม

// Intrinsic State (แชร์ได้)
interface ParticleType {
  color: string;
  sprite: string;
  texture: string;
}

// Flyweight class
class SharedParticleData {
  private static types: Map<string, ParticleType> = new Map();

  static getParticleType(
    color: string,
    sprite: string,
    texture: string
  ): ParticleType {
    const key = `${color}-${sprite}-${texture}`;
    
    if (!this.types.has(key)) {
      console.log(`สร้าง ParticleType ใหม่: ${key}`);
      this.types.set(key, { color, sprite, texture });
    }
    
    return this.types.get(key)!;
  }

  static getTypesCount(): number {
    return this.types.size;
  }
}

// Context (Extrinsic State)
class Particle {
  // Extrinsic state - ต่างกันในแต่ละ particle
  private x: number;
  private y: number;
  private velocity: { x: number; y: number };
  private lifetime: number;

  // Intrinsic state - แชร์ระหว่าง particles
  private type: ParticleType;

  constructor(
    x: number,
    y: number,
    velocityX: number,
    velocityY: number,
    color: string,
    sprite: string,
    texture: string
  ) {
    this.x = x;
    this.y = y;
    this.velocity = { x: velocityX, y: velocityY };
    this.lifetime = 100;
    this.type = SharedParticleData.getParticleType(color, sprite, texture);
  }

  update(): void {
    this.x += this.velocity.x;
    this.y += this.velocity.y;
    this.lifetime--;
  }

  render(): string {
    return `Particle(${this.type.color}) at (${this.x.toFixed(0)}, ${this.y.toFixed(0)}) lifetime:${this.lifetime}`;
  }

  isAlive(): boolean {
    return this.lifetime > 0;
  }
}

// Particle System
class ParticleSystem {
  private particles: Particle[] = [];

  createExplosion(x: number, y: number, count: number): void {
    for (let i = 0; i < count; i++) {
      const angle = (Math.PI * 2 * i) / count;
      const speed = Math.random() * 5 + 1;
      
      this.particles.push(
        new Particle(
          x, y,
          Math.cos(angle) * speed,
          Math.sin(angle) * speed,
          "orange",        // สีเหมือนกันทุก particle ใน explosion นี้
          "explosion.png", // sprite เหมือนกัน
          "fire_texture"   // texture เหมือนกัน
        )
      );
    }
    
    console.log(`สร้าง explosion ที่ (${x}, ${y}) จำนวน ${count} particles`);
    console.log(`จำนวน ParticleType ที่สร้าง: ${SharedParticleData.getTypesCount()}`);
  }

  createSmoke(x: number, y: number, count: number): void {
    for (let i = 0; i < count; i++) {
      this.particles.push(
        new Particle(
          x + Math.random() * 10 - 5,
          y,
          (Math.random() - 0.5) * 2,
          -(Math.random() * 3 + 1),
          "gray",
          "smoke.png",
          "smoke_texture"
        )
      );
    }
    
    console.log(`จำนวน ParticleType ที่สร้าง: ${SharedParticleData.getTypesCount()}`);
  }

  update(): void {
    this.particles = this.particles.filter((p) => {
      p.update();
      return p.isAlive();
    });
  }

  getActiveCount(): number {
    return this.particles.length;
  }
}

// การใช้งาน
const system = new ParticleSystem();
system.createExplosion(100, 100, 1000);
system.createExplosion(200, 150, 500);
system.createSmoke(150, 120, 200);

console.log(`\nParticles ทั้งหมด: ${system.getActiveCount()}`);
console.log(`ParticleTypes ที่ใช้ร่วมกัน: ${SharedParticleData.getTypesCount()}`);
console.log("แทนที่จะเก็บข้อมูล texture/sprite ซ้ำใน 1700 particles, มีเพียง 2 types เท่านั้น!");
```

### ตัวอย่างที่ 2: Font Glyph Cache

```typescript
// ตัวอย่าง: Text renderer ที่ใช้ Flyweight สำหรับ font glyphs

interface GlyphData {
  character: string;
  fontFamily: string;
  fontSize: number;
  bitmap: Uint8Array; // จำลองข้อมูล bitmap จริง
}

class GlyphFactory {
  private static glyphs: Map<string, GlyphData> = new Map();
  private static memoryUsed: number = 0;

  static getGlyph(
    character: string,
    fontFamily: string,
    fontSize: number
  ): GlyphData {
    const key = `${character}-${fontFamily}-${fontSize}`;
    
    if (!this.glyphs.has(key)) {
      // สมมติว่า bitmap ใช้หน่วยความจำ 1KB ต่อ glyph
      const bitmap = new Uint8Array(1024);
      
      const glyph: GlyphData = {
        character,
        fontFamily,
        fontSize,
        bitmap,
      };
      
      this.glyphs.set(key, glyph);
      this.memoryUsed += 1024;
      
      console.log(`สร้าง Glyph: '${character}' (${fontFamily} ${fontSize}px) - หน่วยความจำรวม: ${this.memoryUsed / 1024}KB`);
    }
    
    return this.glyphs.get(key)!;
  }

  static getStats(): { glyphCount: number; memoryKB: number } {
    return {
      glyphCount: this.glyphs.size,
      memoryKB: this.memoryUsed / 1024,
    };
  }
}

class TextCharacter {
  private glyph: GlyphData;
  
  // Extrinsic state
  constructor(
    character: string,
    fontFamily: string,
    fontSize: number,
    private x: number,
    private y: number,
    private color: string
  ) {
    this.glyph = GlyphFactory.getGlyph(character, fontFamily, fontSize);
  }

  render(): string {
    return `'${this.glyph.character}' ที่ (${this.x},${this.y}) สี:${this.color}`;
  }
}

// การใช้งาน - แสดงผลข้อความยาว
function renderLongText(text: string): void {
  const chars: TextCharacter[] = [];
  let x = 0;
  
  for (const char of text) {
    chars.push(new TextCharacter(char, "Sarabun", 16, x, 0, "#333"));
    x += 10;
  }
  
  const stats = GlyphFactory.getStats();
  console.log(`\nแสดงผล ${chars.length} ตัวอักษร`);
  console.log(`Glyphs unique: ${stats.glyphCount} (ใช้หน่วยความจำ ${stats.memoryKB}KB)`);
  console.log(`หากไม่ใช้ Flyweight: ${chars.length}KB`);
  console.log(`ประหยัดหน่วยความจำ: ${chars.length - stats.memoryKB}KB`);
}

const sampleText = "สวัสดีครับ วันนี้เราจะเรียนรู้เกี่ยวกับ TypeScript Design Patterns".repeat(10);
renderLongText(sampleText);
```

---

## 7. Proxy Pattern (รูปแบบตัวแทน)

### แนวคิด

Proxy Pattern สร้าง object ตัวแทน (surrogate) ที่ควบคุมการเข้าถึง object จริง โดยสามารถเพิ่ม functionality เพิ่มเติมก่อนหรือหลังการเรียกใช้ object จริง

### ประเภทของ Proxy

1. **Virtual Proxy** - สร้าง object ที่ใช้ทรัพยากรมากแบบ lazy
2. **Protection Proxy** - ควบคุมการเข้าถึง object
3. **Remote Proxy** - แทน object ที่อยู่บน server อื่น
4. **Caching Proxy** - เก็บ cache ของผลลัพธ์

### ตัวอย่างจริง: Virtual Proxy สำหรับ Image

```typescript
// Virtual Proxy - Lazy Loading

interface Image {
  display(): void;
  getSize(): { width: number; height: number };
}

// Real Image - ใช้เวลาและหน่วยความจำมากในการ load
class RealImage implements Image {
  private imageData: Uint8Array;
  private dimensions: { width: number; height: number };

  constructor(private filename: string) {
    console.log(`⏳ กำลัง load รูปภาพจาก disk: ${filename}`);
    // จำลองการอ่านไฟล์ (ใช้เวลานาน)
    this.imageData = new Uint8Array(1024 * 1024); // 1MB
    this.dimensions = { width: 1920, height: 1080 };
    console.log(`✅ Load รูปภาพสำเร็จ: ${filename}`);
  }

  display(): void {
    console.log(`🖼️ แสดงรูปภาพ: ${this.filename} (${this.dimensions.width}x${this.dimensions.height})`);
  }

  getSize(): { width: number; height: number } {
    return this.dimensions;
  }
}

// Virtual Proxy - ไม่ load จนกว่าจะต้องใช้จริง
class ImageProxy implements Image {
  private realImage: RealImage | null = null;

  constructor(private filename: string) {
    console.log(`📋 สร้าง ImageProxy สำหรับ: ${filename} (ยังไม่ load)`);
  }

  private loadIfNeeded(): RealImage {
    if (!this.realImage) {
      this.realImage = new RealImage(this.filename);
    }
    return this.realImage;
  }

  display(): void {
    this.loadIfNeeded().display();
  }

  getSize(): { width: number; height: number } {
    this.loadIfNeeded().getSize();
    return this.realImage!.getSize();
  }
}

// ตัวอย่าง: Image Gallery
class ImageGallery {
  private images: Image[] = [];

  addImage(filename: string): void {
    // ใช้ Proxy แทน Real Image - ไม่ load จนกว่าจะต้องการ
    this.images.push(new ImageProxy(filename));
  }

  displayImage(index: number): void {
    if (index < this.images.length) {
      this.images[index].display();
    }
  }
}

console.log("=== สร้าง Gallery (ไม่มีการ load รูปภาพ) ===");
const gallery = new ImageGallery();
gallery.addImage("photo1.jpg");
gallery.addImage("photo2.jpg");
gallery.addImage("photo3.jpg");

console.log("\n=== แสดงรูปภาพที่ 1 ===");
gallery.displayImage(0);

console.log("\n=== แสดงรูปภาพที่ 1 อีกครั้ง (ไม่ load ซ้ำ) ===");
gallery.displayImage(0);

console.log("\n=== แสดงรูปภาพที่ 2 (load ครั้งแรก) ===");
gallery.displayImage(1);
```

### ตัวอย่างที่ 2: Protection Proxy

```typescript
// Protection Proxy - ควบคุมสิทธิ์การเข้าถึง

interface DocumentEditor {
  read(documentId: string): string;
  write(documentId: string, content: string): void;
  delete(documentId: string): void;
}

type Role = "admin" | "editor" | "viewer";

interface User {
  id: string;
  name: string;
  role: Role;
}

// Real Document Editor
class RealDocumentEditor implements DocumentEditor {
  private documents: Map<string, string> = new Map([
    ["doc1", "เนื้อหาเอกสาร 1"],
    ["doc2", "เนื้อหาเอกสาร 2 - ลับ"],
  ]);

  read(documentId: string): string {
    return this.documents.get(documentId) || "ไม่พบเอกสาร";
  }

  write(documentId: string, content: string): void {
    this.documents.set(documentId, content);
    console.log(`[DB] บันทึกเอกสาร ${documentId}`);
  }

  delete(documentId: string): void {
    this.documents.delete(documentId);
    console.log(`[DB] ลบเอกสาร ${documentId}`);
  }
}

// Protection Proxy
class SecureDocumentEditor implements DocumentEditor {
  private editor: RealDocumentEditor;
  private currentUser: User;
  private auditLog: string[] = [];

  constructor(editor: RealDocumentEditor, user: User) {
    this.editor = editor;
    this.currentUser = user;
  }

  private log(action: string, documentId: string, success: boolean): void {
    const entry = `[${new Date().toISOString()}] User: ${this.currentUser.name} (${this.currentUser.role}) - ${action} ${documentId}: ${success ? "สำเร็จ" : "ถูกปฏิเสธ"}`;
    this.auditLog.push(entry);
    console.log(entry);
  }

  private hasPermission(action: "read" | "write" | "delete"): boolean {
    const permissions: Record<Role, string[]> = {
      admin: ["read", "write", "delete"],
      editor: ["read", "write"],
      viewer: ["read"],
    };
    
    return permissions[this.currentUser.role].includes(action);
  }

  read(documentId: string): string {
    if (!this.hasPermission("read")) {
      this.log("READ", documentId, false);
      throw new Error("ไม่มีสิทธิ์อ่านเอกสาร");
    }
    
    const content = this.editor.read(documentId);
    this.log("READ", documentId, true);
    return content;
  }

  write(documentId: string, content: string): void {
    if (!this.hasPermission("write")) {
      this.log("WRITE", documentId, false);
      throw new Error("ไม่มีสิทธิ์แก้ไขเอกสาร");
    }
    
    this.editor.write(documentId, content);
    this.log("WRITE", documentId, true);
  }

  delete(documentId: string): void {
    if (!this.hasPermission("delete")) {
      this.log("DELETE", documentId, false);
      throw new Error("ไม่มีสิทธิ์ลบเอกสาร");
    }
    
    this.editor.delete(documentId);
    this.log("DELETE", documentId, true);
  }

  getAuditLog(): string[] {
    return [...this.auditLog];
  }
}

// การใช้งาน
const realEditor = new RealDocumentEditor();

// Admin - มีสิทธิ์ทุกอย่าง
const adminUser: User = { id: "1", name: "สมชาย", role: "admin" };
const adminEditor = new SecureDocumentEditor(realEditor, adminUser);
adminEditor.read("doc1");
adminEditor.write("doc1", "เนื้อหาใหม่");
adminEditor.delete("doc2");

// Viewer - มีสิทธิ์แค่อ่าน
const viewerUser: User = { id: "2", name: "สมหญิง", role: "viewer" };
const viewerEditor = new SecureDocumentEditor(realEditor, viewerUser);
viewerEditor.read("doc1");
try {
  viewerEditor.write("doc1", "พยายามแก้ไข"); // จะ throw error
} catch (error) {
  console.log(`Error: ${error}`);
}
```

### ตัวอย่างที่ 3: Caching Proxy

```typescript
// Caching Proxy สำหรับ API calls

interface WeatherService {
  getWeather(city: string): Promise<WeatherData>;
  getForecast(city: string, days: number): Promise<WeatherData[]>;
}

interface WeatherData {
  city: string;
  temperature: number;
  humidity: number;
  description: string;
  timestamp: Date;
}

// Real Service
class OpenWeatherMapService implements WeatherService {
  async getWeather(city: string): Promise<WeatherData> {
    console.log(`🌐 เรียก API จริง สำหรับ ${city}`);
    // จำลองการเรียก API
    await new Promise((resolve) => setTimeout(resolve, 500));
    
    return {
      city,
      temperature: Math.round(20 + Math.random() * 15),
      humidity: Math.round(50 + Math.random() * 40),
      description: "มีเมฆบางส่วน",
      timestamp: new Date(),
    };
  }

  async getForecast(city: string, days: number): Promise<WeatherData[]> {
    console.log(`🌐 เรียก API พยากรณ์อากาศ ${days} วัน สำหรับ ${city}`);
    await new Promise((resolve) => setTimeout(resolve, 800));
    
    return Array.from({ length: days }, (_, i) => ({
      city,
      temperature: Math.round(20 + Math.random() * 15),
      humidity: Math.round(50 + Math.random() * 40),
      description: "แดดบางส่วน",
      timestamp: new Date(Date.now() + i * 86400000),
    }));
  }
}

// Caching Proxy
class CachedWeatherService implements WeatherService {
  private weatherCache: Map<string, { data: WeatherData; expiry: number }> = new Map();
  private forecastCache: Map<string, { data: WeatherData[]; expiry: number }> = new Map();
  private readonly cacheDurationMs: number;

  constructor(
    private realService: WeatherService,
    cacheDurationMs: number = 10 * 60 * 1000 // 10 นาที
  ) {
    this.cacheDurationMs = cacheDurationMs;
  }

  async getWeather(city: string): Promise<WeatherData> {
    const cacheKey = city.toLowerCase();
    const cached = this.weatherCache.get(cacheKey);
    
    if (cached && Date.now() < cached.expiry) {
      console.log(`💾 ใช้ข้อมูลจาก cache สำหรับ ${city}`);
      return cached.data;
    }

    const data = await this.realService.getWeather(city);
    
    this.weatherCache.set(cacheKey, {
      data,
      expiry: Date.now() + this.cacheDurationMs,
    });
    
    return data;
  }

  async getForecast(city: string, days: number): Promise<WeatherData[]> {
    const cacheKey = `${city.toLowerCase()}-${days}`;
    const cached = this.forecastCache.get(cacheKey);
    
    if (cached && Date.now() < cached.expiry) {
      console.log(`💾 ใช้ข้อมูลพยากรณ์จาก cache สำหรับ ${city}`);
      return cached.data;
    }

    const data = await this.realService.getForecast(city, days);
    
    this.forecastCache.set(cacheKey, {
      data,
      expiry: Date.now() + this.cacheDurationMs,
    });
    
    return data;
  }

  clearCache(): void {
    this.weatherCache.clear();
    this.forecastCache.clear();
    console.log("🗑️ ล้าง cache แล้ว");
  }
}

// การใช้งาน
async function demoWeather() {
  const realService = new OpenWeatherMapService();
  const cachedService = new CachedWeatherService(realService, 30000);

  console.log("--- ครั้งที่ 1 (เรียก API จริง) ---");
  const weather1 = await cachedService.getWeather("กรุงเทพ");
  console.log(`อุณหภูมิ: ${weather1.temperature}°C`);

  console.log("\n--- ครั้งที่ 2 (ใช้ cache) ---");
  const weather2 = await cachedService.getWeather("กรุงเทพ");
  console.log(`อุณหภูมิ: ${weather2.temperature}°C`);

  console.log("\n--- พยากรณ์อากาศ 5 วัน (เรียก API จริง) ---");
  const forecast = await cachedService.getForecast("เชียงใหม่", 5);
  console.log(`พยากรณ์ ${forecast.length} วัน`);
}

demoWeather();
```

---

## สรุป Structural Design Patterns

| Pattern | วัตถุประสงค์ | เมื่อไหรควรใช้ |
|---------|------------|--------------|
| **Adapter** | แปลง interface | เมื่อต้องการใช้ code เก่าร่วมกับระบบใหม่ |
| **Bridge** | แยก abstraction จาก implementation | เมื่อ class มีหลาย dimension ที่ต้องการ vary |
| **Composite** | โครงสร้างต้นไม้ | เมื่อต้องจัดการ object ทั้งแบบเดี่ยวและกลุ่มเหมือนกัน |
| **Decorator** | เพิ่ม behavior แบบ dynamic | เมื่อต้องการ feature เพิ่มเติมโดยไม่แก้ class เดิม |
| **Facade** | Interface ที่ง่ายกว่า | เมื่อ subsystem ซับซ้อนและต้องการ API ที่เรียบง่าย |
| **Flyweight** | ประหยัดหน่วยความจำ | เมื่อต้องสร้าง object จำนวนมากที่มี shared state |
| **Proxy** | ควบคุมการเข้าถึง | เมื่อต้องการ lazy loading, caching, หรือ access control |

## แบบฝึกหัด

1. สร้าง Adapter ระหว่าง REST API และ GraphQL API
2. ออกแบบ Bridge สำหรับระบบ export เอกสาร (PDF, Excel, CSV) ในหลาย format
3. สร้าง Composite สำหรับระบบ UI Component ที่มี nested components
4. ใช้ Decorator เพิ่ม validation, logging, และ caching ให้กับ repository
5. สร้าง Facade สำหรับระบบ authentication (email, social, OTP)
6. ใช้ Flyweight สำหรับ text editor ที่มีตัวอักษรหลายล้านตัว
7. สร้าง Proxy สำหรับ database connection ที่มี connection pooling
