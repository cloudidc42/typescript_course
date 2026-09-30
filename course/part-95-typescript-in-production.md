# Part 95: TypeScript ใน Production

## บทนำ

การ deploy TypeScript application ขึ้น production ต้องคำนึงถึงหลายปัจจัย ตั้งแต่การตั้งค่า TypeScript ที่เหมาะสม ไปจนถึงการ monitor, logging, และ deployment strategies

## ส่วนที่ 1: Production-ready tsconfig

### 1.1 Strict TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    // Target และ Module
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    
    // Output
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    
    // Strict Type Checking
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    
    // Additional Checks
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    
    // Module Resolution
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    
    // Performance
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",
    "skipLibCheck": true,
    
    // Paths
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@config/*": ["./src/config/*"],
      "@modules/*": ["./src/modules/*"],
      "@common/*": ["./src/common/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

### 1.2 tsconfig สำหรับ Build

```json
// tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
    "**/*.spec.ts",
    "test/**/*",
    "jest.config.ts"
  ],
  "compilerOptions": {
    "removeComments": true,
    "declaration": false,
    "declarationMap": false
  }
}
```

## ส่วนที่ 2: Build Optimization

### 2.1 Build Script

```typescript
// scripts/build.ts
import { execSync } from 'child_process';
import { existsSync, rmSync } from 'fs';
import { join } from 'path';

interface BuildConfig {
  srcDir: string;
  outDir: string;
  tsConfig: string;
}

async function build(config: BuildConfig): Promise<void> {
  const { srcDir, outDir, tsConfig } = config;

  console.log('เริ่มต้น build process...');

  // ลบ dist เก่า
  if (existsSync(outDir)) {
    rmSync(outDir, { recursive: true });
    console.log('ลบ dist เก่าแล้ว');
  }

  // Type check
  console.log('กำลัง type check...');
  execSync(`tsc --noEmit -p ${tsConfig}`, { stdio: 'inherit' });
  console.log('Type check ผ่าน');

  // Compile
  console.log('กำลัง compile...');
  execSync(`tsc -p ${tsConfig}`, { stdio: 'inherit' });
  console.log('Compile เสร็จสมบูรณ์');

  // Copy static files
  execSync(`cp -r ${srcDir}/assets ${outDir}/assets 2>/dev/null || true`);

  console.log('Build เสร็จสมบูรณ์!');
}

build({
  srcDir: './src',
  outDir: './dist',
  tsConfig: './tsconfig.build.json',
}).catch((err) => {
  console.error('Build ล้มเหลว:', err);
  process.exit(1);
});
```

### 2.2 Bundle Size Analysis

```typescript
// scripts/analyze-bundle.ts
import { build } from 'esbuild';
import { analyzeMetafile } from 'esbuild';

async function analyzeBundle(): Promise<void> {
  const result = await build({
    entryPoints: ['src/main.ts'],
    bundle: true,
    platform: 'node',
    target: 'node18',
    outfile: 'dist/bundle.js',
    minify: true,
    sourcemap: false,
    metafile: true,
    external: [
      // ไม่รวม packages ใหญ่ๆ
      'aws-sdk',
      'sharp',
      'bcryptjs',
    ],
  });

  if (result.metafile) {
    const analysis = await analyzeMetafile(result.metafile, {
      verbose: true,
    });
    console.log(analysis);

    // แสดงขนาดแต่ละ chunk
    const outputs = result.metafile.outputs;
    for (const [file, output] of Object.entries(outputs)) {
      const sizeKB = (output.bytes / 1024).toFixed(2);
      console.log(`${file}: ${sizeKB} KB`);
    }
  }
}

analyzeBundle();
```

## ส่วนที่ 3: Error Monitoring กับ Sentry

### 3.1 Sentry Setup

```typescript
// src/config/sentry.ts
import * as Sentry from '@sentry/node';
import { ProfilingIntegration } from '@sentry/profiling-node';
import { ConfigService } from '@nestjs/config';

export function initSentry(configService: ConfigService): void {
  const dsn = configService.get('SENTRY_DSN');

  if (!dsn) {
    console.warn('SENTRY_DSN ไม่ได้ตั้งค่า - ปิด Sentry');
    return;
  }

  Sentry.init({
    dsn,
    environment: configService.get('NODE_ENV', 'development'),
    release: configService.get('APP_VERSION', '0.0.0'),
    
    integrations: [
      new Sentry.Integrations.Http({ tracing: true }),
      new Sentry.Integrations.Express({ app: undefined }),
      new ProfilingIntegration(),
    ],

    tracesSampleRate:
      configService.get('NODE_ENV') === 'production' ? 0.1 : 1.0,
    profilesSampleRate: 0.1,

    beforeSend(event, hint) {
      // ไม่ส่ง events ที่เกี่ยวกับ validation errors
      if (event.exception?.values?.[0]?.type === 'ValidationError') {
        return null;
      }

      // กรอง sensitive data
      if (event.request?.data) {
        event.request.data = sanitizeData(event.request.data);
      }

      return event;
    },

    ignoreErrors: [
      'ResizeObserver loop limit exceeded',
      'Network request failed',
      /ChunkLoadError/,
    ],
  });
}

function sanitizeData(data: unknown): unknown {
  if (typeof data !== 'object' || data === null) return data;

  const sensitiveFields = ['password', 'token', 'secret', 'creditCard', 'ssn'];
  const sanitized = { ...data as Record<string, unknown> };

  for (const field of sensitiveFields) {
    if (field in sanitized) {
      sanitized[field] = '[REDACTED]';
    }
  }

  return sanitized;
}
```

### 3.2 Error Handling ใน NestJS

```typescript
// src/common/filters/global-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';
import * as Sentry from '@sentry/node';

interface ErrorResponse {
  statusCode: number;
  message: string;
  error: string;
  timestamp: string;
  path: string;
  requestId?: string;
}

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();
      message =
        typeof exceptionResponse === 'string'
          ? exceptionResponse
          : (exceptionResponse as any).message || message;
    }

    const errorResponse: ErrorResponse = {
      statusCode: status,
      message: Array.isArray(message) ? message.join(', ') : message,
      error: exception instanceof Error ? exception.constructor.name : 'Error',
      timestamp: new Date().toISOString(),
      path: request.url,
      requestId: request.headers['x-request-id'] as string,
    };

    // Log error
    if (status >= 500) {
      this.logger.error(
        `${request.method} ${request.url} - ${status}`,
        exception instanceof Error ? exception.stack : String(exception),
      );

      // ส่งไป Sentry
      Sentry.captureException(exception, {
        tags: {
          endpoint: request.url,
          method: request.method,
          statusCode: status.toString(),
        },
        extra: {
          requestBody: request.body,
          requestParams: request.params,
          requestQuery: request.query,
        },
        user: {
          id: (request as any).user?.id,
          email: (request as any).user?.email,
        },
      });
    }

    response.status(status).json(errorResponse);
  }
}
```

## ส่วนที่ 4: Logging Strategy

### 4.1 Winston Logger

```typescript
// src/config/logger.ts
import { createLogger, format, transports, Logger } from 'winston';
import 'winston-daily-rotate-file';

const { combine, timestamp, printf, colorize, errors, json } = format;

// Format สำหรับ development
const devFormat = combine(
  colorize(),
  timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  errors({ stack: true }),
  printf(({ level, message, timestamp, context, ...meta }) => {
    const ctx = context ? `[${context}]` : '';
    const metaStr = Object.keys(meta).length
      ? `\n${JSON.stringify(meta, null, 2)}`
      : '';
    return `${timestamp} ${level} ${ctx} ${message}${metaStr}`;
  }),
);

// Format สำหรับ production (JSON)
const prodFormat = combine(
  timestamp(),
  errors({ stack: true }),
  json(),
);

export function createAppLogger(nodeEnv: string): Logger {
  const isDev = nodeEnv !== 'production';

  const logger = createLogger({
    level: isDev ? 'debug' : 'info',
    format: isDev ? devFormat : prodFormat,
    transports: [
      new transports.Console({
        silent: nodeEnv === 'test',
      }),
    ],
    exitOnError: false,
  });

  // ใน production - บันทึกลงไฟล์ด้วย rotation
  if (!isDev) {
    logger.add(
      new transports.DailyRotateFile({
        filename: 'logs/error-%DATE%.log',
        datePattern: 'YYYY-MM-DD',
        level: 'error',
        maxSize: '20m',
        maxFiles: '14d',
        zippedArchive: true,
      }) as any,
    );

    logger.add(
      new transports.DailyRotateFile({
        filename: 'logs/combined-%DATE%.log',
        datePattern: 'YYYY-MM-DD',
        maxSize: '20m',
        maxFiles: '30d',
        zippedArchive: true,
      }) as any,
    );
  }

  return logger;
}
```

### 4.2 Request Logging Middleware

```typescript
// src/common/middleware/request-logger.middleware.ts
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestLoggerMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction): void {
    const requestId = (req.headers['x-request-id'] as string) || uuidv4();
    const startTime = Date.now();

    // เพิ่ม request ID
    req.headers['x-request-id'] = requestId;
    res.setHeader('X-Request-ID', requestId);

    // Log request
    this.logger.log({
      type: 'REQUEST',
      requestId,
      method: req.method,
      url: req.originalUrl,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      userId: (req as any).user?.id,
    });

    // Log response
    res.on('finish', () => {
      const duration = Date.now() - startTime;
      const logLevel = res.statusCode >= 500 ? 'error' : 
                       res.statusCode >= 400 ? 'warn' : 'log';

      this.logger[logLevel]({
        type: 'RESPONSE',
        requestId,
        method: req.method,
        url: req.originalUrl,
        statusCode: res.statusCode,
        duration: `${duration}ms`,
        contentLength: res.getHeader('content-length'),
      });
    });

    next();
  }
}
```

## ส่วนที่ 5: Environment Management

### 5.1 Environment Validation

```typescript
// src/config/env.validation.ts
import { plainToClass, Transform } from 'class-transformer';
import {
  IsString,
  IsNumber,
  IsEnum,
  IsOptional,
  IsBoolean,
  validateSync,
  IsUrl,
  Min,
  Max,
} from 'class-validator';

enum Environment {
  Development = 'development',
  Production = 'production',
  Test = 'test',
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment = Environment.Development;

  @IsNumber()
  @Min(1)
  @Max(65535)
  @Transform(({ value }) => parseInt(value))
  PORT: number = 3000;

  @IsString()
  DB_HOST: string;

  @IsNumber()
  @Transform(({ value }) => parseInt(value))
  DB_PORT: number = 5432;

  @IsString()
  DB_USERNAME: string;

  @IsString()
  DB_PASSWORD: string;

  @IsString()
  DB_NAME: string;

  @IsString()
  JWT_SECRET: string;

  @IsString()
  JWT_REFRESH_SECRET: string;

  @IsString()
  @IsOptional()
  SENTRY_DSN?: string;

  @IsString()
  @IsOptional()
  AWS_REGION?: string;

  @IsString()
  @IsOptional()
  AWS_ACCESS_KEY_ID?: string;

  @IsString()
  @IsOptional()
  AWS_SECRET_ACCESS_KEY?: string;

  @IsString()
  @IsOptional()
  AWS_S3_BUCKET?: string;

  @IsNumber()
  @Transform(({ value }) => parseInt(value))
  @IsOptional()
  RATE_LIMIT_MAX: number = 100;

  @IsNumber()
  @Transform(({ value }) => parseInt(value))
  @IsOptional()
  RATE_LIMIT_WINDOW_MS: number = 60000;
}

export function validate(
  config: Record<string, unknown>,
): EnvironmentVariables {
  const validatedConfig = plainToClass(EnvironmentVariables, config, {
    enableImplicitConversion: true,
  });

  const errors = validateSync(validatedConfig, {
    skipMissingProperties: false,
  });

  if (errors.length > 0) {
    const messages = errors
      .map((err) => Object.values(err.constraints || {}).join(', '))
      .join('\n');
    throw new Error(`Environment validation failed:\n${messages}`);
  }

  return validatedConfig;
}
```

### 5.2 Config Service Pattern

```typescript
// src/config/app.config.ts
import { registerAs } from '@nestjs/config';

export const appConfig = registerAs('app', () => ({
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '3000'),
  name: process.env.APP_NAME || 'Blog API',
  version: process.env.APP_VERSION || '1.0.0',
  url: process.env.APP_URL || 'http://localhost:3000',
}));

export const dbConfig = registerAs('db', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  name: process.env.DB_NAME,
  ssl: process.env.NODE_ENV === 'production',
  poolSize: parseInt(process.env.DB_POOL_SIZE || '10'),
}));

export const jwtConfig = registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET,
  refreshSecret: process.env.JWT_REFRESH_SECRET,
  expiresIn: process.env.JWT_EXPIRES_IN || '15m',
  refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN || '7d',
}));

export const s3Config = registerAs('s3', () => ({
  region: process.env.AWS_REGION || 'ap-southeast-1',
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  bucket: process.env.AWS_S3_BUCKET,
  cdnUrl: process.env.CDN_URL,
}));
```

## ส่วนที่ 6: Health Checks

### 6.1 Health Check Controller

```typescript
// src/modules/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HttpHealthIndicator,
  TypeOrmHealthIndicator,
  MemoryHealthIndicator,
  DiskHealthIndicator,
  HealthCheck,
} from '@nestjs/terminus';
import { InjectConnection } from '@nestjs/typeorm';
import { Connection } from 'typeorm';

@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly http: HttpHealthIndicator,
    private readonly db: TypeOrmHealthIndicator,
    private readonly memory: MemoryHealthIndicator,
    private readonly disk: DiskHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      // Database
      () => this.db.pingCheck('database', { timeout: 3000 }),

      // Memory - ไม่เกิน 300MB
      () => this.memory.checkHeap('memory_heap', 300 * 1024 * 1024),
      () => this.memory.checkRSS('memory_rss', 300 * 1024 * 1024),

      // Disk - ไม่เกิน 90%
      () =>
        this.disk.checkStorage('storage', {
          path: '/',
          thresholdPercent: 0.9,
        }),
    ]);
  }

  @Get('ready')
  @HealthCheck()
  readiness() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }

  @Get('live')
  liveness() {
    return {
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      version: process.env.APP_VERSION || '1.0.0',
    };
  }
}
```

### 6.2 Custom Health Indicator

```typescript
// src/modules/health/redis.health.ts
import { Injectable } from '@nestjs/common';
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
} from '@nestjs/terminus';
import { Redis } from 'ioredis';

@Injectable()
export class RedisHealthIndicator extends HealthIndicator {
  constructor(private readonly redis: Redis) {
    super();
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      const start = Date.now();
      await this.redis.ping();
      const latency = Date.now() - start;

      return this.getStatus(key, true, {
        latency: `${latency}ms`,
        status: 'connected',
      });
    } catch (error) {
      throw new HealthCheckError(
        `Redis health check failed`,
        this.getStatus(key, false, {
          message: (error as Error).message,
        }),
      );
    }
  }
}
```

## ส่วนที่ 7: Graceful Shutdown

### 7.1 Shutdown Hooks

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { Logger } from '@nestjs/common';
import { AppModule } from './app.module';

const logger = new Logger('Bootstrap');

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log'],
    bufferLogs: true,
  });

  // Enable graceful shutdown
  app.enableShutdownHooks();

  // Signal handlers
  const signals = ['SIGTERM', 'SIGINT', 'SIGUSR2'] as const;

  for (const signal of signals) {
    process.on(signal, async () => {
      logger.log(`ได้รับ signal: ${signal} - กำลัง shutdown...`);

      try {
        await app.close();
        logger.log('Application ปิดเรียบร้อย');
        process.exit(0);
      } catch (err) {
        logger.error('เกิดข้อผิดพลาดขณะ shutdown:', err);
        process.exit(1);
      }
    });
  }

  const port = process.env.PORT || 3000;
  await app.listen(port);
  logger.log(`Application กำลังทำงานที่ port ${port}`);
}

bootstrap().catch((err) => {
  logger.error('ไม่สามารถเริ่มต้น application ได้:', err);
  process.exit(1);
});
```

### 7.2 Graceful Shutdown ด้วย Connection Draining

```typescript
// src/app.service.ts
import { Injectable, OnApplicationShutdown, Logger } from '@nestjs/common';
import { InjectConnection } from '@nestjs/typeorm';
import { Connection } from 'typeorm';

@Injectable()
export class AppService implements OnApplicationShutdown {
  private readonly logger = new Logger(AppService.name);
  private isShuttingDown = false;

  constructor(
    @InjectConnection()
    private readonly connection: Connection,
  ) {}

  isHealthy(): boolean {
    return !this.isShuttingDown;
  }

  async onApplicationShutdown(signal?: string): Promise<void> {
    this.isShuttingDown = true;
    this.logger.log(`กำลัง shutdown ด้วย signal: ${signal}`);

    // รอให้ requests ที่กำลังดำเนินการเสร็จสิ้น
    await new Promise((resolve) => setTimeout(resolve, 5000));

    // ปิด database connections
    if (this.connection.isInitialized) {
      await this.connection.destroy();
      this.logger.log('Database connection ปิดแล้ว');
    }
  }
}
```

## ส่วนที่ 8: Database Migration Strategy

### 8.1 Migration Workflow

```typescript
// src/database/migrations/helpers.ts
import { QueryRunner, TableColumn } from 'typeorm';

export async function addColumnIfNotExists(
  queryRunner: QueryRunner,
  table: string,
  column: TableColumn,
): Promise<void> {
  const tableObj = await queryRunner.getTable(table);
  const existingColumn = tableObj?.findColumnByName(column.name);

  if (!existingColumn) {
    await queryRunner.addColumn(table, column);
    console.log(`เพิ่มคอลัมน์ ${column.name} ในตาราง ${table}`);
  }
}

export async function dropColumnIfExists(
  queryRunner: QueryRunner,
  table: string,
  columnName: string,
): Promise<void> {
  const tableObj = await queryRunner.getTable(table);
  const column = tableObj?.findColumnByName(columnName);

  if (column) {
    await queryRunner.dropColumn(table, columnName);
    console.log(`ลบคอลัมน์ ${columnName} จากตาราง ${table}`);
  }
}
```

### 8.2 Migration Script

```typescript
// src/database/migrations/002-add-seo-fields.ts
import { MigrationInterface, QueryRunner, TableColumn } from 'typeorm';
import { addColumnIfNotExists } from './helpers';

export class AddSeoFields002 implements MigrationInterface {
  name = 'AddSeoFields002';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // เพิ่ม SEO fields ให้ posts table
    await addColumnIfNotExists(
      queryRunner,
      'posts',
      new TableColumn({
        name: 'metaTitle',
        type: 'varchar',
        length: '60',
        isNullable: true,
      }),
    );

    await addColumnIfNotExists(
      queryRunner,
      'posts',
      new TableColumn({
        name: 'metaDescription',
        type: 'text',
        isNullable: true,
      }),
    );

    // Migration data - copy title เป็น metaTitle
    await queryRunner.query(`
      UPDATE posts 
      SET "metaTitle" = LEFT(title, 60)
      WHERE "metaTitle" IS NULL
    `);

    console.log('Migration 002: เพิ่ม SEO fields เรียบร้อย');
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn('posts', 'metaTitle');
    await queryRunner.dropColumn('posts', 'metaDescription');
  }
}
```

### 8.3 Zero-downtime Migration

```typescript
// scripts/migrate.ts
import { DataSource } from 'typeorm';
import { config } from 'dotenv';

config();

const dataSource = new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  migrations: ['src/database/migrations/*.ts'],
  logging: true,
});

async function runMigrations(): Promise<void> {
  console.log('เชื่อมต่อ database...');
  await dataSource.initialize();

  try {
    console.log('กำลัง run migrations...');
    const migrations = await dataSource.runMigrations({
      transaction: 'each',
    });

    if (migrations.length === 0) {
      console.log('ไม่มี migrations ใหม่');
    } else {
      console.log(`Run ${migrations.length} migrations สำเร็จ:`);
      migrations.forEach((m) => console.log(`  - ${m.name}`));
    }
  } finally {
    await dataSource.destroy();
    console.log('ปิด database connection แล้ว');
  }
}

runMigrations().catch((err) => {
  console.error('Migration ล้มเหลว:', err);
  process.exit(1);
});
```

## ส่วนที่ 9: Performance Benchmarking

### 9.1 Artillery Load Test

```yaml
# load-test.yml
config:
  target: "http://localhost:3000"
  phases:
    - name: "Warm up"
      duration: 30
      arrivalRate: 5

    - name: "Ramp up"
      duration: 60
      arrivalRate: 5
      rampTo: 50

    - name: "Sustained load"
      duration: 120
      arrivalRate: 50

    - name: "Peak load"
      duration: 60
      arrivalRate: 100

  defaults:
    headers:
      Content-Type: "application/json"
      Accept: "application/json"

  variables:
    authToken: "{{ $randomString() }}"

  plugins:
    expect: {}
    metrics-by-endpoint: {}

scenarios:
  - name: "Blog API Load Test"
    weight: 70
    flow:
      - get:
          url: "/api/posts"
          qs:
            page: "{{ $randomInt(1, 10) }}"
            limit: "10"
          expect:
            - statusCode: 200
            - hasProperty: "data"
            - hasProperty: "meta"

      - get:
          url: "/api/posts/{{ $randomString(10) }}"
          expect:
            - statusCode:
                - 200
                - 404

  - name: "Search Load Test"
    weight: 30
    flow:
      - get:
          url: "/api/search"
          qs:
            query: "typescript"
          expect:
            - statusCode: 200
```

### 9.2 Custom Benchmark Tool

```typescript
// scripts/benchmark.ts
import autocannon from 'autocannon';

interface BenchmarkConfig {
  url: string;
  duration: number;
  connections: number;
  title: string;
}

async function runBenchmark(config: BenchmarkConfig): Promise<void> {
  console.log(`\nกำลัง benchmark: ${config.title}`);
  console.log(`URL: ${config.url}`);
  console.log(`Duration: ${config.duration}s, Connections: ${config.connections}`);

  const result = await autocannon({
    url: config.url,
    duration: config.duration,
    connections: config.connections,
    headers: {
      'Content-Type': 'application/json',
    },
  });

  const { latency, requests, throughput, errors } = result;

  console.log('\nผลลัพธ์:');
  console.log(`  Requests/sec: ${Math.round(requests.average)}`);
  console.log(`  Latency avg: ${latency.average}ms`);
  console.log(`  Latency p50: ${latency.p50}ms`);
  console.log(`  Latency p95: ${latency.p95}ms`);
  console.log(`  Latency p99: ${latency.p99}ms`);
  console.log(`  Throughput: ${(throughput.average / 1024 / 1024).toFixed(2)} MB/s`);
  console.log(`  Errors: ${errors}`);
}

// รัน benchmarks
await runBenchmark({
  title: 'GET /api/posts (List)',
  url: 'http://localhost:3000/api/posts',
  duration: 30,
  connections: 50,
});

await runBenchmark({
  title: 'GET /api/search',
  url: 'http://localhost:3000/api/search?query=typescript',
  duration: 30,
  connections: 25,
});
```

## ส่วนที่ 10: Production Checklist

### 10.1 Security Checklist

```typescript
// src/app.module.ts - Security setup
import { Module } from '@nestjs/common';
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';
import helmet from 'helmet';

@Module({
  imports: [
    // Rate limiting
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000, // 1 วินาที
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000, // 10 วินาที
        limit: 20,
      },
      {
        name: 'long',
        ttl: 60000, // 1 นาที
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

// main.ts - Security middleware
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Helmet - HTTP security headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        scriptSrc: ["'self'"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
  }));

  // CORS
  app.enableCors({
    origin: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    credentials: true,
    maxAge: 86400,
  });

  // Compression
  app.use(compression());

  await app.listen(3000);
}
```

### 10.2 Deployment Checklist

```markdown
## Production Deployment Checklist

### TypeScript & Build
- [ ] `strict: true` ใน tsconfig.json
- [ ] ไม่มี `any` types
- [ ] `noUnusedLocals: true`
- [ ] Build ผ่านโดยไม่มี errors
- [ ] Type check ผ่าน

### Security
- [ ] Environment variables ไม่ expose secrets
- [ ] Helmet middleware ตั้งค่าแล้ว
- [ ] CORS ตั้งค่าเฉพาะ domains ที่อนุญาต
- [ ] Rate limiting ตั้งค่าแล้ว
- [ ] SQL Injection protection (parameterized queries)
- [ ] XSS protection (sanitize user input)
- [ ] JWT secret ยาวและซับซ้อน
- [ ] HTTPS เท่านั้น

### Database
- [ ] Migrations run แล้ว
- [ ] Database indexes สร้างแล้ว
- [ ] Connection pool size เหมาะสม
- [ ] Backup strategy ตั้งค่าแล้ว
- [ ] Sensitive data encrypted

### Monitoring
- [ ] Sentry ตั้งค่าแล้ว
- [ ] Health check endpoints พร้อม
- [ ] Logging strategy ตั้งค่าแล้ว
- [ ] Metrics collection ตั้งค่าแล้ว
- [ ] Alerts ตั้งค่าแล้ว

### Performance
- [ ] Caching strategy ตั้งค่าแล้ว
- [ ] Compression เปิดใช้งาน
- [ ] Database queries optimized
- [ ] N+1 queries ไม่มี
- [ ] Load testing ผ่าน

### Reliability
- [ ] Graceful shutdown ทำงานได้
- [ ] Rollback plan พร้อม
- [ ] Zero-downtime deployment ทดสอบแล้ว
- [ ] Circuit breaker ตั้งค่าแล้ว (ถ้าจำเป็น)
```

### 10.3 Rollback Strategy

```typescript
// scripts/rollback.ts
import { execSync } from 'child_process';
import { DataSource } from 'typeorm';

interface RollbackOptions {
  steps?: number;
  migrationName?: string;
}

async function rollback(options: RollbackOptions = {}): Promise<void> {
  const dataSource = new DataSource({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    migrations: ['src/database/migrations/*.ts'],
  });

  await dataSource.initialize();

  try {
    if (options.migrationName) {
      // Rollback ไปยัง migration ที่ระบุ
      console.log(`Rollback ไปยัง migration: ${options.migrationName}`);
      await dataSource.undoLastMigration();
    } else {
      const steps = options.steps || 1;
      console.log(`Rollback ${steps} migrations`);

      for (let i = 0; i < steps; i++) {
        await dataSource.undoLastMigration();
        console.log(`Rollback step ${i + 1} สำเร็จ`);
      }
    }

    console.log('Rollback database เสร็จสมบูรณ์');
  } finally {
    await dataSource.destroy();
  }
}

// Rollback deployment
async function rollbackDeployment(version: string): Promise<void> {
  console.log(`กำลัง rollback ไปยัง version: ${version}`);

  // 1. Rollback database
  await rollback({ steps: 1 });

  // 2. Rollback application (Docker)
  execSync(`docker tag my-app:${version} my-app:latest`);
  execSync('docker service update --image my-app:latest my-app-service');

  console.log(`Rollback ไปยัง version ${version} เสร็จสมบูรณ์`);
}

const version = process.argv[2];
if (!version) {
  console.error('ระบุ version: npm run rollback <version>');
  process.exit(1);
}

rollbackDeployment(version).catch((err) => {
  console.error('Rollback ล้มเหลว:', err);
  process.exit(1);
});
```

## ส่วนที่ 11: CI/CD Pipeline

### 11.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          JWT_SECRET: test-secret
          JWT_REFRESH_SECRET: test-refresh-secret

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /app
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            docker-compose -f docker-compose.prod.yml up -d --no-deps --build api
            docker-compose -f docker-compose.prod.yml exec api npm run migration:run
```

## สรุป

การ deploy TypeScript application ใน Production ต้องคำนึงถึง:

1. **TypeScript Config** - ใช้ strict mode และตั้งค่าที่เหมาะสม
2. **Build Optimization** - incremental compilation, tree shaking
3. **Error Monitoring** - Sentry สำหรับ track errors
4. **Logging** - structured logging ด้วย Winston
5. **Environment** - validate env vars ด้วย class-validator
6. **Health Checks** - /health, /ready, /live endpoints
7. **Graceful Shutdown** - drain connections ก่อน terminate
8. **Database Migrations** - zero-downtime migration strategy
9. **Performance** - benchmark ก่อน deploy
10. **Security** - helmet, CORS, rate limiting, HTTPS
11. **CI/CD** - automated testing และ deployment
12. **Rollback** - plan สำหรับ rollback เสมอ

การทำ production-ready TypeScript application ต้องใส่ใจในรายละเอียดทุกส่วน ตั้งแต่ code quality ไปจนถึง infrastructure และ operational concerns
