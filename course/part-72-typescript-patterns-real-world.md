# ส่วนที่ 72: รูปแบบการออกแบบ TypeScript ในโลกจริง

## บทนำ

รูปแบบการออกแบบ (Design Patterns) ช่วยให้นักพัฒนาสร้างโค้ดที่มีคุณภาพสูง บำรุงรักษาง่าย และขยายได้ ในบทนี้เราจะเรียนรู้รูปแบบที่ใช้บ่อยในการพัฒนาซอฟต์แวร์ระดับ Enterprise ด้วย TypeScript

---

## 1. Repository Pattern

Repository Pattern แยกตรรกะในการเข้าถึงข้อมูลออกจาก Business Logic

### 1.1 Interface และ Implementation

```typescript
// Entity
interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}

// Repository Interface
interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(options?: FindOptions): Promise<User[]>;
  create(data: CreateUserDTO): Promise<User>;
  update(id: string, data: UpdateUserDTO): Promise<User | null>;
  delete(id: string): Promise<boolean>;
  count(filter?: Partial<User>): Promise<number>;
}

interface FindOptions {
  limit?: number;
  offset?: number;
  orderBy?: keyof User;
  order?: 'asc' | 'desc';
}

interface CreateUserDTO {
  name: string;
  email: string;
}

interface UpdateUserDTO {
  name?: string;
  email?: string;
}

// In-Memory Implementation สำหรับ Testing
class InMemoryUserRepository implements IUserRepository {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.users.values()) {
      if (user.email === email) return user;
    }
    return null;
  }

  async findAll(options?: FindOptions): Promise<User[]> {
    let users = Array.from(this.users.values());

    if (options?.orderBy) {
      users.sort((a, b) => {
        const aVal = a[options.orderBy!];
        const bVal = b[options.orderBy!];
        const cmp = String(aVal).localeCompare(String(bVal));
        return options.order === 'desc' ? -cmp : cmp;
      });
    }

    const offset = options?.offset ?? 0;
    const limit = options?.limit ?? users.length;
    return users.slice(offset, offset + limit);
  }

  async create(data: CreateUserDTO): Promise<User> {
    const user: User = {
      id: Math.random().toString(36).substring(2),
      ...data,
      createdAt: new Date()
    };
    this.users.set(user.id, user);
    return user;
  }

  async update(id: string, data: UpdateUserDTO): Promise<User | null> {
    const user = this.users.get(id);
    if (!user) return null;
    const updated = { ...user, ...data };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }

  async count(filter?: Partial<User>): Promise<number> {
    if (!filter) return this.users.size;
    let count = 0;
    for (const user of this.users.values()) {
      if (Object.entries(filter).every(([k, v]) => user[k as keyof User] === v)) {
        count++;
      }
    }
    return count;
  }
}
```

---

## 2. Unit of Work Pattern

Unit of Work ติดตาม objects ที่ถูก register ระหว่าง transaction และ commit/rollback พร้อมกัน

```typescript
interface IEntity {
  id: string;
}

type EntityState = 'new' | 'dirty' | 'deleted' | 'clean';

interface EntityEntry<T extends IEntity> {
  entity: T;
  state: EntityState;
  originalData?: T;
}

class UnitOfWork {
  private entries: Map<string, EntityEntry<IEntity>> = new Map();
  private repositories: Map<string, IUserRepository> = new Map();

  registerNew<T extends IEntity>(entity: T): void {
    this.entries.set(entity.id, { entity, state: 'new' });
  }

  registerDirty<T extends IEntity>(entity: T, original: T): void {
    const existing = this.entries.get(entity.id);
    if (!existing || existing.state === 'new') {
      return; // Keep 'new' state
    }
    this.entries.set(entity.id, {
      entity,
      state: 'dirty',
      originalData: original
    });
  }

  registerDeleted<T extends IEntity>(entity: T): void {
    const existing = this.entries.get(entity.id);
    if (existing?.state === 'new') {
      this.entries.delete(entity.id);
      return;
    }
    this.entries.set(entity.id, { entity, state: 'deleted' });
  }

  async commit(userRepo: IUserRepository): Promise<void> {
    const newEntities = this.getByState('new');
    const dirtyEntities = this.getByState('dirty');
    const deletedEntities = this.getByState('deleted');

    try {
      for (const entry of newEntities) {
        await userRepo.create(entry.entity as any);
      }
      for (const entry of dirtyEntities) {
        await userRepo.update(entry.entity.id, entry.entity as any);
      }
      for (const entry of deletedEntities) {
        await userRepo.delete(entry.entity.id);
      }
      this.clear();
    } catch (error) {
      await this.rollback(userRepo, dirtyEntities);
      throw error;
    }
  }

  private async rollback(
    userRepo: IUserRepository,
    dirtyEntities: EntityEntry<IEntity>[]
  ): Promise<void> {
    for (const entry of dirtyEntities) {
      if (entry.originalData) {
        await userRepo.update(entry.originalData.id, entry.originalData as any);
      }
    }
  }

  private getByState(state: EntityState): EntityEntry<IEntity>[] {
    return Array.from(this.entries.values()).filter(e => e.state === state);
  }

  clear(): void {
    this.entries.clear();
  }
}
```

---

## 3. Specification Pattern

Specification Pattern ช่วยสร้าง Business Rules ที่สามารถรวมกันได้

```typescript
interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean;
  and(other: Specification<T>): Specification<T>;
  or(other: Specification<T>): Specification<T>;
  not(): Specification<T>;
}

abstract class BaseSpecification<T> implements Specification<T> {
  abstract isSatisfiedBy(candidate: T): boolean;

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

class AndSpecification<T> extends BaseSpecification<T> {
  constructor(
    private left: Specification<T>,
    private right: Specification<T>
  ) {
    super();
  }

  isSatisfiedBy(candidate: T): boolean {
    return this.left.isSatisfiedBy(candidate) && this.right.isSatisfiedBy(candidate);
  }
}

class OrSpecification<T> extends BaseSpecification<T> {
  constructor(
    private left: Specification<T>,
    private right: Specification<T>
  ) {
    super();
  }

  isSatisfiedBy(candidate: T): boolean {
    return this.left.isSatisfiedBy(candidate) || this.right.isSatisfiedBy(candidate);
  }
}

class NotSpecification<T> extends BaseSpecification<T> {
  constructor(private spec: Specification<T>) {
    super();
  }

  isSatisfiedBy(candidate: T): boolean {
    return !this.spec.isSatisfiedBy(candidate);
  }
}

// Business Specifications
interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
  discount: number;
}

class PriceRangeSpec extends BaseSpecification<Product> {
  constructor(private min: number, private max: number) {
    super();
  }

  isSatisfiedBy(product: Product): boolean {
    return product.price >= this.min && product.price <= this.max;
  }
}

class InStockSpec extends BaseSpecification<Product> {
  isSatisfiedBy(product: Product): boolean {
    return product.inStock;
  }
}

class CategorySpec extends BaseSpecification<Product> {
  constructor(private category: string) {
    super();
  }

  isSatisfiedBy(product: Product): boolean {
    return product.category === this.category;
  }
}

class OnSaleSpec extends BaseSpecification<Product> {
  isSatisfiedBy(product: Product): boolean {
    return product.discount > 0;
  }
}

// ตัวอย่างการใช้งาน
const products: Product[] = [
  { id: '1', name: 'Laptop', price: 30000, category: 'electronics', inStock: true, discount: 10 },
  { id: '2', name: 'Phone', price: 15000, category: 'electronics', inStock: false, discount: 0 },
  { id: '3', name: 'Book', price: 500, category: 'books', inStock: true, discount: 0 },
];

const affordableElectronics = new PriceRangeSpec(0, 20000)
  .and(new CategorySpec('electronics'))
  .and(new InStockSpec());

const filtered = products.filter(p => affordableElectronics.isSatisfiedBy(p));
console.log(filtered.map(p => p.name)); // ['Phone']... but not in stock, so empty
```

---

## 4. Result/Either Pattern

Result Pattern ช่วยจัดการ errors ได้อย่างชัดเจนโดยไม่ต้อง throw exceptions

```typescript
type Result<T, E = Error> = Ok<T> | Err<E>;

class Ok<T> {
  readonly _type = 'ok' as const;
  constructor(public readonly value: T) {}

  isOk(): this is Ok<T> { return true; }
  isErr(): this is Err<never> { return false; }

  map<U>(fn: (value: T) => U): Result<U> {
    return new Ok(fn(this.value));
  }

  flatMap<U, E>(fn: (value: T) => Result<U, E>): Result<U, E> {
    return fn(this.value);
  }

  mapErr<F>(_fn: (err: never) => F): Result<T, F> {
    return this as unknown as Result<T, F>;
  }

  getOrElse(_defaultValue: T): T {
    return this.value;
  }

  getOrThrow(): T {
    return this.value;
  }
}

class Err<E> {
  readonly _type = 'err' as const;
  constructor(public readonly error: E) {}

  isOk(): this is Ok<never> { return false; }
  isErr(): this is Err<E> { return true; }

  map<U>(_fn: (value: never) => U): Result<U, E> {
    return this as unknown as Result<U, E>;
  }

  flatMap<U>(_fn: (value: never) => Result<U, E>): Result<U, E> {
    return this as unknown as Result<U, E>;
  }

  mapErr<F>(fn: (err: E) => F): Result<never, F> {
    return new Err(fn(this.error));
  }

  getOrElse<T>(defaultValue: T): T {
    return defaultValue;
  }

  getOrThrow(): never {
    throw this.error;
  }
}

// Helper functions
const ok = <T>(value: T): Ok<T> => new Ok(value);
const err = <E>(error: E): Err<E> => new Err(error);

// ตัวอย่างการใช้งาน
type ValidationError = {
  field: string;
  message: string;
};

function validateEmail(email: string): Result<string, ValidationError> {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    return err({ field: 'email', message: 'อีเมลไม่ถูกต้อง' });
  }
  return ok(email.toLowerCase());
}

function validateAge(age: number): Result<number, ValidationError> {
  if (age < 0 || age > 150) {
    return err({ field: 'age', message: 'อายุต้องอยู่ระหว่าง 0-150' });
  }
  return ok(age);
}

// Chain results
const emailResult = validateEmail("user@example.com")
  .map(email => email.trim());

const ageResult = validateAge(25)
  .map(age => age * 365); // แปลงเป็นวัน

if (emailResult.isOk()) {
  console.log('Email valid:', emailResult.value);
} else {
  console.log('Error:', emailResult.error);
}
```

---

## 5. Value Object Pattern

Value Objects เป็น objects ที่กำหนดโดยค่า ไม่ใช่ identity

```typescript
abstract class ValueObject<T> {
  constructor(protected readonly props: T) {
    this.validate(props);
  }

  protected abstract validate(props: T): void;

  equals(other: ValueObject<T>): boolean {
    return JSON.stringify(this.props) === JSON.stringify(other.props);
  }

  getValue(): T {
    return this.props;
  }
}

// Email Value Object
class Email extends ValueObject<string> {
  protected validate(email: string): void {
    if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new Error(`อีเมล '${email}' ไม่ถูกต้อง`);
    }
  }

  toString(): string {
    return this.props;
  }

  getDomain(): string {
    return this.props.split('@')[1];
  }
}

// Money Value Object
interface MoneyProps {
  amount: number;
  currency: string;
}

class Money extends ValueObject<MoneyProps> {
  protected validate(props: MoneyProps): void {
    if (props.amount < 0) {
      throw new Error('จำนวนเงินต้องไม่ติดลบ');
    }
    if (!props.currency || props.currency.length !== 3) {
      throw new Error('สกุลเงินต้องมี 3 ตัวอักษร');
    }
  }

  get amount(): number { return this.props.amount; }
  get currency(): string { return this.props.currency; }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('ไม่สามารถบวกเงินต่างสกุลได้');
    }
    return new Money({ amount: this.amount + other.amount, currency: this.currency });
  }

  subtract(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('ไม่สามารถลบเงินต่างสกุลได้');
    }
    const result = this.amount - other.amount;
    if (result < 0) throw new Error('เงินไม่พอ');
    return new Money({ amount: result, currency: this.currency });
  }

  multiply(factor: number): Money {
    return new Money({ amount: this.amount * factor, currency: this.currency });
  }

  toString(): string {
    return `${this.amount.toFixed(2)} ${this.currency}`;
  }
}

// ตัวอย่าง
const price = new Money({ amount: 100, currency: 'THB' });
const tax = new Money({ amount: 7, currency: 'THB' });
const total = price.add(tax);
console.log(total.toString()); // 107.00 THB
```

---

## 6. Domain Event Pattern

Domain Events แจ้งระบบอื่นๆ เมื่อมีสิ่งสำคัญเกิดขึ้นใน Domain

```typescript
interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  timestamp: Date;
  payload: unknown;
}

type EventHandler<T extends DomainEvent> = (event: T) => Promise<void> | void;

class EventBus {
  private static instance: EventBus;
  private handlers: Map<string, EventHandler<DomainEvent>[]> = new Map();

  static getInstance(): EventBus {
    if (!EventBus.instance) {
      EventBus.instance = new EventBus();
    }
    return EventBus.instance;
  }

  subscribe<T extends DomainEvent>(
    eventType: string,
    handler: EventHandler<T>
  ): () => void {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType)!.push(handler as EventHandler<DomainEvent>);

    // Return unsubscribe function
    return () => {
      const handlers = this.handlers.get(eventType)!;
      const index = handlers.indexOf(handler as EventHandler<DomainEvent>);
      if (index > -1) handlers.splice(index, 1);
    };
  }

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) ?? [];
    await Promise.all(handlers.map(h => h(event)));
  }
}

// Events
interface UserRegisteredEvent extends DomainEvent {
  eventType: 'UserRegistered';
  payload: { userId: string; email: string; name: string };
}

interface OrderPlacedEvent extends DomainEvent {
  eventType: 'OrderPlaced';
  payload: { orderId: string; userId: string; items: Array<{productId: string; quantity: number; price: number}> };
}

// ตัวอย่างการใช้งาน
const eventBus = EventBus.getInstance();

// Subscribe ไปยัง events
const unsubscribeEmail = eventBus.subscribe<UserRegisteredEvent>(
  'UserRegistered',
  async (event) => {
    console.log(`ส่งอีเมลยืนยันไปยัง ${event.payload.email}`);
    // send welcome email logic here
  }
);

const unsubscribeAnalytics = eventBus.subscribe<UserRegisteredEvent>(
  'UserRegistered',
  async (event) => {
    console.log(`บันทึก analytics สำหรับผู้ใช้ ${event.payload.userId}`);
  }
);

// Publish event
await eventBus.publish({
  eventId: 'evt-001',
  eventType: 'UserRegistered',
  aggregateId: 'user-001',
  timestamp: new Date(),
  payload: { userId: 'user-001', email: 'user@example.com', name: 'สมชาย' }
} as UserRegisteredEvent);
```

---

## 7. Outbox Pattern

Outbox Pattern รับประกันว่า domain events จะถูกส่งออกไปอย่างน้อยหนึ่งครั้ง

```typescript
interface OutboxMessage {
  id: string;
  eventType: string;
  payload: string;
  createdAt: Date;
  processedAt: Date | null;
  retryCount: number;
  maxRetries: number;
}

class OutboxRepository {
  private messages: Map<string, OutboxMessage> = new Map();

  async save(eventType: string, payload: unknown): Promise<OutboxMessage> {
    const message: OutboxMessage = {
      id: Math.random().toString(36).substring(2),
      eventType,
      payload: JSON.stringify(payload),
      createdAt: new Date(),
      processedAt: null,
      retryCount: 0,
      maxRetries: 3
    };
    this.messages.set(message.id, message);
    return message;
  }

  async findUnprocessed(): Promise<OutboxMessage[]> {
    return Array.from(this.messages.values())
      .filter(m => !m.processedAt && m.retryCount < m.maxRetries)
      .sort((a, b) => a.createdAt.getTime() - b.createdAt.getTime());
  }

  async markAsProcessed(id: string): Promise<void> {
    const message = this.messages.get(id);
    if (message) {
      message.processedAt = new Date();
    }
  }

  async incrementRetry(id: string): Promise<void> {
    const message = this.messages.get(id);
    if (message) {
      message.retryCount++;
    }
  }
}

class OutboxProcessor {
  private isRunning = false;

  constructor(
    private outboxRepo: OutboxRepository,
    private eventBus: EventBus
  ) {}

  async start(intervalMs: number = 5000): Promise<void> {
    this.isRunning = true;
    while (this.isRunning) {
      await this.processMessages();
      await new Promise(resolve => setTimeout(resolve, intervalMs));
    }
  }

  stop(): void {
    this.isRunning = false;
  }

  private async processMessages(): Promise<void> {
    const messages = await this.outboxRepo.findUnprocessed();

    for (const message of messages) {
      try {
        const payload = JSON.parse(message.payload);
        await this.eventBus.publish({
          eventId: message.id,
          eventType: message.eventType,
          aggregateId: payload.aggregateId ?? message.id,
          timestamp: message.createdAt,
          payload
        });
        await this.outboxRepo.markAsProcessed(message.id);
      } catch (error) {
        console.error(`Failed to process message ${message.id}:`, error);
        await this.outboxRepo.incrementRetry(message.id);
      }
    }
  }
}
```

---

## 8. Saga Pattern

Saga จัดการ long-running business transactions ด้วย compensating transactions

```typescript
type SagaStep<TState> = {
  execute: (state: TState) => Promise<TState>;
  compensate: (state: TState) => Promise<TState>;
  name: string;
};

class Saga<TState> {
  private steps: SagaStep<TState>[] = [];
  private executedSteps: SagaStep<TState>[] = [];

  addStep(step: SagaStep<TState>): this {
    this.steps.push(step);
    return this;
  }

  async execute(initialState: TState): Promise<TState> {
    let state = initialState;
    this.executedSteps = [];

    for (const step of this.steps) {
      try {
        console.log(`Executing step: ${step.name}`);
        state = await step.execute(state);
        this.executedSteps.push(step);
      } catch (error) {
        console.error(`Step ${step.name} failed, compensating...`);
        await this.compensate(state);
        throw error;
      }
    }
    return state;
  }

  private async compensate(state: TState): Promise<void> {
    for (let i = this.executedSteps.length - 1; i >= 0; i--) {
      const step = this.executedSteps[i];
      try {
        console.log(`Compensating step: ${step.name}`);
        state = await step.compensate(state);
      } catch (compensationError) {
        console.error(`Compensation failed for step ${step.name}:`, compensationError);
      }
    }
  }
}

// ตัวอย่าง - Order Saga
interface OrderState {
  orderId: string;
  userId: string;
  amount: number;
  paymentId?: string;
  inventoryReserved?: boolean;
  notificationSent?: boolean;
}

const orderSaga = new Saga<OrderState>()
  .addStep({
    name: 'ReserveInventory',
    execute: async (state) => {
      console.log('กำลัง reserve สินค้า...');
      // ตรวจสอบและ reserve สินค้า
      return { ...state, inventoryReserved: true };
    },
    compensate: async (state) => {
      console.log('กำลัง release สินค้าที่ reserve...');
      return { ...state, inventoryReserved: false };
    }
  })
  .addStep({
    name: 'ProcessPayment',
    execute: async (state) => {
      console.log('กำลังประมวลผลการชำระเงิน...');
      const paymentId = `pay-${Date.now()}`;
      return { ...state, paymentId };
    },
    compensate: async (state) => {
      console.log(`กำลัง refund การชำระเงิน ${state.paymentId}...`);
      return { ...state, paymentId: undefined };
    }
  })
  .addStep({
    name: 'SendNotification',
    execute: async (state) => {
      console.log('กำลังส่ง notification...');
      return { ...state, notificationSent: true };
    },
    compensate: async (state) => {
      return state; // ไม่ต้อง compensate
    }
  });

// รัน saga
const finalState = await orderSaga.execute({
  orderId: 'order-001',
  userId: 'user-001',
  amount: 500
});
```

---

## 9. Circuit Breaker Pattern

Circuit Breaker ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ

```typescript
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerOptions {
  failureThreshold: number;
  recoveryTimeout: number;
  successThreshold: number;
  timeout: number;
}

class CircuitBreaker<T> {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime?: Date;
  private readonly options: CircuitBreakerOptions;

  constructor(options: Partial<CircuitBreakerOptions> = {}) {
    this.options = {
      failureThreshold: 5,
      recoveryTimeout: 30000,
      successThreshold: 2,
      timeout: 10000,
      ...options
    };
  }

  async execute(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (this.shouldAttemptRecovery()) {
        this.state = 'HALF_OPEN';
        console.log('Circuit Breaker: เปลี่ยนสถานะเป็น HALF_OPEN');
      } else {
        throw new Error('Circuit Breaker เปิดอยู่ - ไม่สามารถเรียก service ได้');
      }
    }

    try {
      const result = await this.withTimeout(fn, this.options.timeout);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure(error as Error);
      throw error;
    }
  }

  private async withTimeout<T>(fn: () => Promise<T>, timeout: number): Promise<T> {
    return Promise.race([
      fn(),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error('Request timeout')), timeout)
      )
    ]);
  }

  private onSuccess(): void {
    this.failureCount = 0;
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.state = 'CLOSED';
        this.successCount = 0;
        console.log('Circuit Breaker: เปลี่ยนสถานะเป็น CLOSED');
      }
    }
  }

  private onFailure(error: Error): void {
    this.failureCount++;
    this.lastFailureTime = new Date();

    if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
      this.successCount = 0;
      console.log('Circuit Breaker: เปลี่ยนสถานะกลับเป็น OPEN');
    } else if (this.failureCount >= this.options.failureThreshold) {
      this.state = 'OPEN';
      console.log(`Circuit Breaker: เปิดหลังจากล้มเหลว ${this.failureCount} ครั้ง`);
    }
  }

  private shouldAttemptRecovery(): boolean {
    if (!this.lastFailureTime) return false;
    const elapsed = Date.now() - this.lastFailureTime.getTime();
    return elapsed >= this.options.recoveryTimeout;
  }

  getState(): CircuitState {
    return this.state;
  }

  getStats() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount
    };
  }
}

// ตัวอย่าง
const breaker = new CircuitBreaker<string>({
  failureThreshold: 3,
  recoveryTimeout: 10000
});

const callExternalService = async (): Promise<string> => {
  try {
    return await breaker.execute(async () => {
      // เรียก external service
      const response = await fetch('https://api.example.com/data');
      return response.json();
    });
  } catch (error) {
    return `Error: ${(error as Error).message}`;
  }
};
```

---

## 10. Retry Pattern

```typescript
interface RetryOptions {
  maxAttempts: number;
  delayMs: number;
  backoffFactor: number;
  maxDelayMs: number;
  retryOn?: (error: Error) => boolean;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts: RetryOptions = {
    maxAttempts: 3,
    delayMs: 1000,
    backoffFactor: 2,
    maxDelayMs: 30000,
    ...options
  };

  let lastError: Error;
  let delay = opts.delayMs;

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (opts.retryOn && !opts.retryOn(lastError)) {
        throw lastError;
      }

      if (attempt < opts.maxAttempts) {
        console.log(`Attempt ${attempt} failed. Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        delay = Math.min(delay * opts.backoffFactor, opts.maxDelayMs);
      }
    }
  }

  throw lastError!;
}

// Retry Decorator
function Retry(options: Partial<RetryOptions> = {}) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    descriptor.value = async function (...args: any[]) {
      return withRetry(() => originalMethod.apply(this, args), options);
    };
    return descriptor;
  };
}

class PaymentService {
  @Retry({ maxAttempts: 3, delayMs: 1000 })
  async processPayment(amount: number): Promise<{ success: boolean; transactionId: string }> {
    // payment logic
    return { success: true, transactionId: `txn-${Date.now()}` };
  }
}
```

---

## 11. Bulkhead Pattern

Bulkhead แยก components ออกจากกันเพื่อป้องกัน failure cascade

```typescript
class Bulkhead {
  private activeCount = 0;
  private waitingCount = 0;
  private waitingQueue: Array<{ resolve: () => void; reject: (e: Error) => void }> = [];

  constructor(
    private maxConcurrent: number,
    private maxWaiting: number
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.activeCount < this.maxConcurrent) {
      return this.run(fn);
    }

    if (this.waitingCount >= this.maxWaiting) {
      throw new Error('Bulkhead เต็ม - ไม่สามารถรับ request เพิ่มได้');
    }

    return new Promise((resolve, reject) => {
      this.waitingCount++;
      this.waitingQueue.push({
        resolve: () => {
          this.waitingCount--;
          this.run(fn).then(resolve, reject);
        },
        reject: (e) => {
          this.waitingCount--;
          reject(e);
        }
      });
    });
  }

  private async run<T>(fn: () => Promise<T>): Promise<T> {
    this.activeCount++;
    try {
      return await fn();
    } finally {
      this.activeCount--;
      this.processNext();
    }
  }

  private processNext(): void {
    if (this.waitingQueue.length > 0 && this.activeCount < this.maxConcurrent) {
      const next = this.waitingQueue.shift()!;
      next.resolve();
    }
  }

  getStats() {
    return {
      active: this.activeCount,
      waiting: this.waitingCount,
      maxConcurrent: this.maxConcurrent,
      maxWaiting: this.maxWaiting
    };
  }
}
```

---

## 12. Rate Limiter

```typescript
class TokenBucketRateLimiter {
  private tokens: number;
  private lastRefillTime: number;
  private readonly maxTokens: number;
  private readonly refillRate: number; // tokens per second

  constructor(maxTokens: number, refillRate: number) {
    this.maxTokens = maxTokens;
    this.refillRate = refillRate;
    this.tokens = maxTokens;
    this.lastRefillTime = Date.now();
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = (now - this.lastRefillTime) / 1000;
    const newTokens = elapsed * this.refillRate;
    this.tokens = Math.min(this.maxTokens, this.tokens + newTokens);
    this.lastRefillTime = now;
  }

  tryAcquire(tokens: number = 1): boolean {
    this.refill();
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }
    return false;
  }

  async acquire(tokens: number = 1): Promise<void> {
    while (!this.tryAcquire(tokens)) {
      const waitTime = ((tokens - this.tokens) / this.refillRate) * 1000;
      await new Promise(resolve => setTimeout(resolve, Math.ceil(waitTime)));
    }
  }
}

class SlidingWindowRateLimiter {
  private timestamps: number[] = [];

  constructor(
    private readonly limit: number,
    private readonly windowMs: number
  ) {}

  isAllowed(): boolean {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // ลบ timestamps ที่เก่ากว่า window
    this.timestamps = this.timestamps.filter(t => t > windowStart);

    if (this.timestamps.length < this.limit) {
      this.timestamps.push(now);
      return true;
    }
    return false;
  }

  getRemainingRequests(): number {
    const windowStart = Date.now() - this.windowMs;
    const recentCount = this.timestamps.filter(t => t > windowStart).length;
    return Math.max(0, this.limit - recentCount);
  }
}

// Rate Limiter Middleware
function rateLimitMiddleware(limiter: TokenBucketRateLimiter) {
  return async (req: any, res: any, next: () => void) => {
    const allowed = limiter.tryAcquire();
    if (!allowed) {
      res.status(429).json({
        error: 'Too Many Requests',
        message: 'คุณส่ง request มากเกินไป กรุณาลองใหม่ในอีกสักครู่'
      });
      return;
    }
    next();
  };
}
```

---

## 13. Cache-Aside Pattern

```typescript
interface CacheStore<T> {
  get(key: string): Promise<T | null>;
  set(key: string, value: T, ttlSeconds?: number): Promise<void>;
  delete(key: string): Promise<void>;
  clear(): Promise<void>;
}

class InMemoryCache<T> implements CacheStore<T> {
  private store: Map<string, { value: T; expiresAt?: number }> = new Map();

  async get(key: string): Promise<T | null> {
    const entry = this.store.get(key);
    if (!entry) return null;
    if (entry.expiresAt && Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return null;
    }
    return entry.value;
  }

  async set(key: string, value: T, ttlSeconds?: number): Promise<void> {
    this.store.set(key, {
      value,
      expiresAt: ttlSeconds ? Date.now() + ttlSeconds * 1000 : undefined
    });
  }

  async delete(key: string): Promise<void> {
    this.store.delete(key);
  }

  async clear(): Promise<void> {
    this.store.clear();
  }
}

class CacheAsideRepository<T> {
  constructor(
    private cache: CacheStore<T>,
    private dataSource: { fetch: (key: string) => Promise<T | null> },
    private ttlSeconds: number = 300
  ) {}

  async get(key: string): Promise<T | null> {
    // 1. ตรวจสอบ cache ก่อน
    let data = await this.cache.get(key);
    if (data !== null) {
      console.log(`Cache hit: ${key}`);
      return data;
    }

    // 2. ถ้าไม่มีใน cache ดึงจาก data source
    console.log(`Cache miss: ${key}`);
    data = await this.dataSource.fetch(key);

    // 3. เก็บไว้ใน cache
    if (data !== null) {
      await this.cache.set(key, data, this.ttlSeconds);
    }
    return data;
  }

  async invalidate(key: string): Promise<void> {
    await this.cache.delete(key);
  }

  async update(key: string, value: T): Promise<void> {
    await this.cache.set(key, value, this.ttlSeconds);
  }
}
```

---

## 14. Event-Driven Architecture Patterns

```typescript
// CQRS Pattern
interface Command {
  type: string;
  payload: unknown;
}

interface Query {
  type: string;
  params: unknown;
}

type CommandHandler<TCommand extends Command, TResult> = (
  command: TCommand
) => Promise<TResult>;

type QueryHandler<TQuery extends Query, TResult> = (
  query: TQuery
) => Promise<TResult>;

class CommandBus {
  private handlers: Map<string, CommandHandler<Command, unknown>> = new Map();

  register<TCommand extends Command, TResult>(
    commandType: string,
    handler: CommandHandler<TCommand, TResult>
  ): void {
    this.handlers.set(commandType, handler as CommandHandler<Command, unknown>);
  }

  async dispatch<TResult>(command: Command): Promise<TResult> {
    const handler = this.handlers.get(command.type);
    if (!handler) {
      throw new Error(`ไม่พบ handler สำหรับ command: ${command.type}`);
    }
    return handler(command) as Promise<TResult>;
  }
}

class QueryBus {
  private handlers: Map<string, QueryHandler<Query, unknown>> = new Map();

  register<TQuery extends Query, TResult>(
    queryType: string,
    handler: QueryHandler<TQuery, TResult>
  ): void {
    this.handlers.set(queryType, handler as QueryHandler<Query, unknown>);
  }

  async execute<TResult>(query: Query): Promise<TResult> {
    const handler = this.handlers.get(query.type);
    if (!handler) {
      throw new Error(`ไม่พบ handler สำหรับ query: ${query.type}`);
    }
    return handler(query) as Promise<TResult>;
  }
}

// ตัวอย่าง CQRS
interface CreateOrderCommand extends Command {
  type: 'CreateOrder';
  payload: { userId: string; items: Array<{ productId: string; quantity: number }> };
}

interface GetOrderQuery extends Query {
  type: 'GetOrder';
  params: { orderId: string };
}

const commandBus = new CommandBus();
const queryBus = new QueryBus();

commandBus.register<CreateOrderCommand, string>('CreateOrder', async (command) => {
  const orderId = `order-${Date.now()}`;
  console.log(`สร้าง order ${orderId} สำหรับ user ${command.payload.userId}`);
  return orderId;
});

queryBus.register<GetOrderQuery, { id: string; status: string }>('GetOrder', async (query) => {
  const orderId = (query.params as any).orderId;
  return { id: orderId, status: 'PENDING' };
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Design Patterns ที่สำคัญในโลกจริง:

1. **Repository Pattern** - แยก data access จาก business logic
2. **Unit of Work** - จัดการ transactions หลายอย่างพร้อมกัน
3. **Specification Pattern** - สร้าง reusable business rules
4. **Result/Either Pattern** - จัดการ errors อย่างชัดเจน
5. **Value Object Pattern** - Objects ที่กำหนดโดยค่า
6. **Domain Event Pattern** - แจ้งเหตุการณ์ใน domain
7. **Outbox Pattern** - รับประกันการส่ง events
8. **Saga Pattern** - จัดการ distributed transactions
9. **Circuit Breaker** - ป้องกัน cascade failures
10. **Retry Pattern** - จัดการ transient failures
11. **Bulkhead Pattern** - แยก isolation boundaries
12. **Rate Limiter** - ควบคุมปริมาณ requests
13. **Cache-Aside Pattern** - จัดการ caching
14. **Event-Driven/CQRS** - แยก commands จาก queries
