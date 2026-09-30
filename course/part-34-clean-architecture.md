# Part 34: Clean Architecture ใน TypeScript

## บทนำ

Clean Architecture คือแนวทางการออกแบบซอฟต์แวร์ที่คิดค้นโดย Robert C. Martin (Uncle Bob) มีเป้าหมายคือการสร้างระบบที่:

- **ทดสอบได้** (Testable) - Business rules สามารถทดสอบได้โดยไม่ต้องพึ่ง UI, Database หรือ Framework
- **อิสระจาก Framework** - สามารถเปลี่ยน framework ได้โดยไม่กระทบ business logic
- **อิสระจาก UI** - สามารถเปลี่ยน UI ได้โดยไม่กระทบ business logic
- **อิสระจาก Database** - สามารถเปลี่ยน database ได้ง่าย
- **อิสระจาก External Agencies** - Business rules ไม่รู้จักโลกภายนอก

---

## ชั้น (Layers) ของ Clean Architecture

```
┌─────────────────────────────────────────────┐
│         Frameworks & Drivers                │  ← Web, DB, UI
│  ┌───────────────────────────────────────┐  │
│  │      Interface Adapters               │  │  ← Controllers, Gateways
│  │  ┌─────────────────────────────────┐  │  │
│  │  │       Use Cases                 │  │  │  ← Application Business Rules
│  │  │  ┌───────────────────────────┐  │  │  │
│  │  │  │      Entities             │  │  │  │  ← Enterprise Business Rules
│  │  │  └───────────────────────────┘  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### กฎการพึ่งพา (Dependency Rule)

**"Dependencies ต้องชี้เข้าด้านใน เสมอ"**

- ชั้นนอกสุดรู้จักชั้นในกว่า
- ชั้นในกว่าไม่รู้จักชั้นนอก
- Source code dependencies ชี้เข้าหา higher-level policies

---

## โครงสร้างโฟลเดอร์

```
src/
├── domain/                     # Enterprise Business Rules
│   ├── entities/               # Business Entities
│   │   ├── User.ts
│   │   ├── Order.ts
│   │   └── Product.ts
│   ├── value-objects/          # Value Objects
│   │   ├── Email.ts
│   │   ├── Money.ts
│   │   └── Address.ts
│   └── errors/                 # Domain Errors
│       └── DomainError.ts
│
├── application/                # Application Business Rules
│   ├── use-cases/              # Use Cases
│   │   ├── user/
│   │   │   ├── CreateUser.ts
│   │   │   ├── GetUser.ts
│   │   │   └── UpdateUser.ts
│   │   └── order/
│   │       ├── CreateOrder.ts
│   │       └── CancelOrder.ts
│   ├── interfaces/             # Repository & Service Interfaces
│   │   ├── IUserRepository.ts
│   │   ├── IOrderRepository.ts
│   │   └── IEmailService.ts
│   └── dtos/                   # Data Transfer Objects
│       ├── UserDTO.ts
│       └── OrderDTO.ts
│
├── infrastructure/             # Interface Adapters + Frameworks
│   ├── repositories/           # Repository Implementations
│   │   ├── UserRepository.ts
│   │   └── OrderRepository.ts
│   ├── services/               # External Service Implementations
│   │   └── EmailService.ts
│   ├── database/               # Database setup
│   │   └── connection.ts
│   └── http/                   # HTTP Layer
│       ├── controllers/
│       │   ├── UserController.ts
│       │   └── OrderController.ts
│       ├── middlewares/
│       │   └── auth.ts
│       └── routes/
│           └── index.ts
│
└── main/                       # Composition Root
    ├── app.ts
    └── container.ts
```

---

## Layer 1: Domain Entities

### Value Objects

```typescript
// src/domain/value-objects/Email.ts

export class Email {
  private readonly value: string;

  constructor(email: string) {
    const normalized = email.toLowerCase().trim();
    
    if (!this.isValid(normalized)) {
      throw new Error(`อีเมลไม่ถูกต้อง: ${email}`);
    }
    
    this.value = normalized;
  }

  private isValid(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }

  getValue(): string {
    return this.value;
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}
```

```typescript
// src/domain/value-objects/Money.ts

export class Money {
  private readonly amount: number;
  private readonly currency: string;

  constructor(amount: number, currency: string = "THB") {
    if (amount < 0) {
      throw new Error("จำนวนเงินต้องไม่ติดลบ");
    }
    
    if (!["THB", "USD", "EUR"].includes(currency)) {
      throw new Error(`สกุลเงินไม่รองรับ: ${currency}`);
    }
    
    this.amount = Math.round(amount * 100) / 100; // ปัดเป็น 2 ทศนิยม
    this.currency = currency;
  }

  getAmount(): number {
    return this.amount;
  }

  getCurrency(): string {
    return this.currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("ไม่สามารถบวกเงินต่างสกุลได้");
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  subtract(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("ไม่สามารถลบเงินต่างสกุลได้");
    }
    const result = this.amount - other.amount;
    if (result < 0) {
      throw new Error("ยอดเงินไม่เพียงพอ");
    }
    return new Money(result, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  isGreaterThan(other: Money): boolean {
    if (this.currency !== other.currency) {
      throw new Error("ไม่สามารถเปรียบเทียบเงินต่างสกุลได้");
    }
    return this.amount > other.amount;
  }

  format(): string {
    return new Intl.NumberFormat("th-TH", {
      style: "currency",
      currency: this.currency,
    }).format(this.amount);
  }

  toString(): string {
    return `${this.amount} ${this.currency}`;
  }
}
```

```typescript
// src/domain/value-objects/Address.ts

export interface AddressProps {
  street: string;
  city: string;
  province: string;
  postalCode: string;
  country: string;
}

export class Address {
  private readonly props: AddressProps;

  constructor(props: AddressProps) {
    if (!props.street || props.street.trim().length === 0) {
      throw new Error("ที่อยู่ต้องไม่ว่างเปล่า");
    }
    if (!this.isValidPostalCode(props.postalCode)) {
      throw new Error(`รหัสไปรษณีย์ไม่ถูกต้อง: ${props.postalCode}`);
    }
    
    this.props = {
      street: props.street.trim(),
      city: props.city.trim(),
      province: props.province.trim(),
      postalCode: props.postalCode.trim(),
      country: props.country.trim(),
    };
  }

  private isValidPostalCode(code: string): boolean {
    return /^\d{5}$/.test(code);
  }

  getStreet(): string { return this.props.street; }
  getCity(): string { return this.props.city; }
  getProvince(): string { return this.props.province; }
  getPostalCode(): string { return this.props.postalCode; }
  getCountry(): string { return this.props.country; }

  toString(): string {
    return `${this.props.street}, ${this.props.city}, ${this.props.province} ${this.props.postalCode}, ${this.props.country}`;
  }

  equals(other: Address): boolean {
    return (
      this.props.street === other.props.street &&
      this.props.city === other.props.city &&
      this.props.province === other.props.province &&
      this.props.postalCode === other.props.postalCode
    );
  }
}
```

### Domain Entities

```typescript
// src/domain/entities/User.ts

import { Email } from "../value-objects/Email";
import { Address } from "../value-objects/Address";

export interface UserProps {
  id: string;
  firstName: string;
  lastName: string;
  email: Email;
  phone?: string;
  address?: Address;
  role: "customer" | "admin" | "seller";
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export class User {
  private props: UserProps;

  constructor(props: UserProps) {
    this.validateProps(props);
    this.props = { ...props };
  }

  private validateProps(props: UserProps): void {
    if (!props.firstName || props.firstName.trim().length === 0) {
      throw new Error("ชื่อต้องไม่ว่างเปล่า");
    }
    if (!props.lastName || props.lastName.trim().length === 0) {
      throw new Error("นามสกุลต้องไม่ว่างเปล่า");
    }
    if (props.phone && !/^\+?[\d\s-]{10,15}$/.test(props.phone)) {
      throw new Error("หมายเลขโทรศัพท์ไม่ถูกต้อง");
    }
  }

  // Getters
  getId(): string { return this.props.id; }
  getFirstName(): string { return this.props.firstName; }
  getLastName(): string { return this.props.lastName; }
  getFullName(): string { return `${this.props.firstName} ${this.props.lastName}`; }
  getEmail(): Email { return this.props.email; }
  getPhone(): string | undefined { return this.props.phone; }
  getAddress(): Address | undefined { return this.props.address; }
  getRole(): string { return this.props.role; }
  isUserActive(): boolean { return this.props.isActive; }
  getCreatedAt(): Date { return this.props.createdAt; }

  // Domain Methods
  activate(): void {
    if (this.props.isActive) {
      throw new Error("ผู้ใช้เปิดใช้งานอยู่แล้ว");
    }
    this.props.isActive = true;
    this.props.updatedAt = new Date();
  }

  deactivate(): void {
    if (!this.props.isActive) {
      throw new Error("ผู้ใช้ปิดใช้งานอยู่แล้ว");
    }
    this.props.isActive = false;
    this.props.updatedAt = new Date();
  }

  updateProfile(firstName: string, lastName: string, phone?: string): void {
    if (!firstName || firstName.trim().length === 0) {
      throw new Error("ชื่อต้องไม่ว่างเปล่า");
    }
    this.props.firstName = firstName.trim();
    this.props.lastName = lastName.trim();
    if (phone) this.props.phone = phone;
    this.props.updatedAt = new Date();
  }

  updateAddress(address: Address): void {
    this.props.address = address;
    this.props.updatedAt = new Date();
  }

  promoteToAdmin(): void {
    if (this.props.role !== "customer") {
      throw new Error("สามารถเลื่อนขั้นได้เฉพาะ customer เท่านั้น");
    }
    this.props.role = "admin";
    this.props.updatedAt = new Date();
  }

  canPlaceOrder(): boolean {
    return this.props.isActive && this.props.role !== "admin";
  }

  toJSON(): Record<string, any> {
    return {
      id: this.props.id,
      firstName: this.props.firstName,
      lastName: this.props.lastName,
      email: this.props.email.getValue(),
      phone: this.props.phone,
      role: this.props.role,
      isActive: this.props.isActive,
      createdAt: this.props.createdAt,
      updatedAt: this.props.updatedAt,
    };
  }
}
```

```typescript
// src/domain/entities/Product.ts

import { Money } from "../value-objects/Money";

export interface ProductProps {
  id: string;
  name: string;
  description: string;
  price: Money;
  stock: number;
  category: string;
  sku: string;
  isAvailable: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export class Product {
  private props: ProductProps;

  constructor(props: ProductProps) {
    this.validateProps(props);
    this.props = { ...props };
  }

  private validateProps(props: ProductProps): void {
    if (!props.name || props.name.trim().length === 0) {
      throw new Error("ชื่อสินค้าต้องไม่ว่างเปล่า");
    }
    if (props.stock < 0) {
      throw new Error("จำนวนสินค้าต้องไม่ติดลบ");
    }
    if (!props.sku || props.sku.trim().length === 0) {
      throw new Error("รหัสสินค้าต้องไม่ว่างเปล่า");
    }
  }

  getId(): string { return this.props.id; }
  getName(): string { return this.props.name; }
  getDescription(): string { return this.props.description; }
  getPrice(): Money { return this.props.price; }
  getStock(): number { return this.props.stock; }
  getCategory(): string { return this.props.category; }
  getSku(): string { return this.props.sku; }
  isProductAvailable(): boolean { return this.props.isAvailable && this.props.stock > 0; }

  reserveStock(quantity: number): void {
    if (quantity <= 0) {
      throw new Error("จำนวนต้องมากกว่า 0");
    }
    if (this.props.stock < quantity) {
      throw new Error(`สินค้าไม่เพียงพอ มีเพียง ${this.props.stock} ชิ้น`);
    }
    this.props.stock -= quantity;
    this.props.updatedAt = new Date();
    
    if (this.props.stock === 0) {
      this.props.isAvailable = false;
    }
  }

  restoreStock(quantity: number): void {
    this.props.stock += quantity;
    if (this.props.stock > 0) {
      this.props.isAvailable = true;
    }
    this.props.updatedAt = new Date();
  }

  updatePrice(newPrice: Money): void {
    this.props.price = newPrice;
    this.props.updatedAt = new Date();
  }

  calculateTotalPrice(quantity: number): Money {
    return this.props.price.multiply(quantity);
  }

  toJSON(): Record<string, any> {
    return {
      id: this.props.id,
      name: this.props.name,
      description: this.props.description,
      price: this.props.price.getAmount(),
      currency: this.props.price.getCurrency(),
      stock: this.props.stock,
      category: this.props.category,
      sku: this.props.sku,
      isAvailable: this.props.isAvailable,
    };
  }
}
```

```typescript
// src/domain/entities/Order.ts

import { Money } from "../value-objects/Money";
import { Address } from "../value-objects/Address";

export type OrderStatus =
  | "pending"
  | "confirmed"
  | "paid"
  | "processing"
  | "shipped"
  | "delivered"
  | "cancelled"
  | "refunded";

export interface OrderItem {
  productId: string;
  productName: string;
  quantity: number;
  unitPrice: Money;
  subtotal: Money;
}

export interface OrderProps {
  id: string;
  userId: string;
  items: OrderItem[];
  subtotal: Money;
  shippingCost: Money;
  discount: Money;
  total: Money;
  status: OrderStatus;
  shippingAddress: Address;
  notes?: string;
  createdAt: Date;
  updatedAt: Date;
}

export class Order {
  private props: OrderProps;
  private statusHistory: Array<{ status: OrderStatus; changedAt: Date; reason?: string }> = [];

  constructor(props: OrderProps) {
    this.props = { ...props };
    this.statusHistory.push({ status: props.status, changedAt: props.createdAt });
  }

  static create(
    id: string,
    userId: string,
    items: OrderItem[],
    shippingAddress: Address,
    shippingCost: Money,
    discount: Money,
    notes?: string
  ): Order {
    if (items.length === 0) {
      throw new Error("คำสั่งซื้อต้องมีสินค้าอย่างน้อย 1 รายการ");
    }

    const subtotal = items.reduce(
      (sum, item) => sum.add(item.subtotal),
      new Money(0)
    );
    
    const total = subtotal.add(shippingCost).subtract(discount);

    return new Order({
      id,
      userId,
      items,
      subtotal,
      shippingCost,
      discount,
      total,
      status: "pending",
      shippingAddress,
      notes,
      createdAt: new Date(),
      updatedAt: new Date(),
    });
  }

  getId(): string { return this.props.id; }
  getUserId(): string { return this.props.userId; }
  getItems(): OrderItem[] { return [...this.props.items]; }
  getTotal(): Money { return this.props.total; }
  getSubtotal(): Money { return this.props.subtotal; }
  getShippingCost(): Money { return this.props.shippingCost; }
  getDiscount(): Money { return this.props.discount; }
  getStatus(): OrderStatus { return this.props.status; }
  getShippingAddress(): Address { return this.props.shippingAddress; }
  getCreatedAt(): Date { return this.props.createdAt; }
  getNotes(): string | undefined { return this.props.notes; }

  private changeStatus(newStatus: OrderStatus, reason?: string): void {
    this.props.status = newStatus;
    this.props.updatedAt = new Date();
    this.statusHistory.push({ status: newStatus, changedAt: new Date(), reason });
  }

  confirm(): void {
    if (this.props.status !== "pending") {
      throw new Error(`ไม่สามารถยืนยันคำสั่งซื้อสถานะ ${this.props.status}`);
    }
    this.changeStatus("confirmed");
  }

  markAsPaid(): void {
    if (this.props.status !== "confirmed") {
      throw new Error("ต้องยืนยันก่อนชำระเงิน");
    }
    this.changeStatus("paid");
  }

  startProcessing(): void {
    if (this.props.status !== "paid") {
      throw new Error("ต้องชำระเงินก่อนดำเนินการ");
    }
    this.changeStatus("processing");
  }

  ship(trackingNumber: string): void {
    if (this.props.status !== "processing") {
      throw new Error("ต้องอยู่ในสถานะดำเนินการก่อนจัดส่ง");
    }
    this.changeStatus("shipped", `Tracking: ${trackingNumber}`);
  }

  deliver(): void {
    if (this.props.status !== "shipped") {
      throw new Error("ต้องจัดส่งก่อนส่งมอบ");
    }
    this.changeStatus("delivered");
  }

  cancel(reason: string): void {
    const cancellableStatuses: OrderStatus[] = ["pending", "confirmed", "paid"];
    
    if (!cancellableStatuses.includes(this.props.status)) {
      throw new Error(`ไม่สามารถยกเลิกคำสั่งซื้อสถานะ ${this.props.status}`);
    }
    
    this.changeStatus("cancelled", reason);
  }

  canBeCancelled(): boolean {
    return ["pending", "confirmed", "paid"].includes(this.props.status);
  }

  getStatusHistory() {
    return [...this.statusHistory];
  }

  toJSON(): Record<string, any> {
    return {
      id: this.props.id,
      userId: this.props.userId,
      items: this.props.items.map((item) => ({
        ...item,
        unitPrice: item.unitPrice.getAmount(),
        subtotal: item.subtotal.getAmount(),
      })),
      subtotal: this.props.subtotal.getAmount(),
      shippingCost: this.props.shippingCost.getAmount(),
      discount: this.props.discount.getAmount(),
      total: this.props.total.getAmount(),
      status: this.props.status,
      shippingAddress: this.props.shippingAddress.toString(),
      createdAt: this.props.createdAt,
    };
  }
}
```

---

## Layer 2: Application Use Cases

### Repository Interfaces

```typescript
// src/application/interfaces/IUserRepository.ts

import { User } from "../../domain/entities/User";
import { Email } from "../../domain/value-objects/Email";

export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: Email): Promise<User | null>;
  findAll(options?: {
    page?: number;
    limit?: number;
    role?: string;
    isActive?: boolean;
  }): Promise<{ users: User[]; total: number }>;
  save(user: User): Promise<void>;
  update(user: User): Promise<void>;
  delete(id: string): Promise<void>;
  exists(email: Email): Promise<boolean>;
}
```

```typescript
// src/application/interfaces/IOrderRepository.ts

import { Order, OrderStatus } from "../../domain/entities/Order";

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  findByUserId(userId: string, options?: {
    status?: OrderStatus;
    page?: number;
    limit?: number;
  }): Promise<{ orders: Order[]; total: number }>;
  save(order: Order): Promise<void>;
  update(order: Order): Promise<void>;
  findAll(options?: {
    status?: OrderStatus;
    from?: Date;
    to?: Date;
    page?: number;
    limit?: number;
  }): Promise<{ orders: Order[]; total: number }>;
}
```

```typescript
// src/application/interfaces/IProductRepository.ts

import { Product } from "../../domain/entities/Product";

export interface IProductRepository {
  findById(id: string): Promise<Product | null>;
  findBySku(sku: string): Promise<Product | null>;
  findAll(options?: {
    category?: string;
    available?: boolean;
    page?: number;
    limit?: number;
  }): Promise<{ products: Product[]; total: number }>;
  save(product: Product): Promise<void>;
  update(product: Product): Promise<void>;
  delete(id: string): Promise<void>;
}
```

```typescript
// src/application/interfaces/IEmailService.ts

export interface EmailOptions {
  to: string;
  subject: string;
  htmlBody: string;
  textBody?: string;
}

export interface IEmailService {
  send(options: EmailOptions): Promise<void>;
  sendOrderConfirmation(email: string, orderId: string, total: number): Promise<void>;
  sendPasswordReset(email: string, resetToken: string): Promise<void>;
  sendWelcome(email: string, firstName: string): Promise<void>;
}
```

### DTOs (Data Transfer Objects)

```typescript
// src/application/dtos/UserDTO.ts

export interface CreateUserDTO {
  firstName: string;
  lastName: string;
  email: string;
  password: string;
  phone?: string;
}

export interface UpdateUserDTO {
  firstName?: string;
  lastName?: string;
  phone?: string;
}

export interface UserResponseDTO {
  id: string;
  firstName: string;
  lastName: string;
  fullName: string;
  email: string;
  phone?: string;
  role: string;
  isActive: boolean;
  createdAt: Date;
}
```

```typescript
// src/application/dtos/OrderDTO.ts

export interface CreateOrderDTO {
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
  shippingAddress: {
    street: string;
    city: string;
    province: string;
    postalCode: string;
    country: string;
  };
  notes?: string;
  couponCode?: string;
}

export interface OrderResponseDTO {
  id: string;
  userId: string;
  items: Array<{
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    subtotal: number;
  }>;
  subtotal: number;
  shippingCost: number;
  discount: number;
  total: number;
  status: string;
  shippingAddress: string;
  createdAt: Date;
}
```

### Use Cases

```typescript
// src/application/use-cases/user/CreateUser.ts

import { v4 as uuidv4 } from "uuid";
import { User } from "../../../domain/entities/User";
import { Email } from "../../../domain/value-objects/Email";
import { IUserRepository } from "../../interfaces/IUserRepository";
import { IEmailService } from "../../interfaces/IEmailService";
import { IPasswordHasher } from "../../interfaces/IPasswordHasher";
import { CreateUserDTO, UserResponseDTO } from "../../dtos/UserDTO";

export class CreateUserUseCase {
  constructor(
    private userRepository: IUserRepository,
    private emailService: IEmailService,
    private passwordHasher: IPasswordHasher
  ) {}

  async execute(dto: CreateUserDTO): Promise<UserResponseDTO> {
    // 1. Validate email format (via Value Object)
    const email = new Email(dto.email);

    // 2. Check if user already exists
    const exists = await this.userRepository.exists(email);
    if (exists) {
      throw new Error("อีเมลนี้ถูกใช้งานแล้ว");
    }

    // 3. Hash password
    const hashedPassword = await this.passwordHasher.hash(dto.password);

    // 4. Create User entity
    const user = new User({
      id: uuidv4(),
      firstName: dto.firstName,
      lastName: dto.lastName,
      email,
      phone: dto.phone,
      role: "customer",
      isActive: true,
      createdAt: new Date(),
      updatedAt: new Date(),
    });

    // 5. Save to repository
    await this.userRepository.save(user);

    // 6. Send welcome email
    await this.emailService.sendWelcome(
      email.getValue(),
      user.getFirstName()
    );

    // 7. Return DTO
    return this.toDTO(user);
  }

  private toDTO(user: User): UserResponseDTO {
    return {
      id: user.getId(),
      firstName: user.getFirstName(),
      lastName: user.getLastName(),
      fullName: user.getFullName(),
      email: user.getEmail().getValue(),
      phone: user.getPhone(),
      role: user.getRole(),
      isActive: user.isUserActive(),
      createdAt: user.getCreatedAt(),
    };
  }
}
```

```typescript
// src/application/use-cases/order/CreateOrder.ts

import { v4 as uuidv4 } from "uuid";
import { Order, OrderItem } from "../../../domain/entities/Order";
import { Money } from "../../../domain/value-objects/Money";
import { Address } from "../../../domain/value-objects/Address";
import { IOrderRepository } from "../../interfaces/IOrderRepository";
import { IUserRepository } from "../../interfaces/IUserRepository";
import { IProductRepository } from "../../interfaces/IProductRepository";
import { IEmailService } from "../../interfaces/IEmailService";
import { CreateOrderDTO, OrderResponseDTO } from "../../dtos/OrderDTO";

export class CreateOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository,
    private userRepository: IUserRepository,
    private productRepository: IProductRepository,
    private emailService: IEmailService
  ) {}

  async execute(dto: CreateOrderDTO): Promise<OrderResponseDTO> {
    // 1. Validate user
    const user = await this.userRepository.findById(dto.userId);
    if (!user) {
      throw new Error("ไม่พบผู้ใช้");
    }
    if (!user.canPlaceOrder()) {
      throw new Error("ผู้ใช้นี้ไม่สามารถสั่งซื้อได้");
    }

    // 2. Validate products and calculate totals
    const orderItems: OrderItem[] = [];
    
    for (const itemDto of dto.items) {
      const product = await this.productRepository.findById(itemDto.productId);
      
      if (!product) {
        throw new Error(`ไม่พบสินค้า: ${itemDto.productId}`);
      }
      
      if (!product.isProductAvailable()) {
        throw new Error(`สินค้า "${product.getName()}" ไม่พร้อมจำหน่าย`);
      }
      
      if (product.getStock() < itemDto.quantity) {
        throw new Error(
          `สินค้า "${product.getName()}" มีไม่เพียงพอ ` +
          `(เหลือ ${product.getStock()} ชิ้น)`
        );
      }

      const unitPrice = product.getPrice();
      const subtotal = unitPrice.multiply(itemDto.quantity);

      orderItems.push({
        productId: product.getId(),
        productName: product.getName(),
        quantity: itemDto.quantity,
        unitPrice,
        subtotal,
      });
    }

    // 3. Create shipping address
    const shippingAddress = new Address(dto.shippingAddress);

    // 4. Calculate shipping cost
    const shippingCost = new Money(50); // simplified

    // 5. Calculate discount
    const discount = new Money(0); // simplified

    // 6. Create Order entity
    const order = Order.create(
      uuidv4(),
      dto.userId,
      orderItems,
      shippingAddress,
      shippingCost,
      discount,
      dto.notes
    );

    // 7. Reserve product stock
    for (const itemDto of dto.items) {
      const product = await this.productRepository.findById(itemDto.productId);
      product!.reserveStock(itemDto.quantity);
      await this.productRepository.update(product!);
    }

    // 8. Confirm order
    order.confirm();

    // 9. Save order
    await this.orderRepository.save(order);

    // 10. Send confirmation email
    await this.emailService.sendOrderConfirmation(
      user.getEmail().getValue(),
      order.getId(),
      order.getTotal().getAmount()
    );

    // 11. Return DTO
    return this.toDTO(order);
  }

  private toDTO(order: Order): OrderResponseDTO {
    return {
      id: order.getId(),
      userId: order.getUserId(),
      items: order.getItems().map((item) => ({
        productId: item.productId,
        productName: item.productName,
        quantity: item.quantity,
        unitPrice: item.unitPrice.getAmount(),
        subtotal: item.subtotal.getAmount(),
      })),
      subtotal: order.getSubtotal().getAmount(),
      shippingCost: order.getShippingCost().getAmount(),
      discount: order.getDiscount().getAmount(),
      total: order.getTotal().getAmount(),
      status: order.getStatus(),
      shippingAddress: order.getShippingAddress().toString(),
      createdAt: order.getCreatedAt(),
    };
  }
}
```

```typescript
// src/application/use-cases/order/CancelOrder.ts

import { IOrderRepository } from "../../interfaces/IOrderRepository";
import { IProductRepository } from "../../interfaces/IProductRepository";
import { IEmailService } from "../../interfaces/IEmailService";
import { IUserRepository } from "../../interfaces/IUserRepository";

export interface CancelOrderDTO {
  orderId: string;
  userId: string;
  reason: string;
}

export class CancelOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository,
    private productRepository: IProductRepository,
    private userRepository: IUserRepository,
    private emailService: IEmailService
  ) {}

  async execute(dto: CancelOrderDTO): Promise<void> {
    // 1. Find order
    const order = await this.orderRepository.findById(dto.orderId);
    if (!order) {
      throw new Error("ไม่พบคำสั่งซื้อ");
    }

    // 2. Verify ownership
    if (order.getUserId() !== dto.userId) {
      throw new Error("ไม่มีสิทธิ์ยกเลิกคำสั่งซื้อนี้");
    }

    // 3. Check if can be cancelled
    if (!order.canBeCancelled()) {
      throw new Error(`ไม่สามารถยกเลิกคำสั่งซื้อสถานะ ${order.getStatus()}`);
    }

    // 4. Cancel order
    order.cancel(dto.reason);

    // 5. Restore product stock
    for (const item of order.getItems()) {
      const product = await this.productRepository.findById(item.productId);
      if (product) {
        product.restoreStock(item.quantity);
        await this.productRepository.update(product);
      }
    }

    // 6. Save updated order
    await this.orderRepository.update(order);

    // 7. Notify user
    const user = await this.userRepository.findById(dto.userId);
    if (user) {
      await this.emailService.send({
        to: user.getEmail().getValue(),
        subject: "ยกเลิกคำสั่งซื้อแล้ว",
        htmlBody: `<p>คำสั่งซื้อ #${order.getId()} ถูกยกเลิกแล้ว<br>เหตุผล: ${dto.reason}</p>`,
      });
    }
  }
}
```

---

## Layer 3: Infrastructure (Interface Adapters)

### Repository Implementations

```typescript
// src/infrastructure/repositories/UserRepository.ts

import { User, UserProps } from "../../domain/entities/User";
import { Email } from "../../domain/value-objects/Email";
import { Address } from "../../domain/value-objects/Address";
import { IUserRepository } from "../../application/interfaces/IUserRepository";

// จำลอง Database Model
interface UserModel {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  phone?: string;
  street?: string;
  city?: string;
  province?: string;
  postalCode?: string;
  country?: string;
  role: string;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

// จำลอง Database
class Database {
  private static users: Map<string, UserModel> = new Map();

  static async findOne(id: string): Promise<UserModel | null> {
    return Database.users.get(id) || null;
  }

  static async findByEmail(email: string): Promise<UserModel | null> {
    for (const user of Database.users.values()) {
      if (user.email === email) return user;
    }
    return null;
  }

  static async findAll(options: any): Promise<{ data: UserModel[]; total: number }> {
    let users = [...Database.users.values()];
    if (options.role) users = users.filter((u) => u.role === options.role);
    if (options.isActive !== undefined) users = users.filter((u) => u.isActive === options.isActive);
    const total = users.length;
    const start = ((options.page || 1) - 1) * (options.limit || 10);
    return { data: users.slice(start, start + (options.limit || 10)), total };
  }

  static async save(user: UserModel): Promise<void> {
    Database.users.set(user.id, user);
  }

  static async delete(id: string): Promise<void> {
    Database.users.delete(id);
  }
}

// Mapper
class UserMapper {
  static toDomain(model: UserModel): User {
    const props: UserProps = {
      id: model.id,
      firstName: model.firstName,
      lastName: model.lastName,
      email: new Email(model.email),
      phone: model.phone,
      address: model.street
        ? new Address({
            street: model.street!,
            city: model.city!,
            province: model.province!,
            postalCode: model.postalCode!,
            country: model.country!,
          })
        : undefined,
      role: model.role as any,
      isActive: model.isActive,
      createdAt: model.createdAt,
      updatedAt: model.updatedAt,
    };
    return new User(props);
  }

  static toModel(user: User): UserModel {
    const address = user.getAddress();
    return {
      id: user.getId(),
      firstName: user.getFirstName(),
      lastName: user.getLastName(),
      email: user.getEmail().getValue(),
      phone: user.getPhone(),
      street: address?.getStreet(),
      city: address?.getCity(),
      province: address?.getProvince(),
      postalCode: address?.getPostalCode(),
      country: address?.getCountry(),
      role: user.getRole(),
      isActive: user.isUserActive(),
      createdAt: user.getCreatedAt(),
      updatedAt: new Date(),
    };
  }
}

// Repository Implementation
export class UserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    const model = await Database.findOne(id);
    if (!model) return null;
    return UserMapper.toDomain(model);
  }

  async findByEmail(email: Email): Promise<User | null> {
    const model = await Database.findByEmail(email.getValue());
    if (!model) return null;
    return UserMapper.toDomain(model);
  }

  async findAll(options?: any): Promise<{ users: User[]; total: number }> {
    const result = await Database.findAll(options || {});
    return {
      users: result.data.map(UserMapper.toDomain),
      total: result.total,
    };
  }

  async save(user: User): Promise<void> {
    await Database.save(UserMapper.toModel(user));
  }

  async update(user: User): Promise<void> {
    const exists = await Database.findOne(user.getId());
    if (!exists) throw new Error("ไม่พบผู้ใช้");
    await Database.save(UserMapper.toModel(user));
  }

  async delete(id: string): Promise<void> {
    await Database.delete(id);
  }

  async exists(email: Email): Promise<boolean> {
    const model = await Database.findByEmail(email.getValue());
    return model !== null;
  }
}
```

### HTTP Controller

```typescript
// src/infrastructure/http/controllers/UserController.ts

import { CreateUserUseCase } from "../../../application/use-cases/user/CreateUser";
import { GetUserUseCase } from "../../../application/use-cases/user/GetUser";
import { UpdateUserUseCase } from "../../../application/use-cases/user/UpdateUser";

// จำลอง HTTP types
interface Request {
  params: Record<string, string>;
  body: any;
  query: Record<string, string>;
  user?: { id: string };
}

interface Response {
  status(code: number): Response;
  json(data: any): void;
}

export class UserController {
  constructor(
    private createUserUseCase: CreateUserUseCase,
    private getUserUseCase: GetUserUseCase,
    private updateUserUseCase: UpdateUserUseCase
  ) {}

  async createUser(req: Request, res: Response): Promise<void> {
    try {
      const dto = {
        firstName: req.body.firstName,
        lastName: req.body.lastName,
        email: req.body.email,
        password: req.body.password,
        phone: req.body.phone,
      };

      const user = await this.createUserUseCase.execute(dto);
      
      res.status(201).json({
        success: true,
        data: user,
        message: "สร้างผู้ใช้สำเร็จ",
      });
    } catch (error) {
      if ((error as Error).message.includes("อีเมลนี้ถูกใช้งานแล้ว")) {
        res.status(409).json({
          success: false,
          error: (error as Error).message,
        });
        return;
      }
      
      res.status(400).json({
        success: false,
        error: (error as Error).message,
      });
    }
  }

  async getUser(req: Request, res: Response): Promise<void> {
    try {
      const { id } = req.params;
      const user = await this.getUserUseCase.execute(id);
      
      if (!user) {
        res.status(404).json({
          success: false,
          error: "ไม่พบผู้ใช้",
        });
        return;
      }
      
      res.status(200).json({
        success: true,
        data: user,
      });
    } catch (error) {
      res.status(500).json({
        success: false,
        error: "เกิดข้อผิดพลาดภายในระบบ",
      });
    }
  }

  async updateUser(req: Request, res: Response): Promise<void> {
    try {
      const { id } = req.params;
      const currentUserId = req.user?.id;

      if (currentUserId !== id) {
        res.status(403).json({
          success: false,
          error: "ไม่มีสิทธิ์แก้ไขข้อมูลผู้ใช้นี้",
        });
        return;
      }

      const updatedUser = await this.updateUserUseCase.execute(id, req.body);
      
      res.status(200).json({
        success: true,
        data: updatedUser,
        message: "อัพเดตข้อมูลสำเร็จ",
      });
    } catch (error) {
      res.status(400).json({
        success: false,
        error: (error as Error).message,
      });
    }
  }
}
```

---

## CQRS Pattern Integration

### Command / Query Separation

```typescript
// CQRS สำหรับ Order Management

// Commands
interface ICommand {}
interface ICommandHandler<TCommand extends ICommand, TResult = void> {
  handle(command: TCommand): Promise<TResult>;
}

// Queries
interface IQuery<TResult> {
  __resultType?: TResult;
}
interface IQueryHandler<TQuery extends IQuery<TResult>, TResult> {
  handle(query: TQuery): Promise<TResult>;
}

// Create Order Command
class CreateOrderCommand implements ICommand {
  constructor(
    public readonly userId: string,
    public readonly items: Array<{ productId: string; quantity: number }>,
    public readonly shippingAddress: any,
    public readonly notes?: string
  ) {}
}

class CreateOrderCommandHandler
  implements ICommandHandler<CreateOrderCommand, string>
{
  constructor(private createOrderUseCase: CreateOrderUseCase) {}

  async handle(command: CreateOrderCommand): Promise<string> {
    const result = await this.createOrderUseCase.execute({
      userId: command.userId,
      items: command.items,
      shippingAddress: command.shippingAddress,
      notes: command.notes,
    });
    return result.id;
  }
}

// Get Orders Query
interface OrderSummary {
  id: string;
  total: number;
  status: string;
  createdAt: Date;
  itemCount: number;
}

class GetUserOrdersQuery implements IQuery<OrderSummary[]> {
  constructor(
    public readonly userId: string,
    public readonly page: number = 1,
    public readonly limit: number = 10
  ) {}
}

class GetUserOrdersQueryHandler
  implements IQueryHandler<GetUserOrdersQuery, OrderSummary[]>
{
  constructor(private orderRepository: IOrderRepository) {}

  async handle(query: GetUserOrdersQuery): Promise<OrderSummary[]> {
    const { orders } = await this.orderRepository.findByUserId(query.userId, {
      page: query.page,
      limit: query.limit,
    });

    return orders.map((order) => ({
      id: order.getId(),
      total: order.getTotal().getAmount(),
      status: order.getStatus(),
      createdAt: order.getCreatedAt(),
      itemCount: order.getItems().length,
    }));
  }
}

// Command Bus
class CommandBus {
  private handlers: Map<string, ICommandHandler<any, any>> = new Map();

  register<TCommand extends ICommand>(
    commandClass: new (...args: any[]) => TCommand,
    handler: ICommandHandler<TCommand, any>
  ): void {
    this.handlers.set(commandClass.name, handler);
  }

  async dispatch<TResult>(command: ICommand): Promise<TResult> {
    const handler = this.handlers.get(command.constructor.name);
    
    if (!handler) {
      throw new Error(`ไม่พบ handler สำหรับ ${command.constructor.name}`);
    }
    
    return handler.handle(command);
  }
}

// Query Bus
class QueryBus {
  private handlers: Map<string, IQueryHandler<any, any>> = new Map();

  register<TQuery extends IQuery<TResult>, TResult>(
    queryClass: new (...args: any[]) => TQuery,
    handler: IQueryHandler<TQuery, TResult>
  ): void {
    this.handlers.set(queryClass.name, handler);
  }

  async dispatch<TResult>(query: IQuery<TResult>): Promise<TResult> {
    const handler = this.handlers.get(query.constructor.name);
    
    if (!handler) {
      throw new Error(`ไม่พบ handler สำหรับ ${query.constructor.name}`);
    }
    
    return handler.handle(query);
  }
}
```

---

## Testing Clean Architecture

### Unit Tests สำหรับ Domain Entities

```typescript
// tests/domain/entities/Order.test.ts

import { Order } from "../../../src/domain/entities/Order";
import { Money } from "../../../src/domain/value-objects/Money";
import { Address } from "../../../src/domain/value-objects/Address";

describe("Order Entity", () => {
  const createValidOrderItems = () => [
    {
      productId: "prod-1",
      productName: "MacBook Pro",
      quantity: 1,
      unitPrice: new Money(89000),
      subtotal: new Money(89000),
    },
  ];

  const createValidAddress = () =>
    new Address({
      street: "123 ถนนสุขุมวิท",
      city: "กรุงเทพ",
      province: "กรุงเทพมหานคร",
      postalCode: "10110",
      country: "Thailand",
    });

  describe("Order.create()", () => {
    it("ควรสร้าง Order ได้สำเร็จ", () => {
      const order = Order.create(
        "order-1",
        "user-1",
        createValidOrderItems(),
        createValidAddress(),
        new Money(50),
        new Money(0)
      );

      expect(order.getId()).toBe("order-1");
      expect(order.getStatus()).toBe("pending");
      expect(order.getTotal().getAmount()).toBe(89050);
    });

    it("ควร throw error เมื่อ items ว่าง", () => {
      expect(() =>
        Order.create(
          "order-1",
          "user-1",
          [],
          createValidAddress(),
          new Money(50),
          new Money(0)
        )
      ).toThrow("คำสั่งซื้อต้องมีสินค้าอย่างน้อย 1 รายการ");
    });
  });

  describe("Order.confirm()", () => {
    it("ควร confirm Order ที่ status เป็น pending ได้", () => {
      const order = Order.create(
        "order-1",
        "user-1",
        createValidOrderItems(),
        createValidAddress(),
        new Money(50),
        new Money(0)
      );

      order.confirm();
      expect(order.getStatus()).toBe("confirmed");
    });

    it("ควร throw error เมื่อ confirm Order ที่ไม่ใช่ pending", () => {
      const order = Order.create(
        "order-1",
        "user-1",
        createValidOrderItems(),
        createValidAddress(),
        new Money(50),
        new Money(0)
      );

      order.confirm();
      expect(() => order.confirm()).toThrow();
    });
  });

  describe("Order.cancel()", () => {
    it("ควรยกเลิก Order ที่ pending ได้", () => {
      const order = Order.create(
        "order-1",
        "user-1",
        createValidOrderItems(),
        createValidAddress(),
        new Money(50),
        new Money(0)
      );

      order.cancel("เปลี่ยนใจ");
      expect(order.getStatus()).toBe("cancelled");
    });

    it("ควร throw error เมื่อยกเลิก Order ที่ delivered", () => {
      const order = Order.create(
        "order-1",
        "user-1",
        createValidOrderItems(),
        createValidAddress(),
        new Money(50),
        new Money(0)
      );

      order.confirm();
      order.markAsPaid();
      order.startProcessing();
      order.ship("TRK-001");
      order.deliver();

      expect(() => order.cancel("ต้องการคืน")).toThrow();
    });
  });
});
```

### Unit Tests สำหรับ Use Cases

```typescript
// tests/application/use-cases/CreateUser.test.ts

import { CreateUserUseCase } from "../../../src/application/use-cases/user/CreateUser";
import { IUserRepository } from "../../../src/application/interfaces/IUserRepository";
import { IEmailService } from "../../../src/application/interfaces/IEmailService";
import { IPasswordHasher } from "../../../src/application/interfaces/IPasswordHasher";

// Mock implementations
const createMockUserRepository = (): jest.Mocked<IUserRepository> => ({
  findById: jest.fn(),
  findByEmail: jest.fn(),
  findAll: jest.fn(),
  save: jest.fn(),
  update: jest.fn(),
  delete: jest.fn(),
  exists: jest.fn(),
});

const createMockEmailService = (): jest.Mocked<IEmailService> => ({
  send: jest.fn(),
  sendOrderConfirmation: jest.fn(),
  sendPasswordReset: jest.fn(),
  sendWelcome: jest.fn(),
});

const createMockPasswordHasher = (): jest.Mocked<IPasswordHasher> => ({
  hash: jest.fn(),
  compare: jest.fn(),
});

describe("CreateUserUseCase", () => {
  let useCase: CreateUserUseCase;
  let userRepository: jest.Mocked<IUserRepository>;
  let emailService: jest.Mocked<IEmailService>;
  let passwordHasher: jest.Mocked<IPasswordHasher>;

  beforeEach(() => {
    userRepository = createMockUserRepository();
    emailService = createMockEmailService();
    passwordHasher = createMockPasswordHasher();

    useCase = new CreateUserUseCase(
      userRepository,
      emailService,
      passwordHasher
    );
  });

  it("ควรสร้าง User ใหม่ได้สำเร็จ", async () => {
    userRepository.exists.mockResolvedValue(false);
    passwordHasher.hash.mockResolvedValue("hashed_password");
    userRepository.save.mockResolvedValue(undefined);
    emailService.sendWelcome.mockResolvedValue(undefined);

    const result = await useCase.execute({
      firstName: "สมชาย",
      lastName: "ใจดี",
      email: "somchai@example.com",
      password: "password123",
    });

    expect(result.firstName).toBe("สมชาย");
    expect(result.email).toBe("somchai@example.com");
    expect(result.role).toBe("customer");
    expect(result.isActive).toBe(true);
    expect(userRepository.save).toHaveBeenCalledTimes(1);
    expect(emailService.sendWelcome).toHaveBeenCalledWith(
      "somchai@example.com",
      "สมชาย"
    );
  });

  it("ควร throw error เมื่ออีเมลซ้ำ", async () => {
    userRepository.exists.mockResolvedValue(true);

    await expect(
      useCase.execute({
        firstName: "สมชาย",
        lastName: "ใจดี",
        email: "existing@example.com",
        password: "password123",
      })
    ).rejects.toThrow("อีเมลนี้ถูกใช้งานแล้ว");

    expect(userRepository.save).not.toHaveBeenCalled();
  });

  it("ควร throw error เมื่ออีเมลไม่ถูกต้อง", async () => {
    await expect(
      useCase.execute({
        firstName: "สมชาย",
        lastName: "ใจดี",
        email: "invalid-email",
        password: "password123",
      })
    ).rejects.toThrow();
  });
});
```

---

## ตัวอย่างแอปพลิเคชันสมบูรณ์

### Composition Root

```typescript
// src/main/container.ts - Dependency Injection Setup

import { UserRepository } from "../infrastructure/repositories/UserRepository";
import { OrderRepository } from "../infrastructure/repositories/OrderRepository";
import { ProductRepository } from "../infrastructure/repositories/ProductRepository";
import { EmailService } from "../infrastructure/services/EmailService";
import { PasswordHasher } from "../infrastructure/services/PasswordHasher";
import { CreateUserUseCase } from "../application/use-cases/user/CreateUser";
import { CreateOrderUseCase } from "../application/use-cases/order/CreateOrder";
import { CancelOrderUseCase } from "../application/use-cases/order/CancelOrder";
import { UserController } from "../infrastructure/http/controllers/UserController";
import { OrderController } from "../infrastructure/http/controllers/OrderController";

export class Container {
  // Repositories
  private userRepository = new UserRepository();
  private orderRepository = new OrderRepository();
  private productRepository = new ProductRepository();

  // Services
  private emailService = new EmailService();
  private passwordHasher = new PasswordHasher();

  // Use Cases
  private createUserUseCase = new CreateUserUseCase(
    this.userRepository,
    this.emailService,
    this.passwordHasher
  );

  private createOrderUseCase = new CreateOrderUseCase(
    this.orderRepository,
    this.userRepository,
    this.productRepository,
    this.emailService
  );

  private cancelOrderUseCase = new CancelOrderUseCase(
    this.orderRepository,
    this.productRepository,
    this.userRepository,
    this.emailService
  );

  // Controllers
  getUserController(): UserController {
    return new UserController(
      this.createUserUseCase,
      // ... other use cases
    );
  }

  getOrderController(): OrderController {
    return new OrderController(
      this.createOrderUseCase,
      this.cancelOrderUseCase,
      // ... other use cases
    );
  }
}
```

### Application Setup

```typescript
// src/main/app.ts

import express from "express";
import { Container } from "./container";

const container = new Container();
const app = express();

app.use(express.json());

// Routes
const userController = container.getUserController();
const orderController = container.getOrderController();

app.post("/api/users", (req, res) => userController.createUser(req as any, res as any));
app.get("/api/users/:id", (req, res) => userController.getUser(req as any, res as any));
app.put("/api/users/:id", (req, res) => userController.updateUser(req as any, res as any));

app.post("/api/orders", (req, res) => orderController.createOrder(req as any, res as any));
app.delete("/api/orders/:id", (req, res) => orderController.cancelOrder(req as any, res as any));

// Error handling
app.use((err: Error, req: any, res: any, next: any) => {
  console.error(err.stack);
  res.status(500).json({ error: "เกิดข้อผิดพลาดภายในระบบ" });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 Server กำลังทำงานที่พอร์ต ${PORT}`);
});

export default app;
```

---

## สรุปหลักการ Clean Architecture

### ข้อดีของ Clean Architecture

1. **Testability** - ทดสอบ business logic ได้โดยไม่ต้องพึ่ง database หรือ HTTP
2. **Maintainability** - เปลี่ยน framework หรือ database ได้ง่าย
3. **Scalability** - เพิ่ม features ใหม่ได้โดยไม่กระทบส่วนอื่น
4. **Independence** - แต่ละ layer ทำงานได้อิสระ

### กฎสำคัญ

```
1. Domain Entities ไม่รู้จัก Layer อื่น
2. Use Cases รู้จักแค่ Domain Entities และ Repository Interfaces
3. Controllers รู้จัก Use Cases เท่านั้น
4. Repository Implementations อยู่ใน Infrastructure Layer
5. Dependency Injection ทำใน Composition Root (main/)
```

### เมื่อไหรควรใช้ Clean Architecture

- โปรเจกต์ขนาดกลาง-ใหญ่ที่ต้องการ maintainability สูง
- ระบบที่มีโอกาสเปลี่ยน technology stack
- ระบบที่ต้องการ test coverage สูง
- ทีมที่มีหลายคนทำงานร่วมกัน

## แบบฝึกหัด

1. เพิ่ม domain entity `Category` พร้อม business rules
2. สร้าง use case `ApplyDiscount` ที่มี validation ใน domain
3. เพิ่ม `AuditLog` repository interface และ implementation
4. สร้าง integration test สำหรับ CreateOrder flow ทั้งหมด
5. เพิ่ม CQRS สำหรับ Product management
6. สร้าง Event system สำหรับ domain events
