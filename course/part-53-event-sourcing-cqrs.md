# Part 53: Event Sourcing & CQRS กับ TypeScript

## บทนำ

Event Sourcing และ CQRS (Command Query Responsibility Segregation) เป็นสถาปัตยกรรมที่ทรงพลังสำหรับระบบที่ซับซ้อน โดยแยกการอ่านและเขียนข้อมูลออกจากกัน และเก็บประวัติทุก event ที่เกิดขึ้นในระบบ

---

## 1. CQRS แนวคิดและภาพรวม

### ปัญหาที่แก้ไขด้วย CQRS

```typescript
// ❌ แบบเดิม: Command และ Query ใช้ Model เดียวกัน
class OrderService {
  // Command - เขียนข้อมูล (ซับซ้อน)
  async createOrder(data: CreateOrderDto): Promise<Order> {
    const order = await this.validate(data);
    await this.db.save(order);
    await this.emailService.send(order);
    return order;
  }
  
  // Query - อ่านข้อมูล (ต้องการข้อมูลจากหลาย table)
  async getOrderSummary(customerId: string): Promise<OrderSummary> {
    // ดึงข้อมูลจาก join หลาย table ทำให้ช้า
    return this.db.query(`
      SELECT o.id, o.total, c.name, SUM(oi.quantity) as item_count
      FROM orders o
      JOIN customers c ON o.customer_id = c.id
      JOIN order_items oi ON o.id = oi.order_id
      WHERE o.customer_id = $1
      GROUP BY o.id, c.name
    `, [customerId]);
  }
}
```

```typescript
// ✅ CQRS: แยก Command และ Query
// Command Side (Write Model) - ซับซ้อน มี business logic
class OrderCommandService {
  async createOrder(command: CreateOrderCommand): Promise<void> {
    const order = Order.create(command);
    order.validate();
    await this.orderRepository.save(order);
    // Publish events...
  }
  
  async confirmOrder(command: ConfirmOrderCommand): Promise<void> {
    const order = await this.orderRepository.findById(command.orderId);
    order.confirm();
    await this.orderRepository.save(order);
  }
}

// Query Side (Read Model) - เร็ว ดึงข้อมูลที่ denormalized แล้ว
class OrderQueryService {
  async getOrderSummary(customerId: string): Promise<OrderSummaryView[]> {
    // ดึงจาก Read Model ที่ optimize แล้ว
    return this.readDb.query(
      'SELECT * FROM order_summaries WHERE customer_id = $1',
      [customerId]
    );
  }
  
  async getOrderDetails(orderId: string): Promise<OrderDetailsView | null> {
    return this.readDb.query(
      'SELECT * FROM order_details WHERE id = $1',
      [orderId]
    );
  }
}
```

### สถาปัตยกรรม CQRS

```
┌─────────────────────────────────────────────────┐
│                  Client Request                  │
└──────────────┬──────────────────┬───────────────┘
               │                  │
    ┌──────────▼──────┐   ┌──────▼──────────┐
    │  Command Handler│   │  Query Handler  │
    └──────────┬──────┘   └──────┬──────────┘
               │                  │
    ┌──────────▼──────┐   ┌──────▼──────────┐
    │  Write Model    │   │  Read Model     │
    │  (Complex)      │   │  (Optimized)    │
    └──────────┬──────┘   └──────▲──────────┘
               │                  │
               │    ┌─────────────┘
               └────►  Event/Sync  
```

---

## 2. Command Side

### Commands

```typescript
// Base Command
abstract class Command {
  readonly commandId: string;
  readonly occurredAt: Date;
  
  constructor() {
    this.commandId = `cmd-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    this.occurredAt = new Date();
  }
}

// Concrete Commands
class CreateOrderCommand extends Command {
  constructor(
    readonly customerId: string,
    readonly items: Array<{ productId: string; quantity: number }>,
    readonly shippingAddress: AddressDto,
    readonly promotionCode?: string
  ) {
    super();
  }
}

class AddItemToOrderCommand extends Command {
  constructor(
    readonly orderId: string,
    readonly productId: string,
    readonly quantity: number
  ) {
    super();
  }
}

class ConfirmOrderCommand extends Command {
  constructor(
    readonly orderId: string,
    readonly confirmedBy: string
  ) {
    super();
  }
}

class CancelOrderCommand extends Command {
  constructor(
    readonly orderId: string,
    readonly reason: string,
    readonly cancelledBy: string
  ) {
    super();
  }
}

class ShipOrderCommand extends Command {
  constructor(
    readonly orderId: string,
    readonly trackingNumber: string,
    readonly carrier: string
  ) {
    super();
  }
}

// DTOs
interface AddressDto {
  street: string;
  city: string;
  postalCode: string;
  country: string;
}
```

### Command Handlers

```typescript
// Command Handler Interface
interface CommandHandler<TCommand extends Command, TResult = void> {
  handle(command: TCommand): Promise<TResult>;
}

// Create Order Handler
class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand, string> {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository,
    private customerRepository: CustomerRepository,
    private eventBus: EventBus
  ) {}
  
  async handle(command: CreateOrderCommand): Promise<string> {
    // 1. Validate customer
    const customer = await this.customerRepository.findById(command.customerId);
    if (!customer) {
      throw new Error(`Customer not found: ${command.customerId}`);
    }
    
    // 2. Validate and get products
    const products = new Map<string, { name: string; price: number }>();
    for (const item of command.items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) {
        throw new Error(`Product not found: ${item.productId}`);
      }
      if (product.stockQuantity < item.quantity) {
        throw new Error(`Insufficient stock for product: ${item.productId}`);
      }
      products.set(item.productId, {
        name: product.name,
        price: product.price
      });
    }
    
    // 3. Create order
    const orderId = `order-${Date.now()}`;
    const order = Order.create({
      id: orderId,
      customerId: command.customerId,
      items: command.items.map(item => ({
        productId: item.productId,
        productName: products.get(item.productId)!.name,
        unitPrice: products.get(item.productId)!.price,
        quantity: item.quantity
      })),
      shippingAddress: command.shippingAddress
    });
    
    // 4. Save
    await this.orderRepository.save(order);
    
    // 5. Publish events
    await this.eventBus.publish(new OrderCreatedEvent(orderId, command.customerId));
    
    return orderId;
  }
}

// Confirm Order Handler
class ConfirmOrderCommandHandler implements CommandHandler<ConfirmOrderCommand> {
  constructor(
    private orderRepository: OrderRepository,
    private eventBus: EventBus
  ) {}
  
  async handle(command: ConfirmOrderCommand): Promise<void> {
    const order = await this.orderRepository.findById(command.orderId);
    if (!order) {
      throw new Error(`Order not found: ${command.orderId}`);
    }
    
    order.confirm(command.confirmedBy);
    await this.orderRepository.save(order);
    
    await this.eventBus.publish(
      new OrderConfirmedEvent(command.orderId, command.confirmedBy)
    );
  }
}

// Command Bus
class CommandBus {
  private handlers = new Map<string, CommandHandler<any, any>>();
  
  register<TCommand extends Command>(
    commandType: new (...args: any[]) => TCommand,
    handler: CommandHandler<TCommand, any>
  ): void {
    this.handlers.set(commandType.name, handler);
  }
  
  async dispatch<TCommand extends Command, TResult = void>(
    command: TCommand
  ): Promise<TResult> {
    const handler = this.handlers.get(command.constructor.name);
    if (!handler) {
      throw new Error(`No handler registered for command: ${command.constructor.name}`);
    }
    return handler.handle(command);
  }
}
```

---

## 3. Query Side

### Queries และ Read Models

```typescript
// Query Interface
abstract class Query {
  readonly queryId: string;
  constructor() {
    this.queryId = `qry-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Concrete Queries
class GetOrderByIdQuery extends Query {
  constructor(readonly orderId: string) {
    super();
  }
}

class GetOrdersByCustomerQuery extends Query {
  constructor(
    readonly customerId: string,
    readonly page: number = 1,
    readonly pageSize: number = 10,
    readonly status?: string
  ) {
    super();
  }
}

class GetOrderStatisticsQuery extends Query {
  constructor(
    readonly startDate: Date,
    readonly endDate: Date
  ) {
    super();
  }
}

// Read Models (Optimized Views)
interface OrderSummaryView {
  orderId: string;
  customerId: string;
  customerName: string;
  customerEmail: string;
  status: string;
  totalAmount: number;
  currency: string;
  itemCount: number;
  createdAt: Date;
  confirmedAt?: Date;
  shippedAt?: Date;
  deliveredAt?: Date;
}

interface OrderDetailsView {
  orderId: string;
  customerId: string;
  customerName: string;
  customerEmail: string;
  shippingAddress: {
    street: string;
    city: string;
    postalCode: string;
    country: string;
  };
  items: Array<{
    productId: string;
    productName: string;
    productImage?: string;
    unitPrice: number;
    quantity: number;
    subtotal: number;
  }>;
  discountCode?: string;
  discountAmount: number;
  subtotal: number;
  shippingFee: number;
  totalAmount: number;
  currency: string;
  status: string;
  statusHistory: Array<{
    status: string;
    changedAt: Date;
    changedBy: string;
    note?: string;
  }>;
  trackingNumber?: string;
  carrier?: string;
  createdAt: Date;
}

interface OrderStatisticsView {
  period: string;
  totalOrders: number;
  confirmedOrders: number;
  cancelledOrders: number;
  totalRevenue: number;
  averageOrderValue: number;
  topProducts: Array<{
    productId: string;
    productName: string;
    totalSold: number;
    revenue: number;
  }>;
}
```

### Query Handlers

```typescript
// Query Handler Interface
interface QueryHandler<TQuery extends Query, TResult> {
  handle(query: TQuery): Promise<TResult>;
}

// Get Order Details Handler
class GetOrderDetailsQueryHandler 
  implements QueryHandler<GetOrderByIdQuery, OrderDetailsView | null> {
  
  constructor(private readDb: ReadDatabase) {}
  
  async handle(query: GetOrderByIdQuery): Promise<OrderDetailsView | null> {
    const order = await this.readDb.findOne<OrderDetailsView>(
      'order_details',
      { orderId: query.orderId }
    );
    
    return order;
  }
}

// Get Orders By Customer Handler
class GetOrdersByCustomerQueryHandler
  implements QueryHandler<GetOrdersByCustomerQuery, { items: OrderSummaryView[]; total: number }> {
  
  constructor(private readDb: ReadDatabase) {}
  
  async handle(query: GetOrdersByCustomerQuery): Promise<{ items: OrderSummaryView[]; total: number }> {
    const filter: any = { customerId: query.customerId };
    if (query.status) {
      filter.status = query.status;
    }
    
    const offset = (query.page - 1) * query.pageSize;
    
    const [items, total] = await Promise.all([
      this.readDb.find<OrderSummaryView>('order_summaries', filter, {
        limit: query.pageSize,
        offset,
        orderBy: { field: 'createdAt', direction: 'DESC' }
      }),
      this.readDb.count('order_summaries', filter)
    ]);
    
    return { items, total };
  }
}

// Query Bus
class QueryBus {
  private handlers = new Map<string, QueryHandler<any, any>>();
  
  register<TQuery extends Query, TResult>(
    queryType: new (...args: any[]) => TQuery,
    handler: QueryHandler<TQuery, TResult>
  ): void {
    this.handlers.set(queryType.name, handler);
  }
  
  async dispatch<TQuery extends Query, TResult>(
    query: TQuery
  ): Promise<TResult> {
    const handler = this.handlers.get(query.constructor.name);
    if (!handler) {
      throw new Error(`No handler registered for query: ${query.constructor.name}`);
    }
    return handler.handle(query);
  }
}
```

---

## 4. Read Models และ Projections

```typescript
// Projection ที่แปลง Events เป็น Read Models
abstract class Projection {
  abstract handle(event: DomainEvent): Promise<void>;
}

// Order Summary Projection
class OrderSummaryProjection extends Projection {
  constructor(private readDb: ReadDatabase) {
    super();
  }
  
  async handle(event: DomainEvent): Promise<void> {
    switch (event.constructor.name) {
      case 'OrderCreatedEvent':
        await this.handleOrderCreated(event as OrderCreatedEvent);
        break;
      case 'OrderConfirmedEvent':
        await this.handleOrderConfirmed(event as OrderConfirmedEvent);
        break;
      case 'OrderShippedEvent':
        await this.handleOrderShipped(event as OrderShippedEvent);
        break;
      case 'OrderCancelledEvent':
        await this.handleOrderCancelled(event as OrderCancelledEvent);
        break;
    }
  }
  
  private async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    await this.readDb.upsert('order_summaries', {
      orderId: event.orderId,
      customerId: event.customerId,
      customerName: event.customerName,
      customerEmail: event.customerEmail,
      status: 'PENDING',
      totalAmount: event.totalAmount,
      currency: event.currency,
      itemCount: event.itemCount,
      createdAt: event.occurredAt,
    });
  }
  
  private async handleOrderConfirmed(event: OrderConfirmedEvent): Promise<void> {
    await this.readDb.update('order_summaries',
      { orderId: event.orderId },
      {
        status: 'CONFIRMED',
        confirmedAt: event.occurredAt
      }
    );
  }
  
  private async handleOrderShipped(event: OrderShippedEvent): Promise<void> {
    await this.readDb.update('order_summaries',
      { orderId: event.orderId },
      {
        status: 'SHIPPED',
        shippedAt: event.occurredAt,
        trackingNumber: event.trackingNumber
      }
    );
  }
  
  private async handleOrderCancelled(event: OrderCancelledEvent): Promise<void> {
    await this.readDb.update('order_summaries',
      { orderId: event.orderId },
      {
        status: 'CANCELLED',
        cancelledAt: event.occurredAt
      }
    );
  }
}

// Order Details Projection
class OrderDetailsProjection extends Projection {
  constructor(
    private readDb: ReadDatabase,
    private customerRepository: CustomerQueryRepository
  ) {
    super();
  }
  
  async handle(event: DomainEvent): Promise<void> {
    switch (event.constructor.name) {
      case 'OrderCreatedEvent':
        await this.handleOrderCreated(event as OrderCreatedEvent);
        break;
      case 'OrderItemAddedEvent':
        await this.handleItemAdded(event as OrderItemAddedEvent);
        break;
    }
  }
  
  private async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    const customer = await this.customerRepository.findById(event.customerId);
    
    await this.readDb.insert('order_details', {
      orderId: event.orderId,
      customerId: event.customerId,
      customerName: customer?.name || 'Unknown',
      customerEmail: customer?.email || '',
      shippingAddress: event.shippingAddress,
      items: event.items,
      discountAmount: 0,
      subtotal: event.subtotal,
      shippingFee: event.shippingFee,
      totalAmount: event.totalAmount,
      currency: event.currency,
      status: 'PENDING',
      statusHistory: [{
        status: 'PENDING',
        changedAt: event.occurredAt,
        changedBy: 'System'
      }],
      createdAt: event.occurredAt
    });
  }
  
  private async handleItemAdded(event: OrderItemAddedEvent): Promise<void> {
    const details = await this.readDb.findOne<OrderDetailsView>(
      'order_details',
      { orderId: event.orderId }
    );
    
    if (details) {
      const updatedItems = [...details.items, {
        productId: event.productId,
        productName: event.productName,
        unitPrice: event.unitPrice,
        quantity: event.quantity,
        subtotal: event.unitPrice * event.quantity
      }];
      
      await this.readDb.update('order_details',
        { orderId: event.orderId },
        { items: updatedItems }
      );
    }
  }
}
```

---

## 5. Event Sourcing แนวคิด

```typescript
// แทนที่จะเก็บ state ปัจจุบัน, เก็บ events ที่นำไปสู่ state นั้น

// ❌ แบบเดิม: เก็บแค่ state ปัจจุบัน
interface OrderState {
  id: string;
  status: string;
  total: number;
  items: any[];
}

// ✅ Event Sourcing: เก็บทุก event ที่เกิดขึ้น
interface StoredEvent {
  eventId: string;
  aggregateId: string;
  aggregateType: string;
  eventType: string;
  eventData: any;
  version: number;
  occurredAt: Date;
}

// เมื่อต้องการ state ปัจจุบัน จะ replay events ทั้งหมด
class EventSourcedOrder {
  private id: string = '';
  private status: string = 'PENDING';
  private items: any[] = [];
  private version: number = 0;
  private pendingEvents: DomainEvent[] = [];
  
  // Apply event เพื่อเปลี่ยน state
  private apply(event: DomainEvent): void {
    switch (event.constructor.name) {
      case 'OrderCreatedEvent':
        this.applyOrderCreated(event as OrderCreatedEvent);
        break;
      case 'OrderItemAddedEvent':
        this.applyItemAdded(event as OrderItemAddedEvent);
        break;
      case 'OrderConfirmedEvent':
        this.applyOrderConfirmed(event as OrderConfirmedEvent);
        break;
      case 'OrderCancelledEvent':
        this.applyOrderCancelled(event as OrderCancelledEvent);
        break;
    }
    this.version++;
  }
  
  // Reconstitute from events (Replay)
  static fromEvents(events: DomainEvent[]): EventSourcedOrder {
    const order = new EventSourcedOrder();
    for (const event of events) {
      order.apply(event);
    }
    order.version = events.length;
    return order;
  }
  
  // Business methods ที่สร้าง events
  static create(id: string, customerId: string, items: any[]): EventSourcedOrder {
    const order = new EventSourcedOrder();
    
    const event = new OrderCreatedEvent(id, customerId, items);
    order.apply(event);
    order.pendingEvents.push(event);
    
    return order;
  }
  
  addItem(productId: string, name: string, price: number, quantity: number): void {
    if (this.status !== 'PENDING') {
      throw new Error('Cannot add items to non-pending order');
    }
    
    const event = new OrderItemAddedEvent(this.id, productId, name, price, quantity);
    this.apply(event);
    this.pendingEvents.push(event);
  }
  
  confirm(): void {
    if (this.status !== 'PENDING') {
      throw new Error('Order is not in pending state');
    }
    if (this.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    
    const event = new OrderConfirmedEvent(this.id);
    this.apply(event);
    this.pendingEvents.push(event);
  }
  
  // Apply methods ที่เปลี่ยน state
  private applyOrderCreated(event: OrderCreatedEvent): void {
    this.id = event.orderId;
    this.status = 'PENDING';
    this.items = [];
  }
  
  private applyItemAdded(event: OrderItemAddedEvent): void {
    this.items.push({
      productId: event.productId,
      name: event.productName,
      price: event.unitPrice,
      quantity: event.quantity
    });
  }
  
  private applyOrderConfirmed(event: OrderConfirmedEvent): void {
    this.status = 'CONFIRMED';
  }
  
  private applyOrderCancelled(event: OrderCancelledEvent): void {
    this.status = 'CANCELLED';
  }
  
  // Getters
  getId(): string { return this.id; }
  getStatus(): string { return this.status; }
  getItems(): any[] { return [...this.items]; }
  getVersion(): number { return this.version; }
  getPendingEvents(): DomainEvent[] { return [...this.pendingEvents]; }
  clearPendingEvents(): void { this.pendingEvents = []; }
}
```

---

## 6. Event Store

```typescript
// Event Store Interface
interface EventStore {
  appendEvents(
    aggregateId: string,
    aggregateType: string,
    events: DomainEvent[],
    expectedVersion: number
  ): Promise<void>;
  
  getEvents(
    aggregateId: string,
    fromVersion?: number
  ): Promise<StoredEvent[]>;
  
  getAllEvents(fromPosition?: number): Promise<StoredEvent[]>;
}

// In-Memory Event Store (สำหรับ testing)
class InMemoryEventStore implements EventStore {
  private events: Map<string, StoredEvent[]> = new Map();
  private allEvents: StoredEvent[] = [];
  private position: number = 0;
  
  async appendEvents(
    aggregateId: string,
    aggregateType: string,
    events: DomainEvent[],
    expectedVersion: number
  ): Promise<void> {
    const existingEvents = this.events.get(aggregateId) || [];
    
    // Optimistic concurrency check
    if (existingEvents.length !== expectedVersion) {
      throw new OptimisticConcurrencyError(
        aggregateId,
        expectedVersion,
        existingEvents.length
      );
    }
    
    const storedEvents: StoredEvent[] = events.map((event, index) => ({
      eventId: event.eventId,
      aggregateId,
      aggregateType,
      eventType: event.getEventName(),
      eventData: this.serialize(event),
      version: expectedVersion + index + 1,
      occurredAt: event.occurredAt,
      position: ++this.position
    }));
    
    this.events.set(aggregateId, [...existingEvents, ...storedEvents]);
    this.allEvents.push(...storedEvents);
  }
  
  async getEvents(
    aggregateId: string,
    fromVersion: number = 0
  ): Promise<StoredEvent[]> {
    const events = this.events.get(aggregateId) || [];
    return events.filter(e => e.version > fromVersion);
  }
  
  async getAllEvents(fromPosition: number = 0): Promise<StoredEvent[]> {
    return this.allEvents.filter(e => (e as any).position > fromPosition);
  }
  
  private serialize(event: DomainEvent): any {
    return JSON.parse(JSON.stringify(event));
  }
}

// PostgreSQL Event Store
class PostgresEventStore implements EventStore {
  constructor(private db: Database) {}
  
  async appendEvents(
    aggregateId: string,
    aggregateType: string,
    events: DomainEvent[],
    expectedVersion: number
  ): Promise<void> {
    await this.db.transaction(async (trx) => {
      // Check current version
      const currentVersion = await trx.queryOne(
        'SELECT MAX(version) as version FROM events WHERE aggregate_id = $1',
        [aggregateId]
      );
      
      const currentVer = currentVersion?.version || 0;
      if (currentVer !== expectedVersion) {
        throw new OptimisticConcurrencyError(aggregateId, expectedVersion, currentVer);
      }
      
      // Insert events
      for (let i = 0; i < events.length; i++) {
        const event = events[i];
        await trx.query(
          `INSERT INTO events 
           (event_id, aggregate_id, aggregate_type, event_type, event_data, version, occurred_at)
           VALUES ($1, $2, $3, $4, $5, $6, $7)`,
          [
            event.eventId,
            aggregateId,
            aggregateType,
            event.getEventName(),
            JSON.stringify(event),
            expectedVersion + i + 1,
            event.occurredAt
          ]
        );
      }
    });
  }
  
  async getEvents(
    aggregateId: string,
    fromVersion: number = 0
  ): Promise<StoredEvent[]> {
    return this.db.query(
      'SELECT * FROM events WHERE aggregate_id = $1 AND version > $2 ORDER BY version ASC',
      [aggregateId, fromVersion]
    );
  }
  
  async getAllEvents(fromPosition: number = 0): Promise<StoredEvent[]> {
    return this.db.query(
      'SELECT * FROM events WHERE position > $1 ORDER BY position ASC',
      [fromPosition]
    );
  }
}

class OptimisticConcurrencyError extends Error {
  constructor(
    readonly aggregateId: string,
    readonly expectedVersion: number,
    readonly actualVersion: number
  ) {
    super(
      `Concurrency conflict for aggregate ${aggregateId}: ` +
      `expected version ${expectedVersion}, but found ${actualVersion}`
    );
  }
}
```

---

## 7. Event Replay

```typescript
// Event Replay สำหรับ rebuild state หรือ projection
class EventReplayService {
  constructor(
    private eventStore: EventStore,
    private eventDeserializer: EventDeserializer
  ) {}
  
  // Replay events สำหรับ aggregate เดียว
  async replayAggregate<T>(
    aggregateId: string,
    aggregate: T & { apply: (event: DomainEvent) => void }
  ): Promise<T> {
    const storedEvents = await this.eventStore.getEvents(aggregateId);
    
    for (const storedEvent of storedEvents) {
      const event = this.eventDeserializer.deserialize(storedEvent);
      aggregate.apply(event);
    }
    
    return aggregate;
  }
  
  // Replay ทุก events เพื่อ rebuild projection
  async replayAllEvents(
    projections: Projection[],
    fromPosition: number = 0
  ): Promise<void> {
    const events = await this.eventStore.getAllEvents(fromPosition);
    
    console.log(`Replaying ${events.length} events...`);
    
    for (const storedEvent of events) {
      const event = this.eventDeserializer.deserialize(storedEvent);
      
      for (const projection of projections) {
        try {
          await projection.handle(event);
        } catch (error) {
          console.error(
            `Error in projection ${projection.constructor.name} ` +
            `handling event ${storedEvent.eventType}:`,
            error
          );
        }
      }
    }
    
    console.log('Replay complete');
  }
}

// Event Deserializer
class EventDeserializer {
  private eventFactories: Map<string, (data: any) => DomainEvent> = new Map();
  
  register(eventType: string, factory: (data: any) => DomainEvent): void {
    this.eventFactories.set(eventType, factory);
  }
  
  deserialize(storedEvent: StoredEvent): DomainEvent {
    const factory = this.eventFactories.get(storedEvent.eventType);
    if (!factory) {
      throw new Error(`No deserializer for event type: ${storedEvent.eventType}`);
    }
    return factory(storedEvent.eventData);
  }
}

// ลงทะเบียน Event Factories
const deserializer = new EventDeserializer();

deserializer.register('OrderCreated', (data) => 
  new OrderCreatedEvent(data.orderId, data.customerId, data.items)
);

deserializer.register('OrderConfirmed', (data) =>
  new OrderConfirmedEvent(data.orderId)
);

deserializer.register('OrderCancelled', (data) =>
  new OrderCancelledEvent(data.orderId, data.reason)
);
```

---

## 8. Snapshots

```typescript
// Snapshot เพื่อลดจำนวน events ที่ต้อง replay
interface Snapshot {
  aggregateId: string;
  aggregateType: string;
  state: any;
  version: number;
  createdAt: Date;
}

interface SnapshotStore {
  saveSnapshot(snapshot: Snapshot): Promise<void>;
  getLatestSnapshot(aggregateId: string): Promise<Snapshot | null>;
}

class InMemorySnapshotStore implements SnapshotStore {
  private snapshots: Map<string, Snapshot> = new Map();
  
  async saveSnapshot(snapshot: Snapshot): Promise<void> {
    this.snapshots.set(snapshot.aggregateId, snapshot);
  }
  
  async getLatestSnapshot(aggregateId: string): Promise<Snapshot | null> {
    return this.snapshots.get(aggregateId) || null;
  }
}

// Aggregate ที่รองรับ Snapshots
abstract class SnapshottableAggregate {
  protected version: number = 0;
  private static readonly SNAPSHOT_THRESHOLD = 50;
  
  shouldTakeSnapshot(): boolean {
    return this.version % SnapshottableAggregate.SNAPSHOT_THRESHOLD === 0;
  }
  
  abstract takeSnapshot(): any;
  abstract restoreFromSnapshot(state: any): void;
}

// Repository ที่ใช้ Snapshots
class EventSourcedOrderRepository {
  constructor(
    private eventStore: EventStore,
    private snapshotStore: SnapshotStore,
    private eventDeserializer: EventDeserializer
  ) {}
  
  async findById(orderId: string): Promise<EventSourcedOrder | null> {
    // 1. ลองดึง snapshot ก่อน
    const snapshot = await this.snapshotStore.getLatestSnapshot(orderId);
    
    let order: EventSourcedOrder;
    let fromVersion = 0;
    
    if (snapshot) {
      // Restore จาก snapshot
      order = EventSourcedOrder.fromSnapshot(snapshot.state);
      fromVersion = snapshot.version;
      console.log(`Loading order ${orderId} from snapshot (v${fromVersion})`);
    } else {
      order = new EventSourcedOrder();
    }
    
    // 2. Replay events หลัง snapshot
    const events = await this.eventStore.getEvents(orderId, fromVersion);
    
    if (events.length === 0 && !snapshot) {
      return null;
    }
    
    for (const storedEvent of events) {
      const event = this.eventDeserializer.deserialize(storedEvent);
      order.apply(event);
    }
    
    console.log(`Loaded order ${orderId} (replayed ${events.length} events)`);
    return order;
  }
  
  async save(order: EventSourcedOrder): Promise<void> {
    const events = order.getPendingEvents();
    
    if (events.length === 0) return;
    
    // บันทึก events
    await this.eventStore.appendEvents(
      order.getId(),
      'Order',
      events,
      order.getVersion() - events.length
    );
    
    order.clearPendingEvents();
    
    // บันทึก snapshot ถ้าจำเป็น
    if (order.shouldTakeSnapshot()) {
      await this.snapshotStore.saveSnapshot({
        aggregateId: order.getId(),
        aggregateType: 'Order',
        state: order.takeSnapshot(),
        version: order.getVersion(),
        createdAt: new Date()
      });
      
      console.log(`Snapshot taken for order ${order.getId()} at version ${order.getVersion()}`);
    }
  }
}
```

---

## 9. Sagas และ Process Managers

```typescript
// Saga สำหรับจัดการ long-running processes
// ตัวอย่าง: Order Fulfillment Saga

interface SagaState {
  sagaId: string;
  orderId: string;
  step: string;
  status: 'RUNNING' | 'COMPLETED' | 'FAILED' | 'COMPENSATING';
  data: Record<string, any>;
  startedAt: Date;
  completedAt?: Date;
}

class OrderFulfillmentSaga {
  private state: SagaState;
  
  constructor(orderId: string) {
    this.state = {
      sagaId: `saga-${Date.now()}`,
      orderId,
      step: 'START',
      status: 'RUNNING',
      data: {},
      startedAt: new Date()
    };
  }
  
  // จัดการ events ที่เข้ามา
  async handleEvent(event: DomainEvent): Promise<void> {
    switch (event.getEventName()) {
      case 'OrderConfirmed':
        await this.handleOrderConfirmed(event as OrderConfirmedEvent);
        break;
      case 'PaymentProcessed':
        await this.handlePaymentProcessed(event as PaymentProcessedEvent);
        break;
      case 'PaymentFailed':
        await this.handlePaymentFailed(event as PaymentFailedEvent);
        break;
      case 'InventoryReserved':
        await this.handleInventoryReserved(event as InventoryReservedEvent);
        break;
      case 'InventoryReservationFailed':
        await this.handleInventoryFailed(event as InventoryReservationFailedEvent);
        break;
    }
  }
  
  private async handleOrderConfirmed(event: OrderConfirmedEvent): Promise<void> {
    this.state.step = 'PAYMENT';
    
    // ส่ง command ไปทำ payment
    await this.commandBus.dispatch(new ProcessPaymentCommand(
      this.state.orderId,
      event.totalAmount,
      event.currency
    ));
  }
  
  private async handlePaymentProcessed(event: PaymentProcessedEvent): Promise<void> {
    this.state.step = 'INVENTORY';
    this.state.data.paymentId = event.paymentId;
    
    // ส่ง command ไปจอง inventory
    await this.commandBus.dispatch(new ReserveInventoryCommand(
      this.state.orderId
    ));
  }
  
  private async handlePaymentFailed(event: PaymentFailedEvent): Promise<void> {
    this.state.status = 'FAILED';
    this.state.step = 'COMPENSATE_ORDER';
    
    // Compensating transaction: ยกเลิก order
    await this.commandBus.dispatch(new CancelOrderCommand(
      this.state.orderId,
      `Payment failed: ${event.reason}`
    ));
  }
  
  private async handleInventoryReserved(event: InventoryReservedEvent): Promise<void> {
    this.state.step = 'SHIPPING';
    this.state.data.reservationId = event.reservationId;
    
    // ส่ง command ไปจัดส่ง
    await this.commandBus.dispatch(new CreateShipmentCommand(
      this.state.orderId,
      event.reservationId
    ));
    
    this.state.status = 'COMPLETED';
    this.state.completedAt = new Date();
  }
  
  private async handleInventoryFailed(event: InventoryReservationFailedEvent): Promise<void> {
    this.state.status = 'COMPENSATING';
    this.state.step = 'COMPENSATE_PAYMENT';
    
    // Compensating transaction: คืนเงิน
    await this.commandBus.dispatch(new RefundPaymentCommand(
      this.state.data.paymentId,
      this.state.orderId
    ));
  }
  
  getState(): SagaState {
    return { ...this.state };
  }
  
  isComplete(): boolean {
    return ['COMPLETED', 'FAILED'].includes(this.state.status);
  }
}
```

---

## 10. Integration กับ NestJS

```typescript
// NestJS + CQRS ด้วย @nestjs/cqrs

// app.module.ts
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  imports: [CqrsModule],
  // ...
})
export class AppModule {}

// commands/create-order.command.ts
import { ICommand } from '@nestjs/cqrs';

export class CreateOrderCommand implements ICommand {
  constructor(
    public readonly customerId: string,
    public readonly items: Array<{ productId: string; quantity: number }>,
    public readonly shippingAddress: any
  ) {}
}

// handlers/create-order.handler.ts
import { CommandHandler, ICommandHandler, EventBus } from '@nestjs/cqrs';
import { Injectable } from '@nestjs/common';

@Injectable()
@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly eventBus: EventBus
  ) {}
  
  async execute(command: CreateOrderCommand): Promise<string> {
    const order = Order.create({
      id: `order-${Date.now()}`,
      customerId: command.customerId,
      items: command.items,
      shippingAddress: command.shippingAddress
    });
    
    order.confirm();
    await this.orderRepository.save(order);
    
    // Publish events ผ่าน EventBus ของ NestJS
    const events = order.getDomainEvents();
    events.forEach(event => this.eventBus.publish(event));
    
    return order.getId();
  }
}

// queries/get-order.query.ts
import { IQuery } from '@nestjs/cqrs';

export class GetOrderQuery implements IQuery {
  constructor(public readonly orderId: string) {}
}

// handlers/get-order.handler.ts
import { QueryHandler, IQueryHandler } from '@nestjs/cqrs';

@Injectable()
@QueryHandler(GetOrderQuery)
export class GetOrderQueryHandler implements IQueryHandler<GetOrderQuery> {
  constructor(private readonly readDb: ReadDatabase) {}
  
  async execute(query: GetOrderQuery): Promise<OrderDetailsView | null> {
    return this.readDb.findOne('order_details', { orderId: query.orderId });
  }
}

// events/order-confirmed.event.ts
import { IEvent } from '@nestjs/cqrs';

export class OrderConfirmedNestEvent implements IEvent {
  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly totalAmount: number
  ) {}
}

// sagas/order.saga.ts
import { Injectable } from '@nestjs/common';
import { ICommand, ofType, Saga } from '@nestjs/cqrs';
import { Observable } from 'rxjs';
import { map, filter } from 'rxjs/operators';

@Injectable()
export class OrderSaga {
  @Saga()
  orderConfirmed = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(OrderConfirmedNestEvent),
      map(event => new ProcessPaymentCommand(
        event.orderId,
        event.totalAmount
      ))
    );
  }
}

// order.controller.ts
import { Controller, Post, Get, Body, Param } from '@nestjs/common';
import { CommandBus, QueryBus } from '@nestjs/cqrs';

@Controller('orders')
export class OrderController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly queryBus: QueryBus
  ) {}
  
  @Post()
  async createOrder(@Body() dto: CreateOrderDto): Promise<{ orderId: string }> {
    const orderId = await this.commandBus.execute(
      new CreateOrderCommand(dto.customerId, dto.items, dto.shippingAddress)
    );
    return { orderId };
  }
  
  @Get(':id')
  async getOrder(@Param('id') id: string): Promise<OrderDetailsView | null> {
    return this.queryBus.execute(new GetOrderQuery(id));
  }
  
  @Post(':id/confirm')
  async confirmOrder(@Param('id') id: string): Promise<void> {
    await this.commandBus.execute(new ConfirmOrderCommand(id, 'admin'));
  }
}
```

---

## 11. EventStoreDB Overview

```typescript
// การใช้งาน EventStoreDB กับ TypeScript
// npm install @eventstore/db-client

import {
  EventStoreDBClient,
  jsonEvent,
  JSONEventType,
  ResolvedEvent,
  START,
  FORWARDS
} from '@eventstore/db-client';

// สร้าง client
const client = EventStoreDBClient.connectionString(
  'esdb://localhost:2113?tls=false'
);

// กำหนด Event Types
type OrderCreatedESEvent = JSONEventType<
  'OrderCreated',
  {
    orderId: string;
    customerId: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
    totalAmount: number;
  }
>;

type OrderConfirmedESEvent = JSONEventType<
  'OrderConfirmed',
  {
    orderId: string;
    confirmedBy: string;
    confirmedAt: string;
  }
>;

type OrderEvent = OrderCreatedESEvent | OrderConfirmedESEvent;

// EventStoreDB Repository
class ESDBOrderRepository {
  private getStreamId(orderId: string): string {
    return `Order-${orderId}`;
  }
  
  // บันทึก events
  async appendEvents(
    orderId: string,
    events: DomainEvent[],
    expectedRevision: bigint | 'no_stream' | 'any'
  ): Promise<void> {
    const streamId = this.getStreamId(orderId);
    
    const esEvents = events.map(event => jsonEvent({
      type: event.getEventName(),
      data: this.serializeEvent(event),
      metadata: {
        occurredAt: event.occurredAt.toISOString(),
        eventId: event.eventId
      }
    }));
    
    await client.appendToStream(streamId, esEvents, {
      expectedRevision
    });
  }
  
  // อ่าน events
  async getEvents(orderId: string): Promise<OrderEvent[]> {
    const streamId = this.getStreamId(orderId);
    
    const events = client.readStream<OrderEvent>(streamId, {
      direction: FORWARDS,
      fromRevision: START,
      maxCount: 1000
    });
    
    const result: OrderEvent[] = [];
    for await (const { event } of events) {
      if (event) {
        result.push(event);
      }
    }
    
    return result;
  }
  
  // Subscribe to events (live)
  subscribeToOrderEvents(
    handler: (event: OrderEvent) => Promise<void>
  ): () => void {
    const subscription = client.subscribeToStream<OrderEvent>(
      '$ce-Order',  // Category stream
      {
        fromRevision: 'start',
        resolveLinkTos: true
      }
    );
    
    (async () => {
      for await (const { event } of subscription) {
        if (event) {
          await handler(event);
        }
      }
    })();
    
    return () => subscription.unsubscribe();
  }
  
  private serializeEvent(event: DomainEvent): any {
    return JSON.parse(JSON.stringify(event));
  }
}
```

---

## 12. Complete Example

```typescript
// ตัวอย่าง Event Sourcing + CQRS แบบสมบูรณ์

// Setup
async function setupEventSourcingCQRS() {
  // Infrastructure
  const eventStore = new PostgresEventStore(db);
  const snapshotStore = new PostgresSnapshotStore(db);
  const readDb = new PostgresReadDatabase(readDbConnection);
  const eventBus = new EventBus();
  
  // Deserializer
  const deserializer = new EventDeserializer();
  deserializer.register('OrderCreated', data => new OrderCreatedEvent(
    data.orderId, data.customerId, data.items
  ));
  
  // Repository
  const orderRepo = new EventSourcedOrderRepository(
    eventStore, snapshotStore, deserializer
  );
  
  // Projections
  const orderSummaryProjection = new OrderSummaryProjection(readDb);
  const orderDetailsProjection = new OrderDetailsProjection(readDb, customerRepo);
  
  // Subscribe projections to events
  eventBus.subscribe('*', async (event) => {
    await orderSummaryProjection.handle(event);
    await orderDetailsProjection.handle(event);
  });
  
  // Command Handlers
  const commandBus = new CommandBus();
  commandBus.register(
    CreateOrderCommand,
    new CreateOrderCommandHandler(orderRepo, productRepo, customerRepo, eventBus)
  );
  commandBus.register(
    ConfirmOrderCommand,
    new ConfirmOrderCommandHandler(orderRepo, eventBus)
  );
  
  // Query Handlers
  const queryBus = new QueryBus();
  queryBus.register(
    GetOrderByIdQuery,
    new GetOrderDetailsQueryHandler(readDb)
  );
  queryBus.register(
    GetOrdersByCustomerQuery,
    new GetOrdersByCustomerQueryHandler(readDb)
  );
  
  return { commandBus, queryBus };
}

// การใช้งาน
async function demonstrateEventSourcing() {
  const { commandBus, queryBus } = await setupEventSourcingCQRS();
  
  // 1. สร้าง Order
  const orderId = await commandBus.dispatch(new CreateOrderCommand(
    'cust-001',
    [{ productId: 'prod-001', quantity: 2 }],
    { street: '123 ถนนสุขุมวิท', city: 'กรุงเทพฯ', postalCode: '10110', country: 'TH' }
  ));
  
  console.log(`Order created: ${orderId}`);
  
  // 2. Confirm Order
  await commandBus.dispatch(new ConfirmOrderCommand(orderId, 'system'));
  console.log('Order confirmed');
  
  // 3. Query Order
  const orderDetails = await queryBus.dispatch(new GetOrderByIdQuery(orderId));
  console.log('Order details:', orderDetails);
  
  // 4. Query Customer Orders
  const customerOrders = await queryBus.dispatch(
    new GetOrdersByCustomerQuery('cust-001', 1, 10)
  );
  console.log(`Customer has ${customerOrders.total} orders`);
}
```

---

## สรุป

Event Sourcing และ CQRS เป็นสถาปัตยกรรมที่ทรงพลังสำหรับระบบที่ซับซ้อน:

1. **CQRS** แยก Command (เขียน) และ Query (อ่าน) ออกจากกัน ทำให้สามารถ optimize แต่ละฝั่งได้อิสระ
2. **Event Sourcing** เก็บ events แทน state ทำให้มี audit trail สมบูรณ์ และสามารถ replay ได้
3. **Projections** แปลง events เป็น Read Models ที่ optimize สำหรับ query
4. **Snapshots** ช่วยลดเวลา replay สำหรับ aggregates ที่มี events จำนวนมาก
5. **Sagas** จัดการ distributed transactions และ long-running processes
