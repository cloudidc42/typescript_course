# ตอนที่ 19: Error Handling ใน TypeScript

## บทนำ

การจัดการข้อผิดพลาดที่ดีเป็นสิ่งสำคัญในการพัฒนา software ที่มีคุณภาพ TypeScript ให้เครื่องมือที่ช่วยให้เราจัดการ errors ได้อย่างชัดเจนและปลอดภัยผ่าน type system

---

## 1. TypeScript Error Types

### 1.1 Error Types พื้นฐาน

```typescript
// JavaScript built-in error types
const errors: Error[] = [
  new Error("ข้อความผิดพลาดทั่วไป"),
  new TypeError("ชนิดข้อมูลไม่ถูกต้อง"),
  new RangeError("ค่าอยู่นอกช่วงที่กำหนด"),
  new ReferenceError("ตัวแปรไม่ถูกนิยาม"),
  new SyntaxError("รูปแบบ syntax ผิด"),
  new URIError("URI ไม่ถูกต้อง"),
  new EvalError("เกิดข้อผิดพลาดใน eval()")
];

// ตรวจสอบ error type
function handleError(error: unknown): string {
  if (error instanceof TypeError) {
    return `TypeError: ${error.message}`;
  }
  if (error instanceof RangeError) {
    return `RangeError: ${error.message}`;
  }
  if (error instanceof Error) {
    return `Error: ${error.message}`;
  }
  if (typeof error === 'string') {
    return `String error: ${error}`;
  }
  return `Unknown error: ${String(error)}`;
}
```

### 1.2 Error Properties

```typescript
function analyzeError(error: Error): void {
  console.log('Name:', error.name);
  console.log('Message:', error.message);
  console.log('Stack:', error.stack);
  
  // ใน Node.js
  if ('code' in error) {
    console.log('Code:', (error as NodeJS.ErrnoException).code);
  }
}

// Error ใน TypeScript เป็น unknown ใน catch blocks
try {
  throw new Error("test");
} catch (error) {
  // TypeScript 4.0+ - error เป็น unknown
  if (error instanceof Error) {
    console.log(error.message); // TypeScript รู้ว่า error มี message property
  }
}
```

---

## 2. Custom Error Classes

### 2.1 Custom Error พื้นฐาน

```typescript
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500
  ) {
    super(message);
    this.name = this.constructor.name;
    
    // Fix prototype chain สำหรับ TypeScript transpilation
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

// การใช้งาน
throw new AppError("ผู้ใช้ไม่มีสิทธิ์", "UNAUTHORIZED", 401);
```

### 2.2 Typed Custom Errors

```typescript
interface ErrorMetadata {
  timestamp: Date;
  requestId?: string;
  userId?: number;
  additionalInfo?: Record<string, any>;
}

abstract class BaseError extends Error {
  public readonly timestamp: Date;
  
  constructor(
    message: string,
    public readonly code: string,
    public readonly metadata?: ErrorMetadata
  ) {
    super(message);
    this.name = this.constructor.name;
    this.timestamp = metadata?.timestamp || new Date();
    Object.setPrototypeOf(this, new.target.prototype);
  }
  
  toJSON(): Record<string, any> {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      timestamp: this.timestamp,
      metadata: this.metadata
    };
  }
}

class ValidationError extends BaseError {
  constructor(
    message: string,
    public readonly field: string,
    public readonly value: any
  ) {
    super(message, 'VALIDATION_ERROR');
  }
}

class NotFoundError extends BaseError {
  constructor(
    public readonly resource: string,
    public readonly id: string | number
  ) {
    super(`${resource} with id ${id} not found`, 'NOT_FOUND');
  }
}

class AuthenticationError extends BaseError {
  constructor(message: string = 'Authentication failed') {
    super(message, 'AUTH_ERROR');
  }
}

class AuthorizationError extends BaseError {
  constructor(
    public readonly requiredPermission: string,
    message?: string
  ) {
    super(message || `ต้องการสิทธิ์: ${requiredPermission}`, 'FORBIDDEN');
  }
}

class DatabaseError extends BaseError {
  constructor(
    message: string,
    public readonly query?: string,
    public readonly originalError?: Error
  ) {
    super(message, 'DATABASE_ERROR');
  }
}

class NetworkError extends BaseError {
  constructor(
    message: string,
    public readonly url?: string,
    public readonly statusCode?: number
  ) {
    super(message, 'NETWORK_ERROR');
  }
}
```

---

## 3. Error Class Hierarchy

### 3.1 HTTP Error Hierarchy

```typescript
abstract class HttpError extends BaseError {
  abstract readonly httpStatus: number;
  
  constructor(message: string, code: string) {
    super(message, code);
  }
}

// 4xx Client Errors
class BadRequestError extends HttpError {
  readonly httpStatus = 400;
  
  constructor(message: string = 'Bad Request', public readonly errors?: string[]) {
    super(message, 'BAD_REQUEST');
  }
}

class UnauthorizedError extends HttpError {
  readonly httpStatus = 401;
  
  constructor(message: string = 'Unauthorized') {
    super(message, 'UNAUTHORIZED');
  }
}

class ForbiddenError extends HttpError {
  readonly httpStatus = 403;
  
  constructor(message: string = 'Forbidden') {
    super(message, 'FORBIDDEN');
  }
}

class NotFoundError2 extends HttpError {
  readonly httpStatus = 404;
  
  constructor(resource: string, id?: string | number) {
    super(
      id ? `${resource} (${id}) ไม่พบ` : `${resource} ไม่พบ`,
      'NOT_FOUND'
    );
  }
}

class ConflictError extends HttpError {
  readonly httpStatus = 409;
  
  constructor(message: string = 'Conflict') {
    super(message, 'CONFLICT');
  }
}

class UnprocessableEntityError extends HttpError {
  readonly httpStatus = 422;
  
  constructor(
    message: string = 'Unprocessable Entity',
    public readonly validationErrors?: Record<string, string[]>
  ) {
    super(message, 'UNPROCESSABLE_ENTITY');
  }
}

class TooManyRequestsError extends HttpError {
  readonly httpStatus = 429;
  
  constructor(public readonly retryAfter?: number) {
    super('Too Many Requests', 'RATE_LIMIT_EXCEEDED');
  }
}

// 5xx Server Errors
class InternalServerError extends HttpError {
  readonly httpStatus = 500;
  
  constructor(message: string = 'Internal Server Error', public readonly originalError?: Error) {
    super(message, 'INTERNAL_SERVER_ERROR');
  }
}

class ServiceUnavailableError extends HttpError {
  readonly httpStatus = 503;
  
  constructor(public readonly service: string) {
    super(`${service} service unavailable`, 'SERVICE_UNAVAILABLE');
  }
}

// Error factory function
function createHttpError(statusCode: number, message: string): HttpError {
  switch (statusCode) {
    case 400: return new BadRequestError(message);
    case 401: return new UnauthorizedError(message);
    case 403: return new ForbiddenError(message);
    case 404: return new NotFoundError2(message);
    case 409: return new ConflictError(message);
    case 422: return new UnprocessableEntityError(message);
    case 429: return new TooManyRequestsError();
    case 503: return new ServiceUnavailableError(message);
    default: return new InternalServerError(message);
  }
}
```

---

## 4. try/catch/finally Typing

### 4.1 Typed try/catch

```typescript
// TypeScript 4.0+: catch clause error เป็น unknown
async function fetchData(url: string): Promise<any> {
  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new NetworkError(
        `HTTP ${response.status}: ${response.statusText}`,
        url,
        response.status
      );
    }
    
    return await response.json();
  } catch (error) {
    // narrowing error type
    if (error instanceof NetworkError) {
      console.error(`Network error for ${error.url}: ${error.message}`);
      throw error;
    }
    
    if (error instanceof SyntaxError) {
      throw new Error(`Invalid JSON response from ${url}`);
    }
    
    // unknown error
    throw new InternalServerError(
      'Unexpected error occurred',
      error instanceof Error ? error : undefined
    );
  }
}
```

### 4.2 Error Type Guards

```typescript
function isError(value: unknown): value is Error {
  return value instanceof Error;
}

function isNetworkError(value: unknown): value is NetworkError {
  return value instanceof NetworkError;
}

function isValidationError(value: unknown): value is ValidationError {
  return value instanceof ValidationError;
}

function isHttpError(value: unknown): value is HttpError {
  return value instanceof HttpError;
}

// ใช้ type guards
function handleApiError(error: unknown): never {
  if (isValidationError(error)) {
    console.error(`Validation failed: ${error.field} = ${error.value}`);
  } else if (isNetworkError(error)) {
    console.error(`Network error: ${error.url} (${error.statusCode})`);
  } else if (isHttpError(error)) {
    console.error(`HTTP ${error.httpStatus}: ${error.message}`);
  } else if (isError(error)) {
    console.error(`Error: ${error.message}`);
  } else {
    console.error('Unknown error:', error);
  }
  
  throw error;
}
```

### 4.3 finally กับ Cleanup

```typescript
class ResourceManager {
  private resources: Map<string, { close: () => Promise<void> }> = new Map();
  
  async acquire(id: string, factory: () => Promise<{ close: () => Promise<void> }>): Promise<void> {
    const resource = await factory();
    this.resources.set(id, resource);
  }
  
  async withResource<T>(
    id: string,
    factory: () => Promise<{ close: () => Promise<void>; use: () => Promise<T> }>
  ): Promise<T> {
    const resource = await factory();
    
    try {
      return await resource.use();
    } finally {
      await resource.close(); // ทำงานเสมอ ไม่ว่าจะ success หรือ error
      this.resources.delete(id);
    }
  }
  
  async closeAll(): Promise<void> {
    const closingPromises = Array.from(this.resources.values()).map(r => r.close());
    await Promise.allSettled(closingPromises);
    this.resources.clear();
  }
}

// Database connection example
async function withDatabaseConnection<T>(
  fn: (connection: any) => Promise<T>
): Promise<T> {
  const connection = await createDatabaseConnection();
  
  try {
    const result = await fn(connection);
    await connection.commit();
    return result;
  } catch (error) {
    await connection.rollback();
    throw error;
  } finally {
    await connection.close();
  }
}

async function createDatabaseConnection() {
  return {
    query: async (sql: string) => [],
    commit: async () => {},
    rollback: async () => {},
    close: async () => {}
  };
}
```

---

## 5. Result Type Pattern (Ok/Err)

### 5.1 Result Type พื้นฐาน

```typescript
type Result<T, E extends Error = Error> = Ok<T> | Err<E>;

class Ok<T> {
  readonly _type = 'ok' as const;
  
  constructor(public readonly value: T) {}
  
  isOk(): this is Ok<T> { return true; }
  isErr(): this is Err<never> { return false; }
  
  map<U>(fn: (value: T) => U): Ok<U> {
    return new Ok(fn(this.value));
  }
  
  flatMap<U, E extends Error>(fn: (value: T) => Result<U, E>): Result<U, E> {
    return fn(this.value);
  }
  
  getOrElse(_defaultValue: T): T {
    return this.value;
  }
  
  getOrThrow(): T {
    return this.value;
  }
}

class Err<E extends Error = Error> {
  readonly _type = 'err' as const;
  
  constructor(public readonly error: E) {}
  
  isOk(): this is Ok<never> { return false; }
  isErr(): this is Err<E> { return true; }
  
  map<U>(_fn: (value: never) => U): Err<E> {
    return this;
  }
  
  flatMap<U>(_fn: (value: never) => Result<U, E>): Err<E> {
    return this;
  }
  
  getOrElse<T>(defaultValue: T): T {
    return defaultValue;
  }
  
  getOrThrow(): never {
    throw this.error;
  }
}

// Helper functions
function ok<T>(value: T): Ok<T> {
  return new Ok(value);
}

function err<E extends Error>(error: E): Err<E> {
  return new Err(error);
}
```

### 5.2 Result Pattern ใน Practice

```typescript
interface UserDto {
  name: string;
  email: string;
  age: number;
}

// Validation ที่คืน Result
function validateUser(data: unknown): Result<UserDto, ValidationError> {
  if (!data || typeof data !== 'object') {
    return err(new ValidationError('data ต้องเป็น object', 'data', data));
  }
  
  const obj = data as Record<string, unknown>;
  
  if (!obj.name || typeof obj.name !== 'string') {
    return err(new ValidationError('name ต้องเป็น string ที่ไม่ว่าง', 'name', obj.name));
  }
  
  if (!obj.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(obj.email as string)) {
    return err(new ValidationError('email ไม่ถูกต้อง', 'email', obj.email));
  }
  
  if (!obj.age || typeof obj.age !== 'number' || obj.age < 0 || obj.age > 150) {
    return err(new ValidationError('age ต้องเป็นตัวเลขระหว่าง 0-150', 'age', obj.age));
  }
  
  return ok({
    name: obj.name as string,
    email: obj.email as string,
    age: obj.age as number
  });
}

// Service ที่ใช้ Result
class UserServiceWithResult {
  async createUser(data: unknown): Promise<Result<User, BaseError>> {
    const validationResult = validateUser(data);
    
    if (validationResult.isErr()) {
      return validationResult;
    }
    
    const userDto = validationResult.value;
    
    try {
      const existingUser = await this.findByEmail(userDto.email);
      if (existingUser.isOk()) {
        return err(new ConflictError(`Email ${userDto.email} ถูกใช้แล้ว`));
      }
      
      const user = await this.saveUser(userDto);
      return ok(user);
    } catch (error) {
      return err(new DatabaseError(
        'Failed to create user',
        undefined,
        error instanceof Error ? error : undefined
      ));
    }
  }
  
  private async findByEmail(email: string): Promise<Result<User, NotFoundError>> {
    // mock implementation
    return err(new NotFoundError('User', email));
  }
  
  private async saveUser(dto: UserDto): Promise<User> {
    return { id: 1, ...dto, createdAt: new Date() } as any;
  }
}

// การใช้งาน
const service = new UserServiceWithResult();
const result = await service.createUser({
  name: 'สมชาย',
  email: 'somchai@example.com',
  age: 25
});

if (result.isOk()) {
  console.log('สร้างผู้ใช้สำเร็จ:', result.value);
} else {
  console.error('เกิดข้อผิดพลาด:', result.error.message);
}
```

---

## 6. Either Monad Pattern

### 6.1 Either Type

```typescript
type Either<L, R> = Left<L> | Right<R>;

class Left<L> {
  readonly _tag = 'Left' as const;
  
  constructor(public readonly value: L) {}
  
  isLeft(): this is Left<L> { return true; }
  isRight(): this is Right<never> { return false; }
  
  map<R2>(_fn: (r: never) => R2): Left<L> { return this; }
  mapLeft<L2>(fn: (l: L) => L2): Left<L2> { return new Left(fn(this.value)); }
  
  chain<R2>(_fn: (r: never) => Either<L, R2>): Left<L> { return this; }
  
  fold<B>(onLeft: (l: L) => B, _onRight: (r: never) => B): B {
    return onLeft(this.value);
  }
}

class Right<R> {
  readonly _tag = 'Right' as const;
  
  constructor(public readonly value: R) {}
  
  isLeft(): this is Left<never> { return false; }
  isRight(): this is Right<R> { return true; }
  
  map<R2>(fn: (r: R) => R2): Right<R2> { return new Right(fn(this.value)); }
  mapLeft<L2>(_fn: (l: never) => L2): Right<R> { return this; }
  
  chain<L, R2>(fn: (r: R) => Either<L, R2>): Either<L, R2> { return fn(this.value); }
  
  fold<B>(_onLeft: (l: never) => B, onRight: (r: R) => B): B {
    return onRight(this.value);
  }
}

function left<L>(value: L): Left<L> { return new Left(value); }
function right<R>(value: R): Right<R> { return new Right(value); }

function tryCatch<L extends Error, R>(
  fn: () => R,
  onError: (error: Error) => L
): Either<L, R> {
  try {
    return right(fn());
  } catch (error) {
    return left(onError(error instanceof Error ? error : new Error(String(error))));
  }
}
```

### 6.2 Either ใน Practice

```typescript
type ParseError = { type: 'PARSE_ERROR'; message: string };
type ValidationError2 = { type: 'VALIDATION_ERROR'; field: string; message: string };
type ProcessingError = ParseError | ValidationError2;

function parseJson(json: string): Either<ParseError, any> {
  return tryCatch(
    () => JSON.parse(json),
    (error) => ({ type: 'PARSE_ERROR', message: error.message })
  );
}

function validateAge(data: any): Either<ValidationError2, { age: number }> {
  if (typeof data.age !== 'number' || data.age < 0 || data.age > 150) {
    return left({
      type: 'VALIDATION_ERROR',
      field: 'age',
      message: 'age ต้องเป็นตัวเลขระหว่าง 0-150'
    });
  }
  return right({ age: data.age });
}

// Pipeline ด้วย Either
const processInput = (input: string): Either<ProcessingError, { age: number }> => {
  return parseJson(input).chain(data => validateAge(data));
};

// การใช้งาน
const result2 = processInput('{"age": 25}');
result2.fold(
  error => console.error('Error:', error.message),
  data => console.log('Age:', data.age)
);
```

---

## 7. Validation Errors

### 7.1 Multi-field Validation

```typescript
interface FieldError {
  field: string;
  message: string;
  value?: any;
}

class MultiValidationError extends BaseError {
  constructor(
    public readonly errors: FieldError[],
    message: string = 'Validation failed'
  ) {
    super(message, 'VALIDATION_ERROR');
  }
  
  hasError(field: string): boolean {
    return this.errors.some(e => e.field === field);
  }
  
  getError(field: string): string | undefined {
    return this.errors.find(e => e.field === field)?.message;
  }
  
  toRecord(): Record<string, string> {
    return Object.fromEntries(this.errors.map(e => [e.field, e.message]));
  }
}

// Validator class
class Validator<T extends Record<string, any>> {
  private errors: FieldError[] = [];
  
  constructor(private data: T) {}
  
  required(field: keyof T, message?: string): this {
    const value = this.data[field];
    if (value === null || value === undefined || value === '') {
      this.errors.push({
        field: String(field),
        message: message || `${String(field)} is required`,
        value
      });
    }
    return this;
  }
  
  minLength(field: keyof T, min: number, message?: string): this {
    const value = this.data[field];
    if (typeof value === 'string' && value.length < min) {
      this.errors.push({
        field: String(field),
        message: message || `${String(field)} ต้องมีความยาวอย่างน้อย ${min} ตัวอักษร`,
        value
      });
    }
    return this;
  }
  
  maxLength(field: keyof T, max: number, message?: string): this {
    const value = this.data[field];
    if (typeof value === 'string' && value.length > max) {
      this.errors.push({
        field: String(field),
        message: message || `${String(field)} ต้องมีความยาวไม่เกิน ${max} ตัวอักษร`,
        value
      });
    }
    return this;
  }
  
  isEmail(field: keyof T, message?: string): this {
    const value = this.data[field];
    if (typeof value === 'string' && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      this.errors.push({
        field: String(field),
        message: message || `${String(field)} ต้องเป็น email ที่ถูกต้อง`,
        value
      });
    }
    return this;
  }
  
  custom(field: keyof T, validator: (value: any) => string | null): this {
    const value = this.data[field];
    const error = validator(value);
    if (error) {
      this.errors.push({ field: String(field), message: error, value });
    }
    return this;
  }
  
  validate(): MultiValidationError | null {
    if (this.errors.length > 0) {
      return new MultiValidationError(this.errors);
    }
    return null;
  }
  
  getOrThrow(): T {
    const error = this.validate();
    if (error) throw error;
    return this.data;
  }
}

// การใช้งาน
function validateRegistration(data: any): User {
  return new Validator(data)
    .required('name', 'กรุณาระบุชื่อ')
    .minLength('name', 3, 'ชื่อต้องมีอย่างน้อย 3 ตัวอักษร')
    .required('email', 'กรุณาระบุ email')
    .isEmail('email', 'รูปแบบ email ไม่ถูกต้อง')
    .required('password', 'กรุณาระบุรหัสผ่าน')
    .minLength('password', 8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    .custom('age', (v) => {
      if (v !== undefined && (v < 0 || v > 150)) {
        return 'อายุต้องอยู่ระหว่าง 0-150 ปี';
      }
      return null;
    })
    .getOrThrow();
}
```

---

## 8. HTTP Error Handling

### 8.1 Express Error Handler

```typescript
import express, { Request, Response, NextFunction } from 'express';

interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: any;
    timestamp: string;
    requestId?: string;
  };
}

function errorHandler(
  error: unknown,
  req: Request,
  res: Response,
  _next: NextFunction
): void {
  const requestId = req.headers['x-request-id'] as string;
  
  if (error instanceof MultiValidationError) {
    res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: error.message,
        details: error.errors,
        timestamp: new Date().toISOString(),
        requestId
      }
    } as ErrorResponse);
    return;
  }
  
  if (error instanceof HttpError) {
    res.status(error.httpStatus).json({
      error: {
        code: error.code,
        message: error.message,
        timestamp: new Date().toISOString(),
        requestId
      }
    } as ErrorResponse);
    return;
  }
  
  if (error instanceof BaseError) {
    res.status(500).json({
      error: {
        code: error.code,
        message: error.message,
        timestamp: new Date().toISOString(),
        requestId
      }
    } as ErrorResponse);
    return;
  }
  
  // Unknown error
  console.error('Unhandled error:', error);
  
  res.status(500).json({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'เกิดข้อผิดพลาดภายในเซิร์ฟเวอร์',
      timestamp: new Date().toISOString(),
      requestId
    }
  } as ErrorResponse);
}

// Async error wrapper
function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<void>
) {
  return (req: Request, res: Response, next: NextFunction) => {
    fn(req, res, next).catch(next);
  };
}

// การใช้งาน
const router = express.Router();

router.get('/users/:id', asyncHandler(async (req, res) => {
  const id = parseInt(req.params.id);
  
  if (isNaN(id)) {
    throw new BadRequestError('ID ต้องเป็นตัวเลข');
  }
  
  const user = await findUserById(id);
  
  if (!user) {
    throw new NotFoundError2('User', id);
  }
  
  res.json(user);
}));

async function findUserById(id: number): Promise<User | null> {
  return null; // mock
}
```

---

## 9. Error Logging

### 9.1 Logger Interface และ Implementation

```typescript
type LogLevel = 'debug' | 'info' | 'warn' | 'error' | 'fatal';

interface LogEntry {
  level: LogLevel;
  message: string;
  timestamp: Date;
  context?: Record<string, any>;
  error?: {
    name: string;
    message: string;
    stack?: string;
    code?: string;
  };
}

interface Logger {
  debug(message: string, context?: Record<string, any>): void;
  info(message: string, context?: Record<string, any>): void;
  warn(message: string, context?: Record<string, any>): void;
  error(message: string, error?: unknown, context?: Record<string, any>): void;
  fatal(message: string, error?: unknown, context?: Record<string, any>): void;
}

class ConsoleLogger implements Logger {
  constructor(private minLevel: LogLevel = 'info') {}
  
  private shouldLog(level: LogLevel): boolean {
    const levels: LogLevel[] = ['debug', 'info', 'warn', 'error', 'fatal'];
    return levels.indexOf(level) >= levels.indexOf(this.minLevel);
  }
  
  private log(level: LogLevel, message: string, error?: unknown, context?: Record<string, any>): void {
    if (!this.shouldLog(level)) return;
    
    const entry: LogEntry = {
      level,
      message,
      timestamp: new Date(),
      context
    };
    
    if (error) {
      if (error instanceof Error) {
        entry.error = {
          name: error.name,
          message: error.message,
          stack: error.stack,
          code: (error as any).code
        };
      } else {
        entry.error = { name: 'UnknownError', message: String(error) };
      }
    }
    
    const output = JSON.stringify(entry, null, 2);
    
    if (level === 'error' || level === 'fatal') {
      console.error(output);
    } else if (level === 'warn') {
      console.warn(output);
    } else {
      console.log(output);
    }
  }
  
  debug(message: string, context?: Record<string, any>): void {
    this.log('debug', message, undefined, context);
  }
  
  info(message: string, context?: Record<string, any>): void {
    this.log('info', message, undefined, context);
  }
  
  warn(message: string, context?: Record<string, any>): void {
    this.log('warn', message, undefined, context);
  }
  
  error(message: string, error?: unknown, context?: Record<string, any>): void {
    this.log('error', message, error, context);
  }
  
  fatal(message: string, error?: unknown, context?: Record<string, any>): void {
    this.log('fatal', message, error, context);
  }
}

// การใช้งาน
const logger: Logger = new ConsoleLogger('debug');

logger.info('แอปพลิเคชันเริ่มทำงาน', { port: 3000, env: 'production' });
logger.warn('Memory usage สูง', { usagePercent: 85 });
logger.error('Database connection failed', new DatabaseError('Connection timeout'), {
  host: 'localhost',
  port: 5432
});
```

---

## 10. Never Returning Functions

### 10.1 never Return Type

```typescript
// Function ที่ throw เสมอ ต้องมี return type เป็น never
function throwError(message: string): never {
  throw new Error(message);
}

function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

// ใช้กับ exhaustive checking
type Shape = 'circle' | 'rectangle' | 'triangle';

function getArea(shape: Shape, ...dimensions: number[]): number {
  switch (shape) {
    case 'circle':
      return Math.PI * dimensions[0] ** 2;
    case 'rectangle':
      return dimensions[0] * dimensions[1];
    case 'triangle':
      return 0.5 * dimensions[0] * dimensions[1];
    default:
      return assertNever(shape); // TypeScript จะแจ้ง error ถ้า shape มีค่าอื่นที่ไม่ได้ handle
  }
}
```

### 10.2 never ใน Error Handling

```typescript
// Process exit function
function exitProcess(code: number, message?: string): never {
  if (message) {
    console.error(`Fatal: ${message}`);
  }
  process.exit(code);
}

// Fatal error handler
function handleFatalError(error: unknown): never {
  const logger = new ConsoleLogger('fatal');
  logger.fatal('Fatal error occurred', error);
  
  if (process.env.NODE_ENV === 'production') {
    exitProcess(1);
  }
  
  throw error; // ใน development mode, re-throw
}

// Type narrowing กับ never
function processEvent(event: { type: 'click' } | { type: 'keypress'; key: string }): void {
  switch (event.type) {
    case 'click':
      console.log('Clicked!');
      break;
    case 'keypress':
      console.log(`Key pressed: ${event.key}`);
      break;
    default:
      // TypeScript รู้ว่า event ไม่มี type อื่นนอกจากที่ระบุ
      assertNever(event);
  }
}
```

---

## 11. Error Boundary Pattern

### 11.1 Application Error Boundary

```typescript
type ErrorHandler = (error: unknown) => void;

class ErrorBoundary {
  private handlers: Map<Function, ErrorHandler[]> = new Map();
  private defaultHandler?: ErrorHandler;
  
  catch<E extends Error>(
    ErrorClass: new (...args: any[]) => E,
    handler: (error: E) => void
  ): this {
    const existing = this.handlers.get(ErrorClass) || [];
    this.handlers.set(ErrorClass, [...existing, handler as ErrorHandler]);
    return this;
  }
  
  default(handler: ErrorHandler): this {
    this.defaultHandler = handler;
    return this;
  }
  
  handle(error: unknown): void {
    for (const [ErrorClass, handlers] of this.handlers.entries()) {
      if (error instanceof ErrorClass) {
        handlers.forEach(h => h(error));
        return;
      }
    }
    
    if (this.defaultHandler) {
      this.defaultHandler(error);
    } else {
      console.error('Unhandled error:', error);
    }
  }
  
  wrap<T>(fn: () => T): T | undefined {
    try {
      return fn();
    } catch (error) {
      this.handle(error);
      return undefined;
    }
  }
  
  async wrapAsync<T>(fn: () => Promise<T>): Promise<T | undefined> {
    try {
      return await fn();
    } catch (error) {
      this.handle(error);
      return undefined;
    }
  }
}

// การใช้งาน
const boundary = new ErrorBoundary()
  .catch(ValidationError, error => {
    console.error(`Validation: ${error.field} = ${error.message}`);
    // แสดง user-friendly message
  })
  .catch(UnauthorizedError, error => {
    console.error('Unauthorized:', error.message);
    // redirect ไปหน้า login
  })
  .catch(DatabaseError, error => {
    console.error('Database error:', error.message, error.query);
    // alert admin
  })
  .default(error => {
    console.error('Unexpected error:', error);
    // log ไปยัง error tracking service
  });

// ใช้ boundary
const user = await boundary.wrapAsync(async () => {
  return await fetchUser(1);
});
```

---

## 12. Error Aggregation

### 12.1 การรวม Errors หลายตัว

```typescript
class AggregateError2 extends BaseError {
  constructor(
    public readonly errors: Error[],
    message?: string
  ) {
    super(message || `${errors.length} errors occurred`, 'AGGREGATE_ERROR');
  }
  
  getErrors(): Error[] {
    return [...this.errors];
  }
  
  hasErrorOfType<E extends Error>(ErrorClass: new (...args: any[]) => E): boolean {
    return this.errors.some(e => e instanceof ErrorClass);
  }
  
  getErrorsOfType<E extends Error>(ErrorClass: new (...args: any[]) => E): E[] {
    return this.errors.filter((e): e is E => e instanceof ErrorClass);
  }
}

// Function ที่รวม errors จากหลาย operations
async function processAllUsers(userIds: number[]): Promise<{
  succeeded: number[];
  failed: Array<{ id: number; error: Error }>;
}> {
  const results = await Promise.allSettled(
    userIds.map(async id => {
      const user = await fetchUser(id);
      return id;
    })
  );
  
  const succeeded: number[] = [];
  const failed: Array<{ id: number; error: Error }> = [];
  
  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      succeeded.push(result.value);
    } else {
      failed.push({
        id: userIds[index],
        error: result.reason instanceof Error
          ? result.reason
          : new Error(String(result.reason))
      });
    }
  });
  
  if (failed.length > 0 && failed.length === userIds.length) {
    throw new AggregateError2(
      failed.map(f => f.error),
      `All ${userIds.length} user operations failed`
    );
  }
  
  return { succeeded, failed };
}
```

---

## 13. ตัวอย่างจริง: Complete Error Handling System

```typescript
// Full error handling system สำหรับ web application

// Setup error tracking (Sentry-like)
class ErrorTracker {
  private events: Array<{
    error: Error;
    context: Record<string, any>;
    timestamp: Date;
  }> = [];
  
  capture(error: Error, context: Record<string, any> = {}): void {
    this.events.push({
      error,
      context: {
        ...context,
        url: typeof window !== 'undefined' ? window.location.href : undefined,
        userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : undefined
      },
      timestamp: new Date()
    });
    
    console.error('[ErrorTracker]', {
      name: error.name,
      message: error.message,
      code: (error as any).code,
      stack: error.stack,
      context
    });
  }
  
  getEvents() {
    return [...this.events];
  }
}

const tracker = new ErrorTracker();

// Global error handler
async function globalErrorHandler(error: unknown, context?: Record<string, any>): Promise<void> {
  const err = error instanceof Error ? error : new Error(String(error));
  
  // Log
  const logger = new ConsoleLogger();
  logger.error('Global error', err, context);
  
  // Track
  tracker.capture(err, context || {});
  
  // ส่ง alert สำหรับ critical errors
  if (error instanceof InternalServerError || error instanceof DatabaseError) {
    await sendAlert(err);
  }
}

async function sendAlert(error: Error): Promise<void> {
  console.log(`[ALERT] Critical error: ${error.message}`);
  // จริงๆ ส่ง Slack/Email notification
}

// ตัวอย่างการใช้งานทั้งหมด
async function createUserSafe(data: unknown): Promise<User | null> {
  try {
    // Validate
    const validData = validateRegistration(data);
    
    // Create
    const service = new UserServiceWithResult();
    const result = await service.createUser(validData);
    
    if (result.isErr()) {
      logger.warn('User creation failed', { error: result.error.message });
      return null;
    }
    
    logger.info('User created successfully', { userId: result.value.id });
    return result.value;
    
  } catch (error) {
    await globalErrorHandler(error, { action: 'createUser', data });
    return null;
  }
}

const logger2 = new ConsoleLogger();
```

---

## สรุป

Error Handling ใน TypeScript มีเครื่องมือและ patterns หลากหลาย:

1. **Custom Error Classes** - สร้าง error types ที่มีความหมายชัดเจน
2. **Error Hierarchy** - จัดระเบียบ errors ด้วย inheritance
3. **try/catch/finally** - จัดการ errors พร้อม cleanup
4. **Result Type** - แสดง errors ผ่าน return type แทน exceptions
5. **Either Monad** - functional programming approach
6. **Validation Errors** - จัดการ validation ที่ซับซ้อน
7. **HTTP Error Handling** - จัดการ HTTP errors อย่างเป็นระบบ
8. **Error Logging** - บันทึก errors อย่างมีโครงสร้าง
9. **Never Type** - exhaustive type checking
10. **Error Boundaries** - จำกัดผลกระทบของ errors

การเลือก pattern ที่เหมาะสมขึ้นอยู่กับบริบทและความต้องการของแต่ละ project
