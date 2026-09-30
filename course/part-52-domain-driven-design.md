# Part 52: Domain-Driven Design (DDD) กับ TypeScript

## บทนำ

Domain-Driven Design (DDD) คือแนวทางการออกแบบซอฟต์แวร์ที่เน้นการสร้างแบบจำลอง (model) ที่สะท้อนความเป็นจริงของธุรกิจ (business domain) โดย Eric Evans ได้นำเสนอแนวคิดนี้ในหนังสือ "Domain-Driven Design: Tackling Complexity in the Heart of Software" ในปี 2003

DDD เหมาะสำหรับระบบที่มีความซับซ้อนสูงและมีตรรกะทางธุรกิจมาก

---

## 1. DDD Overview และแนวคิดพื้นฐาน

### ทำไมต้องใช้ DDD?

```typescript
// ปัญหาที่เกิดขึ้นโดยไม่มี DDD
// โค้ดที่ผสมผสาน business logic กับ infrastructure
class OrderController {
  async createOrder(req: Request, res: Response) {
    const { userId, items } = req.body;
    
    // ตรวจสอบ user จาก database โดยตรง
    const user = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    // คำนวณราคาโดยตรงใน controller
    let total = 0;
    for (const item of items) {
      const product = await db.query(`SELECT * FROM products WHERE id = ${item.productId}`);
      total += product.price * item.quantity;
    }
    
    // บันทึก order โดยตรง
    await db.query(`INSERT INTO orders (user_id, total) VALUES (${userId}, ${total})`);
    
    res.json({ success: true });
  }
}
```

```typescript
// แนวทาง DDD - แยก concerns ชัดเจน
// Domain layer มี business logic
class Order {
  private readonly id: OrderId;
  private readonly customerId: CustomerId;
  private items: OrderItem[];
  private status: OrderStatus;
  
  constructor(id: OrderId, customerId: CustomerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
    this.status = OrderStatus.PENDING;
  }
  
  addItem(product: Product, quantity: number): void {
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    
    const existingItem = this.items.find(i => i.productId.equals(product.id));
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.push(new OrderItem(product.id, product.price, quantity));
    }
  }
  
  calculateTotal(): Money {
    return this.items.reduce(
      (total, item) => total.add(item.subtotal()),
      Money.zero()
    );
  }
  
  confirm(): void {
    if (this.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    this.status = OrderStatus.CONFIRMED;
  }
}
```

### สามชั้นหลักของ DDD

```
┌─────────────────────────────────────┐
│         User Interface              │  ← Presentation Layer
├─────────────────────────────────────┤
│         Application Layer           │  ← Use Cases / Application Services
├─────────────────────────────────────┤
│         Domain Layer                │  ← Business Logic (หัวใจของ DDD)
├─────────────────────────────────────┤
│         Infrastructure Layer        │  ← Database, API, External Services
└─────────────────────────────────────┘
```

---

## 2. Strategic Design

### Ubiquitous Language (ภาษาร่วม)

Ubiquitous Language คือภาษาที่ทีมพัฒนาและผู้เชี่ยวชาญด้านธุรกิจใช้ร่วมกัน

```typescript
// ❌ ภาษาที่ไม่ชัดเจน - technical jargon
interface UserRecord {
  uid: number;
  uname: string;
  stat: number;  // 0 = active, 1 = inactive, 2 = banned
  regDate: Date;
}

// ✅ Ubiquitous Language - ภาษาที่ทุกคนเข้าใจ
interface Customer {
  customerId: CustomerId;
  fullName: PersonName;
  accountStatus: AccountStatus;
  registrationDate: Date;
}

enum AccountStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  SUSPENDED = 'SUSPENDED'
}

// ตัวอย่าง: ภาษาในโดเมน E-commerce
// ใช้คำว่า "Place Order" ไม่ใช่ "Create Order Record"
// ใช้คำว่า "Customer" ไม่ใช่ "User"
// ใช้คำว่า "Product" ไม่ใช่ "Item"
// ใช้คำว่า "Shopping Cart" ไม่ใช่ "Temporary Order"
```

```typescript
// สร้าง Glossary ของ Ubiquitous Language
const ecommerceGlossary = {
  customer: 'บุคคลที่ซื้อสินค้าและมีบัญชีในระบบ',
  order: 'คำสั่งซื้อที่ลูกค้ายืนยันแล้ว',
  cart: 'รายการสินค้าที่ลูกค้ากำลังพิจารณาซื้อ',
  product: 'สินค้าที่จำหน่ายในร้าน',
  sku: 'รหัสสินค้าเฉพาะสำหรับแต่ละขนาด/สี',
  fulfillment: 'กระบวนการจัดส่งสินค้าให้ลูกค้า',
  refund: 'การคืนเงินให้ลูกค้า',
  discount: 'ส่วนลดที่ใช้กับคำสั่งซื้อ',
};
```

### Bounded Context (บริบทที่มีขอบเขต)

```typescript
// Bounded Context 1: Order Management
namespace OrderManagement {
  // ใน context นี้ "Product" คือสิ่งที่ถูกสั่งซื้อ
  interface Product {
    productId: string;
    name: string;
    price: number;
    availableQuantity: number;
  }
  
  interface Order {
    orderId: string;
    customerId: string;
    items: OrderItem[];
    status: OrderStatus;
  }
}

// Bounded Context 2: Inventory Management
namespace InventoryManagement {
  // ใน context นี้ "Product" คือสินค้าที่ต้องจัดการสต็อก
  interface Product {
    sku: string;
    warehouseLocation: string;
    stockQuantity: number;
    reorderPoint: number;
    supplier: Supplier;
  }
}

// Bounded Context 3: Catalog Management
namespace CatalogManagement {
  // ใน context นี้ "Product" คือสินค้าสำหรับแสดงในร้านค้า
  interface Product {
    productId: string;
    title: string;
    description: string;
    images: Image[];
    categories: Category[];
    seoMetadata: SeoMetadata;
  }
}
```

```typescript
// แต่ละ Bounded Context มี Model ของตัวเอง
// และสื่อสารกันผ่าน API หรือ Events

// การแมป Context ระหว่าง Order และ Inventory
class OrderToInventoryAdapter {
  // แปลง Order Management Product เป็น Inventory request
  adaptOrderItem(item: OrderManagement.OrderItem): InventoryManagement.ReservationRequest {
    return {
      sku: item.productId,  // ต้องแมป field ที่ต่างกัน
      quantity: item.quantity,
      orderId: item.orderId
    };
  }
}
```

---

## 3. Context Mapping

Context Mapping แสดงความสัมพันธ์ระหว่าง Bounded Contexts

### ประเภทของ Context Map

```typescript
// 1. Shared Kernel - แบ่งปัน model บางส่วน
// ทั้ง Order และ Customer ใช้ CustomerId ร่วมกัน
namespace SharedKernel {
  export class CustomerId {
    constructor(private readonly value: string) {
      if (!value || value.trim() === '') {
        throw new Error('CustomerId cannot be empty');
      }
    }
    
    getValue(): string {
      return this.value;
    }
    
    equals(other: CustomerId): boolean {
      return this.value === other.value;
    }
    
    toString(): string {
      return this.value;
    }
  }
}

// 2. Customer-Supplier - มีความสัมพันธ์แบบ upstream/downstream
// Order Context (Downstream) ขึ้นอยู่กับ Catalog Context (Upstream)
interface CatalogProductData {
  productId: string;
  name: string;
  price: number;
}

// Order Context ใช้ข้อมูลจาก Catalog
class OrderService {
  constructor(private catalogService: CatalogService) {}
  
  async addProductToOrder(
    orderId: string,
    productId: string,
    quantity: number
  ): Promise<void> {
    // ดึงข้อมูลจาก Catalog Context
    const product = await this.catalogService.getProduct(productId);
    // ใช้ใน Order Context
    const order = await this.orderRepository.findById(orderId);
    order.addItem(product, quantity);
    await this.orderRepository.save(order);
  }
}
```

```typescript
// 3. Anti-Corruption Layer (ACL) - ป้องกันโดเมนจากระบบภายนอก
// เมื่อต้องผสานกับระบบเก่า (Legacy System)

// Legacy System มี model แบบเก่า
interface LegacyOrderData {
  ord_no: string;
  cust_id: number;
  ord_dt: string;
  items: Array<{
    prod_cd: string;
    qty: number;
    unit_prc: number;
  }>;
  tot_amt: number;
  stat_cd: string;  // 'P' = Pending, 'C' = Confirmed, 'D' = Delivered
}

// ACL แปลงข้อมูลจาก Legacy เป็น Domain Model
class LegacyOrderACL {
  translateOrder(legacyOrder: LegacyOrderData): Order {
    return new Order(
      new OrderId(legacyOrder.ord_no),
      new CustomerId(legacyOrder.cust_id.toString()),
      this.translateItems(legacyOrder.items),
      this.translateStatus(legacyOrder.stat_cd),
      new Date(legacyOrder.ord_dt)
    );
  }
  
  private translateItems(legacyItems: LegacyOrderData['items']): OrderItem[] {
    return legacyItems.map(item => new OrderItem(
      new ProductId(item.prod_cd),
      new Money(item.unit_prc, 'THB'),
      item.qty
    ));
  }
  
  private translateStatus(statusCode: string): OrderStatus {
    const statusMap: Record<string, OrderStatus> = {
      'P': OrderStatus.PENDING,
      'C': OrderStatus.CONFIRMED,
      'D': OrderStatus.DELIVERED
    };
    
    const status = statusMap[statusCode];
    if (!status) {
      throw new Error(`Unknown status code: ${statusCode}`);
    }
    return status;
  }
}
```

---

## 4. Tactical Design - Value Objects

Value Objects คือ objects ที่ไม่มี identity แต่มีค่าที่สำคัญ

```typescript
// Value Object: Money
class Money {
  constructor(
    private readonly amount: number,
    private readonly currency: string
  ) {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    if (!currency || currency.length !== 3) {
      throw new Error('Currency must be a 3-letter code');
    }
    
    // ทำให้ immutable
    Object.freeze(this);
  }
  
  static zero(currency: string = 'THB'): Money {
    return new Money(0, currency);
  }
  
  static of(amount: number, currency: string): Money {
    return new Money(amount, currency);
  }
  
  add(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this.amount + other.amount, this.currency);
  }
  
  subtract(other: Money): Money {
    this.ensureSameCurrency(other);
    const result = this.amount - other.amount;
    if (result < 0) {
      throw new Error('Insufficient funds');
    }
    return new Money(result, this.currency);
  }
  
  multiply(factor: number): Money {
    if (factor < 0) {
      throw new Error('Factor cannot be negative');
    }
    return new Money(Math.round(this.amount * factor * 100) / 100, this.currency);
  }
  
  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }
  
  isGreaterThan(other: Money): boolean {
    this.ensureSameCurrency(other);
    return this.amount > other.amount;
  }
  
  private ensureSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new Error(
        `Cannot operate on different currencies: ${this.currency} vs ${other.currency}`
      );
    }
  }
  
  getAmount(): number { return this.amount; }
  getCurrency(): string { return this.currency; }
  
  toString(): string {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }
}

// การใช้งาน Money Value Object
const price = new Money(100, 'THB');
const discount = new Money(10, 'THB');
const finalPrice = price.subtract(discount); // THB 90.00
console.log(finalPrice.toString()); // "THB 90.00"
```

```typescript
// Value Object: Email
class Email {
  private readonly value: string;
  
  constructor(email: string) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      throw new Error(`Invalid email: ${email}`);
    }
    this.value = email.toLowerCase().trim();
    Object.freeze(this);
  }
  
  equals(other: Email): boolean {
    return this.value === other.value;
  }
  
  getDomain(): string {
    return this.value.split('@')[1];
  }
  
  toString(): string {
    return this.value;
  }
}

// Value Object: Address
class Address {
  constructor(
    private readonly street: string,
    private readonly city: string,
    private readonly postalCode: string,
    private readonly country: string
  ) {
    if (!street || !city || !postalCode || !country) {
      throw new Error('All address fields are required');
    }
    Object.freeze(this);
  }
  
  equals(other: Address): boolean {
    return (
      this.street === other.street &&
      this.city === other.city &&
      this.postalCode === other.postalCode &&
      this.country === other.country
    );
  }
  
  format(): string {
    return `${this.street}, ${this.city} ${this.postalCode}, ${this.country}`;
  }
}
```

```typescript
// Value Object: DateRange
class DateRange {
  constructor(
    private readonly start: Date,
    private readonly end: Date
  ) {
    if (start >= end) {
      throw new Error('Start date must be before end date');
    }
    Object.freeze(this);
  }
  
  contains(date: Date): boolean {
    return date >= this.start && date <= this.end;
  }
  
  overlaps(other: DateRange): boolean {
    return this.start <= other.end && this.end >= other.start;
  }
  
  getDurationInDays(): number {
    const msPerDay = 24 * 60 * 60 * 1000;
    return Math.floor((this.end.getTime() - this.start.getTime()) / msPerDay);
  }
  
  equals(other: DateRange): boolean {
    return (
      this.start.getTime() === other.start.getTime() &&
      this.end.getTime() === other.end.getTime()
    );
  }
}
```

---

## 5. Entities

Entities คือ objects ที่มี identity เฉพาะตัว และ identity นั้นคงที่แม้ attributes จะเปลี่ยนแปลง

```typescript
// Base Entity class
abstract class Entity<TId> {
  constructor(protected readonly id: TId) {}
  
  equals(other: Entity<TId>): boolean {
    if (!(other instanceof Entity)) return false;
    return this.id === other.id;
  }
  
  getId(): TId {
    return this.id;
  }
}

// Entity: Customer
class Customer extends Entity<CustomerId> {
  private name: PersonName;
  private email: Email;
  private address: Address;
  private loyaltyPoints: number;
  private createdAt: Date;
  
  constructor(
    id: CustomerId,
    name: PersonName,
    email: Email,
    address: Address
  ) {
    super(id);
    this.name = name;
    this.email = email;
    this.address = address;
    this.loyaltyPoints = 0;
    this.createdAt = new Date();
  }
  
  changeName(newName: PersonName): void {
    this.name = newName;
  }
  
  changeEmail(newEmail: Email): void {
    this.email = newEmail;
  }
  
  changeAddress(newAddress: Address): void {
    this.address = newAddress;
  }
  
  addLoyaltyPoints(points: number): void {
    if (points <= 0) {
      throw new Error('Points must be positive');
    }
    this.loyaltyPoints += points;
  }
  
  redeemLoyaltyPoints(points: number): void {
    if (points > this.loyaltyPoints) {
      throw new Error('Insufficient loyalty points');
    }
    this.loyaltyPoints -= points;
  }
  
  // Getters
  getName(): PersonName { return this.name; }
  getEmail(): Email { return this.email; }
  getAddress(): Address { return this.address; }
  getLoyaltyPoints(): number { return this.loyaltyPoints; }
  getCreatedAt(): Date { return this.createdAt; }
}

// PersonName Value Object
class PersonName {
  constructor(
    private readonly firstName: string,
    private readonly lastName: string
  ) {
    if (!firstName.trim() || !lastName.trim()) {
      throw new Error('Name cannot be empty');
    }
    Object.freeze(this);
  }
  
  getFullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }
  
  getFirstName(): string { return this.firstName; }
  getLastName(): string { return this.lastName; }
}
```

---

## 6. Aggregates

Aggregates คือกลุ่มของ Entities และ Value Objects ที่รวมกันเป็น unit และมี Aggregate Root เป็นจุดเข้าถึงหลัก

```typescript
// Aggregate Root: Order
class Order extends Entity<OrderId> {
  private readonly customerId: CustomerId;
  private items: OrderItem[];
  private status: OrderStatus;
  private shippingAddress: Address;
  private discount: Discount | null;
  private readonly createdAt: Date;
  private domainEvents: DomainEvent[];
  
  constructor(
    id: OrderId,
    customerId: CustomerId,
    shippingAddress: Address
  ) {
    super(id);
    this.customerId = customerId;
    this.items = [];
    this.status = OrderStatus.PENDING;
    this.shippingAddress = shippingAddress;
    this.discount = null;
    this.createdAt = new Date();
    this.domainEvents = [];
  }
  
  // Business methods
  addItem(productId: ProductId, productName: string, price: Money, quantity: number): void {
    this.ensureNotConfirmed();
    
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    
    const existingItem = this.items.find(i => i.getProductId().equals(productId));
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      const itemId = new OrderItemId(this.generateItemId());
      this.items.push(new OrderItem(itemId, productId, productName, price, quantity));
    }
    
    this.addDomainEvent(new OrderItemAddedEvent(this.id, productId, quantity));
  }
  
  removeItem(productId: ProductId): void {
    this.ensureNotConfirmed();
    
    const index = this.items.findIndex(i => i.getProductId().equals(productId));
    if (index === -1) {
      throw new Error('Item not found in order');
    }
    
    this.items.splice(index, 1);
    this.addDomainEvent(new OrderItemRemovedEvent(this.id, productId));
  }
  
  applyDiscount(discount: Discount): void {
    this.ensureNotConfirmed();
    
    if (!discount.isApplicable(this)) {
      throw new Error('Discount is not applicable to this order');
    }
    
    this.discount = discount;
  }
  
  confirm(): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error('Order can only be confirmed from pending status');
    }
    
    if (this.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    
    this.status = OrderStatus.CONFIRMED;
    this.addDomainEvent(new OrderConfirmedEvent(
      this.id,
      this.customerId,
      this.calculateTotal()
    ));
  }
  
  ship(trackingNumber: TrackingNumber): void {
    if (this.status !== OrderStatus.CONFIRMED) {
      throw new Error('Order must be confirmed before shipping');
    }
    
    this.status = OrderStatus.SHIPPED;
    this.addDomainEvent(new OrderShippedEvent(this.id, trackingNumber));
  }
  
  cancel(reason: string): void {
    if (this.status === OrderStatus.DELIVERED) {
      throw new Error('Cannot cancel delivered order');
    }
    
    this.status = OrderStatus.CANCELLED;
    this.addDomainEvent(new OrderCancelledEvent(this.id, reason));
  }
  
  calculateTotal(): Money {
    const subtotal = this.items.reduce(
      (total, item) => total.add(item.getSubtotal()),
      Money.zero('THB')
    );
    
    if (this.discount) {
      return this.discount.apply(subtotal);
    }
    
    return subtotal;
  }
  
  // Domain Events
  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event);
  }
  
  getDomainEvents(): DomainEvent[] {
    return [...this.domainEvents];
  }
  
  clearDomainEvents(): void {
    this.domainEvents = [];
  }
  
  // Invariant enforcement
  private ensureNotConfirmed(): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error('Cannot modify confirmed order');
    }
  }
  
  private generateItemId(): string {
    return `${this.id.getValue()}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  // Getters
  getCustomerId(): CustomerId { return this.customerId; }
  getItems(): ReadonlyArray<OrderItem> { return [...this.items]; }
  getStatus(): OrderStatus { return this.status; }
  getShippingAddress(): Address { return this.shippingAddress; }
  getCreatedAt(): Date { return this.createdAt; }
}

// OrderItem Entity (ภายใน Order Aggregate)
class OrderItem extends Entity<OrderItemId> {
  private readonly productId: ProductId;
  private readonly productName: string;
  private readonly unitPrice: Money;
  private quantity: number;
  
  constructor(
    id: OrderItemId,
    productId: ProductId,
    productName: string,
    unitPrice: Money,
    quantity: number
  ) {
    super(id);
    this.productId = productId;
    this.productName = productName;
    this.unitPrice = unitPrice;
    this.quantity = quantity;
  }
  
  increaseQuantity(amount: number): void {
    this.quantity += amount;
  }
  
  getSubtotal(): Money {
    return this.unitPrice.multiply(this.quantity);
  }
  
  getProductId(): ProductId { return this.productId; }
  getProductName(): string { return this.productName; }
  getUnitPrice(): Money { return this.unitPrice; }
  getQuantity(): number { return this.quantity; }
}
```

---

## 7. Domain Events

Domain Events แสดงถึงเหตุการณ์สำคัญที่เกิดขึ้นใน domain

```typescript
// Base Domain Event
abstract class DomainEvent {
  readonly occurredAt: Date;
  readonly eventId: string;
  
  constructor() {
    this.occurredAt = new Date();
    this.eventId = `evt-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  abstract getEventName(): string;
}

// Concrete Domain Events
class OrderCreatedEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly shippingAddress: Address
  ) {
    super();
  }
  
  getEventName(): string {
    return 'OrderCreated';
  }
}

class OrderItemAddedEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly productId: ProductId,
    readonly quantity: number
  ) {
    super();
  }
  
  getEventName(): string {
    return 'OrderItemAdded';
  }
}

class OrderConfirmedEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly totalAmount: Money
  ) {
    super();
  }
  
  getEventName(): string {
    return 'OrderConfirmed';
  }
}

class OrderShippedEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly trackingNumber: TrackingNumber
  ) {
    super();
  }
  
  getEventName(): string {
    return 'OrderShipped';
  }
}

class OrderCancelledEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly reason: string
  ) {
    super();
  }
  
  getEventName(): string {
    return 'OrderCancelled';
  }
}

// Domain Event Publisher
interface DomainEventHandler<T extends DomainEvent> {
  handle(event: T): Promise<void>;
}

class DomainEventPublisher {
  private static instance: DomainEventPublisher;
  private handlers: Map<string, DomainEventHandler<any>[]> = new Map();
  
  static getInstance(): DomainEventPublisher {
    if (!DomainEventPublisher.instance) {
      DomainEventPublisher.instance = new DomainEventPublisher();
    }
    return DomainEventPublisher.instance;
  }
  
  subscribe<T extends DomainEvent>(
    eventName: string,
    handler: DomainEventHandler<T>
  ): void {
    const handlers = this.handlers.get(eventName) || [];
    handlers.push(handler);
    this.handlers.set(eventName, handlers);
  }
  
  async publish(event: DomainEvent): Promise<void> {
    const eventHandlers = this.handlers.get(event.getEventName()) || [];
    await Promise.all(eventHandlers.map(handler => handler.handle(event)));
  }
  
  async publishAll(events: DomainEvent[]): Promise<void> {
    await Promise.all(events.map(event => this.publish(event)));
  }
}

// ตัวอย่าง Event Handler
class SendOrderConfirmationEmailHandler 
  implements DomainEventHandler<OrderConfirmedEvent> {
  
  constructor(private emailService: EmailService) {}
  
  async handle(event: OrderConfirmedEvent): Promise<void> {
    await this.emailService.sendOrderConfirmation(
      event.customerId,
      event.orderId,
      event.totalAmount
    );
  }
}

class UpdateInventoryOnOrderConfirmedHandler
  implements DomainEventHandler<OrderConfirmedEvent> {
  
  constructor(private inventoryService: InventoryService) {}
  
  async handle(event: OrderConfirmedEvent): Promise<void> {
    // ลด stock เมื่อ order ถูก confirm
    console.log(`Updating inventory for order ${event.orderId}`);
  }
}
```

---

## 8. Repositories

Repositories เป็น interface สำหรับเข้าถึงและบันทึก Aggregates

```typescript
// Repository Interface (Domain Layer)
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;
  findByStatus(status: OrderStatus): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
  nextId(): OrderId;
}

// Repository Implementation (Infrastructure Layer)
class PostgresOrderRepository implements OrderRepository {
  constructor(private db: Database) {}
  
  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.query(
      'SELECT * FROM orders WHERE id = $1',
      [id.getValue()]
    );
    
    if (!row) return null;
    
    return this.toDomain(row);
  }
  
  async findByCustomerId(customerId: CustomerId): Promise<Order[]> {
    const rows = await this.db.query(
      'SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC',
      [customerId.getValue()]
    );
    
    return rows.map(row => this.toDomain(row));
  }
  
  async findByStatus(status: OrderStatus): Promise<Order[]> {
    const rows = await this.db.query(
      'SELECT * FROM orders WHERE status = $1',
      [status]
    );
    
    return rows.map(row => this.toDomain(row));
  }
  
  async save(order: Order): Promise<void> {
    const orderData = this.toPersistence(order);
    
    await this.db.transaction(async (trx) => {
      // บันทึก order
      await trx.query(
        `INSERT INTO orders (id, customer_id, status, total, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, NOW())
         ON CONFLICT (id) DO UPDATE SET
           status = EXCLUDED.status,
           total = EXCLUDED.total,
           updated_at = NOW()`,
        [
          orderData.id,
          orderData.customerId,
          orderData.status,
          orderData.total,
          orderData.createdAt
        ]
      );
      
      // บันทึก order items
      await trx.query('DELETE FROM order_items WHERE order_id = $1', [orderData.id]);
      
      for (const item of orderData.items) {
        await trx.query(
          `INSERT INTO order_items (id, order_id, product_id, product_name, unit_price, quantity)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [item.id, item.orderId, item.productId, item.productName, item.unitPrice, item.quantity]
        );
      }
    });
    
    // Publish domain events
    const events = order.getDomainEvents();
    order.clearDomainEvents();
    await DomainEventPublisher.getInstance().publishAll(events);
  }
  
  async delete(id: OrderId): Promise<void> {
    await this.db.query('DELETE FROM orders WHERE id = $1', [id.getValue()]);
  }
  
  nextId(): OrderId {
    return new OrderId(`order-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`);
  }
  
  // Mapper functions
  private toDomain(row: any): Order {
    // แปลงจาก database row เป็น Domain Object
    return Order.reconstitute(
      new OrderId(row.id),
      new CustomerId(row.customer_id),
      new Address(row.street, row.city, row.postal_code, row.country),
      row.items.map((item: any) => ({
        id: new OrderItemId(item.id),
        productId: new ProductId(item.product_id),
        productName: item.product_name,
        price: new Money(item.unit_price, 'THB'),
        quantity: item.quantity
      })),
      row.status as OrderStatus,
      row.created_at
    );
  }
  
  private toPersistence(order: Order): any {
    return {
      id: order.getId().getValue(),
      customerId: order.getCustomerId().getValue(),
      status: order.getStatus(),
      total: order.calculateTotal().getAmount(),
      createdAt: order.getCreatedAt(),
      items: order.getItems().map(item => ({
        id: item.getId().getValue(),
        orderId: order.getId().getValue(),
        productId: item.getProductId().getValue(),
        productName: item.getProductName(),
        unitPrice: item.getUnitPrice().getAmount(),
        quantity: item.getQuantity()
      }))
    };
  }
}
```

---

## 9. Domain Services

Domain Services มี business logic ที่ไม่เหมาะจะอยู่ใน Entity หรือ Value Object

```typescript
// Domain Service: PricingService
interface PricingService {
  calculateOrderPrice(order: Order, customer: Customer): Money;
  applyPromotion(order: Order, promotionCode: string): Discount;
}

class DefaultPricingService implements PricingService {
  constructor(
    private promotionRepository: PromotionRepository,
    private taxService: TaxService
  ) {}
  
  calculateOrderPrice(order: Order, customer: Customer): Money {
    let total = order.calculateTotal();
    
    // ใช้ loyalty discount
    if (customer.getLoyaltyPoints() >= 1000) {
      const loyaltyDiscount = new Discount('loyalty', DiscountType.PERCENTAGE, 5);
      total = loyaltyDiscount.apply(total);
    }
    
    // เพิ่มภาษี
    const tax = this.taxService.calculateTax(total, order.getShippingAddress());
    return total.add(tax);
  }
  
  async applyPromotion(order: Order, promotionCode: string): Promise<Discount> {
    const promotion = await this.promotionRepository.findByCode(promotionCode);
    
    if (!promotion) {
      throw new Error(`Promotion code not found: ${promotionCode}`);
    }
    
    if (!promotion.isValid()) {
      throw new Error(`Promotion code has expired: ${promotionCode}`);
    }
    
    if (!promotion.isApplicableToOrder(order)) {
      throw new Error(`Promotion code is not applicable to this order`);
    }
    
    return promotion.createDiscount();
  }
}

// Domain Service: InventoryService
interface InventoryCheckService {
  checkAvailability(productId: ProductId, quantity: number): Promise<boolean>;
  reserveItems(items: Array<{ productId: ProductId; quantity: number }>): Promise<ReservationId>;
}

class DefaultInventoryCheckService implements InventoryCheckService {
  constructor(private inventoryRepository: InventoryRepository) {}
  
  async checkAvailability(productId: ProductId, quantity: number): Promise<boolean> {
    const inventory = await this.inventoryRepository.findByProductId(productId);
    
    if (!inventory) {
      return false;
    }
    
    return inventory.getAvailableQuantity() >= quantity;
  }
  
  async reserveItems(
    items: Array<{ productId: ProductId; quantity: number }>
  ): Promise<ReservationId> {
    // ตรวจสอบทุก items ก่อน
    for (const item of items) {
      const available = await this.checkAvailability(item.productId, item.quantity);
      if (!available) {
        throw new Error(`Product ${item.productId} is not available in requested quantity`);
      }
    }
    
    // จอง items
    const reservation = await this.inventoryRepository.createReservation(items);
    return reservation.getId();
  }
}
```

---

## 10. Application Services

Application Services ประสาน Domain Objects เพื่อทำให้ Use Cases สำเร็จ

```typescript
// Application Service: OrderApplicationService
interface PlaceOrderCommand {
  customerId: string;
  shippingAddress: {
    street: string;
    city: string;
    postalCode: string;
    country: string;
  };
  items: Array<{
    productId: string;
    quantity: number;
  }>;
  promotionCode?: string;
}

interface PlaceOrderResult {
  orderId: string;
  total: number;
  currency: string;
}

class OrderApplicationService {
  constructor(
    private orderRepository: OrderRepository,
    private customerRepository: CustomerRepository,
    private productRepository: ProductRepository,
    private pricingService: PricingService,
    private inventoryService: InventoryCheckService
  ) {}
  
  async placeOrder(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    // 1. ตรวจสอบ Customer
    const customer = await this.customerRepository.findById(
      new CustomerId(command.customerId)
    );
    
    if (!customer) {
      throw new Error('Customer not found');
    }
    
    // 2. ตรวจสอบสินค้า
    const products = await Promise.all(
      command.items.map(item => 
        this.productRepository.findById(new ProductId(item.productId))
      )
    );
    
    for (const product of products) {
      if (!product) {
        throw new Error('Product not found');
      }
    }
    
    // 3. ตรวจสอบ inventory
    await this.inventoryService.reserveItems(
      command.items.map(item => ({
        productId: new ProductId(item.productId),
        quantity: item.quantity
      }))
    );
    
    // 4. สร้าง Order
    const orderId = this.orderRepository.nextId();
    const shippingAddress = new Address(
      command.shippingAddress.street,
      command.shippingAddress.city,
      command.shippingAddress.postalCode,
      command.shippingAddress.country
    );
    
    const order = new Order(orderId, customer.getId(), shippingAddress);
    
    // 5. เพิ่มสินค้า
    for (let i = 0; i < command.items.length; i++) {
      const product = products[i]!;
      const item = command.items[i];
      
      order.addItem(
        product.getId(),
        product.getName(),
        product.getPrice(),
        item.quantity
      );
    }
    
    // 6. ใช้ promotion code ถ้ามี
    if (command.promotionCode) {
      const discount = await this.pricingService.applyPromotion(
        order,
        command.promotionCode
      );
      order.applyDiscount(discount);
    }
    
    // 7. Confirm order
    order.confirm();
    
    // 8. บันทึก order
    await this.orderRepository.save(order);
    
    // 9. เพิ่ม loyalty points ให้ customer
    const loyaltyPoints = Math.floor(order.calculateTotal().getAmount() / 100);
    customer.addLoyaltyPoints(loyaltyPoints);
    await this.customerRepository.save(customer);
    
    return {
      orderId: order.getId().getValue(),
      total: order.calculateTotal().getAmount(),
      currency: order.calculateTotal().getCurrency()
    };
  }
  
  async cancelOrder(orderId: string, reason: string): Promise<void> {
    const order = await this.orderRepository.findById(new OrderId(orderId));
    
    if (!order) {
      throw new Error('Order not found');
    }
    
    order.cancel(reason);
    await this.orderRepository.save(order);
  }
}
```

---

## 11. Factories

Factories สร้าง complex Domain Objects

```typescript
// Factory Method สำหรับสร้าง Order
class OrderFactory {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository
  ) {}
  
  async createFromCart(
    cart: ShoppingCart,
    customer: Customer,
    shippingAddress: Address
  ): Promise<Order> {
    const orderId = this.orderRepository.nextId();
    const order = new Order(orderId, customer.getId(), shippingAddress);
    
    for (const cartItem of cart.getItems()) {
      const product = await this.productRepository.findById(cartItem.getProductId());
      
      if (!product) {
        throw new Error(`Product not found: ${cartItem.getProductId()}`);
      }
      
      if (!product.isAvailable()) {
        throw new Error(`Product is not available: ${product.getName()}`);
      }
      
      order.addItem(
        product.getId(),
        product.getName(),
        product.getPrice(),
        cartItem.getQuantity()
      );
    }
    
    return order;
  }
  
  createDraftOrder(customerId: CustomerId, shippingAddress: Address): Order {
    const orderId = this.orderRepository.nextId();
    return new Order(orderId, customerId, shippingAddress);
  }
}

// Abstract Factory สำหรับ E-commerce entities
interface EcommerceEntityFactory {
  createOrder(customerId: CustomerId, address: Address): Order;
  createProduct(name: string, price: Money, category: string): Product;
  createCustomer(name: PersonName, email: Email, address: Address): Customer;
}

class DefaultEcommerceFactory implements EcommerceEntityFactory {
  createOrder(customerId: CustomerId, address: Address): Order {
    const id = new OrderId(`order-${Date.now()}`);
    return new Order(id, customerId, address);
  }
  
  createProduct(name: string, price: Money, category: string): Product {
    const id = new ProductId(`prod-${Date.now()}`);
    return new Product(id, name, price, category);
  }
  
  createCustomer(name: PersonName, email: Email, address: Address): Customer {
    const id = new CustomerId(`cust-${Date.now()}`);
    return new Customer(id, name, email, address);
  }
}
```

---

## 12. Anti-Corruption Layer (ACL)

```typescript
// สถานการณ์: ระบบใหม่ต้องทำงานร่วมกับ Legacy Payment System

// Legacy Payment System Interface (เราไม่ควบคุม)
interface LegacyPaymentGateway {
  processPayment(data: {
    tx_id: string;
    amt: number;
    curr: string;
    card_no: string;
    exp_mo: string;
    exp_yr: string;
    cvv: string;
    merchant_id: string;
  }): Promise<{
    status: 'SUCCESS' | 'FAILED' | 'PENDING';
    txn_ref: string;
    err_code?: string;
    err_msg?: string;
  }>;
}

// Domain Models ของเรา
interface PaymentRequest {
  orderId: OrderId;
  amount: Money;
  paymentMethod: CreditCard;
}

interface PaymentResult {
  transactionId: string;
  status: PaymentStatus;
  errorMessage?: string;
}

enum PaymentStatus {
  SUCCESSFUL = 'SUCCESSFUL',
  FAILED = 'FAILED',
  PENDING = 'PENDING'
}

// ACL ที่ป้องกัน Domain ของเราจาก Legacy
class LegacyPaymentACL {
  private readonly MERCHANT_ID = 'MERCHANT_001';
  
  constructor(private legacyGateway: LegacyPaymentGateway) {}
  
  async processPayment(request: PaymentRequest): Promise<PaymentResult> {
    // แปลงจาก Domain Model เป็น Legacy Format
    const legacyRequest = this.toLeagcyRequest(request);
    
    const legacyResponse = await this.legacyGateway.processPayment(legacyRequest);
    
    // แปลง Legacy Response เป็น Domain Model
    return this.toDomainResult(legacyResponse);
  }
  
  private toLeagcyRequest(request: PaymentRequest): any {
    const card = request.paymentMethod;
    
    return {
      tx_id: `TX-${request.orderId.getValue()}-${Date.now()}`,
      amt: request.amount.getAmount(),
      curr: request.amount.getCurrency(),
      card_no: card.getNumber(),
      exp_mo: card.getExpiryMonth().toString().padStart(2, '0'),
      exp_yr: card.getExpiryYear().toString(),
      cvv: card.getCvv(),
      merchant_id: this.MERCHANT_ID
    };
  }
  
  private toDomainResult(legacyResponse: any): PaymentResult {
    const statusMap: Record<string, PaymentStatus> = {
      'SUCCESS': PaymentStatus.SUCCESSFUL,
      'FAILED': PaymentStatus.FAILED,
      'PENDING': PaymentStatus.PENDING
    };
    
    return {
      transactionId: legacyResponse.txn_ref,
      status: statusMap[legacyResponse.status] || PaymentStatus.FAILED,
      errorMessage: legacyResponse.err_msg
    };
  }
}
```

---

## 13. Complete DDD Example - E-commerce System

```typescript
// ===== Domain Layer =====

// Value Objects
class ProductId {
  constructor(private readonly value: string) {
    if (!value) throw new Error('ProductId cannot be empty');
    Object.freeze(this);
  }
  getValue(): string { return this.value; }
  equals(other: ProductId): boolean { return this.value === other.value; }
  toString(): string { return this.value; }
}

class OrderId {
  constructor(private readonly value: string) {
    if (!value) throw new Error('OrderId cannot be empty');
    Object.freeze(this);
  }
  getValue(): string { return this.value; }
  equals(other: OrderId): boolean { return this.value === other.value; }
  toString(): string { return this.value; }
}

class CustomerId {
  constructor(private readonly value: string) {
    if (!value) throw new Error('CustomerId cannot be empty');
    Object.freeze(this);
  }
  getValue(): string { return this.value; }
  equals(other: CustomerId): boolean { return this.value === other.value; }
  toString(): string { return this.value; }
}

class OrderItemId {
  constructor(private readonly value: string) {
    Object.freeze(this);
  }
  getValue(): string { return this.value; }
  equals(other: OrderItemId): boolean { return this.value === other.value; }
}

// Enums
enum OrderStatus {
  PENDING = 'PENDING',
  CONFIRMED = 'CONFIRMED',
  PROCESSING = 'PROCESSING',
  SHIPPED = 'SHIPPED',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED',
  REFUNDED = 'REFUNDED'
}

// Complete Product Entity
class Product extends Entity<ProductId> {
  private name: string;
  private description: string;
  private price: Money;
  private category: string;
  private isActive: boolean;
  private stockQuantity: number;
  
  constructor(
    id: ProductId,
    name: string,
    description: string,
    price: Money,
    category: string,
    stockQuantity: number
  ) {
    super(id);
    this.name = name;
    this.description = description;
    this.price = price;
    this.category = category;
    this.isActive = true;
    this.stockQuantity = stockQuantity;
  }
  
  isAvailable(): boolean {
    return this.isActive && this.stockQuantity > 0;
  }
  
  isAvailableInQuantity(quantity: number): boolean {
    return this.isActive && this.stockQuantity >= quantity;
  }
  
  decreaseStock(quantity: number): void {
    if (quantity > this.stockQuantity) {
      throw new Error('Insufficient stock');
    }
    this.stockQuantity -= quantity;
  }
  
  increaseStock(quantity: number): void {
    this.stockQuantity += quantity;
  }
  
  deactivate(): void {
    this.isActive = false;
  }
  
  getName(): string { return this.name; }
  getDescription(): string { return this.description; }
  getPrice(): Money { return this.price; }
  getCategory(): string { return this.category; }
  getStockQuantity(): number { return this.stockQuantity; }
}

// Complete implementation test
async function runDDDExample() {
  // สร้าง customer
  const customerId = new CustomerId('cust-001');
  const name = new PersonName('สมชาย', 'ใจดี');
  const email = new Email('somchai@example.com');
  const address = new Address('123 ถนนสุขุมวิท', 'กรุงเทพฯ', '10110', 'Thailand');
  const customer = new Customer(customerId, name, email, address);
  
  // สร้าง products
  const productId1 = new ProductId('prod-001');
  const product1 = new Product(
    productId1,
    'MacBook Pro',
    'Laptop รุ่นใหม่',
    new Money(59990, 'THB'),
    'Electronics',
    10
  );
  
  // สร้าง order
  const orderId = new OrderId(`order-${Date.now()}`);
  const order = new Order(orderId, customerId, address);
  
  // เพิ่ม items
  order.addItem(
    product1.getId(),
    product1.getName(),
    product1.getPrice(),
    1
  );
  
  // Confirm order
  order.confirm();
  
  console.log(`Order ${order.getId()} confirmed`);
  console.log(`Total: ${order.calculateTotal()}`);
  
  // แสดง domain events
  const events = order.getDomainEvents();
  console.log('Domain Events:', events.map(e => e.getEventName()));
}
```

---

## 14. DDD กับ TypeScript Best Practices

```typescript
// 1. ใช้ Nominal Typing เพื่อป้องกันการสับสน IDs
type Brand<T, BrandName extends string> = T & { __brand: BrandName };

type OrderIdType = Brand<string, 'OrderId'>;
type CustomerIdType = Brand<string, 'CustomerId'>;
type ProductIdType = Brand<string, 'ProductId'>;

function createOrderId(value: string): OrderIdType {
  if (!value) throw new Error('OrderId cannot be empty');
  return value as OrderIdType;
}

function createCustomerId(value: string): CustomerIdType {
  if (!value) throw new Error('CustomerId cannot be empty');
  return value as CustomerIdType;
}

// TypeScript จะ error ถ้าใส่ผิดประเภท
function processOrder(orderId: OrderIdType, customerId: CustomerIdType) {
  console.log(`Processing order ${orderId} for customer ${customerId}`);
}

const orderId = createOrderId('order-001');
const customerId = createCustomerId('cust-001');

// ✅ ถูกต้อง
processOrder(orderId, customerId);

// ❌ Error: Argument of type 'CustomerIdType' is not assignable to parameter of type 'OrderIdType'
// processOrder(customerId, orderId);
```

```typescript
// 2. Result Type สำหรับ Domain Operations
type Result<T, E = Error> = 
  | { success: true; value: T }
  | { success: false; error: E };

function ok<T>(value: T): Result<T> {
  return { success: true, value };
}

function fail<E = Error>(error: E): Result<never, E> {
  return { success: false, error };
}

// ใช้ใน Domain
class Order {
  tryAddItem(
    productId: ProductId,
    price: Money,
    quantity: number
  ): Result<void, string> {
    if (this.status !== OrderStatus.PENDING) {
      return fail('Cannot add items to confirmed order');
    }
    
    if (quantity <= 0) {
      return fail('Quantity must be positive');
    }
    
    // เพิ่ม item...
    return ok(undefined);
  }
}

// การใช้งาน
const result = order.tryAddItem(productId, price, 2);
if (!result.success) {
  console.error('Failed to add item:', result.error);
} else {
  console.log('Item added successfully');
}
```

```typescript
// 3. Specification Pattern สำหรับ business rules
interface Specification<T> {
  isSatisfiedBy(entity: T): boolean;
  and(other: Specification<T>): Specification<T>;
  or(other: Specification<T>): Specification<T>;
  not(): Specification<T>;
}

abstract class AbstractSpecification<T> implements Specification<T> {
  abstract isSatisfiedBy(entity: T): boolean;
  
  and(other: Specification<T>): Specification<T> {
    return new AndSpecification(this, other);
  }
  
  or(other: Specification<T>): Specification<T> {
    return new OrSpecification(this, other);
  }
  
  not(): Specification<T> {
    return new NotSpecification(this);
  }
}

class AndSpecification<T> extends AbstractSpecification<T> {
  constructor(
    private left: Specification<T>,
    private right: Specification<T>
  ) {
    super();
  }
  
  isSatisfiedBy(entity: T): boolean {
    return this.left.isSatisfiedBy(entity) && this.right.isSatisfiedBy(entity);
  }
}

class OrSpecification<T> extends AbstractSpecification<T> {
  constructor(
    private left: Specification<T>,
    private right: Specification<T>
  ) {
    super();
  }
  
  isSatisfiedBy(entity: T): boolean {
    return this.left.isSatisfiedBy(entity) || this.right.isSatisfiedBy(entity);
  }
}

class NotSpecification<T> extends AbstractSpecification<T> {
  constructor(private spec: Specification<T>) {
    super();
  }
  
  isSatisfiedBy(entity: T): boolean {
    return !this.spec.isSatisfiedBy(entity);
  }
}

// Concrete Specifications
class EligibleForFreeShippingSpec extends AbstractSpecification<Order> {
  private readonly MINIMUM_ORDER_AMOUNT = 500;
  
  isSatisfiedBy(order: Order): boolean {
    return order.calculateTotal().getAmount() >= this.MINIMUM_ORDER_AMOUNT;
  }
}

class HasValidPaymentMethodSpec extends AbstractSpecification<Order> {
  isSatisfiedBy(order: Order): boolean {
    // ตรวจสอบว่ามี payment method ที่ valid
    return true; // simplified
  }
}

// การใช้งาน Specifications
const freeShippingSpec = new EligibleForFreeShippingSpec();
const validPaymentSpec = new HasValidPaymentMethodSpec();
const canCheckoutSpec = freeShippingSpec.and(validPaymentSpec);

if (canCheckoutSpec.isSatisfiedBy(order)) {
  console.log('Order can proceed to checkout');
}
```

---

## สรุป

DDD เป็นแนวทางที่ทรงพลังสำหรับระบบที่ซับซ้อน โดยมีหลักการสำคัญ:

1. **Ubiquitous Language** - ใช้ภาษาเดียวกันทั้ง developer และ business
2. **Bounded Context** - แบ่งระบบออกเป็นส่วนที่มีขอบเขตชัดเจน
3. **Value Objects** - immutable objects ที่ defined โดย attributes
4. **Entities** - objects ที่มี identity เฉพาะตัว
5. **Aggregates** - กลุ่มของ objects ที่มี consistency boundary
6. **Domain Events** - เหตุการณ์ที่เกิดขึ้นใน domain
7. **Repositories** - abstraction สำหรับ data access
8. **Domain Services** - business logic ที่ไม่เป็นของ entity ใด
9. **Application Services** - ประสาน use cases
10. **Anti-Corruption Layer** - ป้องกันโดเมนจากระบบภายนอก
