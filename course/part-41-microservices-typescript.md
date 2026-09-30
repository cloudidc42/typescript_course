# ตอนที่ 41: Microservices กับ TypeScript

## บทนำ

Microservices architecture คือรูปแบบการออกแบบซอฟต์แวร์ที่แบ่งแอปพลิเคชันออกเป็น services ขนาดเล็กที่ทำงานอิสระและสื่อสารกันผ่าน network TypeScript เหมาะอย่างยิ่งกับ Microservices เนื่องจาก type safety ช่วยลดข้อผิดพลาดในการสื่อสารระหว่าง services

---

## 41.1 Microservices Architecture Overview

### ข้อดีของ Microservices

```
แอปพลิเคชัน Monolith:
┌─────────────────────────────────────┐
│           Monolith App              │
│  ┌────────┐ ┌────────┐ ┌────────┐  │
│  │ Users  │ │ Orders │ │Payment │  │
│  └────────┘ └────────┘ └────────┘  │
│  ┌────────────────────────────────┐ │
│  │         Database               │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘

แอปพลิเคชัน Microservices:
┌──────────┐    ┌──────────┐    ┌──────────┐
│  User    │    │  Order   │    │ Payment  │
│ Service  │◄──►│ Service  │◄──►│ Service  │
│          │    │          │    │          │
│  DB      │    │  DB      │    │  DB      │
└──────────┘    └──────────┘    └──────────┘
     ▲                ▲               ▲
     └────────────────┴───────────────┘
              Message Broker
```

### โครงสร้างโปรเจค

```
microservices-app/
├── packages/
│   ├── shared/          # Shared types & utilities
│   ├── api-gateway/     # API Gateway service
│   ├── user-service/    # User management
│   ├── order-service/   # Order processing
│   ├── payment-service/ # Payment processing
│   └── notification-service/ # Notifications
├── docker-compose.yml
├── package.json         # Root package.json (monorepo)
└── tsconfig.base.json
```

### Shared Types Package

```typescript
// packages/shared/src/types/index.ts

// Base entity
export interface BaseEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

// User types
export interface User extends BaseEntity {
  name: string;
  email: string;
  role: UserRole;
}

export enum UserRole {
  ADMIN = 'ADMIN',
  USER = 'USER',
}

// Order types
export interface Order extends BaseEntity {
  userId: string;
  items: OrderItem[];
  total: number;
  status: OrderStatus;
}

export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

export enum OrderStatus {
  PENDING = 'PENDING',
  CONFIRMED = 'CONFIRMED',
  PROCESSING = 'PROCESSING',
  SHIPPED = 'SHIPPED',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED',
}

// Message patterns
export interface MessagePattern {
  cmd: string;
}

export interface EventPattern {
  event: string;
}

// Service responses
export interface ServiceResponse<T> {
  success: boolean;
  data?: T;
  error?: ServiceError;
}

export interface ServiceError {
  code: string;
  message: string;
  details?: Record<string, any>;
}
```

---

## 41.2 NestJS Microservices

### การติดตั้ง

```bash
npm install @nestjs/microservices @nestjs/common @nestjs/core
npm install -D @types/node typescript ts-node
```

### User Service

```typescript
// packages/user-service/src/main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { UserModule } from './user.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    UserModule,
    {
      transport: Transport.TCP,
      options: {
        host: '0.0.0.0',
        port: 3001,
      },
    }
  );

  await app.listen();
  console.log('🚀 User Microservice is running on port 3001');
}

bootstrap().catch(console.error);
```

```typescript
// packages/user-service/src/user.module.ts
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';

@Module({
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

```typescript
// packages/user-service/src/user.controller.ts
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { UserService } from './user.service';
import {
  CreateUserDto,
  UpdateUserDto,
  FindUserDto,
} from './dto';

@Controller()
export class UserController {
  constructor(private readonly userService: UserService) {}

  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() data: FindUserDto) {
    return this.userService.findById(data.id);
  }

  @MessagePattern({ cmd: 'get_all_users' })
  async getAllUsers() {
    return this.userService.findAll();
  }

  @MessagePattern({ cmd: 'create_user' })
  async createUser(@Payload() data: CreateUserDto) {
    return this.userService.create(data);
  }

  @MessagePattern({ cmd: 'update_user' })
  async updateUser(@Payload() data: { id: string; input: UpdateUserDto }) {
    return this.userService.update(data.id, data.input);
  }

  @MessagePattern({ cmd: 'delete_user' })
  async deleteUser(@Payload() data: { id: string }) {
    return this.userService.delete(data.id);
  }

  @MessagePattern({ cmd: 'validate_user' })
  async validateUser(@Payload() data: { email: string; password: string }) {
    return this.userService.validateCredentials(data.email, data.password);
  }
}
```

```typescript
// packages/user-service/src/user.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid';
import { hash, compare } from 'bcryptjs';
import { User, UserRole } from '@myapp/shared';
import { CreateUserDto, UpdateUserDto } from './dto';

@Injectable()
export class UserService {
  private users = new Map<string, User & { password: string }>();

  async findById(id: string): Promise<User | null> {
    const user = this.users.get(id);
    if (!user) return null;

    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }

  async findAll(): Promise<User[]> {
    return Array.from(this.users.values()).map(({ password, ...user }) => user);
  }

  async findByEmail(email: string): Promise<(User & { password: string }) | null> {
    return Array.from(this.users.values()).find(u => u.email === email) || null;
  }

  async create(data: CreateUserDto): Promise<User> {
    const existing = await this.findByEmail(data.email);
    if (existing) {
      throw new Error('อีเมลนี้ถูกใช้งานแล้ว');
    }

    const hashedPassword = await hash(data.password, 12);
    const user: User & { password: string } = {
      id: uuidv4(),
      name: data.name,
      email: data.email,
      password: hashedPassword,
      role: UserRole.USER,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.users.set(user.id, user);

    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }

  async update(id: string, data: UpdateUserDto): Promise<User | null> {
    const user = this.users.get(id);
    if (!user) return null;

    const updated = {
      ...user,
      ...data,
      updatedAt: new Date(),
    };

    this.users.set(id, updated);

    const { password, ...userWithoutPassword } = updated;
    return userWithoutPassword;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }

  async validateCredentials(
    email: string,
    password: string
  ): Promise<User | null> {
    const user = await this.findByEmail(email);
    if (!user) return null;

    const isValid = await compare(password, user.password);
    if (!isValid) return null;

    const { password: _, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }
}
```

---

## 41.3 Message Patterns

### Pattern ต่างๆ ใน NestJS Microservices

```typescript
// Request-Response Pattern
@MessagePattern({ cmd: 'get_user' })
async getUser(@Payload() data: { id: string }) {
  // Returns a value - caller waits for response
  return this.userService.findById(data.id);
}

// Event Pattern (Fire and forget)
@EventPattern('user_created')
async handleUserCreated(@Payload() data: UserCreatedEvent) {
  // No response needed
  await this.notificationService.sendWelcomeEmail(data.email);
}
```

### DTO กับ Validation

```typescript
// packages/user-service/src/dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, IsOptional, IsEnum } from 'class-validator';
import { UserRole } from '@myapp/shared';

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  name!: string;

  @IsEmail()
  email!: string;

  @IsString()
  @MinLength(8)
  password!: string;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;
}

export class UpdateUserDto {
  @IsOptional()
  @IsString()
  @MinLength(2)
  name?: string;

  @IsOptional()
  @IsEmail()
  email?: string;
}

export class FindUserDto {
  @IsString()
  id!: string;
}
```

---

## 41.4 TCP Transport

### API Gateway กับ TCP Client

```typescript
// packages/api-gateway/src/app.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { ApiController } from './api.controller';
import { UserClientService } from './clients/user-client.service';
import { OrderClientService } from './clients/order-client.service';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USER_SERVICE',
        transport: Transport.TCP,
        options: {
          host: process.env.USER_SERVICE_HOST || 'localhost',
          port: parseInt(process.env.USER_SERVICE_PORT || '3001'),
        },
      },
      {
        name: 'ORDER_SERVICE',
        transport: Transport.TCP,
        options: {
          host: process.env.ORDER_SERVICE_HOST || 'localhost',
          port: parseInt(process.env.ORDER_SERVICE_PORT || '3002'),
        },
      },
      {
        name: 'PAYMENT_SERVICE',
        transport: Transport.TCP,
        options: {
          host: process.env.PAYMENT_SERVICE_HOST || 'localhost',
          port: parseInt(process.env.PAYMENT_SERVICE_PORT || '3003'),
        },
      },
    ]),
  ],
  controllers: [ApiController],
  providers: [UserClientService, OrderClientService],
})
export class AppModule {}
```

```typescript
// packages/api-gateway/src/clients/user-client.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { firstValueFrom, timeout } from 'rxjs';
import { User } from '@myapp/shared';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UserClientService {
  constructor(
    @Inject('USER_SERVICE') private readonly client: ClientProxy
  ) {}

  async getUser(id: string): Promise<User | null> {
    return firstValueFrom(
      this.client
        .send<User | null>({ cmd: 'get_user' }, { id })
        .pipe(timeout(5000))
    );
  }

  async getAllUsers(): Promise<User[]> {
    return firstValueFrom(
      this.client
        .send<User[]>({ cmd: 'get_all_users' }, {})
        .pipe(timeout(5000))
    );
  }

  async createUser(data: CreateUserDto): Promise<User> {
    return firstValueFrom(
      this.client
        .send<User>({ cmd: 'create_user' }, data)
        .pipe(timeout(5000))
    );
  }

  async validateUser(email: string, password: string): Promise<User | null> {
    return firstValueFrom(
      this.client
        .send<User | null>({ cmd: 'validate_user' }, { email, password })
        .pipe(timeout(5000))
    );
  }
}
```

### API Controller

```typescript
// packages/api-gateway/src/api.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  UseGuards,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { UserClientService } from './clients/user-client.service';
import { OrderClientService } from './clients/order-client.service';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { CurrentUser } from './decorators/current-user.decorator';
import { User } from '@myapp/shared';

@Controller('api')
export class ApiController {
  constructor(
    private readonly userService: UserClientService,
    private readonly orderService: OrderClientService,
  ) {}

  // User endpoints
  @Get('users')
  @UseGuards(JwtAuthGuard)
  async getUsers() {
    return this.userService.getAllUsers();
  }

  @Get('users/:id')
  async getUser(@Param('id') id: string) {
    const user = await this.userService.getUser(id);
    if (!user) {
      return { error: 'ไม่พบผู้ใช้' };
    }
    return user;
  }

  @Post('users')
  @HttpCode(HttpStatus.CREATED)
  async createUser(@Body() body: any) {
    return this.userService.createUser(body);
  }

  // Order endpoints
  @Get('orders')
  @UseGuards(JwtAuthGuard)
  async getUserOrders(@CurrentUser() user: User) {
    return this.orderService.getOrdersByUser(user.id);
  }

  @Post('orders')
  @UseGuards(JwtAuthGuard)
  @HttpCode(HttpStatus.CREATED)
  async createOrder(
    @Body() body: any,
    @CurrentUser() user: User
  ) {
    return this.orderService.createOrder({
      ...body,
      userId: user.id,
    });
  }
}
```

---

## 41.5 Redis Transport

### การตั้งค่า Redis Transport

```bash
npm install ioredis
npm install -D @types/ioredis
```

```typescript
// packages/notification-service/src/main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { NotificationModule } from './notification.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    NotificationModule,
    {
      transport: Transport.REDIS,
      options: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT || '6379'),
        password: process.env.REDIS_PASSWORD,
        retryAttempts: 5,
        retryDelay: 3000,
      },
    }
  );

  await app.listen();
  console.log('📧 Notification Service ready (Redis)');
}

bootstrap();
```

```typescript
// packages/notification-service/src/notification.controller.ts
import { Controller } from '@nestjs/common';
import { EventPattern, Payload, MessagePattern } from '@nestjs/microservices';
import { NotificationService } from './notification.service';

@Controller()
export class NotificationController {
  constructor(private readonly notificationService: NotificationService) {}

  @EventPattern('user.created')
  async handleUserCreated(
    @Payload() data: { userId: string; email: string; name: string }
  ) {
    console.log(`📧 Processing welcome email for ${data.email}`);
    await this.notificationService.sendWelcomeEmail(data);
  }

  @EventPattern('order.confirmed')
  async handleOrderConfirmed(
    @Payload() data: { orderId: string; userId: string; items: any[] }
  ) {
    console.log(`📦 Sending order confirmation for order ${data.orderId}`);
    await this.notificationService.sendOrderConfirmation(data);
  }

  @EventPattern('payment.failed')
  async handlePaymentFailed(
    @Payload() data: { orderId: string; userId: string; reason: string }
  ) {
    console.log(`⚠️ Payment failed notification for order ${data.orderId}`);
    await this.notificationService.sendPaymentFailedNotification(data);
  }

  @MessagePattern('notification.send')
  async sendNotification(
    @Payload() data: { userId: string; type: string; message: string }
  ) {
    return this.notificationService.send(data);
  }
}
```

### Redis Event Publisher

```typescript
// packages/shared/src/events/event-publisher.ts
import { Injectable, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';

export interface EventPublisher {
  publish(event: string, data: any): void;
}

@Injectable()
export class RedisEventPublisher implements EventPublisher {
  constructor(
    @Inject('EVENT_BUS') private readonly client: ClientProxy
  ) {}

  publish(event: string, data: any): void {
    // Fire and forget
    this.client.emit(event, {
      ...data,
      timestamp: new Date().toISOString(),
    });
  }
}
```

---

## 41.6 RabbitMQ Integration

### การตั้งค่า RabbitMQ

```bash
npm install amqplib amqp-connection-manager
npm install -D @types/amqplib
```

```typescript
// packages/order-service/src/main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { OrderModule } from './order.module';

async function bootstrap() {
  // HTTP server สำหรับ health checks
  const app = await NestFactory.create(OrderModule);

  // RabbitMQ microservice
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.RMQ,
    options: {
      urls: [process.env.RABBITMQ_URL || 'amqp://localhost:5672'],
      queue: 'order_queue',
      queueOptions: {
        durable: true,
        arguments: {
          'x-message-ttl': 86400000, // 24 hours
          'x-dead-letter-exchange': 'order_dlx',
        },
      },
      prefetchCount: 10,
      noAck: false,
    },
  });

  await app.startAllMicroservices();
  await app.listen(3002);
  console.log('📦 Order Service running');
}

bootstrap();
```

```typescript
// packages/order-service/src/order.controller.ts
import { Controller } from '@nestjs/common';
import { MessagePattern, EventPattern, Payload, Ctx, RmqContext } from '@nestjs/microservices';
import { OrderService } from './order.service';
import { CreateOrderDto } from './dto';

@Controller()
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @MessagePattern('order.create')
  async createOrder(
    @Payload() data: CreateOrderDto,
    @Ctx() context: RmqContext
  ) {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();

    try {
      const order = await this.orderService.create(data);
      // Acknowledge message manually
      channel.ack(originalMsg);
      return order;
    } catch (error) {
      // Reject และส่งไป dead letter queue
      channel.nack(originalMsg, false, false);
      throw error;
    }
  }

  @EventPattern('payment.completed')
  async handlePaymentCompleted(
    @Payload() data: { orderId: string; transactionId: string },
    @Ctx() context: RmqContext
  ) {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();

    try {
      await this.orderService.confirmOrder(data.orderId, data.transactionId);
      channel.ack(originalMsg);
    } catch (error) {
      console.error('Failed to confirm order:', error);
      channel.nack(originalMsg, false, true); // requeue
    }
  }
}
```

---

## 41.7 Event-Driven Communication

### Event Bus Pattern

```typescript
// packages/shared/src/events/types.ts
export interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  aggregateType: string;
  timestamp: Date;
  version: number;
  payload: Record<string, any>;
}

// User Events
export interface UserCreatedEvent extends DomainEvent {
  eventType: 'USER_CREATED';
  payload: {
    userId: string;
    name: string;
    email: string;
    role: string;
  };
}

export interface UserUpdatedEvent extends DomainEvent {
  eventType: 'USER_UPDATED';
  payload: {
    userId: string;
    changes: Record<string, any>;
  };
}

// Order Events
export interface OrderCreatedEvent extends DomainEvent {
  eventType: 'ORDER_CREATED';
  payload: {
    orderId: string;
    userId: string;
    items: Array<{
      productId: string;
      quantity: number;
      price: number;
    }>;
    total: number;
  };
}

export interface OrderStatusChangedEvent extends DomainEvent {
  eventType: 'ORDER_STATUS_CHANGED';
  payload: {
    orderId: string;
    previousStatus: string;
    newStatus: string;
    reason?: string;
  };
}

// Payment Events
export interface PaymentProcessedEvent extends DomainEvent {
  eventType: 'PAYMENT_PROCESSED';
  payload: {
    paymentId: string;
    orderId: string;
    amount: number;
    currency: string;
    method: string;
    status: 'SUCCESS' | 'FAILED';
    transactionId?: string;
    failureReason?: string;
  };
}
```

### Event Store

```typescript
// packages/shared/src/events/event-store.ts
import { v4 as uuidv4 } from 'uuid';
import { DomainEvent } from './types';

export class EventStore {
  private events: DomainEvent[] = [];

  async save(
    aggregateId: string,
    aggregateType: string,
    eventType: string,
    payload: Record<string, any>
  ): Promise<DomainEvent> {
    const event: DomainEvent = {
      eventId: uuidv4(),
      eventType,
      aggregateId,
      aggregateType,
      timestamp: new Date(),
      version: this.getNextVersion(aggregateId),
      payload,
    };

    this.events.push(event);
    return event;
  }

  getEvents(aggregateId: string): DomainEvent[] {
    return this.events
      .filter(e => e.aggregateId === aggregateId)
      .sort((a, b) => a.version - b.version);
  }

  getAllEventsSince(timestamp: Date): DomainEvent[] {
    return this.events
      .filter(e => e.timestamp > timestamp)
      .sort((a, b) => a.timestamp.getTime() - b.timestamp.getTime());
  }

  private getNextVersion(aggregateId: string): number {
    const events = this.getEvents(aggregateId);
    return events.length > 0
      ? Math.max(...events.map(e => e.version)) + 1
      : 1;
  }
}
```

### Saga Pattern สำหรับ Distributed Transactions

```typescript
// packages/order-service/src/sagas/order.saga.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { ClientProxy } from '@nestjs/microservices';
import { Inject } from '@nestjs/common';
import { firstValueFrom } from 'rxjs';

export enum SagaStatus {
  PENDING = 'PENDING',
  PAYMENT_PROCESSING = 'PAYMENT_PROCESSING',
  PAYMENT_FAILED = 'PAYMENT_FAILED',
  INVENTORY_RESERVING = 'INVENTORY_RESERVING',
  COMPLETED = 'COMPLETED',
  COMPENSATING = 'COMPENSATING',
  FAILED = 'FAILED',
}

export interface OrderSagaState {
  sagaId: string;
  orderId: string;
  userId: string;
  items: any[];
  total: number;
  status: SagaStatus;
  paymentId?: string;
  reservationId?: string;
  createdAt: Date;
  updatedAt: Date;
}

@Injectable()
export class OrderSaga {
  private sagas = new Map<string, OrderSagaState>();

  constructor(
    @Inject('PAYMENT_SERVICE') private paymentClient: ClientProxy,
    @Inject('INVENTORY_SERVICE') private inventoryClient: ClientProxy,
    @Inject('NOTIFICATION_SERVICE') private notificationClient: ClientProxy,
  ) {}

  async startOrderSaga(orderData: {
    orderId: string;
    userId: string;
    items: any[];
    total: number;
    paymentMethod: string;
  }): Promise<void> {
    const sagaId = `saga_${orderData.orderId}`;
    
    const state: OrderSagaState = {
      sagaId,
      orderId: orderData.orderId,
      userId: orderData.userId,
      items: orderData.items,
      total: orderData.total,
      status: SagaStatus.PENDING,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.sagas.set(sagaId, state);

    try {
      // Step 1: Reserve inventory
      await this.updateSagaStatus(sagaId, SagaStatus.INVENTORY_RESERVING);
      const reservation = await firstValueFrom(
        this.inventoryClient.send('inventory.reserve', {
          orderId: orderData.orderId,
          items: orderData.items,
        })
      );

      state.reservationId = reservation.reservationId;
      this.sagas.set(sagaId, state);

      // Step 2: Process payment
      await this.updateSagaStatus(sagaId, SagaStatus.PAYMENT_PROCESSING);
      const payment = await firstValueFrom(
        this.paymentClient.send('payment.process', {
          orderId: orderData.orderId,
          userId: orderData.userId,
          amount: orderData.total,
          method: orderData.paymentMethod,
        })
      );

      if (payment.status === 'FAILED') {
        throw new Error(`Payment failed: ${payment.reason}`);
      }

      state.paymentId = payment.paymentId;

      // Step 3: Complete saga
      await this.updateSagaStatus(sagaId, SagaStatus.COMPLETED);
      
      // Send confirmation notification
      this.notificationClient.emit('order.confirmed', {
        orderId: orderData.orderId,
        userId: orderData.userId,
      });

    } catch (error) {
      console.error(`Saga ${sagaId} failed:`, error);
      await this.compensate(sagaId);
    }
  }

  private async compensate(sagaId: string): Promise<void> {
    const state = this.sagas.get(sagaId);
    if (!state) return;

    await this.updateSagaStatus(sagaId, SagaStatus.COMPENSATING);

    // ยกเลิก reservation ถ้ามี
    if (state.reservationId) {
      try {
        await firstValueFrom(
          this.inventoryClient.send('inventory.release', {
            reservationId: state.reservationId,
          })
        );
      } catch (error) {
        console.error('Failed to release inventory:', error);
      }
    }

    // ยกเลิก payment ถ้ามี
    if (state.paymentId) {
      try {
        await firstValueFrom(
          this.paymentClient.send('payment.refund', {
            paymentId: state.paymentId,
          })
        );
      } catch (error) {
        console.error('Failed to refund payment:', error);
      }
    }

    await this.updateSagaStatus(sagaId, SagaStatus.FAILED);

    // Send failure notification
    this.notificationClient.emit('order.failed', {
      orderId: state.orderId,
      userId: state.userId,
    });
  }

  private async updateSagaStatus(
    sagaId: string,
    status: SagaStatus
  ): Promise<void> {
    const state = this.sagas.get(sagaId);
    if (!state) return;

    state.status = status;
    state.updatedAt = new Date();
    this.sagas.set(sagaId, state);
    
    console.log(`Saga ${sagaId}: ${status}`);
  }
}
```

---

## 41.8 API Gateway Pattern

### Kong-style API Gateway

```typescript
// packages/api-gateway/src/gateway/gateway.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Request } from 'express';

interface ServiceConfig {
  name: string;
  host: string;
  port: number;
  healthEndpoint: string;
  timeout: number;
  retries: number;
}

interface RouteConfig {
  path: string;
  methods: string[];
  service: string;
  auth: boolean;
  rateLimit?: {
    windowMs: number;
    max: number;
  };
}

@Injectable()
export class GatewayService {
  private readonly logger = new Logger(GatewayService.name);

  private readonly services: Map<string, ServiceConfig> = new Map([
    ['users', {
      name: 'user-service',
      host: process.env.USER_SERVICE_HOST || 'localhost',
      port: 3001,
      healthEndpoint: '/health',
      timeout: 5000,
      retries: 3,
    }],
    ['orders', {
      name: 'order-service',
      host: process.env.ORDER_SERVICE_HOST || 'localhost',
      port: 3002,
      healthEndpoint: '/health',
      timeout: 10000,
      retries: 2,
    }],
    ['payments', {
      name: 'payment-service',
      host: process.env.PAYMENT_SERVICE_HOST || 'localhost',
      port: 3003,
      healthEndpoint: '/health',
      timeout: 15000,
      retries: 1,
    }],
  ]);

  private readonly routes: RouteConfig[] = [
    { path: '/api/users', methods: ['GET', 'POST'], service: 'users', auth: false },
    { path: '/api/users/:id', methods: ['GET', 'PUT', 'DELETE'], service: 'users', auth: true },
    { path: '/api/orders', methods: ['GET', 'POST'], service: 'orders', auth: true },
    { path: '/api/payments', methods: ['POST'], service: 'payments', auth: true },
  ];

  getServiceForRequest(req: Request): ServiceConfig | null {
    const route = this.routes.find(r => {
      const pathMatch = this.matchPath(req.path, r.path);
      const methodMatch = r.methods.includes(req.method);
      return pathMatch && methodMatch;
    });

    if (!route) return null;
    return this.services.get(route.service) || null;
  }

  private matchPath(requestPath: string, routePath: string): boolean {
    const routeParts = routePath.split('/');
    const requestParts = requestPath.split('/');

    if (routeParts.length !== requestParts.length) return false;

    return routeParts.every((part, i) => {
      if (part.startsWith(':')) return true;
      return part === requestParts[i];
    });
  }

  async checkServiceHealth(serviceName: string): Promise<boolean> {
    const service = this.services.get(serviceName);
    if (!service) return false;

    try {
      const response = await fetch(
        `http://${service.host}:${service.port}${service.healthEndpoint}`,
        { signal: AbortSignal.timeout(2000) }
      );
      return response.ok;
    } catch {
      return false;
    }
  }

  async getAllServicesHealth(): Promise<Record<string, boolean>> {
    const healthChecks: Record<string, boolean> = {};

    await Promise.all(
      Array.from(this.services.entries()).map(async ([name, _]) => {
        healthChecks[name] = await this.checkServiceHealth(name);
      })
    );

    return healthChecks;
  }
}
```

---

## 41.9 Circuit Breaker Pattern

```typescript
// packages/shared/src/patterns/circuit-breaker.ts

export enum CircuitState {
  CLOSED = 'CLOSED',    // ปกติ
  OPEN = 'OPEN',        // ปิดวงจร (fail fast)
  HALF_OPEN = 'HALF_OPEN', // ทดสอบ
}

export interface CircuitBreakerConfig {
  failureThreshold: number;    // จำนวนครั้งที่ fail ก่อน open
  successThreshold: number;    // จำนวนครั้งที่ success ใน half-open
  timeout: number;             // ms ก่อน half-open
  requestTimeout: number;      // timeout ต่อ request
}

export class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failures = 0;
  private successes = 0;
  private lastFailureTime: Date | null = null;
  private readonly config: CircuitBreakerConfig;

  constructor(
    private readonly name: string,
    config: Partial<CircuitBreakerConfig> = {}
  ) {
    this.config = {
      failureThreshold: 5,
      successThreshold: 2,
      timeout: 30000,        // 30 seconds
      requestTimeout: 5000,  // 5 seconds
      ...config,
    };
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      const timeSinceLastFailure = this.lastFailureTime
        ? Date.now() - this.lastFailureTime.getTime()
        : Infinity;

      if (timeSinceLastFailure > this.config.timeout) {
        console.log(`[CircuitBreaker:${this.name}] Moving to HALF_OPEN`);
        this.state = CircuitState.HALF_OPEN;
        this.successes = 0;
      } else {
        throw new Error(
          `Circuit breaker ${this.name} is OPEN. ` +
          `Retry after ${Math.ceil((this.config.timeout - timeSinceLastFailure) / 1000)}s`
        );
      }
    }

    try {
      const result = await Promise.race([
        fn(),
        new Promise<never>((_, reject) =>
          setTimeout(
            () => reject(new Error(`Request timeout after ${this.config.requestTimeout}ms`)),
            this.config.requestTimeout
          )
        ),
      ]);

      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.successes++;
      if (this.successes >= this.config.successThreshold) {
        console.log(`[CircuitBreaker:${this.name}] Moving to CLOSED`);
        this.state = CircuitState.CLOSED;
        this.successes = 0;
      }
    }
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = new Date();

    if (
      this.state === CircuitState.CLOSED &&
      this.failures >= this.config.failureThreshold
    ) {
      console.log(`[CircuitBreaker:${this.name}] Moving to OPEN after ${this.failures} failures`);
      this.state = CircuitState.OPEN;
    } else if (this.state === CircuitState.HALF_OPEN) {
      console.log(`[CircuitBreaker:${this.name}] Back to OPEN`);
      this.state = CircuitState.OPEN;
      this.successes = 0;
    }
  }

  getState(): CircuitState {
    return this.state;
  }

  getStats() {
    return {
      name: this.name,
      state: this.state,
      failures: this.failures,
      successes: this.successes,
      lastFailureTime: this.lastFailureTime,
    };
  }
}

// Factory
export class CircuitBreakerFactory {
  private static breakers = new Map<string, CircuitBreaker>();

  static get(name: string, config?: Partial<CircuitBreakerConfig>): CircuitBreaker {
    if (!this.breakers.has(name)) {
      this.breakers.set(name, new CircuitBreaker(name, config));
    }
    return this.breakers.get(name)!;
  }

  static getAll(): CircuitBreaker[] {
    return Array.from(this.breakers.values());
  }
}

// การใช้งาน
export async function callWithCircuitBreaker<T>(
  serviceName: string,
  fn: () => Promise<T>
): Promise<T> {
  const breaker = CircuitBreakerFactory.get(serviceName, {
    failureThreshold: 5,
    timeout: 30000,
  });

  return breaker.execute(fn);
}
```

---

## 41.10 Health Checks

```typescript
// packages/shared/src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HealthCheck, HealthIndicator } from '@nestjs/terminus';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
  ) {}

  @Get()
  @HealthCheck()
  async check() {
    return this.health.check([
      // Database health
      async () => {
        try {
          // Check DB connection here
          return { database: { status: 'up' } };
        } catch {
          return { database: { status: 'down' } };
        }
      },

      // Memory health
      async () => {
        const used = process.memoryUsage();
        const heapUsedMB = used.heapUsed / 1024 / 1024;
        const isHealthy = heapUsedMB < 500; // 500 MB threshold

        return {
          memory: {
            status: isHealthy ? 'up' : 'down',
            heapUsedMB: Math.round(heapUsedMB),
          },
        };
      },

      // External service health
      async () => {
        try {
          const response = await fetch('https://api.example.com/health', {
            signal: AbortSignal.timeout(2000),
          });
          return { externalService: { status: response.ok ? 'up' : 'down' } };
        } catch {
          return { externalService: { status: 'down' } };
        }
      },
    ]);
  }

  @Get('ready')
  async readiness() {
    return {
      status: 'ready',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    };
  }

  @Get('live')
  async liveness() {
    return {
      status: 'alive',
      pid: process.pid,
      timestamp: new Date().toISOString(),
    };
  }
}
```

---

## 41.11 Service Discovery

```typescript
// packages/shared/src/discovery/service-registry.ts

export interface ServiceInstance {
  id: string;
  name: string;
  host: string;
  port: number;
  metadata: Record<string, string>;
  health: 'healthy' | 'unhealthy' | 'unknown';
  registeredAt: Date;
  lastHeartbeat: Date;
}

export class ServiceRegistry {
  private instances = new Map<string, ServiceInstance[]>();
  private heartbeatInterval: NodeJS.Timer | null = null;

  register(instance: Omit<ServiceInstance, 'registeredAt' | 'lastHeartbeat'>): void {
    const fullInstance: ServiceInstance = {
      ...instance,
      registeredAt: new Date(),
      lastHeartbeat: new Date(),
    };

    const existing = this.instances.get(instance.name) || [];
    existing.push(fullInstance);
    this.instances.set(instance.name, existing);

    console.log(`Service registered: ${instance.name}@${instance.host}:${instance.port}`);
  }

  deregister(serviceId: string): void {
    for (const [name, instances] of this.instances.entries()) {
      const filtered = instances.filter(i => i.id !== serviceId);
      if (filtered.length !== instances.length) {
        this.instances.set(name, filtered);
        console.log(`Service deregistered: ${serviceId}`);
      }
    }
  }

  discover(serviceName: string): ServiceInstance[] {
    return (this.instances.get(serviceName) || []).filter(
      i => i.health === 'healthy'
    );
  }

  // Round-robin load balancing
  getNextInstance(serviceName: string): ServiceInstance | null {
    const healthy = this.discover(serviceName);
    if (healthy.length === 0) return null;

    // Simple round-robin
    const index = Date.now() % healthy.length;
    return healthy[index];
  }

  heartbeat(serviceId: string): void {
    for (const instances of this.instances.values()) {
      const instance = instances.find(i => i.id === serviceId);
      if (instance) {
        instance.lastHeartbeat = new Date();
        instance.health = 'healthy';
      }
    }
  }

  startHealthCheck(intervalMs = 30000): void {
    this.heartbeatInterval = setInterval(() => {
      const now = Date.now();
      const timeout = intervalMs * 3; // 3 missed heartbeats

      for (const instances of this.instances.values()) {
        instances.forEach(instance => {
          const timeSinceHeartbeat = now - instance.lastHeartbeat.getTime();
          if (timeSinceHeartbeat > timeout) {
            instance.health = 'unhealthy';
          }
        });
      }
    }, intervalMs);
  }

  stopHealthCheck(): void {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval as NodeJS.Timeout);
    }
  }

  getAllServices(): Record<string, ServiceInstance[]> {
    const result: Record<string, ServiceInstance[]> = {};
    this.instances.forEach((instances, name) => {
      result[name] = instances;
    });
    return result;
  }
}

export const registry = new ServiceRegistry();
```

---

## 41.12 Distributed Tracing

```typescript
// packages/shared/src/tracing/tracer.ts
import { v4 as uuidv4 } from 'uuid';

export interface Span {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  operationName: string;
  serviceName: string;
  startTime: Date;
  endTime?: Date;
  duration?: number;
  tags: Record<string, any>;
  logs: SpanLog[];
  status: 'ok' | 'error';
}

export interface SpanLog {
  timestamp: Date;
  level: 'info' | 'warn' | 'error';
  message: string;
  data?: Record<string, any>;
}

export class Tracer {
  constructor(private readonly serviceName: string) {}

  startSpan(
    operationName: string,
    parentContext?: { traceId: string; spanId: string }
  ): ActiveSpan {
    const span: Span = {
      traceId: parentContext?.traceId || uuidv4(),
      spanId: uuidv4(),
      parentSpanId: parentContext?.spanId,
      operationName,
      serviceName: this.serviceName,
      startTime: new Date(),
      tags: {},
      logs: [],
      status: 'ok',
    };

    return new ActiveSpan(span, this);
  }

  finish(span: Span): void {
    span.endTime = new Date();
    span.duration = span.endTime.getTime() - span.startTime.getTime();

    // ส่ง span ไปยัง tracing backend (Jaeger, Zipkin, etc.)
    this.export(span);
  }

  private export(span: Span): void {
    // TODO: ส่งไปยัง Jaeger หรือ Zipkin
    console.log(`[Trace] ${span.traceId}/${span.spanId} ${span.operationName} ${span.duration}ms`);
  }
}

export class ActiveSpan {
  constructor(
    private span: Span,
    private tracer: Tracer
  ) {}

  setTag(key: string, value: any): this {
    this.span.tags[key] = value;
    return this;
  }

  log(level: SpanLog['level'], message: string, data?: Record<string, any>): this {
    this.span.logs.push({
      timestamp: new Date(),
      level,
      message,
      data,
    });
    return this;
  }

  setError(error: Error): this {
    this.span.status = 'error';
    this.span.tags['error'] = true;
    this.span.tags['error.message'] = error.message;
    this.log('error', error.message, { stack: error.stack });
    return this;
  }

  finish(): void {
    this.tracer.finish(this.span);
  }

  getContext(): { traceId: string; spanId: string } {
    return {
      traceId: this.span.traceId,
      spanId: this.span.spanId,
    };
  }
}

// Middleware สำหรับ NestJS
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class TracingMiddleware implements NestMiddleware {
  private tracer = new Tracer('api-gateway');

  use(req: Request, res: Response, next: NextFunction) {
    // ดึง trace context จาก headers
    const parentTraceId = req.headers['x-trace-id'] as string;
    const parentSpanId = req.headers['x-span-id'] as string;

    const parentContext = parentTraceId
      ? { traceId: parentTraceId, spanId: parentSpanId }
      : undefined;

    const span = this.tracer.startSpan(
      `${req.method} ${req.path}`,
      parentContext
    );

    span
      .setTag('http.method', req.method)
      .setTag('http.url', req.path)
      .setTag('http.user_agent', req.headers['user-agent']);

    // ใส่ trace context ใน request
    (req as any).traceContext = span.getContext();

    // ส่ง trace headers ไปยัง downstream services
    res.setHeader('x-trace-id', span.getContext().traceId);

    res.on('finish', () => {
      span
        .setTag('http.status_code', res.statusCode)
        .finish();
    });

    next();
  }
}
```

---

## 41.13 Docker Compose สำหรับ Microservices

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Message Broker
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Cache & Session Store
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # User Service
  user-service:
    build:
      context: ./packages/user-service
      dockerfile: Dockerfile
    container_name: user-service
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - PORT=3001
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/users
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Order Service
  order-service:
    build:
      context: ./packages/order-service
      dockerfile: Dockerfile
    container_name: order-service
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=production
      - PORT=3002
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/orders
      - RABBITMQ_URL=amqp://admin:password@rabbitmq:5672
      - USER_SERVICE_HOST=user-service
      - USER_SERVICE_PORT=3001
    depends_on:
      - postgres
      - rabbitmq
    restart: unless-stopped

  # Payment Service
  payment-service:
    build:
      context: ./packages/payment-service
      dockerfile: Dockerfile
    container_name: payment-service
    ports:
      - "3003:3003"
    environment:
      - NODE_ENV=production
      - RABBITMQ_URL=amqp://admin:password@rabbitmq:5672
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
    depends_on:
      - rabbitmq
    restart: unless-stopped

  # Notification Service
  notification-service:
    build:
      context: ./packages/notification-service
      dockerfile: Dockerfile
    container_name: notification-service
    environment:
      - RABBITMQ_URL=amqp://admin:password@rabbitmq:5672
      - REDIS_HOST=redis
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
    depends_on:
      - rabbitmq
      - redis
    restart: unless-stopped

  # API Gateway
  api-gateway:
    build:
      context: ./packages/api-gateway
      dockerfile: Dockerfile
    container_name: api-gateway
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - USER_SERVICE_HOST=user-service
      - USER_SERVICE_PORT=3001
      - ORDER_SERVICE_HOST=order-service
      - ORDER_SERVICE_PORT=3002
      - PAYMENT_SERVICE_HOST=payment-service
      - PAYMENT_SERVICE_PORT=3003
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - user-service
      - order-service
      - payment-service
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Database
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_MULTIPLE_DATABASES: users,orders,products
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sh:/docker-entrypoint-initdb.d/init-db.sh
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3005:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  grafana_data:

networks:
  default:
    name: microservices-network
```

---

## 41.14 Metrics กับ Prometheus

```typescript
// packages/shared/src/metrics/metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Counter, Histogram, Gauge, register } from 'prom-client';

@Injectable()
export class MetricsService {
  private httpRequestCounter: Counter;
  private httpRequestDuration: Histogram;
  private activeConnections: Gauge;
  private errorCounter: Counter;

  constructor(serviceName: string) {
    this.httpRequestCounter = new Counter({
      name: `${serviceName}_http_requests_total`,
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'path', 'status'],
    });

    this.httpRequestDuration = new Histogram({
      name: `${serviceName}_http_request_duration_seconds`,
      help: 'HTTP request duration in seconds',
      labelNames: ['method', 'path', 'status'],
      buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
    });

    this.activeConnections = new Gauge({
      name: `${serviceName}_active_connections`,
      help: 'Number of active connections',
    });

    this.errorCounter = new Counter({
      name: `${serviceName}_errors_total`,
      help: 'Total number of errors',
      labelNames: ['type'],
    });
  }

  recordRequest(method: string, path: string, status: number, duration: number): void {
    this.httpRequestCounter.inc({ method, path, status: status.toString() });
    this.httpRequestDuration.observe(
      { method, path, status: status.toString() },
      duration / 1000
    );
  }

  incrementConnections(): void {
    this.activeConnections.inc();
  }

  decrementConnections(): void {
    this.activeConnections.dec();
  }

  recordError(type: string): void {
    this.errorCounter.inc({ type });
  }

  async getMetrics(): Promise<string> {
    return register.metrics();
  }
}
```

---

## สรุป

Microservices กับ TypeScript มีข้อดีหลายประการ:

1. **Type Safety ข้ามบริการ** - Shared types ทำให้ communication ปลอดภัย
2. **NestJS Microservices** - Framework ที่รองรับ pattern ต่างๆ ครบครัน
3. **Message Patterns** - TCP, Redis, RabbitMQ ตามความต้องการ
4. **Saga Pattern** - จัดการ distributed transactions
5. **Circuit Breaker** - Resilience pattern สำคัญ
6. **Service Discovery** - Dynamic service registration
7. **Distributed Tracing** - ติดตาม requests ข้ามบริการ
8. **Docker Compose** - ง่ายต่อการ development และ deployment

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ WebSockets และ Real-time communication
