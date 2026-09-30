# ตอนที่ 21: Express.js REST API กับ TypeScript

## บทนำ

Express.js เป็น web framework ที่ได้รับความนิยมมากที่สุดสำหรับ Node.js การใช้งาน Express ร่วมกับ TypeScript จะช่วยให้โค้ดมีความปลอดภัยด้านประเภทข้อมูลและมีโครงสร้างที่ดีขึ้น ในบทนี้เราจะสร้าง REST API ที่สมบูรณ์พร้อม authentication, validation, และ error handling

---

## 21.1 การติดตั้งและตั้งค่า Express กับ TypeScript

### การติดตั้ง packages

```bash
# สร้างโปรเจกต์ใหม่
mkdir express-ts-api
cd express-ts-api
npm init -y

# ติดตั้ง Express และ dependencies
npm install express cors helmet morgan dotenv
npm install --save-dev typescript @types/express @types/cors @types/morgan @types/node ts-node-dev

# ติดตั้ง packages เพิ่มเติม
npm install bcryptjs jsonwebtoken uuid
npm install --save-dev @types/bcryptjs @types/jsonwebtoken @types/uuid
```

### package.json สมบูรณ์

```json
{
  "name": "express-ts-api",
  "version": "1.0.0",
  "description": "Express.js REST API with TypeScript",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "dev:debug": "ts-node-dev --respawn --transpile-only --inspect src/index.ts",
    "clean": "rm -rf dist",
    "prebuild": "npm run clean",
    "lint": "eslint src --ext .ts --fix",
    "typecheck": "tsc --noEmit",
    "test": "jest --coverage"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "express": "^4.18.2",
    "helmet": "^7.0.0",
    "jsonwebtoken": "^9.0.0",
    "morgan": "^1.10.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/bcryptjs": "^2.4.3",
    "@types/cors": "^2.8.13",
    "@types/express": "^4.17.17",
    "@types/jsonwebtoken": "^9.0.2",
    "@types/morgan": "^1.9.5",
    "@types/node": "^20.4.5",
    "@types/uuid": "^9.0.2",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.1.6"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "baseUrl": "./src",
    "paths": {
      "@controllers/*": ["controllers/*"],
      "@services/*": ["services/*"],
      "@models/*": ["models/*"],
      "@middleware/*": ["middleware/*"],
      "@routes/*": ["routes/*"],
      "@utils/*": ["utils/*"],
      "@config/*": ["config/*"],
      "@types/*": ["types/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 21.2 การกำหนด Types สำหรับ Express

### src/types/express.d.ts - ขยาย Express Request type

```typescript
import { User } from './index';

// ขยาย Express namespace เพื่อเพิ่ม custom properties
declare global {
  namespace Express {
    interface Request {
      user?: AuthUser;
      requestId?: string;
      startTime?: number;
    }
  }
}

export interface AuthUser {
  id: string;
  email: string;
  role: UserRole;
}

export type UserRole = 'admin' | 'moderator' | 'user';

export {};
```

### src/types/index.ts

```typescript
// Base types
export interface BaseEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

// User types
export interface User extends BaseEntity {
  name: string;
  email: string;
  password: string;
  role: UserRole;
  isActive: boolean;
  lastLoginAt?: Date;
}

export type UserRole = 'admin' | 'moderator' | 'user';

export type PublicUser = Omit<User, 'password'>;

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
  role?: UserRole;
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
  role?: UserRole;
  isActive?: boolean;
}

export interface LoginDto {
  email: string;
  password: string;
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

// API Response types
export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  message?: string;
  errors?: ValidationError[];
}

export interface PaginatedResponse<T> {
  success: boolean;
  data: T[];
  meta: PaginationMeta;
}

export interface PaginationMeta {
  total: number;
  page: number;
  limit: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}

export interface ValidationError {
  field: string;
  message: string;
}

export interface PaginationQuery {
  page?: string;
  limit?: string;
  sort?: string;
  order?: 'asc' | 'desc';
  search?: string;
}
```

---

## 21.3 Middleware Typing

### src/middleware/requestLogger.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

interface RequestLogData {
  requestId: string;
  method: string;
  url: string;
  ip: string;
  userAgent: string;
  userId?: string;
  duration?: number;
  statusCode?: number;
}

export function requestLogger() {
  return (req: Request, res: Response, next: NextFunction): void => {
    // เพิ่ม requestId และ startTime ให้กับ request
    req.requestId = uuidv4();
    req.startTime = Date.now();
    
    const logData: RequestLogData = {
      requestId: req.requestId,
      method: req.method,
      url: req.originalUrl,
      ip: req.ip || req.socket.remoteAddress || 'unknown',
      userAgent: req.get('user-agent') || 'unknown',
    };
    
    console.log('[REQUEST]', JSON.stringify(logData));
    
    // บันทึก response เมื่อ request เสร็จสิ้น
    res.on('finish', () => {
      const duration = Date.now() - (req.startTime || 0);
      console.log('[RESPONSE]', JSON.stringify({
        ...logData,
        userId: req.user?.id,
        duration,
        statusCode: res.statusCode,
      }));
    });
    
    next();
  };
}
```

### src/middleware/authenticate.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { AuthUser } from '../types/express';

interface JwtPayload {
  userId: string;
  email: string;
  role: string;
  iat?: number;
  exp?: number;
}

export class AuthMiddleware {
  private readonly jwtSecret: string;

  constructor(jwtSecret: string) {
    this.jwtSecret = jwtSecret;
  }

  // Middleware สำหรับตรวจสอบ token (required)
  authenticate = (req: Request, res: Response, next: NextFunction): void => {
    const token = this.extractToken(req);
    
    if (!token) {
      res.status(401).json({
        success: false,
        message: 'Authentication token is required',
      });
      return;
    }
    
    try {
      const decoded = jwt.verify(token, this.jwtSecret) as JwtPayload;
      
      req.user = {
        id: decoded.userId,
        email: decoded.email,
        role: decoded.role as AuthUser['role'],
      };
      
      next();
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        res.status(401).json({
          success: false,
          message: 'Token has expired',
        });
        return;
      }
      
      res.status(401).json({
        success: false,
        message: 'Invalid authentication token',
      });
    }
  };

  // Middleware สำหรับตรวจสอบ token (optional)
  optionalAuthenticate = (req: Request, res: Response, next: NextFunction): void => {
    const token = this.extractToken(req);
    
    if (token) {
      try {
        const decoded = jwt.verify(token, this.jwtSecret) as JwtPayload;
        req.user = {
          id: decoded.userId,
          email: decoded.email,
          role: decoded.role as AuthUser['role'],
        };
      } catch {
        // ไม่ทำอะไรถ้า token ไม่ valid (optional auth)
      }
    }
    
    next();
  };

  // ตรวจสอบสิทธิ์ตาม roles
  authorize = (...roles: AuthUser['role'][]) => {
    return (req: Request, res: Response, next: NextFunction): void => {
      if (!req.user) {
        res.status(401).json({
          success: false,
          message: 'Authentication required',
        });
        return;
      }
      
      if (!roles.includes(req.user.role)) {
        res.status(403).json({
          success: false,
          message: 'Insufficient permissions',
        });
        return;
      }
      
      next();
    };
  };

  private extractToken(req: Request): string | null {
    const authHeader = req.headers.authorization;
    
    if (authHeader?.startsWith('Bearer ')) {
      return authHeader.substring(7);
    }
    
    // ตรวจสอบใน cookie ด้วย
    if (req.cookies?.accessToken) {
      return req.cookies.accessToken;
    }
    
    return null;
  }
}

// Factory function
export function createAuthMiddleware(jwtSecret: string): AuthMiddleware {
  return new AuthMiddleware(jwtSecret);
}
```

### src/middleware/errorHandler.ts

```typescript
import { Request, Response, NextFunction } from 'express';

export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly errors?: Record<string, string[]>;

  constructor(
    message: string,
    statusCode: number = 500,
    isOperational: boolean = true,
    errors?: Record<string, string[]>
  ) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    this.errors = errors;
    
    // แก้ไข prototype chain สำหรับ instanceof check
    Object.setPrototypeOf(this, AppError.prototype);
    Error.captureStackTrace(this, this.constructor);
  }

  static badRequest(message: string, errors?: Record<string, string[]>): AppError {
    return new AppError(message, 400, true, errors);
  }

  static unauthorized(message: string = 'Unauthorized'): AppError {
    return new AppError(message, 401);
  }

  static forbidden(message: string = 'Forbidden'): AppError {
    return new AppError(message, 403);
  }

  static notFound(message: string = 'Resource not found'): AppError {
    return new AppError(message, 404);
  }

  static conflict(message: string): AppError {
    return new AppError(message, 409);
  }

  static internal(message: string = 'Internal server error'): AppError {
    return new AppError(message, 500, false);
  }
}

interface ErrorResponse {
  success: false;
  message: string;
  errors?: Record<string, string[]>;
  stack?: string;
}

export function errorHandler(
  error: Error | AppError,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  let statusCode = 500;
  let message = 'Internal server error';
  let errors: Record<string, string[]> | undefined;
  
  if (error instanceof AppError) {
    statusCode = error.statusCode;
    message = error.message;
    errors = error.errors;
  }
  
  const isDevelopment = process.env.NODE_ENV === 'development';
  
  const response: ErrorResponse = {
    success: false,
    message,
    ...(errors && { errors }),
    ...(isDevelopment && { stack: error.stack }),
  };
  
  // Log error
  if (statusCode >= 500) {
    console.error('[ERROR]', {
      requestId: req.requestId,
      method: req.method,
      url: req.originalUrl,
      statusCode,
      message,
      stack: error.stack,
    });
  }
  
  res.status(statusCode).json(response);
}

export function notFoundHandler(req: Request, res: Response): void {
  res.status(404).json({
    success: false,
    message: `Route ${req.method} ${req.originalUrl} not found`,
  });
}
```

### src/middleware/validate.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { ValidationError } from '../types';

type ValidationRule = {
  required?: boolean;
  type?: 'string' | 'number' | 'boolean' | 'email' | 'url';
  minLength?: number;
  maxLength?: number;
  min?: number;
  max?: number;
  pattern?: RegExp;
  custom?: (value: unknown) => boolean | string;
};

type ValidationSchema = {
  [field: string]: ValidationRule;
};

function validateField(
  value: unknown,
  field: string,
  rules: ValidationRule
): string | null {
  // Required check
  if (rules.required && (value === undefined || value === null || value === '')) {
    return `${field} is required`;
  }
  
  if (value === undefined || value === null || value === '') {
    return null; // ถ้าไม่ required และไม่มีค่า ข้ามการตรวจสอบ
  }
  
  // Type checks
  if (rules.type === 'string' && typeof value !== 'string') {
    return `${field} must be a string`;
  }
  
  if (rules.type === 'number' && typeof value !== 'number') {
    return `${field} must be a number`;
  }
  
  if (rules.type === 'email' && typeof value === 'string') {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value)) {
      return `${field} must be a valid email address`;
    }
  }
  
  // String length checks
  if (rules.minLength && typeof value === 'string' && value.length < rules.minLength) {
    return `${field} must be at least ${rules.minLength} characters`;
  }
  
  if (rules.maxLength && typeof value === 'string' && value.length > rules.maxLength) {
    return `${field} must not exceed ${rules.maxLength} characters`;
  }
  
  // Number range checks
  if (rules.min !== undefined && typeof value === 'number' && value < rules.min) {
    return `${field} must be at least ${rules.min}`;
  }
  
  if (rules.max !== undefined && typeof value === 'number' && value > rules.max) {
    return `${field} must not exceed ${rules.max}`;
  }
  
  // Pattern check
  if (rules.pattern && typeof value === 'string' && !rules.pattern.test(value)) {
    return `${field} format is invalid`;
  }
  
  // Custom validation
  if (rules.custom) {
    const result = rules.custom(value);
    if (typeof result === 'string') {
      return result;
    }
    if (result === false) {
      return `${field} is invalid`;
    }
  }
  
  return null;
}

export function validateBody(schema: ValidationSchema) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const errors: ValidationError[] = [];
    
    for (const [field, rules] of Object.entries(schema)) {
      const value = req.body[field];
      const error = validateField(value, field, rules);
      
      if (error) {
        errors.push({ field, message: error });
      }
    }
    
    if (errors.length > 0) {
      res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors,
      });
      return;
    }
    
    next();
  };
}

export function validateQuery(schema: ValidationSchema) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const errors: ValidationError[] = [];
    
    for (const [field, rules] of Object.entries(schema)) {
      const value = req.query[field];
      const error = validateField(value, field, rules);
      
      if (error) {
        errors.push({ field, message: error });
      }
    }
    
    if (errors.length > 0) {
      res.status(400).json({
        success: false,
        message: 'Query validation failed',
        errors,
      });
      return;
    }
    
    next();
  };
}
```

---

## 21.4 Controller Pattern

### src/controllers/baseController.ts

```typescript
import { Request, Response } from 'express';
import { ApiResponse, PaginatedResponse, PaginationMeta } from '../types';

export abstract class BaseController {
  protected sendSuccess<T>(
    res: Response,
    data: T,
    statusCode: number = 200,
    message?: string
  ): void {
    const response: ApiResponse<T> = {
      success: true,
      data,
      ...(message && { message }),
    };
    res.status(statusCode).json(response);
  }

  protected sendCreated<T>(res: Response, data: T, message?: string): void {
    this.sendSuccess(res, data, 201, message);
  }

  protected sendNoContent(res: Response): void {
    res.status(204).send();
  }

  protected sendPaginated<T>(
    res: Response,
    data: T[],
    total: number,
    page: number,
    limit: number
  ): void {
    const totalPages = Math.ceil(total / limit);
    
    const meta: PaginationMeta = {
      total,
      page,
      limit,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1,
    };
    
    const response: PaginatedResponse<T> = {
      success: true,
      data,
      meta,
    };
    
    res.status(200).json(response);
  }

  protected parsePagination(query: { page?: string; limit?: string }): {
    page: number;
    limit: number;
    skip: number;
  } {
    const page = Math.max(1, parseInt(query.page || '1', 10));
    const limit = Math.min(100, Math.max(1, parseInt(query.limit || '10', 10)));
    const skip = (page - 1) * limit;
    
    return { page, limit, skip };
  }
}
```

### src/controllers/userController.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { BaseController } from './baseController';
import { UserService } from '../services/userService';
import { CreateUserDto, UpdateUserDto, PaginationQuery } from '../types';
import { AppError } from '../middleware/errorHandler';

export class UserController extends BaseController {
  constructor(private readonly userService: UserService) {
    super();
  }

  // GET /users
  getAll = async (req: Request<{}, {}, {}, PaginationQuery>, res: Response, next: NextFunction): Promise<void> => {
    try {
      const { page, limit, skip } = this.parsePagination(req.query);
      const search = req.query.search;
      
      const { users, total } = await this.userService.findAll({
        skip,
        limit,
        search,
      });
      
      this.sendPaginated(res, users, total, page, limit);
    } catch (error) {
      next(error);
    }
  };

  // GET /users/:id
  getById = async (req: Request<{ id: string }>, res: Response, next: NextFunction): Promise<void> => {
    try {
      const user = await this.userService.findById(req.params.id);
      
      if (!user) {
        throw AppError.notFound('User not found');
      }
      
      this.sendSuccess(res, user);
    } catch (error) {
      next(error);
    }
  };

  // POST /users
  create = async (req: Request<{}, {}, CreateUserDto>, res: Response, next: NextFunction): Promise<void> => {
    try {
      const user = await this.userService.create(req.body);
      this.sendCreated(res, user, 'User created successfully');
    } catch (error) {
      next(error);
    }
  };

  // PUT /users/:id
  update = async (
    req: Request<{ id: string }, {}, UpdateUserDto>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const user = await this.userService.update(req.params.id, req.body);
      
      if (!user) {
        throw AppError.notFound('User not found');
      }
      
      this.sendSuccess(res, user, 'User updated successfully');
    } catch (error) {
      next(error);
    }
  };

  // DELETE /users/:id
  delete = async (req: Request<{ id: string }>, res: Response, next: NextFunction): Promise<void> => {
    try {
      // ป้องกัน user ลบตัวเอง
      if (req.user?.id === req.params.id) {
        throw AppError.badRequest('Cannot delete your own account');
      }
      
      const deleted = await this.userService.delete(req.params.id);
      
      if (!deleted) {
        throw AppError.notFound('User not found');
      }
      
      this.sendNoContent(res);
    } catch (error) {
      next(error);
    }
  };

  // GET /users/me
  getProfile = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      if (!req.user) {
        throw AppError.unauthorized();
      }
      
      const user = await this.userService.findById(req.user.id);
      
      if (!user) {
        throw AppError.notFound('User not found');
      }
      
      this.sendSuccess(res, user);
    } catch (error) {
      next(error);
    }
  };

  // PATCH /users/me
  updateProfile = async (
    req: Request<{}, {}, UpdateUserDto>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      if (!req.user) {
        throw AppError.unauthorized();
      }
      
      // ป้องกัน user เปลี่ยน role ของตัวเอง
      const { role, ...safeUpdateData } = req.body;
      
      const user = await this.userService.update(req.user.id, safeUpdateData);
      this.sendSuccess(res, user, 'Profile updated successfully');
    } catch (error) {
      next(error);
    }
  };
}
```

---

## 21.5 Service Layer

### src/services/userService.ts

```typescript
import bcrypt from 'bcryptjs';
import { v4 as uuidv4 } from 'uuid';
import { User, PublicUser, CreateUserDto, UpdateUserDto } from '../types';
import { AppError } from '../middleware/errorHandler';

interface FindAllOptions {
  skip: number;
  limit: number;
  search?: string;
}

interface FindAllResult {
  users: PublicUser[];
  total: number;
}

// In-memory store (ในงานจริงจะใช้ database)
const usersStore: User[] = [];

export class UserService {
  private readonly saltRounds = 12;

  async findAll(options: FindAllOptions): Promise<FindAllResult> {
    let filtered = [...usersStore];
    
    // ค้นหาตาม name หรือ email
    if (options.search) {
      const searchLower = options.search.toLowerCase();
      filtered = filtered.filter(
        user =>
          user.name.toLowerCase().includes(searchLower) ||
          user.email.toLowerCase().includes(searchLower)
      );
    }
    
    const total = filtered.length;
    const users = filtered
      .slice(options.skip, options.skip + options.limit)
      .map(this.toPublicUser);
    
    return { users, total };
  }

  async findById(id: string): Promise<PublicUser | null> {
    const user = usersStore.find(u => u.id === id);
    return user ? this.toPublicUser(user) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return usersStore.find(u => u.email === email) || null;
  }

  async create(data: CreateUserDto): Promise<PublicUser> {
    // ตรวจสอบว่า email ซ้ำหรือไม่
    const existingUser = await this.findByEmail(data.email);
    if (existingUser) {
      throw AppError.conflict('Email is already in use');
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(data.password, this.saltRounds);
    
    const newUser: User = {
      id: uuidv4(),
      name: data.name,
      email: data.email.toLowerCase(),
      password: hashedPassword,
      role: data.role || 'user',
      isActive: true,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    
    usersStore.push(newUser);
    return this.toPublicUser(newUser);
  }

  async update(id: string, data: UpdateUserDto): Promise<PublicUser | null> {
    const index = usersStore.findIndex(u => u.id === id);
    if (index === -1) return null;
    
    // ถ้ามีการเปลี่ยน email ต้องตรวจสอบว่าซ้ำหรือไม่
    if (data.email) {
      const existingUser = await this.findByEmail(data.email);
      if (existingUser && existingUser.id !== id) {
        throw AppError.conflict('Email is already in use');
      }
    }
    
    usersStore[index] = {
      ...usersStore[index],
      ...data,
      updatedAt: new Date(),
    };
    
    return this.toPublicUser(usersStore[index]);
  }

  async delete(id: string): Promise<boolean> {
    const index = usersStore.findIndex(u => u.id === id);
    if (index === -1) return false;
    
    usersStore.splice(index, 1);
    return true;
  }

  async validatePassword(user: User, password: string): Promise<boolean> {
    return bcrypt.compare(password, user.password);
  }

  private toPublicUser(user: User): PublicUser {
    const { password, ...publicUser } = user;
    return publicUser;
  }
}
```

---

## 21.6 Authentication Controller

### src/controllers/authController.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { BaseController } from './baseController';
import { UserService } from '../services/userService';
import { CreateUserDto, LoginDto, AuthTokens } from '../types';
import { AppError } from '../middleware/errorHandler';

interface JwtPayload {
  userId: string;
  email: string;
  role: string;
}

export class AuthController extends BaseController {
  private readonly jwtSecret: string;
  private readonly jwtExpiresIn: string;

  constructor(private readonly userService: UserService) {
    super();
    this.jwtSecret = process.env.JWT_SECRET || 'default-secret';
    this.jwtExpiresIn = process.env.JWT_EXPIRES_IN || '7d';
  }

  // POST /auth/register
  register = async (
    req: Request<{}, {}, CreateUserDto>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const user = await this.userService.create(req.body);
      const tokens = this.generateTokens({
        userId: user.id,
        email: user.email,
        role: user.role,
      });
      
      this.sendCreated(res, { user, ...tokens }, 'Registration successful');
    } catch (error) {
      next(error);
    }
  };

  // POST /auth/login
  login = async (
    req: Request<{}, {}, LoginDto>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const { email, password } = req.body;
      
      // ค้นหา user ด้วย email
      const user = await this.userService.findByEmail(email);
      if (!user) {
        throw AppError.unauthorized('Invalid email or password');
      }
      
      // ตรวจสอบ password
      const isValidPassword = await this.userService.validatePassword(user, password);
      if (!isValidPassword) {
        throw AppError.unauthorized('Invalid email or password');
      }
      
      // ตรวจสอบว่า account active
      if (!user.isActive) {
        throw AppError.unauthorized('Account is deactivated');
      }
      
      const tokens = this.generateTokens({
        userId: user.id,
        email: user.email,
        role: user.role,
      });
      
      const { password: _, ...publicUser } = user;
      
      this.sendSuccess(res, { user: publicUser, ...tokens }, 200, 'Login successful');
    } catch (error) {
      next(error);
    }
  };

  // POST /auth/refresh
  refreshToken = async (
    req: Request<{}, {}, { refreshToken: string }>,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const { refreshToken } = req.body;
      
      if (!refreshToken) {
        throw AppError.badRequest('Refresh token is required');
      }
      
      const decoded = jwt.verify(refreshToken, this.jwtSecret + '_refresh') as JwtPayload;
      
      const user = await this.userService.findById(decoded.userId);
      if (!user) {
        throw AppError.unauthorized('User not found');
      }
      
      const tokens = this.generateTokens({
        userId: decoded.userId,
        email: decoded.email,
        role: decoded.role,
      });
      
      this.sendSuccess(res, tokens, 200, 'Token refreshed successfully');
    } catch (error) {
      if (error instanceof jwt.JsonWebTokenError) {
        next(AppError.unauthorized('Invalid refresh token'));
        return;
      }
      next(error);
    }
  };

  // POST /auth/logout
  logout = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      // ในงานจริง ควร blacklist token นี้
      this.sendSuccess(res, null, 200, 'Logged out successfully');
    } catch (error) {
      next(error);
    }
  };

  private generateTokens(payload: JwtPayload): AuthTokens {
    const accessToken = jwt.sign(payload, this.jwtSecret, {
      expiresIn: this.jwtExpiresIn as jwt.SignOptions['expiresIn'],
    });
    
    const refreshToken = jwt.sign(payload, this.jwtSecret + '_refresh', {
      expiresIn: '30d',
    });
    
    return { accessToken, refreshToken };
  }
}
```

---

## 21.7 Router Typing

### src/routes/userRoutes.ts

```typescript
import { Router } from 'express';
import { UserController } from '../controllers/userController';
import { UserService } from '../services/userService';
import { AuthMiddleware } from '../middleware/authenticate';
import { validateBody } from '../middleware/validate';

export function createUserRouter(authMiddleware: AuthMiddleware): Router {
  const router = Router();
  const userService = new UserService();
  const userController = new UserController(userService);
  
  // Validation schemas
  const createUserSchema = {
    name: { required: true, type: 'string' as const, minLength: 2, maxLength: 50 },
    email: { required: true, type: 'email' as const },
    password: { required: true, type: 'string' as const, minLength: 8, maxLength: 100 },
  };
  
  const updateUserSchema = {
    name: { type: 'string' as const, minLength: 2, maxLength: 50 },
    email: { type: 'email' as const },
  };
  
  // Public routes (ไม่ต้อง authenticate)
  router.post(
    '/',
    validateBody(createUserSchema),
    userController.create
  );
  
  // Protected routes (ต้อง authenticate)
  router.use(authMiddleware.authenticate);
  
  router.get('/me', userController.getProfile);
  router.patch('/me', validateBody(updateUserSchema), userController.updateProfile);
  
  // Admin only routes
  router.get('/', authMiddleware.authorize('admin'), userController.getAll);
  router.get('/:id', authMiddleware.authorize('admin'), userController.getById);
  router.put(
    '/:id',
    authMiddleware.authorize('admin'),
    validateBody(updateUserSchema),
    userController.update
  );
  router.delete('/:id', authMiddleware.authorize('admin'), userController.delete);
  
  return router;
}
```

### src/routes/authRoutes.ts

```typescript
import { Router } from 'express';
import { AuthController } from '../controllers/authController';
import { UserService } from '../services/userService';
import { validateBody } from '../middleware/validate';

export function createAuthRouter(): Router {
  const router = Router();
  const userService = new UserService();
  const authController = new AuthController(userService);
  
  // Validation schemas
  const registerSchema = {
    name: { required: true, type: 'string' as const, minLength: 2, maxLength: 50 },
    email: { required: true, type: 'email' as const },
    password: { required: true, type: 'string' as const, minLength: 8 },
  };
  
  const loginSchema = {
    email: { required: true, type: 'email' as const },
    password: { required: true, type: 'string' as const },
  };
  
  router.post('/register', validateBody(registerSchema), authController.register);
  router.post('/login', validateBody(loginSchema), authController.login);
  router.post('/refresh', authController.refreshToken);
  router.post('/logout', authController.logout);
  
  return router;
}
```

### src/routes/index.ts

```typescript
import { Router } from 'express';
import { createAuthRouter } from './authRoutes';
import { createUserRouter } from './userRoutes';
import { AuthMiddleware } from '../middleware/authenticate';

export function createApiRouter(authMiddleware: AuthMiddleware): Router {
  const router = Router();
  
  // Mount routes
  router.use('/auth', createAuthRouter());
  router.use('/users', createUserRouter(authMiddleware));
  
  return router;
}
```

---

## 21.8 CRUD API Complete Example - Product Management

### src/types/product.ts

```typescript
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  stock: number;
  category: ProductCategory;
  images: string[];
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export type ProductCategory = 'electronics' | 'clothing' | 'food' | 'books' | 'other';

export interface CreateProductDto {
  name: string;
  description: string;
  price: number;
  stock: number;
  category: ProductCategory;
  images?: string[];
}

export interface UpdateProductDto {
  name?: string;
  description?: string;
  price?: number;
  stock?: number;
  category?: ProductCategory;
  images?: string[];
  isActive?: boolean;
}

export interface ProductFilter {
  category?: ProductCategory;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  search?: string;
  page?: number;
  limit?: number;
  sort?: 'name' | 'price' | 'createdAt';
  order?: 'asc' | 'desc';
}
```

### src/services/productService.ts

```typescript
import { v4 as uuidv4 } from 'uuid';
import { Product, CreateProductDto, UpdateProductDto, ProductFilter } from '../types/product';
import { AppError } from '../middleware/errorHandler';

const productsStore: Product[] = [
  {
    id: uuidv4(),
    name: 'MacBook Pro 14"',
    description: 'โน้ตบุ๊คสำหรับมืออาชีพ ประสิทธิภาพสูง',
    price: 59900,
    stock: 10,
    category: 'electronics',
    images: ['macbook-1.jpg', 'macbook-2.jpg'],
    isActive: true,
    createdAt: new Date('2024-01-01'),
    updatedAt: new Date('2024-01-01'),
  },
  {
    id: uuidv4(),
    name: 'เสื้อยืด Cotton 100%',
    description: 'เสื้อยืดผ้าฝ้ายแท้ สวมใส่สบาย',
    price: 290,
    stock: 100,
    category: 'clothing',
    images: ['shirt-1.jpg'],
    isActive: true,
    createdAt: new Date('2024-01-02'),
    updatedAt: new Date('2024-01-02'),
  },
];

interface FindAllResult {
  products: Product[];
  total: number;
}

export class ProductService {
  async findAll(filter: ProductFilter): Promise<FindAllResult> {
    let filtered = productsStore.filter(p => p.isActive);
    
    // ค้นหา
    if (filter.search) {
      const searchLower = filter.search.toLowerCase();
      filtered = filtered.filter(
        p =>
          p.name.toLowerCase().includes(searchLower) ||
          p.description.toLowerCase().includes(searchLower)
      );
    }
    
    // กรองตาม category
    if (filter.category) {
      filtered = filtered.filter(p => p.category === filter.category);
    }
    
    // กรองตาม price range
    if (filter.minPrice !== undefined) {
      filtered = filtered.filter(p => p.price >= filter.minPrice!);
    }
    
    if (filter.maxPrice !== undefined) {
      filtered = filtered.filter(p => p.price <= filter.maxPrice!);
    }
    
    // กรองตาม stock
    if (filter.inStock !== undefined) {
      filtered = filtered.filter(p => filter.inStock ? p.stock > 0 : p.stock === 0);
    }
    
    // เรียงลำดับ
    const sortField = filter.sort || 'createdAt';
    const sortOrder = filter.order || 'desc';
    
    filtered.sort((a, b) => {
      let aVal: string | number | Date = a[sortField as keyof Product] as string | number | Date;
      let bVal: string | number | Date = b[sortField as keyof Product] as string | number | Date;
      
      if (aVal instanceof Date) aVal = aVal.getTime();
      if (bVal instanceof Date) bVal = bVal.getTime();
      
      if (typeof aVal === 'string' && typeof bVal === 'string') {
        return sortOrder === 'asc'
          ? aVal.localeCompare(bVal)
          : bVal.localeCompare(aVal);
      }
      
      return sortOrder === 'asc'
        ? (aVal as number) - (bVal as number)
        : (bVal as number) - (aVal as number);
    });
    
    const total = filtered.length;
    const page = filter.page || 1;
    const limit = filter.limit || 10;
    const skip = (page - 1) * limit;
    
    const products = filtered.slice(skip, skip + limit);
    
    return { products, total };
  }

  async findById(id: string): Promise<Product | null> {
    return productsStore.find(p => p.id === id) || null;
  }

  async create(data: CreateProductDto): Promise<Product> {
    const newProduct: Product = {
      id: uuidv4(),
      ...data,
      images: data.images || [],
      isActive: true,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    
    productsStore.push(newProduct);
    return newProduct;
  }

  async update(id: string, data: UpdateProductDto): Promise<Product | null> {
    const index = productsStore.findIndex(p => p.id === id);
    if (index === -1) return null;
    
    productsStore[index] = {
      ...productsStore[index],
      ...data,
      updatedAt: new Date(),
    };
    
    return productsStore[index];
  }

  async delete(id: string): Promise<boolean> {
    const index = productsStore.findIndex(p => p.id === id);
    if (index === -1) return false;
    
    // Soft delete
    productsStore[index].isActive = false;
    productsStore[index].updatedAt = new Date();
    return true;
  }

  async updateStock(id: string, quantity: number): Promise<Product | null> {
    const product = productsStore.find(p => p.id === id);
    if (!product) return null;
    
    const newStock = product.stock + quantity;
    if (newStock < 0) {
      throw AppError.badRequest('Insufficient stock');
    }
    
    product.stock = newStock;
    product.updatedAt = new Date();
    
    return product;
  }
}
```

---

## 21.9 File Upload กับ TypeScript

### การติดตั้ง multer

```bash
npm install multer
npm install --save-dev @types/multer
```

### src/middleware/upload.ts

```typescript
import multer, { FileFilterCallback } from 'multer';
import path from 'path';
import { Request } from 'express';
import { AppError } from './errorHandler';

// ประเภทไฟล์ที่อนุญาต
const ALLOWED_IMAGE_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
const ALLOWED_DOC_TYPES = ['application/pdf', 'application/msword'];
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

// กำหนด storage
const diskStorage = multer.diskStorage({
  destination: (req: Request, file: Express.Multer.File, callback: (error: Error | null, destination: string) => void) => {
    const uploadDir = path.join(process.cwd(), 'uploads');
    callback(null, uploadDir);
  },
  filename: (req: Request, file: Express.Multer.File, callback: (error: Error | null, filename: string) => void) => {
    const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
    const extension = path.extname(file.originalname);
    const filename = `${file.fieldname}-${uniqueSuffix}${extension}`;
    callback(null, filename);
  },
});

// Memory storage (สำหรับ upload ไปที่ cloud)
const memoryStorage = multer.memoryStorage();

// File filter function
function createFileFilter(allowedTypes: string[]) {
  return (req: Request, file: Express.Multer.File, callback: FileFilterCallback): void => {
    if (allowedTypes.includes(file.mimetype)) {
      callback(null, true);
    } else {
      callback(new AppError(`File type ${file.mimetype} is not allowed`, 400));
    }
  };
}

// Image upload middleware
export const uploadImage = multer({
  storage: diskStorage,
  limits: {
    fileSize: MAX_FILE_SIZE,
  },
  fileFilter: createFileFilter(ALLOWED_IMAGE_TYPES),
});

// Multiple images upload
export const uploadImages = multer({
  storage: diskStorage,
  limits: {
    fileSize: MAX_FILE_SIZE,
    files: 5, // สูงสุด 5 ไฟล์
  },
  fileFilter: createFileFilter(ALLOWED_IMAGE_TYPES),
});

// Document upload
export const uploadDocument = multer({
  storage: diskStorage,
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB
  },
  fileFilter: createFileFilter(ALLOWED_DOC_TYPES),
});

// Memory storage upload (สำหรับ process ก่อน upload)
export const uploadToMemory = multer({
  storage: memoryStorage,
  limits: {
    fileSize: MAX_FILE_SIZE,
  },
  fileFilter: createFileFilter(ALLOWED_IMAGE_TYPES),
});
```

### ตัวอย่าง Controller ที่รับ file upload

```typescript
import { Request, Response, NextFunction } from 'express';
import path from 'path';
import { BaseController } from './baseController';
import { AppError } from '../middleware/errorHandler';

export class FileController extends BaseController {
  // POST /files/upload
  uploadSingle = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      if (!req.file) {
        throw AppError.badRequest('No file uploaded');
      }
      
      const fileInfo = {
        originalName: req.file.originalname,
        filename: req.file.filename,
        mimetype: req.file.mimetype,
        size: req.file.size,
        path: req.file.path,
        url: `/uploads/${req.file.filename}`,
      };
      
      this.sendCreated(res, fileInfo, 'File uploaded successfully');
    } catch (error) {
      next(error);
    }
  };

  // POST /files/upload-multiple
  uploadMultiple = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const files = req.files as Express.Multer.File[];
      
      if (!files || files.length === 0) {
        throw AppError.badRequest('No files uploaded');
      }
      
      const filesInfo = files.map(file => ({
        originalName: file.originalname,
        filename: file.filename,
        mimetype: file.mimetype,
        size: file.size,
        url: `/uploads/${file.filename}`,
      }));
      
      this.sendCreated(res, filesInfo, `${files.length} files uploaded successfully`);
    } catch (error) {
      next(error);
    }
  };
}
```

---

## 21.10 Main Application

### src/app.ts

```typescript
import express, { Application } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import path from 'path';
import { createApiRouter } from './routes';
import { errorHandler, notFoundHandler } from './middleware/errorHandler';
import { requestLogger } from './middleware/requestLogger';
import { createAuthMiddleware } from './middleware/authenticate';

export function createApp(): Application {
  const app = express();
  const jwtSecret = process.env.JWT_SECRET || 'default-secret';
  const authMiddleware = createAuthMiddleware(jwtSecret);
  
  // Security middleware
  app.use(helmet());
  
  // CORS configuration
  app.use(cors({
    origin: process.env.CORS_ORIGIN || '*',
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,
  }));
  
  // Body parsing
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true, limit: '10mb' }));
  
  // Logging
  if (process.env.NODE_ENV !== 'test') {
    app.use(morgan('dev'));
  }
  app.use(requestLogger());
  
  // Static files
  app.use('/uploads', express.static(path.join(process.cwd(), 'uploads')));
  
  // Health check
  app.get('/health', (req, res) => {
    res.json({
      success: true,
      data: {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        environment: process.env.NODE_ENV,
        version: process.env.npm_package_version,
        uptime: process.uptime(),
      },
    });
  });
  
  // API routes
  app.use('/api/v1', createApiRouter(authMiddleware));
  
  // Error handlers (ต้องอยู่ท้ายสุด)
  app.use(notFoundHandler);
  app.use(errorHandler);
  
  return app;
}
```

### src/index.ts

```typescript
import 'dotenv/config';
import { createApp } from './app';

const PORT = parseInt(process.env.PORT || '3000', 10);
const HOST = process.env.HOST || '0.0.0.0';

async function bootstrap(): Promise<void> {
  const app = createApp();
  
  const server = app.listen(PORT, HOST, () => {
    console.log(`
🚀 Server is running!
📍 URL: http://${HOST}:${PORT}
🌍 Environment: ${process.env.NODE_ENV || 'development'}
    `);
  });
  
  // Graceful shutdown
  const shutdown = async (signal: string): Promise<void> => {
    console.log(`\n${signal} received, shutting down gracefully...`);
    
    server.close(() => {
      console.log('HTTP server closed');
      process.exit(0);
    });
    
    // Force shutdown หลังจาก 10 วินาที
    setTimeout(() => {
      console.error('Could not close connections in time, forcefully shutting down');
      process.exit(1);
    }, 10000);
  };
  
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
  
  process.on('uncaughtException', (error: Error) => {
    console.error('Uncaught Exception:', error);
    process.exit(1);
  });
  
  process.on('unhandledRejection', (reason: unknown) => {
    console.error('Unhandled Rejection:', reason);
    process.exit(1);
  });
}

bootstrap();
```

---

## 21.11 Rate Limiting

```typescript
// src/middleware/rateLimit.ts
import { Request, Response, NextFunction } from 'express';

interface RateLimitStore {
  [key: string]: {
    count: number;
    resetTime: number;
  };
}

interface RateLimitOptions {
  windowMs: number;    // ช่วงเวลาในหน่วย milliseconds
  max: number;         // จำนวนสูงสุดต่อ window
  message?: string;
  keyGenerator?: (req: Request) => string;
}

const store: RateLimitStore = {};

export function rateLimit(options: RateLimitOptions) {
  const {
    windowMs,
    max,
    message = 'Too many requests, please try again later',
    keyGenerator = (req) => req.ip || 'unknown',
  } = options;
  
  return (req: Request, res: Response, next: NextFunction): void => {
    const key = keyGenerator(req);
    const now = Date.now();
    
    // ลบ expired entries
    if (store[key] && store[key].resetTime < now) {
      delete store[key];
    }
    
    // สร้าง entry ใหม่ถ้าไม่มี
    if (!store[key]) {
      store[key] = {
        count: 0,
        resetTime: now + windowMs,
      };
    }
    
    store[key].count++;
    
    // เพิ่ม headers
    res.set({
      'X-RateLimit-Limit': max.toString(),
      'X-RateLimit-Remaining': Math.max(0, max - store[key].count).toString(),
      'X-RateLimit-Reset': new Date(store[key].resetTime).toISOString(),
    });
    
    if (store[key].count > max) {
      res.status(429).json({
        success: false,
        message,
      });
      return;
    }
    
    next();
  };
}

// Preset rate limiters
export const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  max: 100,
});

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  max: 10,
  message: 'Too many authentication attempts',
});

export const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 นาที
  max: 60,
});
```

---

## 21.12 การทดสอบ API ด้วย curl

```bash
# ลงทะเบียน user ใหม่
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com","password":"password123"}'

# เข้าสู่ระบบ
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"password123"}'

# ดึงรายการ users (ต้องใช้ admin token)
curl -X GET http://localhost:3000/api/v1/users \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

# ดึงข้อมูล profile ของตัวเอง
curl -X GET http://localhost:3000/api/v1/users/me \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

# อัพเดต profile
curl -X PATCH http://localhost:3000/api/v1/users/me \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{"name":"John Updated"}'

# Health check
curl http://localhost:3000/health
```

---

## สรุปบทที่ 21

ในบทนี้เราได้เรียนรู้:

1. **Express + TypeScript Setup** - การติดตั้งและตั้งค่าพื้นฐาน
2. **Request/Response Types** - การ typing สำหรับ Express objects
3. **Custom Request Extensions** - การเพิ่ม custom properties ใน Request
4. **Middleware Typing** - การสร้าง middleware ที่ type-safe
5. **Controller Pattern** - โครงสร้าง controller ที่มี base class
6. **Service Layer** - logic layer ที่แยกออกจาก controller
7. **Router Typing** - การสร้างและรวม routers
8. **Authentication** - JWT-based authentication system
9. **File Upload** - การรับ file uploads ด้วย multer
10. **Error Handling** - ระบบ error handling ที่ครบถ้วน
11. **Rate Limiting** - การป้องกัน abuse

ในบทถัดไปเราจะเรียนรู้การใช้งาน TypeORM กับ TypeScript เพื่อทำงานกับฐานข้อมูล
