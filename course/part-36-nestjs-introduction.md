# ตอนที่ 36: การแนะนำ NestJS (NestJS Introduction)

## NestJS คืออะไร?

NestJS คือ framework สำหรับการสร้าง server-side applications ที่มีประสิทธิภาพและสามารถ scale ได้ด้วย Node.js โดยใช้ TypeScript เป็นภาษาหลัก NestJS ได้รับแรงบันดาลใจจาก Angular ซึ่งทำให้มีโครงสร้างที่ชัดเจนและการออกแบบที่ดี

NestJS ใช้ประโยชน์จาก Express.js (หรือ Fastify) เป็น HTTP server ด้านล่าง แต่ยังเพิ่ม abstraction layer ที่ทำให้การพัฒนาแอปพลิเคชันง่ายขึ้น

### คุณสมบัติหลักของ NestJS

- **TypeScript-first**: รองรับ TypeScript อย่างเต็มรูปแบบ
- **Decorator-based**: ใช้ decorators ในการกำหนดโครงสร้าง
- **Modular architecture**: การออกแบบแบบ modular ที่ชัดเจน
- **Dependency Injection**: ระบบ DI ที่ทรงพลัง
- **Testable**: รองรับการทดสอบทุกระดับ
- **Extensible**: สามารถขยายได้ง่าย

```typescript
// ตัวอย่างแรก: NestJS Application พื้นฐาน
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
  console.log('แอปพลิเคชันกำลังทำงานที่ http://localhost:3000');
}

bootstrap();
```

---

## สถาปัตยกรรมของ NestJS (NestJS Architecture)

### 1. Modules (โมดูล)
Modules คือหน่วยพื้นฐานในการจัดระเบียบโค้ด ทุกแอปพลิเคชัน NestJS มี root module อย่างน้อยหนึ่งตัว

```typescript
// app.module.ts - โมดูลหลักของแอปพลิเคชัน
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],        // โมดูลที่ต้องการ import
  controllers: [AppController],  // controllers ในโมดูลนี้
  providers: [AppService],       // providers/services
  exports: [],        // สิ่งที่ต้องการ export ให้โมดูลอื่น
})
export class AppModule {}
```

### 2. Controllers (คอนโทรลเลอร์)
Controllers รับผิดชอบในการจัดการ HTTP requests และส่ง responses กลับ

```typescript
// app.controller.ts
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { AppService } from './app.service';

@Controller('app')
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Get(':id')
  getById(@Param('id') id: string): string {
    return `ข้อมูล ID: ${id}`;
  }

  @Post()
  create(@Body() data: any): any {
    return { message: 'สร้างสำเร็จ', data };
  }
}
```

### 3. Providers/Services (ผู้ให้บริการ/เซอร์วิส)
Services มีหน้าที่ประมวลผล business logic

```typescript
// app.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'สวัสดีจาก NestJS!';
  }

  findAll(): string[] {
    return ['ข้อมูล 1', 'ข้อมูล 2', 'ข้อมูล 3'];
  }
}
```

---

## การติดตั้ง NestJS และ CLI (Installation and CLI)

### การติดตั้ง NestJS CLI
```bash
# ติดตั้ง NestJS CLI ทั่วทั้งระบบ
npm install -g @nestjs/cli

# ตรวจสอบเวอร์ชัน
nest --version
```

### การสร้างโปรเจกต์ใหม่
```bash
# สร้างโปรเจกต์ใหม่
nest new my-nestjs-app

# เลือก package manager (npm, yarn, หรือ pnpm)
# ? Which package manager would you ❤️  to use?
# ❯ npm
#   yarn
#   pnpm
```

### คำสั่ง NestJS CLI ที่ใช้บ่อย
```bash
# สร้าง module ใหม่
nest generate module users
# หรือ
nest g mo users

# สร้าง controller ใหม่
nest generate controller users
# หรือ
nest g co users

# สร้าง service ใหม่
nest generate service users
# หรือ
nest g s users

# สร้าง resource ครบชุด (CRUD)
nest generate resource users
# หรือ
nest g res users

# สร้าง middleware
nest generate middleware logger
# หรือ
nest g mi logger

# สร้าง pipe
nest generate pipe validation
# หรือ
nest g pi validation

# สร้าง guard
nest generate guard auth
# หรือ
nest g gu auth

# สร้าง interceptor
nest generate interceptor logging
# หรือ
nest g in logging

# รันแอปพลิเคชัน
npm run start

# รันในโหมด development (watch mode)
npm run start:dev

# รันในโหมด production
npm run start:prod

# รัน tests
npm run test

# รัน e2e tests
npm run test:e2e
```

---

## โครงสร้างโปรเจกต์ (Project Structure)

เมื่อสร้างโปรเจกต์ใหม่ด้วย NestJS CLI จะได้โครงสร้างดังนี้:

```
my-nestjs-app/
├── src/
│   ├── app.controller.spec.ts   # ไฟล์ทดสอบ controller
│   ├── app.controller.ts        # Controller หลัก
│   ├── app.module.ts            # Module หลัก
│   ├── app.service.ts           # Service หลัก
│   └── main.ts                  # Entry point
├── test/
│   ├── app.e2e-spec.ts          # End-to-end tests
│   └── jest-e2e.json            # Jest config สำหรับ e2e
├── .eslintrc.js                 # ESLint configuration
├── .gitignore
├── .prettierrc                  # Prettier configuration
├── nest-cli.json                # NestJS CLI configuration
├── package.json
├── README.md
├── tsconfig.build.json          # TypeScript config สำหรับ build
└── tsconfig.json                # TypeScript configuration
```

### ไฟล์ main.ts - จุดเริ่มต้นของแอปพลิเคชัน
```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // เพิ่ม global validation pipe
  app.useGlobalPipes(new ValidationPipe());
  
  // เพิ่ม global prefix
  app.setGlobalPrefix('api/v1');
  
  // เปิดใช้ CORS
  app.enableCors({
    origin: ['http://localhost:3000', 'https://myapp.com'],
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
    credentials: true,
  });
  
  const port = process.env.PORT || 3000;
  await app.listen(port);
  console.log(`แอปพลิเคชันกำลังทำงานที่ port ${port}`);
}

bootstrap();
```

### ไฟล์ tsconfig.json
```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": false,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false
  }
}
```

---

## แอปพลิเคชัน NestJS แรกของคุณ (First NestJS Application)

มาสร้างแอปพลิเคชัน "Hello World" แบบสมบูรณ์กัน:

### ขั้นตอนที่ 1: สร้างโปรเจกต์
```bash
nest new hello-world-app
cd hello-world-app
```

### ขั้นตอนที่ 2: แก้ไข AppService
```typescript
// src/app.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  private messages: string[] = [];

  getHello(): string {
    return 'สวัสดีโลก! จาก NestJS';
  }

  addMessage(message: string): { id: number; message: string; timestamp: Date } {
    this.messages.push(message);
    return {
      id: this.messages.length,
      message,
      timestamp: new Date(),
    };
  }

  getMessages(): string[] {
    return this.messages;
  }
}
```

### ขั้นตอนที่ 3: แก้ไข AppController
```typescript
// src/app.controller.ts
import { Controller, Get, Post, Body, HttpCode } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Post('messages')
  @HttpCode(201)
  addMessage(@Body('message') message: string) {
    return this.appService.addMessage(message);
  }

  @Get('messages')
  getMessages() {
    return {
      messages: this.appService.getMessages(),
      total: this.appService.getMessages().length,
    };
  }
}
```

### ขั้นตอนที่ 4: รันแอปพลิเคชัน
```bash
npm run start:dev
```

---

## Decorators ใน NestJS (Decorators Overview)

Decorators คือ feature พิเศษของ TypeScript ที่ NestJS ใช้อย่างมาก

### Class Decorators (Decorators สำหรับ Class)
```typescript
// @Module - กำหนดว่า class นี้เป็น module
@Module({
  imports: [DatabaseModule],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}

// @Controller - กำหนดว่า class นี้เป็น controller
@Controller('users')
export class UserController {}

// @Injectable - กำหนดว่า class นี้สามารถ inject ได้
@Injectable()
export class UserService {}
```

### Method Decorators (Decorators สำหรับ Method)
```typescript
import {
  Get, Post, Put, Delete, Patch,
  HttpCode, Header, Redirect
} from '@nestjs/common';

@Controller('products')
export class ProductController {
  // GET /products
  @Get()
  findAll() {}

  // POST /products
  @Post()
  @HttpCode(201)  // กำหนด HTTP status code
  create() {}

  // GET /products/:id
  @Get(':id')
  findOne() {}

  // PUT /products/:id
  @Put(':id')
  update() {}

  // DELETE /products/:id
  @Delete(':id')
  remove() {}

  // PATCH /products/:id
  @Patch(':id')
  partialUpdate() {}

  // Redirect
  @Get('old-path')
  @Redirect('/new-path', 301)
  oldPath() {}

  // กำหนด Header
  @Get('download')
  @Header('Content-Type', 'application/octet-stream')
  download() {}
}
```

### Parameter Decorators (Decorators สำหรับ Parameters)
```typescript
import {
  Param, Query, Body, Headers,
  Req, Res, Ip, HostParam
} from '@nestjs/common';
import { Request, Response } from 'express';

@Controller('demo')
export class DemoController {
  @Get(':id')
  getWithParam(@Param('id') id: string) {
    // รับค่าจาก URL parameter เช่น /demo/123
    return `ID: ${id}`;
  }

  @Get()
  getWithQuery(@Query('page') page: number, @Query('limit') limit: number) {
    // รับค่าจาก query string เช่น /demo?page=1&limit=10
    return { page, limit };
  }

  @Post()
  createWithBody(@Body() body: any) {
    // รับค่าจาก request body
    return body;
  }

  @Post('partial')
  createPartial(@Body('name') name: string, @Body('email') email: string) {
    // รับเฉพาะบางส่วนจาก request body
    return { name, email };
  }

  @Get('headers-demo')
  getHeaders(@Headers('authorization') auth: string) {
    // รับค่าจาก headers
    return { auth };
  }

  @Get('request-demo')
  getRequest(@Req() req: Request) {
    // รับ request object ทั้งหมด
    return { method: req.method, url: req.url };
  }

  @Get('response-demo')
  getResponse(@Res() res: Response) {
    // ใช้ response object โดยตรง
    res.json({ message: 'custom response' });
  }

  @Get('ip-demo')
  getIp(@Ip() ip: string) {
    return { ip };
  }
}
```

### Custom Decorators (Decorators ที่สร้างเอง)
```typescript
// decorators/user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

// สร้าง decorator สำหรับดึงข้อมูล user จาก request
export const CurrentUser = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);

// การใช้งาน
@Controller('profile')
export class ProfileController {
  @Get()
  getProfile(@CurrentUser() user: any) {
    return user;
  }

  @Get('email')
  getEmail(@CurrentUser('email') email: string) {
    return { email };
  }
}
```

---

## ระบบ Modules (Modules System)

### Root Module (โมดูลหลัก)
```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersModule } from './users/users.module';
import { ProductsModule } from './products/products.module';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    // โมดูลสำหรับ configuration
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
    // โมดูลสำหรับ database
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST || 'localhost',
      port: parseInt(process.env.DB_PORT) || 5432,
      username: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASSWORD || 'password',
      database: process.env.DB_NAME || 'mydb',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: process.env.NODE_ENV !== 'production',
    }),
    // feature modules
    UsersModule,
    ProductsModule,
    AuthModule,
  ],
})
export class AppModule {}
```

### Feature Module (โมดูลสำหรับ feature)
```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],  // export service เพื่อให้โมดูลอื่นใช้ได้
})
export class UsersModule {}
```

### Shared Module (โมดูลที่ใช้ร่วมกัน)
```typescript
// src/shared/shared.module.ts
import { Module, Global } from '@nestjs/common';
import { EmailService } from './services/email.service';
import { LoggerService } from './services/logger.service';
import { UtilsService } from './services/utils.service';

@Global()  // ทำให้โมดูลนี้เข้าถึงได้จากทุกที่โดยไม่ต้อง import ซ้ำ
@Module({
  providers: [EmailService, LoggerService, UtilsService],
  exports: [EmailService, LoggerService, UtilsService],
})
export class SharedModule {}
```

### Dynamic Module (โมดูลแบบ dynamic)
```typescript
// src/database/database.module.ts
import { DynamicModule, Module } from '@nestjs/common';
import { DatabaseService } from './database.service';

export interface DatabaseOptions {
  host: string;
  port: number;
  database: string;
}

@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
        DatabaseService,
      ],
      exports: [DatabaseService],
      global: true,
    };
  }
}

// การใช้งาน
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432,
      database: 'mydb',
    }),
  ],
})
export class AppModule {}
```

---

## Controllers พื้นฐาน (Controllers Basics)

### Controller แบบสมบูรณ์
```typescript
// src/products/products.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Patch,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
  ParseIntPipe,
  ValidationPipe,
  UseGuards,
  UseInterceptors,
} from '@nestjs/common';
import { ProductsService } from './products.service';
import { CreateProductDto } from './dto/create-product.dto';
import { UpdateProductDto } from './dto/update-product.dto';

@Controller('products')
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  // GET /products
  @Get()
  findAll(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 10,
    @Query('search') search?: string,
  ) {
    return this.productsService.findAll({ page, limit, search });
  }

  // GET /products/:id
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.productsService.findOne(id);
  }

  // POST /products
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createProductDto: CreateProductDto) {
    return this.productsService.create(createProductDto);
  }

  // PUT /products/:id
  @Put(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateProductDto: UpdateProductDto,
  ) {
    return this.productsService.update(id, updateProductDto);
  }

  // PATCH /products/:id
  @Patch(':id')
  partialUpdate(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateProductDto: Partial<UpdateProductDto>,
  ) {
    return this.productsService.update(id, updateProductDto);
  }

  // DELETE /products/:id
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.productsService.remove(id);
  }
}
```

### DTO (Data Transfer Object)
```typescript
// src/products/dto/create-product.dto.ts
import { IsString, IsNumber, IsOptional, Min, MaxLength } from 'class-validator';

export class CreateProductDto {
  @IsString()
  @MaxLength(100)
  name: string;

  @IsString()
  @IsOptional()
  description?: string;

  @IsNumber()
  @Min(0)
  price: number;

  @IsNumber()
  @Min(0)
  stock: number;
}

// src/products/dto/update-product.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateProductDto } from './create-product.dto';

export class UpdateProductDto extends PartialType(CreateProductDto) {}
```

---

## Providers และ Services พื้นฐาน (Providers/Services Basics)

### Service พื้นฐาน
```typescript
// src/products/products.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { CreateProductDto } from './dto/create-product.dto';
import { UpdateProductDto } from './dto/update-product.dto';

interface Product {
  id: number;
  name: string;
  description?: string;
  price: number;
  stock: number;
  createdAt: Date;
  updatedAt: Date;
}

@Injectable()
export class ProductsService {
  private products: Product[] = [];
  private nextId = 1;

  findAll(options: { page: number; limit: number; search?: string }) {
    let items = [...this.products];

    if (options.search) {
      const search = options.search.toLowerCase();
      items = items.filter(
        (p) =>
          p.name.toLowerCase().includes(search) ||
          p.description?.toLowerCase().includes(search),
      );
    }

    const total = items.length;
    const start = (options.page - 1) * options.limit;
    const end = start + options.limit;
    const data = items.slice(start, end);

    return {
      data,
      total,
      page: options.page,
      limit: options.limit,
      totalPages: Math.ceil(total / options.limit),
    };
  }

  findOne(id: number): Product {
    const product = this.products.find((p) => p.id === id);
    if (!product) {
      throw new NotFoundException(`ไม่พบสินค้า ID: ${id}`);
    }
    return product;
  }

  create(createProductDto: CreateProductDto): Product {
    const product: Product = {
      id: this.nextId++,
      ...createProductDto,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.products.push(product);
    return product;
  }

  update(id: number, updateProductDto: UpdateProductDto): Product {
    const index = this.products.findIndex((p) => p.id === id);
    if (index === -1) {
      throw new NotFoundException(`ไม่พบสินค้า ID: ${id}`);
    }

    this.products[index] = {
      ...this.products[index],
      ...updateProductDto,
      updatedAt: new Date(),
    };

    return this.products[index];
  }

  remove(id: number): void {
    const index = this.products.findIndex((p) => p.id === id);
    if (index === -1) {
      throw new NotFoundException(`ไม่พบสินค้า ID: ${id}`);
    }
    this.products.splice(index, 1);
  }
}
```

---

## แอปพลิเคชัน Hello World แบบสมบูรณ์ (Complete Hello World App)

มาสร้างแอปพลิเคชันที่ครบสมบูรณ์ด้วยการจัดการ tasks:

### 1. สร้าง Task Module
```bash
nest g resource tasks
```

### 2. Task Entity/Interface
```typescript
// src/tasks/interfaces/task.interface.ts
export enum TaskStatus {
  TODO = 'TODO',
  IN_PROGRESS = 'IN_PROGRESS',
  DONE = 'DONE',
}

export interface Task {
  id: string;
  title: string;
  description: string;
  status: TaskStatus;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3. Task DTOs
```typescript
// src/tasks/dto/create-task.dto.ts
import { IsString, IsNotEmpty, IsOptional, IsEnum } from 'class-validator';
import { TaskStatus } from '../interfaces/task.interface';

export class CreateTaskDto {
  @IsString()
  @IsNotEmpty({ message: 'กรุณาระบุชื่อ task' })
  title: string;

  @IsString()
  @IsOptional()
  description?: string;
}

// src/tasks/dto/update-task-status.dto.ts
import { IsEnum } from 'class-validator';
import { TaskStatus } from '../interfaces/task.interface';

export class UpdateTaskStatusDto {
  @IsEnum(TaskStatus, { message: 'สถานะไม่ถูกต้อง' })
  status: TaskStatus;
}

// src/tasks/dto/get-tasks-filter.dto.ts
import { IsOptional, IsString, IsEnum } from 'class-validator';
import { TaskStatus } from '../interfaces/task.interface';

export class GetTasksFilterDto {
  @IsOptional()
  @IsEnum(TaskStatus)
  status?: TaskStatus;

  @IsOptional()
  @IsString()
  search?: string;
}
```

### 4. Task Service
```typescript
// src/tasks/tasks.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid';
import { Task, TaskStatus } from './interfaces/task.interface';
import { CreateTaskDto } from './dto/create-task.dto';
import { GetTasksFilterDto } from './dto/get-tasks-filter.dto';

@Injectable()
export class TasksService {
  private tasks: Task[] = [];

  getAllTasks(): Task[] {
    return this.tasks;
  }

  getTasksWithFilters(filterDto: GetTasksFilterDto): Task[] {
    const { status, search } = filterDto;
    let tasks = this.getAllTasks();

    if (status) {
      tasks = tasks.filter((task) => task.status === status);
    }

    if (search) {
      tasks = tasks.filter(
        (task) =>
          task.title.includes(search) || task.description.includes(search),
      );
    }

    return tasks;
  }

  getTaskById(id: string): Task {
    const task = this.tasks.find((task) => task.id === id);
    if (!task) {
      throw new NotFoundException(`Task "${id}" ไม่พบ`);
    }
    return task;
  }

  createTask(createTaskDto: CreateTaskDto): Task {
    const { title, description } = createTaskDto;
    const task: Task = {
      id: uuidv4(),
      title,
      description: description || '',
      status: TaskStatus.TODO,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.tasks.push(task);
    return task;
  }

  deleteTask(id: string): void {
    const task = this.getTaskById(id);
    this.tasks = this.tasks.filter((t) => t.id !== task.id);
  }

  updateTaskStatus(id: string, status: TaskStatus): Task {
    const task = this.getTaskById(id);
    task.status = status;
    task.updatedAt = new Date();
    return task;
  }
}
```

### 5. Task Controller
```typescript
// src/tasks/tasks.controller.ts
import {
  Controller,
  Get,
  Post,
  Delete,
  Patch,
  Body,
  Param,
  Query,
  UsePipes,
  ValidationPipe,
  ParseUUIDPipe,
} from '@nestjs/common';
import { TasksService } from './tasks.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskStatusDto } from './dto/update-task-status.dto';
import { GetTasksFilterDto } from './dto/get-tasks-filter.dto';

@Controller('tasks')
@UsePipes(ValidationPipe)
export class TasksController {
  constructor(private tasksService: TasksService) {}

  @Get()
  getTasks(@Query() filterDto: GetTasksFilterDto) {
    if (Object.keys(filterDto).length) {
      return this.tasksService.getTasksWithFilters(filterDto);
    } else {
      return this.tasksService.getAllTasks();
    }
  }

  @Get('/:id')
  getTaskById(@Param('id', ParseUUIDPipe) id: string) {
    return this.tasksService.getTaskById(id);
  }

  @Post()
  createTask(@Body() createTaskDto: CreateTaskDto) {
    return this.tasksService.createTask(createTaskDto);
  }

  @Delete('/:id')
  deleteTask(@Param('id', ParseUUIDPipe) id: string): void {
    this.tasksService.deleteTask(id);
  }

  @Patch('/:id/status')
  updateTaskStatus(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updateTaskStatusDto: UpdateTaskStatusDto,
  ) {
    const { status } = updateTaskStatusDto;
    return this.tasksService.updateTaskStatus(id, status);
  }
}
```

### 6. Task Module
```typescript
// src/tasks/tasks.module.ts
import { Module } from '@nestjs/common';
import { TasksController } from './tasks.controller';
import { TasksService } from './tasks.service';

@Module({
  controllers: [TasksController],
  providers: [TasksService],
})
export class TasksModule {}
```

### 7. App Module (อัปเดต)
```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TasksModule } from './tasks/tasks.module';

@Module({
  imports: [TasksModule],
})
export class AppModule {}
```

---

## ประโยชน์ของ TypeScript ใน NestJS (TypeScript Benefits)

### 1. Type Safety (ความปลอดภัยของ Type)
```typescript
// ตัวอย่าง: TypeScript ช่วยให้เราหาข้อผิดพลาดก่อน compile
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

@Injectable()
export class UsersService {
  private users: User[] = [];

  // TypeScript ตรวจสอบ return type
  findById(id: number): User | undefined {
    return this.users.find(u => u.id === id);
  }

  // TypeScript ช่วยป้องกัน error
  getUserName(id: number): string {
    const user = this.findById(id);
    if (!user) {
      throw new NotFoundException('ไม่พบผู้ใช้');
    }
    // TypeScript รู้ว่า user ไม่ใช่ undefined แล้ว
    return user.name; // ไม่มี error
  }
}
```

### 2. Interface และ Type Definitions
```typescript
// กำหนด interfaces สำหรับ API responses
interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}

interface PaginatedResponse<T> {
  data: T[];
  meta: PaginationMeta;
}

@Injectable()
export class ProductsService {
  findAll(page: number, limit: number): PaginatedResponse<Product> {
    const products = this.getProducts();
    return {
      data: products.slice((page - 1) * limit, page * limit),
      meta: {
        page,
        limit,
        total: products.length,
        totalPages: Math.ceil(products.length / limit),
      },
    };
  }
}
```

### 3. Generics
```typescript
// Generic response wrapper
interface ApiResponse<T> {
  success: boolean;
  data: T;
  message: string;
  timestamp: string;
}

// Generic service method
function createApiResponse<T>(data: T, message: string): ApiResponse<T> {
  return {
    success: true,
    data,
    message,
    timestamp: new Date().toISOString(),
  };
}

// การใช้งาน
@Controller('api')
export class ApiController {
  @Get('users')
  getUsers(): ApiResponse<User[]> {
    const users = []; // สมมติว่าได้จาก service
    return createApiResponse(users, 'ดึงข้อมูลผู้ใช้สำเร็จ');
  }
}
```

### 4. Decorators และ Metadata
```typescript
// NestJS ใช้ TypeScript decorators อย่างกว้างขวาง
import { SetMetadata } from '@nestjs/common';

// สร้าง custom decorator
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// สร้าง guard ที่ใช้ metadata
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (!requiredRoles) {
      return true;
    }
    
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}

// การใช้งาน
@Controller('admin')
@UseGuards(RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin', 'superadmin')  // กำหนด role ที่ต้องการ
  getAllUsers() {
    return [];
  }
}
```

### 5. Abstract Classes
```typescript
// สร้าง abstract base service
abstract class BaseService<T, CreateDto, UpdateDto> {
  protected abstract findById(id: number): Promise<T>;
  protected abstract create(dto: CreateDto): Promise<T>;
  protected abstract update(id: number, dto: UpdateDto): Promise<T>;
  protected abstract delete(id: number): Promise<void>;
  
  // Shared logic
  async findOrFail(id: number): Promise<T> {
    const entity = await this.findById(id);
    if (!entity) {
      throw new NotFoundException(`ไม่พบข้อมูล ID: ${id}`);
    }
    return entity;
  }
}

// Implement ใน concrete service
@Injectable()
export class UsersService extends BaseService<User, CreateUserDto, UpdateUserDto> {
  protected async findById(id: number): Promise<User> {
    // implementation
    return null;
  }
  
  protected async create(dto: CreateUserDto): Promise<User> {
    // implementation
    return null;
  }
  
  protected async update(id: number, dto: UpdateUserDto): Promise<User> {
    // implementation
    return null;
  }
  
  protected async delete(id: number): Promise<void> {
    // implementation
  }
}
```

---

## Pipes ใน NestJS (Validation & Transformation)

### Built-in Pipes
```typescript
import {
  ParseIntPipe,
  ParseFloatPipe,
  ParseBoolPipe,
  ParseArrayPipe,
  ParseUUIDPipe,
  ParseEnumPipe,
  DefaultValuePipe,
  ValidationPipe,
} from '@nestjs/common';

@Controller('demo')
export class DemoController {
  // ParseIntPipe - แปลง string เป็น integer
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return { id, type: typeof id }; // id จะเป็น number
  }

  // ParseFloatPipe - แปลง string เป็น float
  @Get('price/:amount')
  getPrice(@Param('amount', ParseFloatPipe) amount: number) {
    return { amount };
  }

  // ParseBoolPipe - แปลง string เป็น boolean
  @Get('active/:status')
  getActive(@Param('status', ParseBoolPipe) active: boolean) {
    return { active };
  }

  // DefaultValuePipe - กำหนดค่า default
  @Get('search')
  search(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
  ) {
    return { page, limit };
  }

  // ParseUUIDPipe - ตรวจสอบ UUID format
  @Get('uuid/:id')
  getByUuid(@Param('id', ParseUUIDPipe) id: string) {
    return { id };
  }
}
```

### Custom Pipe
```typescript
// src/pipes/trim.pipe.ts
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if (typeof value === 'string') {
      return value.trim();
    }
    if (typeof value === 'object' && value !== null) {
      Object.keys(value).forEach((key) => {
        if (typeof value[key] === 'string') {
          value[key] = value[key].trim();
        }
      });
    }
    return value;
  }
}

// การใช้งาน
@Controller('users')
export class UsersController {
  @Post()
  create(@Body(TrimPipe) createUserDto: CreateUserDto) {
    return { message: 'สร้างผู้ใช้สำเร็จ', user: createUserDto };
  }
}
```

---

## Exception Filters (การจัดการข้อผิดพลาด)

### Built-in Exceptions
```typescript
import {
  HttpException,
  HttpStatus,
  NotFoundException,
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  ConflictException,
  InternalServerErrorException,
} from '@nestjs/common';

@Injectable()
export class UsersService {
  findOne(id: number) {
    if (!id) {
      throw new BadRequestException('ID ไม่ถูกต้อง');
    }
    
    const user = this.findById(id);
    if (!user) {
      throw new NotFoundException(`ไม่พบผู้ใช้ ID: ${id}`);
    }
    
    return user;
  }

  create(email: string) {
    const existing = this.findByEmail(email);
    if (existing) {
      throw new ConflictException('Email นี้ถูกใช้แล้ว');
    }
    // ...
  }
}
```

### Custom Exception Filter
```typescript
// src/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const error =
      typeof exceptionResponse === 'string'
        ? { message: exceptionResponse }
        : (exceptionResponse as object);

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      ...error,
    });
  }
}

// การใช้งาน - Global filter
// main.ts
app.useGlobalFilters(new HttpExceptionFilter());

// หรือ Controller level
@Controller('users')
@UseFilters(HttpExceptionFilter)
export class UsersController {}

// หรือ Method level
@Get(':id')
@UseFilters(HttpExceptionFilter)
findOne(@Param('id') id: string) {}
```

---

## Interceptors (ตัวดักการทำงาน)

```typescript
// src/interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
  success: boolean;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        data,
        success: true,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}

// Logging Interceptor
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const start = Date.now();

    console.log(`[${method}] ${url} - เริ่มต้น`);

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        console.log(`[${method}] ${url} - สำเร็จ ${duration}ms`);
      }),
    );
  }
}
```

---

## Middleware ใน NestJS

```typescript
// src/middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl, ip } = req;
    const userAgent = req.get('user-agent') || '';
    const startTime = Date.now();

    res.on('finish', () => {
      const { statusCode } = res;
      const duration = Date.now() - startTime;
      console.log(
        `${method} ${originalUrl} ${statusCode} ${duration}ms - ${ip} "${userAgent}"`,
      );
    });

    next();
  }
}

// การใช้ middleware ใน module
import { MiddlewareConsumer, NestModule, RequestMethod } from '@nestjs/common';

@Module({
  imports: [UsersModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: '*', method: RequestMethod.ALL });
    
    // หรือ apply เฉพาะบาง route
    consumer
      .apply(AuthMiddleware)
      .exclude(
        { path: 'auth/login', method: RequestMethod.POST },
        { path: 'auth/register', method: RequestMethod.POST },
      )
      .forRoutes(UsersController);
  }
}
```

---

## Guards ใน NestJS

```typescript
// src/guards/auth.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];
    
    if (!token) {
      throw new UnauthorizedException('ต้องการ authentication token');
    }
    
    try {
      // ตรวจสอบ token (ในความเป็นจริงควรใช้ JWT verify)
      const user = this.validateToken(token);
      request.user = user;
      return true;
    } catch {
      throw new UnauthorizedException('Token ไม่ถูกต้อง');
    }
  }

  private validateToken(token: string): any {
    // Logic ตรวจสอบ token
    return { id: 1, email: 'user@example.com', roles: ['user'] };
  }
}

// การใช้งาน
@Controller('protected')
@UseGuards(AuthGuard)
export class ProtectedController {
  @Get()
  getData() {
    return 'ข้อมูลที่ต้องการ authentication';
  }
}
```

---

## สรุป

NestJS เป็น framework ที่ทรงพลังสำหรับการสร้าง Node.js applications ด้วย TypeScript มีคุณสมบัติหลักที่ควรรู้:

1. **Module System**: การจัดระเบียบโค้ดด้วย modules
2. **Controllers**: จัดการ HTTP requests
3. **Services/Providers**: Business logic และ Dependency Injection
4. **Decorators**: การกำหนดโครงสร้างด้วย decorators
5. **Pipes**: การตรวจสอบและแปลงข้อมูล
6. **Guards**: การควบคุม access
7. **Interceptors**: การดักและแปลงข้อมูล
8. **Filters**: การจัดการ exceptions
9. **Middleware**: การประมวลผลก่อน/หลัง request

ในบทต่อไปเราจะเรียนรู้เรื่อง Modules และ Controllers อย่างละเอียดมากขึ้น
