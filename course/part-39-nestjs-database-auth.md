# ตอนที่ 39: NestJS Database & Authentication

## TypeORM Integration (การผสาน TypeORM)

TypeORM คือ ORM ที่นิยมใช้กับ NestJS รองรับหลายฐานข้อมูล เช่น PostgreSQL, MySQL, SQLite, MongoDB

### การติดตั้ง TypeORM
```bash
# ติดตั้ง TypeORM และ database driver
npm install @nestjs/typeorm typeorm pg
# หรือสำหรับ MySQL
npm install @nestjs/typeorm typeorm mysql2
# หรือสำหรับ SQLite
npm install @nestjs/typeorm typeorm sqlite3
```

### การตั้งค่า TypeORM ใน AppModule
```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('DB_HOST', 'localhost'),
        port: configService.get<number>('DB_PORT', 5432),
        username: configService.get('DB_USER', 'postgres'),
        password: configService.get('DB_PASSWORD', 'password'),
        database: configService.get('DB_NAME', 'mydb'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: configService.get('NODE_ENV') !== 'production',
        logging: configService.get('NODE_ENV') === 'development',
        ssl: configService.get('NODE_ENV') === 'production'
          ? { rejectUnauthorized: false }
          : false,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

---

## Entity Setup (การตั้งค่า Entity)

### Base Entity
```typescript
// src/common/entities/base.entity.ts
import {
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
} from 'typeorm';

export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @DeleteDateColumn({ name: 'deleted_at', nullable: true })
  deletedAt?: Date;
}
```

### User Entity
```typescript
// src/users/entities/user.entity.ts
import {
  Entity,
  Column,
  Index,
  OneToMany,
  BeforeInsert,
  BeforeUpdate,
} from 'typeorm';
import * as bcrypt from 'bcrypt';
import { BaseEntity } from '../../common/entities/base.entity';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

export enum UserStatus {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
  BANNED = 'banned',
}

@Entity('users')
@Index(['email'], { unique: true })
export class User extends BaseEntity {
  @Column({ length: 100 })
  name: string;

  @Column({ unique: true, length: 255 })
  email: string;

  @Column({ select: false }) // ไม่ return ใน query ปกติ
  password: string;

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.USER,
  })
  role: UserRole;

  @Column({
    type: 'enum',
    enum: UserStatus,
    default: UserStatus.ACTIVE,
  })
  status: UserStatus;

  @Column({ nullable: true, name: 'avatar_url' })
  avatarUrl?: string;

  @Column({ nullable: true, name: 'refresh_token' })
  refreshToken?: string;

  @Column({ nullable: true, name: 'last_login_at' })
  lastLoginAt?: Date;

  @Column({ default: false, name: 'email_verified' })
  emailVerified: boolean;

  @Column({ nullable: true, name: 'email_verify_token' })
  emailVerifyToken?: string;

  // Relations
  @OneToMany('Order', 'user')
  orders: any[];

  @OneToMany('Post', 'author')
  posts: any[];

  // Hooks
  @BeforeInsert()
  @BeforeUpdate()
  async hashPassword() {
    if (this.password) {
      this.password = await bcrypt.hash(this.password, 12);
    }
  }

  // Methods
  async validatePassword(password: string): Promise<boolean> {
    return bcrypt.compare(password, this.password);
  }

  toSafeObject() {
    const { password, refreshToken, emailVerifyToken, ...safe } = this;
    return safe;
  }
}
```

### Product Entity
```typescript
// src/products/entities/product.entity.ts
import {
  Entity,
  Column,
  Index,
  ManyToOne,
  OneToMany,
  JoinColumn,
  ManyToMany,
  JoinTable,
} from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';

export enum ProductStatus {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
  OUT_OF_STOCK = 'out_of_stock',
}

@Entity('products')
@Index(['slug'], { unique: true })
@Index(['status'])
@Index(['categoryId'])
export class Product extends BaseEntity {
  @Column({ length: 200 })
  name: string;

  @Column({ unique: true, length: 250 })
  slug: string;

  @Column({ type: 'text', nullable: true })
  description?: string;

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  price: number;

  @Column({ type: 'decimal', precision: 10, scale: 2, nullable: true, name: 'compare_price' })
  comparePrice?: number;

  @Column({ default: 0 })
  stock: number;

  @Column({ nullable: true, name: 'image_url' })
  imageUrl?: string;

  @Column('simple-array', { nullable: true })
  images?: string[];

  @Column({
    type: 'enum',
    enum: ProductStatus,
    default: ProductStatus.ACTIVE,
  })
  status: ProductStatus;

  @Column({ default: 0, name: 'view_count' })
  viewCount: number;

  @Column({ nullable: true, name: 'category_id' })
  categoryId?: string;

  // Relations
  @ManyToOne('Category', 'products', { nullable: true, eager: false })
  @JoinColumn({ name: 'category_id' })
  category?: any;

  @ManyToMany('Tag', { eager: false })
  @JoinTable({
    name: 'product_tags',
    joinColumn: { name: 'product_id' },
    inverseJoinColumn: { name: 'tag_id' },
  })
  tags?: any[];

  @OneToMany('OrderItem', 'product')
  orderItems?: any[];
}
```

---

## Repository Pattern (รูปแบบ Repository)

```typescript
// src/users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, FindOptionsWhere, Like, ILike } from 'typeorm';
import { User, UserRole, UserStatus } from './entities/user.entity';

export interface UserFilter {
  search?: string;
  role?: UserRole;
  status?: UserStatus;
  page?: number;
  limit?: number;
  orderBy?: keyof User;
  order?: 'ASC' | 'DESC';
}

@Injectable()
export class UsersRepository {
  constructor(
    @InjectRepository(User)
    private readonly repository: Repository<User>,
  ) {}

  async findAll(filter: UserFilter = {}) {
    const {
      search,
      role,
      status,
      page = 1,
      limit = 10,
      orderBy = 'createdAt',
      order = 'DESC',
    } = filter;

    const where: FindOptionsWhere<User> = {};

    if (role) where.role = role;
    if (status) where.status = status;

    const queryBuilder = this.repository.createQueryBuilder('user');
    
    if (search) {
      queryBuilder.where(
        '(user.name ILIKE :search OR user.email ILIKE :search)',
        { search: `%${search}%` }
      );
    }

    if (role) queryBuilder.andWhere('user.role = :role', { role });
    if (status) queryBuilder.andWhere('user.status = :status', { status });

    queryBuilder
      .orderBy(`user.${orderBy}`, order)
      .skip((page - 1) * limit)
      .take(limit);

    const [users, total] = await queryBuilder.getManyAndCount();

    return {
      data: users.map(u => u.toSafeObject()),
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async findById(id: string): Promise<User | null> {
    return this.repository.findOne({ where: { id } });
  }

  async findByEmail(email: string, includePassword = false): Promise<User | null> {
    const query = this.repository.createQueryBuilder('user')
      .where('user.email = :email', { email: email.toLowerCase() });
    
    if (includePassword) {
      query.addSelect('user.password');
    }
    
    return query.getOne();
  }

  async findByRefreshToken(token: string): Promise<User | null> {
    return this.repository.findOne({ where: { refreshToken: token } });
  }

  async create(data: Partial<User>): Promise<User> {
    const user = this.repository.create(data);
    return this.repository.save(user);
  }

  async update(id: string, data: Partial<User>): Promise<User> {
    await this.repository.update(id, data);
    return this.findById(id);
  }

  async softDelete(id: string): Promise<void> {
    await this.repository.softDelete(id);
  }

  async hardDelete(id: string): Promise<void> {
    await this.repository.delete(id);
  }

  async countByRole(role: UserRole): Promise<number> {
    return this.repository.count({ where: { role } });
  }
}
```

---

## Migrations (การ Migrate ฐานข้อมูล)

### ตั้งค่า TypeORM CLI
```typescript
// typeorm.config.ts
import { DataSource } from 'typeorm';
import * as dotenv from 'dotenv';

dotenv.config();

export default new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT) || 5432,
  username: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'password',
  database: process.env.DB_NAME || 'mydb',
  entities: ['src/**/*.entity{.ts,.js}'],
  migrations: ['src/migrations/*{.ts,.js}'],
  synchronize: false,
  logging: true,
});
```

```json
// package.json scripts
{
  "scripts": {
    "typeorm": "ts-node -r tsconfig-paths/register ./node_modules/.bin/typeorm",
    "migration:generate": "npm run typeorm -- migration:generate -d typeorm.config.ts",
    "migration:run": "npm run typeorm -- migration:run -d typeorm.config.ts",
    "migration:revert": "npm run typeorm -- migration:revert -d typeorm.config.ts",
    "migration:create": "npm run typeorm -- migration:create"
  }
}
```

### Migration File
```typescript
// src/migrations/1700000000000-CreateUsersTable.ts
import { MigrationInterface, QueryRunner, Table, TableIndex } from 'typeorm';

export class CreateUsersTable1700000000000 implements MigrationInterface {
  name = 'CreateUsersTable1700000000000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // สร้าง enum types
    await queryRunner.query(`
      CREATE TYPE "users_role_enum" AS ENUM('admin', 'user', 'moderator')
    `);
    
    await queryRunner.query(`
      CREATE TYPE "users_status_enum" AS ENUM('active', 'inactive', 'banned')
    `);

    // สร้าง table
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()',
          },
          { name: 'name', type: 'varchar', length: '100' },
          { name: 'email', type: 'varchar', length: '255', isUnique: true },
          { name: 'password', type: 'varchar' },
          {
            name: 'role',
            type: 'enum',
            enum: ['admin', 'user', 'moderator'],
            default: "'user'",
          },
          {
            name: 'status',
            type: 'enum',
            enum: ['active', 'inactive', 'banned'],
            default: "'active'",
          },
          { name: 'avatar_url', type: 'varchar', isNullable: true },
          { name: 'refresh_token', type: 'varchar', isNullable: true },
          { name: 'last_login_at', type: 'timestamp', isNullable: true },
          { name: 'email_verified', type: 'boolean', default: false },
          { name: 'email_verify_token', type: 'varchar', isNullable: true },
          { name: 'created_at', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
          { name: 'updated_at', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
          { name: 'deleted_at', type: 'timestamp', isNullable: true },
        ],
      }),
    );

    // สร้าง indexes
    await queryRunner.createIndex(
      'users',
      new TableIndex({ name: 'IDX_USERS_EMAIL', columnNames: ['email'] }),
    );

    await queryRunner.createIndex(
      'users',
      new TableIndex({ name: 'IDX_USERS_STATUS', columnNames: ['status'] }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
    await queryRunner.query('DROP TYPE "users_role_enum"');
    await queryRunner.query('DROP TYPE "users_status_enum"');
  }
}
```

---

## Prisma Integration (การผสาน Prisma)

### การติดตั้ง Prisma
```bash
npm install prisma @prisma/client
npx prisma init
```

### Prisma Schema
```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id              String    @id @default(uuid())
  name            String
  email           String    @unique
  password        String
  role            Role      @default(USER)
  status          UserStatus @default(ACTIVE)
  avatarUrl       String?   @map("avatar_url")
  refreshToken    String?   @map("refresh_token")
  lastLoginAt     DateTime? @map("last_login_at")
  emailVerified   Boolean   @default(false) @map("email_verified")
  createdAt       DateTime  @default(now()) @map("created_at")
  updatedAt       DateTime  @updatedAt @map("updated_at")
  
  posts     Post[]
  orders    Order[]

  @@map("users")
}

model Post {
  id          String      @id @default(uuid())
  title       String
  content     String
  published   Boolean     @default(false)
  authorId    String      @map("author_id")
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  
  author      User        @relation(fields: [authorId], references: [id])
  
  @@map("posts")
}

enum Role {
  ADMIN
  USER
  MODERATOR
}

enum UserStatus {
  ACTIVE
  INACTIVE
  BANNED
}
```

### Prisma Service
```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    super({
      log: process.env.NODE_ENV === 'development'
        ? ['query', 'info', 'warn', 'error']
        : ['error'],
    });
  }

  async onModuleInit() {
    await this.$connect();
    console.log('เชื่อมต่อ Prisma สำเร็จ');
  }

  async onModuleDestroy() {
    await this.$disconnect();
    console.log('ตัดการเชื่อมต่อ Prisma');
  }

  async cleanDatabase() {
    if (process.env.NODE_ENV !== 'test') {
      throw new Error('cleanDatabase สามารถใช้ได้เฉพาะใน test environment');
    }
    const models = Reflect.ownKeys(this).filter((key) => key[0] !== '_');
    return Promise.all(models.map((modelKey) => (this as any)[modelKey].deleteMany()));
  }
}
```

### การใช้ Prisma Service
```typescript
// src/users/users.service.ts (Prisma version)
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { User, Prisma } from '@prisma/client';
import * as bcrypt from 'bcrypt';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async findAll(params?: {
    skip?: number;
    take?: number;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }) {
    const { skip = 0, take = 10, where, orderBy } = params || {};
    
    const [users, total] = await Promise.all([
      this.prisma.user.findMany({
        skip,
        take,
        where,
        orderBy: orderBy || { createdAt: 'desc' },
        select: {
          id: true,
          name: true,
          email: true,
          role: true,
          status: true,
          createdAt: true,
          updatedAt: true,
        },
      }),
      this.prisma.user.count({ where }),
    ]);

    return { users, total };
  }

  async findById(id: string): Promise<Omit<User, 'password'>> {
    const user = await this.prisma.user.findUnique({
      where: { id },
      select: {
        id: true,
        name: true,
        email: true,
        role: true,
        status: true,
        avatarUrl: true,
        emailVerified: true,
        lastLoginAt: true,
        createdAt: true,
        updatedAt: true,
      },
    });

    if (!user) {
      throw new NotFoundException(`ไม่พบผู้ใช้ ID: ${id}`);
    }

    return user as any;
  }

  async create(data: Prisma.UserCreateInput): Promise<Omit<User, 'password'>> {
    const existing = await this.prisma.user.findUnique({
      where: { email: data.email },
    });

    if (existing) {
      throw new ConflictException('Email นี้ถูกใช้แล้ว');
    }

    const hashedPassword = await bcrypt.hash(data.password, 12);
    
    const user = await this.prisma.user.create({
      data: { ...data, password: hashedPassword },
      select: {
        id: true,
        name: true,
        email: true,
        role: true,
        status: true,
        createdAt: true,
      },
    });

    return user as any;
  }
}
```

---

## JWT Authentication (การยืนยันตัวตนด้วย JWT)

### การติดตั้ง packages
```bash
npm install @nestjs/passport @nestjs/jwt passport passport-jwt passport-local
npm install @types/passport-jwt @types/passport-local -D
```

### Auth Module Setup
```typescript
// src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { JwtStrategy } from './strategies/jwt.strategy';
import { LocalStrategy } from './strategies/local.strategy';
import { JwtRefreshStrategy } from './strategies/jwt-refresh.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get('JWT_SECRET', 'default-secret'),
        signOptions: {
          expiresIn: configService.get('JWT_EXPIRES_IN', '15m'),
        },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [
    AuthService,
    JwtStrategy,
    LocalStrategy,
    JwtRefreshStrategy,
  ],
  exports: [AuthService, JwtModule],
})
export class AuthModule {}
```

### JWT Strategy
```typescript
// src/auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { UsersService } from '../../users/users.service';

export interface JwtPayload {
  sub: string;         // User ID
  email: string;
  role: string;
  iat?: number;
  exp?: number;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    private configService: ConfigService,
    private usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('JWT_SECRET', 'default-secret'),
    });
  }

  async validate(payload: JwtPayload) {
    const user = await this.usersService.findById(payload.sub);
    
    if (!user) {
      throw new UnauthorizedException('Token ไม่ถูกต้อง');
    }

    return {
      id: payload.sub,
      email: payload.email,
      role: payload.role,
    };
  }
}

// src/auth/strategies/local.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AuthService } from '../auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy, 'local') {
  constructor(private authService: AuthService) {
    super({ usernameField: 'email' });
  }

  async validate(email: string, password: string) {
    const user = await this.authService.validateUser(email, password);
    if (!user) {
      throw new UnauthorizedException('Email หรือรหัสผ่านไม่ถูกต้อง');
    }
    return user;
  }
}

// src/auth/strategies/jwt-refresh.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { Request } from 'express';
import { ConfigService } from '@nestjs/config';
import { AuthService } from '../auth.service';

@Injectable()
export class JwtRefreshStrategy extends PassportStrategy(Strategy, 'jwt-refresh') {
  constructor(
    private configService: ConfigService,
    private authService: AuthService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([
        (request: Request) => {
          return request?.cookies?.refreshToken;
        },
        ExtractJwt.fromBodyField('refreshToken'),
      ]),
      secretOrKey: configService.get('JWT_REFRESH_SECRET', 'refresh-secret'),
      passReqToCallback: true,
    });
  }

  async validate(req: Request, payload: any) {
    const refreshToken =
      req.cookies?.refreshToken || req.body?.refreshToken;
    
    const user = await this.authService.validateRefreshToken(
      payload.sub,
      refreshToken,
    );
    
    if (!user) {
      throw new UnauthorizedException('Refresh token ไม่ถูกต้อง');
    }
    
    return user;
  }
}
```

---

## Guards (JwtAuthGuard)

```typescript
// src/auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // ตรวจสอบว่า route เป็น public หรือไม่
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    return super.canActivate(context);
  }

  handleRequest(err: any, user: any, info: any) {
    if (err || !user) {
      throw err || new UnauthorizedException('ต้องการการยืนยันตัวตน');
    }
    return user;
  }
}

// src/auth/guards/local-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}

// src/auth/guards/jwt-refresh.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtRefreshGuard extends AuthGuard('jwt-refresh') {}

// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();

    if (!user) {
      throw new ForbiddenException('ต้องการการยืนยันตัวตน');
    }

    const hasRole = requiredRoles.some(role => user.role === role);

    if (!hasRole) {
      throw new ForbiddenException('ไม่มีสิทธิ์เข้าถึง');
    }

    return true;
  }
}
```

---

## Decorators (@CurrentUser)

```typescript
// src/auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    if (!user) return undefined;

    return data ? user[data] : user;
  },
);

// src/auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// src/auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// src/auth/decorators/get-refresh-token.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { Request } from 'express';

export const GetRefreshToken = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest<Request>();
    return request.cookies?.refreshToken || request.body?.refreshToken;
  },
);
```

---

## Auth Service

```typescript
// src/auth/auth.service.ts
import {
  Injectable,
  UnauthorizedException,
  BadRequestException,
  ConflictException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import * as bcrypt from 'bcrypt';
import { v4 as uuidv4 } from 'uuid';
import { UsersService } from '../users/users.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

export interface TokenResponse {
  accessToken: string;
  refreshToken: string;
  user: {
    id: string;
    name: string;
    email: string;
    role: string;
  };
}

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
    private configService: ConfigService,
  ) {}

  async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmailWithPassword(email);
    
    if (!user) {
      return null;
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);
    
    if (!isPasswordValid) {
      return null;
    }

    const { password: _, ...result } = user;
    return result;
  }

  async register(registerDto: RegisterDto): Promise<TokenResponse> {
    const { name, email, password } = registerDto;
    
    const existingUser = await this.usersService.findByEmail(email);
    if (existingUser) {
      throw new ConflictException('Email นี้ถูกใช้แล้ว');
    }

    const hashedPassword = await bcrypt.hash(password, 12);
    
    const user = await this.usersService.create({
      name,
      email,
      password: hashedPassword,
    });

    return this.generateTokens(user);
  }

  async login(user: any): Promise<TokenResponse> {
    const tokens = await this.generateTokens(user);
    
    // บันทึก refresh token ลงฐานข้อมูล (hashed)
    const hashedRefreshToken = await bcrypt.hash(tokens.refreshToken, 10);
    await this.usersService.updateRefreshToken(user.id, hashedRefreshToken);
    
    // อัปเดต lastLoginAt
    await this.usersService.updateLastLogin(user.id);

    return tokens;
  }

  async logout(userId: string): Promise<void> {
    await this.usersService.updateRefreshToken(userId, null);
  }

  async refreshTokens(userId: string, refreshToken: string): Promise<TokenResponse> {
    const user = await this.usersService.findByIdWithRefreshToken(userId);
    
    if (!user || !user.refreshToken) {
      throw new UnauthorizedException('ไม่มีสิทธิ์เข้าถึง');
    }

    const isRefreshTokenValid = await bcrypt.compare(refreshToken, user.refreshToken);
    
    if (!isRefreshTokenValid) {
      throw new UnauthorizedException('Refresh token ไม่ถูกต้อง');
    }

    const tokens = await this.generateTokens(user);
    
    const hashedRefreshToken = await bcrypt.hash(tokens.refreshToken, 10);
    await this.usersService.updateRefreshToken(user.id, hashedRefreshToken);

    return tokens;
  }

  async validateRefreshToken(userId: string, refreshToken: string) {
    const user = await this.usersService.findByIdWithRefreshToken(userId);
    
    if (!user?.refreshToken) return null;
    
    const isValid = await bcrypt.compare(refreshToken, user.refreshToken);
    return isValid ? user : null;
  }

  private async generateTokens(user: any): Promise<TokenResponse> {
    const payload = {
      sub: user.id,
      email: user.email,
      role: user.role,
    };

    const [accessToken, refreshToken] = await Promise.all([
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_SECRET'),
        expiresIn: this.configService.get('JWT_EXPIRES_IN', '15m'),
      }),
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
        expiresIn: this.configService.get('JWT_REFRESH_EXPIRES_IN', '7d'),
      }),
    ]);

    return {
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role,
      },
    };
  }

  async changePassword(userId: string, oldPassword: string, newPassword: string): Promise<void> {
    const user = await this.usersService.findByIdWithPassword(userId);
    
    const isOldPasswordValid = await bcrypt.compare(oldPassword, user.password);
    if (!isOldPasswordValid) {
      throw new BadRequestException('รหัสผ่านเดิมไม่ถูกต้อง');
    }

    const hashedNewPassword = await bcrypt.hash(newPassword, 12);
    await this.usersService.updatePassword(userId, hashedNewPassword);
    
    // Invalidate all refresh tokens
    await this.usersService.updateRefreshToken(userId, null);
  }

  async forgotPassword(email: string): Promise<void> {
    const user = await this.usersService.findByEmail(email);
    
    if (!user) {
      // ไม่บอกว่าไม่พบ email เพื่อความปลอดภัย
      return;
    }

    const resetToken = uuidv4();
    const resetTokenExpiry = new Date(Date.now() + 3600000); // 1 ชั่วโมง
    
    await this.usersService.setPasswordResetToken(
      user.id,
      await bcrypt.hash(resetToken, 10),
      resetTokenExpiry,
    );

    // ส่ง email (ต้อง implement EmailService)
    console.log(`Reset token สำหรับ ${email}: ${resetToken}`);
  }

  async resetPassword(token: string, newPassword: string): Promise<void> {
    const user = await this.usersService.findByPasswordResetToken(token);
    
    if (!user) {
      throw new BadRequestException('Token ไม่ถูกต้องหรือหมดอายุ');
    }

    const hashedPassword = await bcrypt.hash(newPassword, 12);
    await this.usersService.updatePassword(user.id, hashedPassword);
    await this.usersService.clearPasswordResetToken(user.id);
    await this.usersService.updateRefreshToken(user.id, null);
  }
}
```

---

## Auth Controller

```typescript
// src/auth/auth.controller.ts
import {
  Controller,
  Post,
  Get,
  Body,
  UseGuards,
  HttpCode,
  HttpStatus,
  Res,
  Req,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { ChangePasswordDto } from './dto/change-password.dto';
import { ForgotPasswordDto } from './dto/forgot-password.dto';
import { ResetPasswordDto } from './dto/reset-password.dto';
import { LocalAuthGuard } from './guards/local-auth.guard';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { JwtRefreshGuard } from './guards/jwt-refresh.guard';
import { CurrentUser } from './decorators/current-user.decorator';
import { Public } from './decorators/public.decorator';
import { GetRefreshToken } from './decorators/get-refresh-token.decorator';

@Controller('auth')
@UseGuards(JwtAuthGuard) // Global guard สำหรับ controller นี้
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  @Public()
  @HttpCode(HttpStatus.CREATED)
  async register(
    @Body() registerDto: RegisterDto,
    @Res({ passthrough: true }) res: Response,
  ) {
    const result = await this.authService.register(registerDto);
    
    // Set refresh token เป็น HTTP-only cookie
    res.cookie('refreshToken', result.refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 วัน
    });

    return {
      accessToken: result.accessToken,
      user: result.user,
      message: 'ลงทะเบียนสำเร็จ',
    };
  }

  @Post('login')
  @Public()
  @UseGuards(LocalAuthGuard)
  @HttpCode(HttpStatus.OK)
  async login(
    @CurrentUser() user: any,
    @Res({ passthrough: true }) res: Response,
  ) {
    const result = await this.authService.login(user);

    res.cookie('refreshToken', result.refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });

    return {
      accessToken: result.accessToken,
      user: result.user,
      message: 'เข้าสู่ระบบสำเร็จ',
    };
  }

  @Post('logout')
  @HttpCode(HttpStatus.OK)
  async logout(
    @CurrentUser('id') userId: string,
    @Res({ passthrough: true }) res: Response,
  ) {
    await this.authService.logout(userId);

    res.clearCookie('refreshToken');

    return { message: 'ออกจากระบบสำเร็จ' };
  }

  @Post('refresh')
  @Public()
  @UseGuards(JwtRefreshGuard)
  @HttpCode(HttpStatus.OK)
  async refreshTokens(
    @CurrentUser('id') userId: string,
    @GetRefreshToken() refreshToken: string,
    @Res({ passthrough: true }) res: Response,
  ) {
    const result = await this.authService.refreshTokens(userId, refreshToken);

    res.cookie('refreshToken', result.refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });

    return {
      accessToken: result.accessToken,
      message: 'ต่ออายุ token สำเร็จ',
    };
  }

  @Get('me')
  getProfile(@CurrentUser() user: any) {
    return { user };
  }

  @Post('change-password')
  @HttpCode(HttpStatus.OK)
  async changePassword(
    @CurrentUser('id') userId: string,
    @Body() changePasswordDto: ChangePasswordDto,
  ) {
    await this.authService.changePassword(
      userId,
      changePasswordDto.oldPassword,
      changePasswordDto.newPassword,
    );
    return { message: 'เปลี่ยนรหัสผ่านสำเร็จ' };
  }

  @Post('forgot-password')
  @Public()
  @HttpCode(HttpStatus.OK)
  async forgotPassword(@Body() forgotPasswordDto: ForgotPasswordDto) {
    await this.authService.forgotPassword(forgotPasswordDto.email);
    return { message: 'ส่ง email รีเซ็ตรหัสผ่านแล้ว (ถ้า email มีอยู่ในระบบ)' };
  }

  @Post('reset-password')
  @Public()
  @HttpCode(HttpStatus.OK)
  async resetPassword(@Body() resetPasswordDto: ResetPasswordDto) {
    await this.authService.resetPassword(
      resetPasswordDto.token,
      resetPasswordDto.newPassword,
    );
    return { message: 'รีเซ็ตรหัสผ่านสำเร็จ' };
  }
}
```

---

## Auth DTOs

```typescript
// src/auth/dto/register.dto.ts
import {
  IsString,
  IsEmail,
  IsNotEmpty,
  MinLength,
  MaxLength,
  Matches,
} from 'class-validator';
import { Transform } from 'class-transformer';

export class RegisterDto {
  @IsString()
  @IsNotEmpty({ message: 'กรุณาระบุชื่อ' })
  @MaxLength(100, { message: 'ชื่อต้องไม่เกิน 100 ตัวอักษร' })
  @Transform(({ value }) => value?.trim())
  name: string;

  @IsEmail({}, { message: 'Email ไม่ถูกต้อง' })
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(8, { message: 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร' })
  @MaxLength(50, { message: 'รหัสผ่านต้องไม่เกิน 50 ตัวอักษร' })
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'รหัสผ่านต้องมีตัวพิมพ์เล็ก ตัวพิมพ์ใหญ่ และตัวเลข',
  })
  password: string;
}

// src/auth/dto/login.dto.ts
import { IsEmail, IsString, IsNotEmpty } from 'class-validator';

export class LoginDto {
  @IsEmail({}, { message: 'Email ไม่ถูกต้อง' })
  email: string;

  @IsString()
  @IsNotEmpty({ message: 'กรุณาระบุรหัสผ่าน' })
  password: string;
}

// src/auth/dto/change-password.dto.ts
import { IsString, IsNotEmpty, MinLength } from 'class-validator';

export class ChangePasswordDto {
  @IsString()
  @IsNotEmpty()
  oldPassword: string;

  @IsString()
  @MinLength(8)
  newPassword: string;
}

// src/auth/dto/forgot-password.dto.ts
import { IsEmail } from 'class-validator';

export class ForgotPasswordDto {
  @IsEmail()
  email: string;
}

// src/auth/dto/reset-password.dto.ts
import { IsString, IsNotEmpty, MinLength } from 'class-validator';

export class ResetPasswordDto {
  @IsString()
  @IsNotEmpty()
  token: string;

  @IsString()
  @MinLength(8)
  newPassword: string;
}
```

---

## Role-Based Authorization (การควบคุมสิทธิ์ตาม Role)

```typescript
// src/auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

export const ROLES_KEY = 'roles';
export const Roles = (...roles: UserRole[]) => SetMetadata(ROLES_KEY, roles);

// การใช้งาน Role-based authorization
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles(UserRole.ADMIN)
  getAllUsers() {
    return { users: [] };
  }

  @Post('users/:id/ban')
  @Roles(UserRole.ADMIN, UserRole.MODERATOR)
  banUser(@Param('id') id: string) {
    return { message: `ระงับการใช้งาน user ${id}` };
  }

  @Get('dashboard')
  @Roles(UserRole.ADMIN)
  getDashboard() {
    return {
      totalUsers: 100,
      activeUsers: 80,
      totalOrders: 500,
    };
  }
}

// Permission-based Guard (ละเอียดกว่า Role-based)
export const PERMISSIONS_KEY = 'permissions';
export const RequirePermissions = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermissions) return true;

    const { user } = context.switchToHttp().getRequest();

    if (!user?.permissions) {
      throw new ForbiddenException('ไม่มีสิทธิ์เข้าถึง');
    }

    const hasPermission = requiredPermissions.every(permission =>
      user.permissions.includes(permission),
    );

    if (!hasPermission) {
      throw new ForbiddenException('ไม่มีสิทธิ์ในการดำเนินการนี้');
    }

    return true;
  }
}
```

---

## Password Hashing (การเข้ารหัสรหัสผ่าน)

```typescript
// src/common/utils/password.util.ts
import * as bcrypt from 'bcrypt';
import * as crypto from 'crypto';

export class PasswordUtil {
  static readonly SALT_ROUNDS = 12;

  static async hash(password: string): Promise<string> {
    return bcrypt.hash(password, this.SALT_ROUNDS);
  }

  static async compare(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }

  static generateRandomPassword(length: number = 16): string {
    return crypto.randomBytes(length).toString('base64').slice(0, length);
  }

  static generateResetToken(): { token: string; hashedToken: string } {
    const token = crypto.randomBytes(32).toString('hex');
    const hashedToken = crypto
      .createHash('sha256')
      .update(token)
      .digest('hex');
    return { token, hashedToken };
  }

  static isStrongPassword(password: string): {
    isStrong: boolean;
    errors: string[];
  } {
    const errors: string[] = [];

    if (password.length < 8) {
      errors.push('รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร');
    }
    if (!/[a-z]/.test(password)) {
      errors.push('ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว');
    }
    if (!/[A-Z]/.test(password)) {
      errors.push('ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว');
    }
    if (!/\d/.test(password)) {
      errors.push('ต้องมีตัวเลขอย่างน้อย 1 ตัว');
    }
    if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
      errors.push('ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว');
    }

    return { isStrong: errors.length === 0, errors };
  }
}
```

---

## Refresh Tokens (การจัดการ Refresh Tokens)

```typescript
// src/auth/token.service.ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import * as bcrypt from 'bcrypt';

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
  accessTokenExpiry: Date;
  refreshTokenExpiry: Date;
}

@Injectable()
export class TokenService {
  constructor(
    private jwtService: JwtService,
    private configService: ConfigService,
  ) {}

  async generateTokenPair(payload: {
    sub: string;
    email: string;
    role: string;
  }): Promise<TokenPair> {
    const accessTokenExpiry = new Date(Date.now() + 15 * 60 * 1000); // 15 นาที
    const refreshTokenExpiry = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // 7 วัน

    const [accessToken, refreshToken] = await Promise.all([
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_SECRET'),
        expiresIn: '15m',
      }),
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
        expiresIn: '7d',
      }),
    ]);

    return {
      accessToken,
      refreshToken,
      accessTokenExpiry,
      refreshTokenExpiry,
    };
  }

  async verifyAccessToken(token: string): Promise<any> {
    return this.jwtService.verifyAsync(token, {
      secret: this.configService.get('JWT_SECRET'),
    });
  }

  async verifyRefreshToken(token: string): Promise<any> {
    return this.jwtService.verifyAsync(token, {
      secret: this.configService.get('JWT_REFRESH_SECRET'),
    });
  }

  async hashRefreshToken(token: string): Promise<string> {
    return bcrypt.hash(token, 10);
  }

  async compareRefreshToken(token: string, hash: string): Promise<boolean> {
    return bcrypt.compare(token, hash);
  }

  decodeToken(token: string): any {
    return this.jwtService.decode(token);
  }
}
```

---

## Complete Auth Flow (กระบวนการ Auth แบบสมบูรณ์)

### .env file
```bash
# .env
DATABASE_URL=postgresql://postgres:password@localhost:5432/mydb
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=mydb

JWT_SECRET=your-super-secret-jwt-key-change-this-in-production
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=your-super-secret-refresh-key-change-this-in-production
JWT_REFRESH_EXPIRES_IN=7d

NODE_ENV=development
PORT=3000
```

### Global Setup (main.ts)
```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe, ClassSerializerInterceptor } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import * as cookieParser from 'cookie-parser';
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Security headers
  app.use(helmet());

  // Cookie parser
  app.use(cookieParser());

  // CORS
  app.enableCors({
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
  });

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,           // ตัด properties ที่ไม่อยู่ใน DTO
      forbidNonWhitelisted: true, // throw error ถ้ามี extra properties
      transform: true,           // auto-transform types
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  // Global serializer interceptor
  app.useGlobalInterceptors(
    new ClassSerializerInterceptor(app.get(Reflector)),
  );

  // Global prefix
  app.setGlobalPrefix('api/v1');

  const port = process.env.PORT || 3000;
  await app.listen(port);
  
  console.log(`แอปพลิเคชันกำลังทำงานที่ http://localhost:${port}/api/v1`);
}

bootstrap();
```

### ตัวอย่าง Protected Routes
```typescript
// src/users/users.controller.ts
import {
  Controller,
  Get,
  Put,
  Delete,
  Body,
  Param,
  UseGuards,
  ParseUUIDPipe,
  ForbiddenException,
} from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { Roles } from '../auth/decorators/roles.decorator';
import { UserRole } from '../auth/decorators/roles.decorator';
import { UsersService } from './users.service';

@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  constructor(private usersService: UsersService) {}

  // ผู้ใช้ทั่วไปดูข้อมูลตัวเองได้
  @Get('profile')
  getMyProfile(@CurrentUser() user: any) {
    return this.usersService.findById(user.id);
  }

  // Admin เท่านั้นที่ดูรายการผู้ใช้ทั้งหมดได้
  @Get()
  @UseGuards(RolesGuard)
  @Roles(UserRole.ADMIN)
  findAll() {
    return this.usersService.findAll();
  }

  // Admin หรือเจ้าของบัญชีเท่านั้นที่แก้ไขได้
  @Put(':id')
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updateDto: any,
    @CurrentUser() currentUser: any,
  ) {
    if (currentUser.id !== id && currentUser.role !== UserRole.ADMIN) {
      throw new ForbiddenException('ไม่มีสิทธิ์แก้ไขข้อมูลผู้ใช้นี้');
    }
    return this.usersService.update(id, updateDto);
  }

  // Admin เท่านั้นที่ลบได้
  @Delete(':id')
  @UseGuards(RolesGuard)
  @Roles(UserRole.ADMIN)
  async remove(
    @Param('id', ParseUUIDPipe) id: string,
    @CurrentUser('id') currentUserId: string,
  ) {
    if (id === currentUserId) {
      throw new ForbiddenException('ไม่สามารถลบบัญชีตัวเองได้');
    }
    return this.usersService.delete(id);
  }
}
```

---

## Rate Limiting สำหรับ Auth

```typescript
// ติดตั้ง throttler
// npm install @nestjs/throttler

// src/app.module.ts - เพิ่ม ThrottlerModule
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,       // 1 วินาที
        limit: 3,        // 3 requests
      },
      {
        name: 'medium',
        ttl: 10000,      // 10 วินาที
        limit: 20,
      },
      {
        name: 'long',
        ttl: 60000,      // 1 นาที
        limit: 100,
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}

// ใช้ throttle เฉพาะ auth endpoints
import { Throttle, SkipThrottle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle({ short: { limit: 5, ttl: 60000 } }) // 5 ครั้งต่อนาที
  async login() {}

  @Post('register')
  @Throttle({ short: { limit: 3, ttl: 60000 } }) // 3 ครั้งต่อนาที
  async register() {}

  @Post('forgot-password')
  @Throttle({ short: { limit: 3, ttl: 300000 } }) // 3 ครั้งต่อ 5 นาที
  async forgotPassword() {}

  @Get('me')
  @SkipThrottle() // ไม่ต้อง throttle
  getProfile() {}
}
```

---

## การทดสอบ Auth (Testing Auth)

```typescript
// src/auth/auth.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { UnauthorizedException, ConflictException } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';

const mockUsersService = {
  findByEmail: jest.fn(),
  findByEmailWithPassword: jest.fn(),
  findById: jest.fn(),
  create: jest.fn(),
  updateRefreshToken: jest.fn(),
  updateLastLogin: jest.fn(),
};

const mockJwtService = {
  signAsync: jest.fn(),
  verifyAsync: jest.fn(),
};

const mockConfigService = {
  get: jest.fn().mockImplementation((key: string, defaultValue?: any) => {
    const configs: Record<string, any> = {
      JWT_SECRET: 'test-secret',
      JWT_EXPIRES_IN: '15m',
      JWT_REFRESH_SECRET: 'test-refresh-secret',
      JWT_REFRESH_EXPIRES_IN: '7d',
    };
    return configs[key] || defaultValue;
  }),
};

describe('AuthService', () => {
  let authService: AuthService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: UsersService, useValue: mockUsersService },
        { provide: JwtService, useValue: mockJwtService },
        { provide: ConfigService, useValue: mockConfigService },
      ],
    }).compile();

    authService = module.get<AuthService>(AuthService);
    jest.clearAllMocks();
  });

  describe('validateUser', () => {
    it('should return user without password on valid credentials', async () => {
      const hashedPassword = await bcrypt.hash('password123', 10);
      const mockUser = {
        id: 'user-1',
        email: 'test@example.com',
        password: hashedPassword,
        name: 'Test User',
        role: 'user',
      };

      mockUsersService.findByEmailWithPassword.mockResolvedValue(mockUser);

      const result = await authService.validateUser('test@example.com', 'password123');

      expect(result).toBeDefined();
      expect(result.email).toBe('test@example.com');
      expect((result as any).password).toBeUndefined();
    });

    it('should return null for invalid password', async () => {
      const hashedPassword = await bcrypt.hash('correctpassword', 10);
      mockUsersService.findByEmailWithPassword.mockResolvedValue({
        password: hashedPassword,
      });

      const result = await authService.validateUser('test@example.com', 'wrongpassword');
      expect(result).toBeNull();
    });

    it('should return null for non-existent user', async () => {
      mockUsersService.findByEmailWithPassword.mockResolvedValue(null);

      const result = await authService.validateUser('notfound@example.com', 'password');
      expect(result).toBeNull();
    });
  });

  describe('register', () => {
    it('should register new user successfully', async () => {
      mockUsersService.findByEmail.mockResolvedValue(null);
      mockUsersService.create.mockResolvedValue({
        id: 'new-user-id',
        name: 'New User',
        email: 'new@example.com',
        role: 'user',
      });
      mockJwtService.signAsync.mockResolvedValue('mock-token');

      const result = await authService.register({
        name: 'New User',
        email: 'new@example.com',
        password: 'Password123!',
      });

      expect(result.accessToken).toBeDefined();
      expect(result.user.email).toBe('new@example.com');
    });

    it('should throw ConflictException for existing email', async () => {
      mockUsersService.findByEmail.mockResolvedValue({ id: 'existing' });

      await expect(
        authService.register({
          name: 'Test',
          email: 'existing@example.com',
          password: 'Password123!',
        }),
      ).rejects.toThrow(ConflictException);
    });
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **TypeORM Integration**: การเชื่อมต่อและใช้งาน TypeORM
2. **Prisma Integration**: การใช้งาน Prisma ORM
3. **Entity Setup**: การสร้าง entities พร้อม decorators
4. **Repository Pattern**: การจัดการ database queries
5. **Migrations**: การจัดการ database schema changes
6. **JWT Authentication**: การตั้งค่า JWT strategy
7. **Guards**: JwtAuthGuard, RolesGuard, PermissionsGuard
8. **Decorators**: @CurrentUser, @Public, @Roles
9. **Role-Based Authorization**: การควบคุมสิทธิ์ตาม role
10. **Password Hashing**: การเข้ารหัสรหัสผ่าน
11. **Refresh Tokens**: การจัดการ token renewal
12. **Complete Auth Flow**: กระบวนการ authentication แบบสมบูรณ์
13. **Rate Limiting**: การป้องกัน brute force attacks
14. **Testing**: การทดสอบ auth service

นี่คือพื้นฐานสำคัญสำหรับการสร้างแอปพลิเคชัน NestJS ที่มีความปลอดภัยและสามารถ scale ได้
