# ตอนที่ 37: NestJS Modules & Controllers

## Modules ใน NestJS (Module Definition and Decorators)

Module คือหน่วยพื้นฐานในการจัดระเบียบโค้ดใน NestJS แต่ละ module มีหน้าที่เฉพาะและสามารถนำมารวมกันเพื่อสร้างแอปพลิเคชันที่ซับซ้อนได้

### @Module Decorator

```typescript
// ไวยากรณ์พื้นฐานของ @Module
import { Module } from '@nestjs/common';

@Module({
  imports: [],       // โมดูลอื่นที่ต้อง import
  controllers: [],   // controllers ในโมดูลนี้
  providers: [],     // providers/services ในโมดูลนี้
  exports: [],       // สิ่งที่ export ให้โมดูลอื่น
})
export class MyModule {}
```

### ตัวอย่าง Module ที่สมบูรณ์
```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserRepository } from './user.repository';
import { User } from './entities/user.entity';
import { EmailModule } from '../email/email.module';
import { LoggerModule } from '../logger/logger.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([User]),  // import TypeORM entities
    EmailModule,                        // import email module
    LoggerModule,                       // import logger module
  ],
  controllers: [UsersController],
  providers: [
    UsersService,
    UserRepository,
    {
      provide: 'USER_CONFIG',
      useValue: { maxUsers: 1000 },
    },
  ],
  exports: [UsersService],  // export UsersService เพื่อให้โมดูลอื่นใช้ได้
})
export class UsersModule {}
```

---

## Feature Modules (โมดูลสำหรับ Feature)

Feature modules ช่วยจัดระเบียบโค้ดตาม business domain

### โครงสร้าง Feature Module
```
src/
├── users/
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   ├── entities/
│   │   └── user.entity.ts
│   ├── users.controller.ts
│   ├── users.module.ts
│   └── users.service.ts
├── products/
│   ├── dto/
│   ├── entities/
│   ├── products.controller.ts
│   ├── products.module.ts
│   └── products.service.ts
└── app.module.ts
```

### ตัวอย่าง Feature Module สำหรับ Blog
```typescript
// src/blog/blog.module.ts
import { Module } from '@nestjs/common';
import { PostsController } from './posts/posts.controller';
import { PostsService } from './posts/posts.service';
import { CommentsController } from './comments/comments.controller';
import { CommentsService } from './comments/comments.service';
import { CategoriesController } from './categories/categories.controller';
import { CategoriesService } from './categories/categories.service';
import { TagsService } from './tags/tags.service';

@Module({
  controllers: [
    PostsController,
    CommentsController,
    CategoriesController,
  ],
  providers: [
    PostsService,
    CommentsService,
    CategoriesService,
    TagsService,
  ],
  exports: [
    PostsService,
    CategoriesService,
    TagsService,
  ],
})
export class BlogModule {}

// src/app.module.ts
import { Module } from '@nestjs/common';
import { BlogModule } from './blog/blog.module';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    BlogModule,
    UsersModule,
    AuthModule,
  ],
})
export class AppModule {}
```

### Posts Module ในระบบ Blog
```typescript
// src/blog/posts/posts.module.ts
import { Module } from '@nestjs/common';
import { PostsController } from './posts.controller';
import { PostsService } from './posts.service';
import { CommentsModule } from '../comments/comments.module';

@Module({
  imports: [CommentsModule],
  controllers: [PostsController],
  providers: [PostsService],
  exports: [PostsService],
})
export class PostsModule {}
```

---

## Shared Modules (โมดูลที่ใช้ร่วมกัน)

Shared modules มี providers ที่ต้องการใช้ในหลาย modules

```typescript
// src/shared/shared.module.ts
import { Module } from '@nestjs/common';
import { EmailService } from './services/email.service';
import { SmsService } from './services/sms.service';
import { FileService } from './services/file.service';
import { CacheService } from './services/cache.service';

@Module({
  providers: [
    EmailService,
    SmsService,
    FileService,
    CacheService,
  ],
  exports: [
    EmailService,
    SmsService,
    FileService,
    CacheService,
  ],
})
export class SharedModule {}

// การใช้งาน SharedModule ใน feature module
@Module({
  imports: [SharedModule],
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

### Email Service
```typescript
// src/shared/services/email.service.ts
import { Injectable } from '@nestjs/common';

interface EmailOptions {
  to: string;
  subject: string;
  body: string;
  html?: string;
}

@Injectable()
export class EmailService {
  async sendEmail(options: EmailOptions): Promise<void> {
    console.log(`ส่ง email ไปยัง ${options.to}: ${options.subject}`);
    // ในความเป็นจริงจะใช้ nodemailer หรือ service อื่น
  }

  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    await this.sendEmail({
      to: email,
      subject: `ยินดีต้อนรับ ${name}!`,
      body: `สวัสดี ${name}, ขอบคุณที่สมัครสมาชิก`,
      html: `<h1>สวัสดี ${name}</h1><p>ขอบคุณที่สมัครสมาชิก</p>`,
    });
  }

  async sendPasswordResetEmail(email: string, token: string): Promise<void> {
    await this.sendEmail({
      to: email,
      subject: 'รีเซ็ตรหัสผ่าน',
      body: `ใช้ลิงก์นี้เพื่อรีเซ็ตรหัสผ่าน: /reset-password?token=${token}`,
    });
  }
}
```

---

## Global Modules (โมดูลระดับ Global)

Global modules ไม่จำเป็นต้อง import ในทุก module

```typescript
// src/config/config.module.ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Global()  // ทำให้ module นี้เป็น global
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}

// src/config/config.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class ConfigService {
  private readonly config: Record<string, any>;

  constructor() {
    this.config = {
      database: {
        host: process.env.DB_HOST || 'localhost',
        port: parseInt(process.env.DB_PORT) || 5432,
        name: process.env.DB_NAME || 'mydb',
      },
      jwt: {
        secret: process.env.JWT_SECRET || 'secret',
        expiresIn: process.env.JWT_EXPIRES_IN || '1d',
      },
      app: {
        port: parseInt(process.env.PORT) || 3000,
        env: process.env.NODE_ENV || 'development',
      },
    };
  }

  get<T>(key: string): T {
    const keys = key.split('.');
    let value: any = this.config;
    
    for (const k of keys) {
      value = value?.[k];
    }
    
    return value as T;
  }

  isDevelopment(): boolean {
    return this.get<string>('app.env') === 'development';
  }

  isProduction(): boolean {
    return this.get<string>('app.env') === 'production';
  }
}

// หลังจากลงทะเบียน ConfigModule เป็น global
// สามารถ inject ConfigService ได้ทุกที่โดยไม่ต้อง import ConfigModule ซ้ำ
@Injectable()
export class UsersService {
  constructor(private configService: ConfigService) {
    const dbHost = this.configService.get<string>('database.host');
    console.log(`เชื่อมต่อ database ที่ ${dbHost}`);
  }
}
```

---

## Dynamic Modules (โมดูลแบบ Dynamic)

Dynamic modules ช่วยสร้าง configurable modules

```typescript
// src/mailer/mailer.module.ts
import { DynamicModule, Module, Provider } from '@nestjs/common';
import { MailerService } from './mailer.service';

export interface MailerOptions {
  host: string;
  port: number;
  user: string;
  password: string;
  from: string;
}

export const MAILER_OPTIONS = 'MAILER_OPTIONS';

@Module({})
export class MailerModule {
  // สำหรับ synchronous configuration
  static forRoot(options: MailerOptions): DynamicModule {
    return {
      module: MailerModule,
      providers: [
        {
          provide: MAILER_OPTIONS,
          useValue: options,
        },
        MailerService,
      ],
      exports: [MailerService],
    };
  }

  // สำหรับ async configuration (เช่น ดึงค่าจาก ConfigService)
  static forRootAsync(options: {
    useFactory: (...args: any[]) => Promise<MailerOptions> | MailerOptions;
    inject?: any[];
    imports?: any[];
  }): DynamicModule {
    return {
      module: MailerModule,
      imports: options.imports || [],
      providers: [
        {
          provide: MAILER_OPTIONS,
          useFactory: options.useFactory,
          inject: options.inject || [],
        },
        MailerService,
      ],
      exports: [MailerService],
    };
  }
}

// src/mailer/mailer.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { MAILER_OPTIONS, MailerOptions } from './mailer.module';

@Injectable()
export class MailerService {
  constructor(
    @Inject(MAILER_OPTIONS) private options: MailerOptions,
  ) {}

  async send(to: string, subject: string, text: string): Promise<void> {
    console.log(`ส่งจาก ${this.options.from} ไปยัง ${to}: ${subject}`);
  }
}

// การใช้งาน
@Module({
  imports: [
    // Synchronous
    MailerModule.forRoot({
      host: 'smtp.gmail.com',
      port: 587,
      user: 'myemail@gmail.com',
      password: 'password',
      from: 'noreply@myapp.com',
    }),

    // หรือ Async
    MailerModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        host: configService.get('MAIL_HOST'),
        port: configService.get('MAIL_PORT'),
        user: configService.get('MAIL_USER'),
        password: configService.get('MAIL_PASSWORD'),
        from: configService.get('MAIL_FROM'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

---

## Controllers ใน NestJS (Controller Decorators)

Controllers จัดการ incoming HTTP requests และส่ง responses

### Controller พื้นฐาน
```typescript
// src/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Patch,
  Head,
  Options,
  All,
} from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get()          // GET /users
  findAll() {}

  @Get('profile') // GET /users/profile
  getProfile() {}

  @Get(':id')     // GET /users/:id
  findOne() {}

  @Post()         // POST /users
  create() {}

  @Put(':id')     // PUT /users/:id
  update() {}

  @Patch(':id')   // PATCH /users/:id
  partialUpdate() {}

  @Delete(':id')  // DELETE /users/:id
  remove() {}

  @Head(':id')    // HEAD /users/:id
  checkExists() {}

  @Options()      // OPTIONS /users
  options() {}

  @All('*')       // จับทุก method
  handleAll() {}
}
```

---

## Route Parameters (พารามิเตอร์ใน Route)

```typescript
import { Controller, Get, Param, ParseIntPipe, ParseUUIDPipe } from '@nestjs/common';

@Controller('products')
export class ProductsController {
  // พารามิเตอร์เดี่ยว
  @Get(':id')
  findOne(@Param('id') id: string) {
    return { id };
  }

  // พารามิเตอร์ที่แปลงเป็น number
  @Get('number/:id')
  findById(@Param('id', ParseIntPipe) id: number) {
    return { id, type: typeof id }; // number
  }

  // พารามิเตอร์ UUID
  @Get('uuid/:uuid')
  findByUuid(@Param('uuid', ParseUUIDPipe) uuid: string) {
    return { uuid };
  }

  // หลายพารามิเตอร์
  @Get(':category/:id')
  findByCategoryAndId(
    @Param('category') category: string,
    @Param('id', ParseIntPipe) id: number,
  ) {
    return { category, id };
  }

  // รับพารามิเตอร์ทั้งหมดเป็น object
  @Get('all/:category/:subcategory/:id')
  findAll(@Param() params: { category: string; subcategory: string; id: string }) {
    return params;
  }

  // Wildcard routes
  @Get('search/*')
  search() {
    return 'search result';
  }
}
```

---

## Query Parameters (พารามิเตอร์ใน Query String)

```typescript
import {
  Controller,
  Get,
  Query,
  ParseIntPipe,
  ParseBoolPipe,
  DefaultValuePipe,
} from '@nestjs/common';

// DTO สำหรับ query parameters
class ProductQueryDto {
  page?: number;
  limit?: number;
  search?: string;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  category?: string;
}

@Controller('products')
export class ProductsController {
  // รับ query parameters ทีละตัว
  @Get()
  findAll(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
    @Query('search') search?: string,
  ) {
    return { page, limit, search };
  }

  // รับ query parameters เป็น object
  @Get('advanced')
  advancedSearch(@Query() query: ProductQueryDto) {
    return { 
      query,
      message: `ค้นหาสินค้าด้วย: ${JSON.stringify(query)}` 
    };
  }

  // รับ array จาก query string
  // เช่น /products/filter?tags[]=shoes&tags[]=sports
  @Get('filter')
  filterByTags(@Query('tags') tags: string | string[]) {
    const tagArray = Array.isArray(tags) ? tags : tags ? [tags] : [];
    return { tags: tagArray };
  }

  // Boolean query parameter
  @Get('active')
  getActive(
    @Query('isActive', new DefaultValuePipe(true), ParseBoolPipe) isActive: boolean,
  ) {
    return { isActive };
  }
}
```

---

## Request Body (ข้อมูลใน Request Body)

```typescript
import {
  Controller,
  Post,
  Put,
  Patch,
  Body,
  UsePipes,
  ValidationPipe,
} from '@nestjs/common';
import { IsString, IsEmail, IsNumber, IsOptional, Min, Max, IsEnum } from 'class-validator';
import { Transform, Type } from 'class-transformer';

// DTOs
enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

class CreateUserDto {
  @IsString()
  @Transform(({ value }) => value?.trim())
  name: string;

  @IsEmail({}, { message: 'Email ไม่ถูกต้อง' })
  email: string;

  @IsString()
  password: string;

  @IsNumber()
  @Min(0)
  @Max(120)
  @IsOptional()
  @Type(() => Number)
  age?: number;

  @IsEnum(UserRole, { message: 'Role ไม่ถูกต้อง' })
  @IsOptional()
  role?: UserRole = UserRole.USER;
}

class UpdateUserDto {
  @IsString()
  @IsOptional()
  name?: string;

  @IsNumber()
  @Min(0)
  @Max(120)
  @IsOptional()
  age?: number;
}

@Controller('users')
@UsePipes(new ValidationPipe({ transform: true }))
export class UsersController {
  // รับ body ทั้งหมด
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return { message: 'สร้างผู้ใช้สำเร็จ', user: createUserDto };
  }

  // รับเฉพาะบางส่วนของ body
  @Post('quick')
  quickCreate(
    @Body('name') name: string,
    @Body('email') email: string,
  ) {
    return { name, email };
  }

  // PUT - full update
  @Put(':id')
  update(@Body() updateUserDto: CreateUserDto) {
    return { message: 'อัปเดตผู้ใช้สำเร็จ', user: updateUserDto };
  }

  // PATCH - partial update
  @Patch(':id')
  partialUpdate(@Body() updateUserDto: UpdateUserDto) {
    return { message: 'อัปเดตบางส่วนสำเร็จ', updates: updateUserDto };
  }
}
```

---

## Response Objects (การส่ง Response)

```typescript
import {
  Controller,
  Get,
  Post,
  Res,
  HttpCode,
  HttpStatus,
  Header,
  Redirect,
  StreamableFile,
} from '@nestjs/common';
import { Response } from 'express';
import { createReadStream } from 'fs';
import { join } from 'path';

@Controller('demo')
export class DemoController {
  // ส่ง response object ปกติ (แนะนำ)
  @Get()
  getSimple() {
    return { message: 'Hello' }; // NestJS จัดการ serialization ให้อัตโนมัติ
  }

  // กำหนด HTTP status code
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create() {
    return { id: 1, message: 'สร้างสำเร็จ' };
  }

  // กำหนด headers
  @Get('download')
  @Header('Content-Disposition', 'attachment; filename="data.csv"')
  @Header('Content-Type', 'text/csv')
  downloadCsv() {
    return 'id,name\n1,John\n2,Jane';
  }

  // Redirect
  @Get('old-route')
  @Redirect('/new-route', 301)
  oldRoute() {}

  // Dynamic redirect
  @Get('dynamic-redirect')
  dynamicRedirect() {
    return { url: '/destination', statusCode: 302 };
  }

  // ใช้ Express Response object โดยตรง
  @Get('express')
  expressResponse(@Res() res: Response) {
    res.status(200).json({
      message: 'Express response',
      data: { id: 1 },
    });
  }

  // Passthrough (ใช้ @Res แต่ยังต้องการ response interceptors)
  @Get('passthrough')
  passthroughResponse(@Res({ passthrough: true }) res: Response) {
    res.cookie('token', 'my-jwt-token', { httpOnly: true });
    return { message: 'Set cookie และ return ปกติ' };
  }

  // Stream response
  @Get('stream')
  streamFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

---

## HTTP Methods (HTTP Methods ทั้งหมด)

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Patch,
  Head,
  Options,
  All,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';

@Controller('resources')
export class ResourcesController {
  // GET - ดึงข้อมูล
  @Get()
  findAll() {
    return [];
  }

  @Get(':id')
  findOne() {
    return {};
  }

  // POST - สร้างข้อมูลใหม่
  @Post()
  @HttpCode(HttpStatus.CREATED) // 201
  create() {
    return { id: 1 };
  }

  // PUT - แทนที่ข้อมูลทั้งหมด
  @Put(':id')
  replace() {
    return {};
  }

  // PATCH - อัปเดตบางส่วน
  @Patch(':id')
  update() {
    return {};
  }

  // DELETE - ลบข้อมูล
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT) // 204
  remove() {}

  // HEAD - เหมือน GET แต่ไม่ส่ง body
  @Head(':id')
  checkExists() {}

  // OPTIONS - ข้อมูล CORS และ allowed methods
  @Options()
  options() {
    return {
      methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
    };
  }

  // ALL - จับทุก HTTP method
  @All('catch-all')
  handleAll() {
    return 'handles everything';
  }
}
```

---

## Route Prefixes (คำนำหน้า Route)

```typescript
// src/main.ts - Global prefix
app.setGlobalPrefix('api/v1');
// ทุก route จะกลายเป็น /api/v1/...

// Controller-level prefix
@Controller('users')           // /api/v1/users
@Controller('admin/users')     // /api/v1/admin/users
@Controller({ path: 'users', host: 'admin.example.com' })  // host-based routing

// ตัวอย่างการจัดการ versioning
@Controller({ path: 'users', version: '1' })
export class UsersV1Controller {
  @Get()
  findAll() {
    return { version: 1, users: [] };
  }
}

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller {
  @Get()
  findAll() {
    return { version: 2, users: [], meta: {} };
  }
}

// main.ts - เปิดใช้ versioning
import { VersioningType } from '@nestjs/common';

app.enableVersioning({
  type: VersioningType.URI,
  // type: VersioningType.HEADER,
  // type: VersioningType.MEDIA_TYPE,
  // type: VersioningType.CUSTOM,
});
```

---

## Sub-Routes (เส้นทางย่อย)

```typescript
@Controller('posts')
export class PostsController {
  // /posts
  @Get()
  findAll() { return []; }

  // /posts/:postId
  @Get(':postId')
  findOne(@Param('postId') postId: string) { return {}; }

  // /posts/:postId/comments
  @Get(':postId/comments')
  getComments(@Param('postId') postId: string) {
    return { postId, comments: [] };
  }

  // /posts/:postId/comments/:commentId
  @Get(':postId/comments/:commentId')
  getComment(
    @Param('postId') postId: string,
    @Param('commentId') commentId: string,
  ) {
    return { postId, commentId };
  }

  // /posts/:postId/comments
  @Post(':postId/comments')
  addComment(
    @Param('postId') postId: string,
    @Body() body: any,
  ) {
    return { postId, comment: body };
  }

  // /posts/:postId/like
  @Post(':postId/like')
  likePost(@Param('postId') postId: string) {
    return { postId, liked: true };
  }

  // /posts/:postId/tags
  @Get(':postId/tags')
  getTags(@Param('postId') postId: string) {
    return { postId, tags: [] };
  }
}
```

---

## Complete CRUD Controller Example (ตัวอย่าง CRUD Controller แบบสมบูรณ์)

มาสร้าง CRUD Controller สำหรับการจัดการ Blog Posts อย่างสมบูรณ์:

### DTOs
```typescript
// src/posts/dto/create-post.dto.ts
import {
  IsString,
  IsNotEmpty,
  IsOptional,
  IsArray,
  IsBoolean,
  IsEnum,
  MaxLength,
  MinLength,
} from 'class-validator';
import { Transform } from 'class-transformer';

export enum PostStatus {
  DRAFT = 'draft',
  PUBLISHED = 'published',
  ARCHIVED = 'archived',
}

export class CreatePostDto {
  @IsString()
  @IsNotEmpty({ message: 'กรุณาระบุชื่อบทความ' })
  @MaxLength(200, { message: 'ชื่อบทความต้องไม่เกิน 200 ตัวอักษร' })
  title: string;

  @IsString()
  @IsNotEmpty({ message: 'กรุณาระบุเนื้อหา' })
  @MinLength(10, { message: 'เนื้อหาต้องมีอย่างน้อย 10 ตัวอักษร' })
  content: string;

  @IsString()
  @IsOptional()
  excerpt?: string;

  @IsString()
  @IsOptional()
  coverImage?: string;

  @IsArray()
  @IsOptional()
  @IsString({ each: true })
  tags?: string[];

  @IsEnum(PostStatus)
  @IsOptional()
  status?: PostStatus = PostStatus.DRAFT;

  @IsBoolean()
  @IsOptional()
  allowComments?: boolean = true;

  @IsString()
  @IsOptional()
  categoryId?: string;
}

// src/posts/dto/update-post.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreatePostDto } from './create-post.dto';

export class UpdatePostDto extends PartialType(CreatePostDto) {}

// src/posts/dto/query-posts.dto.ts
import { IsOptional, IsString, IsNumber, IsEnum, Min } from 'class-validator';
import { Type } from 'class-transformer';
import { PostStatus } from './create-post.dto';

export class QueryPostsDto {
  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(1)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsEnum(PostStatus)
  status?: PostStatus;

  @IsOptional()
  @IsString()
  authorId?: string;

  @IsOptional()
  @IsString()
  categoryId?: string;

  @IsOptional()
  @IsString()
  tag?: string;

  @IsOptional()
  @IsString()
  sortBy?: string = 'createdAt';

  @IsOptional()
  @IsEnum(['asc', 'desc'])
  sortOrder?: 'asc' | 'desc' = 'desc';
}
```

### Post Entity
```typescript
// src/posts/entities/post.entity.ts
export interface PostAuthor {
  id: string;
  name: string;
  email: string;
  avatar?: string;
}

export interface Post {
  id: string;
  title: string;
  slug: string;
  content: string;
  excerpt?: string;
  coverImage?: string;
  tags: string[];
  status: string;
  allowComments: boolean;
  viewCount: number;
  likeCount: number;
  commentCount: number;
  authorId: string;
  author?: PostAuthor;
  categoryId?: string;
  publishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

### Post Service
```typescript
// src/posts/posts.service.ts
import {
  Injectable,
  NotFoundException,
  ForbiddenException,
  BadRequestException,
} from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid';
import { CreatePostDto, PostStatus } from './dto/create-post.dto';
import { UpdatePostDto } from './dto/update-post.dto';
import { QueryPostsDto } from './dto/query-posts.dto';
import { Post } from './entities/post.entity';

@Injectable()
export class PostsService {
  private posts: Post[] = [];

  private generateSlug(title: string): string {
    return title
      .toLowerCase()
      .replace(/[^a-z0-9ก-ฮ\s-]/g, '')
      .replace(/\s+/g, '-')
      .replace(/-+/g, '-')
      .trim();
  }

  async findAll(query: QueryPostsDto): Promise<{
    data: Post[];
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  }> {
    let posts = [...this.posts];

    if (query.search) {
      const search = query.search.toLowerCase();
      posts = posts.filter(
        p =>
          p.title.toLowerCase().includes(search) ||
          p.content.toLowerCase().includes(search) ||
          p.excerpt?.toLowerCase().includes(search),
      );
    }

    if (query.status) {
      posts = posts.filter(p => p.status === query.status);
    }

    if (query.authorId) {
      posts = posts.filter(p => p.authorId === query.authorId);
    }

    if (query.tag) {
      posts = posts.filter(p => p.tags.includes(query.tag));
    }

    // Sorting
    posts.sort((a, b) => {
      const field = query.sortBy || 'createdAt';
      const order = query.sortOrder === 'asc' ? 1 : -1;
      const aVal = (a as any)[field];
      const bVal = (b as any)[field];
      if (aVal < bVal) return -1 * order;
      if (aVal > bVal) return 1 * order;
      return 0;
    });

    const total = posts.length;
    const page = query.page || 1;
    const limit = query.limit || 10;
    const start = (page - 1) * limit;
    const data = posts.slice(start, start + limit);

    return {
      data,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async findOne(id: string): Promise<Post> {
    const post = this.posts.find(p => p.id === id);
    if (!post) {
      throw new NotFoundException(`ไม่พบบทความ ID: ${id}`);
    }
    return post;
  }

  async findBySlug(slug: string): Promise<Post> {
    const post = this.posts.find(p => p.slug === slug);
    if (!post) {
      throw new NotFoundException(`ไม่พบบทความ: ${slug}`);
    }
    return post;
  }

  async create(createPostDto: CreatePostDto, authorId: string): Promise<Post> {
    const slug = this.generateSlug(createPostDto.title);
    const existingSlug = this.posts.find(p => p.slug === slug);
    
    const post: Post = {
      id: uuidv4(),
      title: createPostDto.title,
      slug: existingSlug ? `${slug}-${Date.now()}` : slug,
      content: createPostDto.content,
      excerpt: createPostDto.excerpt,
      coverImage: createPostDto.coverImage,
      tags: createPostDto.tags || [],
      status: createPostDto.status || PostStatus.DRAFT,
      allowComments: createPostDto.allowComments ?? true,
      viewCount: 0,
      likeCount: 0,
      commentCount: 0,
      authorId,
      categoryId: createPostDto.categoryId,
      publishedAt: createPostDto.status === PostStatus.PUBLISHED ? new Date() : undefined,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.posts.push(post);
    return post;
  }

  async update(id: string, updatePostDto: UpdatePostDto, userId: string): Promise<Post> {
    const post = await this.findOne(id);

    if (post.authorId !== userId) {
      throw new ForbiddenException('คุณไม่มีสิทธิ์แก้ไขบทความนี้');
    }

    const index = this.posts.findIndex(p => p.id === id);
    
    this.posts[index] = {
      ...post,
      ...updatePostDto,
      updatedAt: new Date(),
      publishedAt: updatePostDto.status === PostStatus.PUBLISHED && !post.publishedAt
        ? new Date()
        : post.publishedAt,
    };

    return this.posts[index];
  }

  async remove(id: string, userId: string): Promise<void> {
    const post = await this.findOne(id);

    if (post.authorId !== userId) {
      throw new ForbiddenException('คุณไม่มีสิทธิ์ลบบทความนี้');
    }

    this.posts = this.posts.filter(p => p.id !== id);
  }

  async incrementViewCount(id: string): Promise<void> {
    const index = this.posts.findIndex(p => p.id === id);
    if (index !== -1) {
      this.posts[index].viewCount++;
    }
  }

  async likePost(id: string): Promise<{ liked: boolean; likeCount: number }> {
    const post = await this.findOne(id);
    const index = this.posts.findIndex(p => p.id === id);
    this.posts[index].likeCount++;
    return { liked: true, likeCount: this.posts[index].likeCount };
  }
}
```

### Complete CRUD Controller
```typescript
// src/posts/posts.controller.ts
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
  ParseUUIDPipe,
  UseGuards,
  UsePipes,
  ValidationPipe,
  UseInterceptors,
} from '@nestjs/common';
import { PostsService } from './posts.service';
import { CreatePostDto } from './dto/create-post.dto';
import { UpdatePostDto } from './dto/update-post.dto';
import { QueryPostsDto } from './dto/query-posts.dto';

// Mock decorator สำหรับตัวอย่าง
const AuthGuard = () => (target: any) => target;
const CurrentUser = () => (target: any, key: string, index: number) => {};

@Controller('posts')
@UsePipes(new ValidationPipe({ transform: true, whitelist: true }))
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  // GET /posts
  @Get()
  async findAll(@Query() query: QueryPostsDto) {
    return this.postsService.findAll(query);
  }

  // GET /posts/featured
  @Get('featured')
  async getFeatured() {
    return this.postsService.findAll({
      status: 'published' as any,
      limit: 5,
      sortBy: 'viewCount',
      sortOrder: 'desc',
    });
  }

  // GET /posts/slug/:slug
  @Get('slug/:slug')
  async findBySlug(@Param('slug') slug: string) {
    const post = await this.postsService.findBySlug(slug);
    await this.postsService.incrementViewCount(post.id);
    return post;
  }

  // GET /posts/:id
  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.postsService.findOne(id);
  }

  // POST /posts
  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(
    @Body() createPostDto: CreatePostDto,
    // @CurrentUser('id') userId: string, // จะใช้เมื่อมี auth
  ) {
    const userId = 'mock-user-id'; // สำหรับตัวอย่าง
    return this.postsService.create(createPostDto, userId);
  }

  // PUT /posts/:id
  @Put(':id')
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updatePostDto: UpdatePostDto,
  ) {
    const userId = 'mock-user-id';
    return this.postsService.update(id, updatePostDto, userId);
  }

  // PATCH /posts/:id
  @Patch(':id')
  async partialUpdate(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updatePostDto: UpdatePostDto,
  ) {
    const userId = 'mock-user-id';
    return this.postsService.update(id, updatePostDto, userId);
  }

  // DELETE /posts/:id
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param('id', ParseUUIDPipe) id: string) {
    const userId = 'mock-user-id';
    await this.postsService.remove(id, userId);
  }

  // POST /posts/:id/like
  @Post(':id/like')
  async likePost(@Param('id', ParseUUIDPipe) id: string) {
    return this.postsService.likePost(id);
  }

  // GET /posts/:id/comments
  @Get(':id/comments')
  async getComments(@Param('id', ParseUUIDPipe) id: string) {
    // ตรวจสอบว่าบทความมีอยู่จริง
    await this.postsService.findOne(id);
    return { postId: id, comments: [] };
  }

  // POST /posts/:id/comments
  @Post(':id/comments')
  @HttpCode(HttpStatus.CREATED)
  async addComment(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() body: { content: string },
  ) {
    await this.postsService.findOne(id);
    return { postId: id, comment: { id: 'new-comment-id', ...body } };
  }

  // PATCH /posts/:id/publish
  @Patch(':id/publish')
  async publishPost(@Param('id', ParseUUIDPipe) id: string) {
    const userId = 'mock-user-id';
    return this.postsService.update(id, { status: 'published' as any }, userId);
  }

  // PATCH /posts/:id/archive
  @Patch(':id/archive')
  async archivePost(@Param('id', ParseUUIDPipe) id: string) {
    const userId = 'mock-user-id';
    return this.postsService.update(id, { status: 'archived' as any }, userId);
  }
}
```

### Posts Module
```typescript
// src/posts/posts.module.ts
import { Module } from '@nestjs/common';
import { PostsController } from './posts.controller';
import { PostsService } from './posts.service';

@Module({
  controllers: [PostsController],
  providers: [PostsService],
  exports: [PostsService],
})
export class PostsModule {}
```

---

## Advanced Controller Patterns (รูปแบบ Controller ขั้นสูง)

### Controller พร้อม Swagger/OpenAPI Documentation
```typescript
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiParam,
  ApiQuery,
  ApiBearerAuth,
  ApiBody,
} from '@nestjs/swagger';

@ApiTags('Users')
@ApiBearerAuth()
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'ดึงรายการผู้ใช้ทั้งหมด' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiQuery({ name: 'limit', required: false, type: Number })
  @ApiResponse({ status: 200, description: 'ดึงข้อมูลสำเร็จ' })
  findAll(@Query('page') page: number, @Query('limit') limit: number) {
    return [];
  }

  @Get(':id')
  @ApiOperation({ summary: 'ดึงข้อมูลผู้ใช้ตาม ID' })
  @ApiParam({ name: 'id', description: 'User ID', type: String })
  @ApiResponse({ status: 200, description: 'พบข้อมูลผู้ใช้' })
  @ApiResponse({ status: 404, description: 'ไม่พบผู้ใช้' })
  findOne(@Param('id') id: string) {
    return {};
  }

  @Post()
  @ApiOperation({ summary: 'สร้างผู้ใช้ใหม่' })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({ status: 201, description: 'สร้างผู้ใช้สำเร็จ' })
  @ApiResponse({ status: 400, description: 'ข้อมูลไม่ถูกต้อง' })
  @ApiResponse({ status: 409, description: 'Email ซ้ำ' })
  create(@Body() createUserDto: any) {
    return {};
  }
}
```

### Controller พร้อม Caching
```typescript
import { CacheInterceptor, CacheTTL, CacheKey } from '@nestjs/cache-manager';

@Controller('products')
@UseInterceptors(CacheInterceptor)
export class ProductsController {
  // Cache 5 นาที
  @Get()
  @CacheTTL(300)
  findAll() {
    return [];
  }

  // Custom cache key
  @Get('featured')
  @CacheKey('featured-products')
  @CacheTTL(600)  // 10 นาที
  getFeatured() {
    return [];
  }

  // ไม่ cache (override)
  @Get('realtime')
  @CacheTTL(0)
  getRealtime() {
    return { timestamp: new Date() };
  }
}
```

### Controller พร้อม Rate Limiting
```typescript
import { Throttle, SkipThrottle } from '@nestjs/throttler';

@Controller('api')
@Throttle({ default: { limit: 100, ttl: 60000 } })  // 100 requests ต่อ 1 นาที
export class ApiController {
  @Get('data')
  getData() {
    return { data: [] };
  }

  // เพิ่ม limit สำหรับ endpoint นี้
  @Post('intensive')
  @Throttle({ default: { limit: 5, ttl: 60000 } })  // เฉพาะ 5 requests ต่อนาที
  intensiveOperation() {
    return { result: 'done' };
  }

  // ข้าม throttle
  @Get('health')
  @SkipThrottle()
  healthCheck() {
    return { status: 'ok' };
  }
}
```

---

## Error Handling ใน Controllers

```typescript
import {
  HttpException,
  HttpStatus,
  NotFoundException,
  BadRequestException,
  ConflictException,
  UnauthorizedException,
  ForbiddenException,
} from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string) {
    // Throw specific exceptions
    if (!id || isNaN(Number(id))) {
      throw new BadRequestException('ID ต้องเป็นตัวเลข');
    }

    const user = null; // สมมติว่าหาไม่พบ
    if (!user) {
      throw new NotFoundException({
        message: `ไม่พบผู้ใช้ ID: ${id}`,
        code: 'USER_NOT_FOUND',
      });
    }

    return user;
  }

  @Post()
  async create(@Body() body: any) {
    const existing = null; // สมมติว่าตรวจ email แล้ว
    if (existing) {
      throw new ConflictException('Email นี้ถูกใช้แล้ว');
    }

    // Custom HTTP exception
    if (body.age < 18) {
      throw new HttpException(
        {
          status: HttpStatus.FORBIDDEN,
          error: 'ต้องมีอายุอย่างน้อย 18 ปี',
          code: 'UNDERAGE_USER',
        },
        HttpStatus.FORBIDDEN,
      );
    }

    return { message: 'สร้างสำเร็จ' };
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **Module System**: การสร้างและจัดระเบียบ modules
2. **Feature Modules**: การแบ่ง modules ตาม business domain
3. **Shared Modules**: การแชร์ providers ระหว่าง modules
4. **Global Modules**: modules ที่ใช้ได้ทั่วทั้งแอปพลิเคชัน
5. **Dynamic Modules**: modules ที่ configure ได้ตามต้องการ
6. **Controllers**: การจัดการ HTTP requests
7. **Route Parameters**: การรับค่าจาก URL
8. **Query Parameters**: การรับค่าจาก query string
9. **Request Body**: การรับข้อมูลจาก request body
10. **Response Objects**: การส่ง responses
11. **HTTP Methods**: GET, POST, PUT, DELETE, PATCH และอื่นๆ
12. **Complete CRUD**: ตัวอย่างการสร้าง CRUD controller แบบสมบูรณ์

ในบทต่อไปเราจะเรียนรู้เรื่อง Services และ Providers อย่างละเอียด
