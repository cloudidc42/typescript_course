# ส่วนที่ 73: รูปแบบการออกแบบ API ใน TypeScript

## บทนำ

การออกแบบ API ที่ดีเป็นสิ่งสำคัญสำหรับความสำเร็จของ software product ในบทนี้เราจะเรียนรู้รูปแบบและ best practices ในการออกแบบ API ด้วย TypeScript

---

## 1. RESTful API Design พื้นฐาน

### 1.1 Route Structure และ Naming Conventions

```typescript
import express, { Request, Response, Router } from 'express';

// Resource-based routing
const router = Router();

// GET /users - ดึง users ทั้งหมด
router.get('/users', async (req: Request, res: Response) => {
  const users = await userService.findAll();
  res.json({ data: users });
});

// GET /users/:id - ดึง user เฉพาะ
router.get('/users/:id', async (req: Request, res: Response) => {
  const user = await userService.findById(req.params.id);
  if (!user) {
    return res.status(404).json({ error: 'ไม่พบผู้ใช้' });
  }
  res.json({ data: user });
});

// POST /users - สร้าง user ใหม่
router.post('/users', async (req: Request, res: Response) => {
  const user = await userService.create(req.body);
  res.status(201).json({ data: user });
});

// PUT /users/:id - อัปเดต user (แทนที่ทั้งหมด)
router.put('/users/:id', async (req: Request, res: Response) => {
  const user = await userService.replace(req.params.id, req.body);
  res.json({ data: user });
});

// PATCH /users/:id - อัปเดต user บางส่วน
router.patch('/users/:id', async (req: Request, res: Response) => {
  const user = await userService.update(req.params.id, req.body);
  res.json({ data: user });
});

// DELETE /users/:id - ลบ user
router.delete('/users/:id', async (req: Request, res: Response) => {
  await userService.delete(req.params.id);
  res.status(204).send();
});

// Nested resources
// GET /users/:userId/posts - ดึง posts ของ user
router.get('/users/:userId/posts', async (req: Request, res: Response) => {
  const posts = await postService.findByUserId(req.params.userId);
  res.json({ data: posts });
});
```

### 1.2 HTTP Status Codes

```typescript
enum HttpStatus {
  // 2xx Success
  OK = 200,
  CREATED = 201,
  ACCEPTED = 202,
  NO_CONTENT = 204,

  // 3xx Redirection
  MOVED_PERMANENTLY = 301,
  NOT_MODIFIED = 304,

  // 4xx Client Errors
  BAD_REQUEST = 400,
  UNAUTHORIZED = 401,
  FORBIDDEN = 403,
  NOT_FOUND = 404,
  METHOD_NOT_ALLOWED = 405,
  CONFLICT = 409,
  GONE = 410,
  UNPROCESSABLE_ENTITY = 422,
  TOO_MANY_REQUESTS = 429,

  // 5xx Server Errors
  INTERNAL_SERVER_ERROR = 500,
  NOT_IMPLEMENTED = 501,
  BAD_GATEWAY = 502,
  SERVICE_UNAVAILABLE = 503,
}

// Type-safe response helpers
class ApiResponse {
  static ok<T>(res: Response, data: T) {
    return res.status(HttpStatus.OK).json({ success: true, data });
  }

  static created<T>(res: Response, data: T) {
    return res.status(HttpStatus.CREATED).json({ success: true, data });
  }

  static noContent(res: Response) {
    return res.status(HttpStatus.NO_CONTENT).send();
  }

  static badRequest(res: Response, message: string, errors?: unknown[]) {
    return res.status(HttpStatus.BAD_REQUEST).json({
      success: false,
      error: { code: 'BAD_REQUEST', message, errors }
    });
  }

  static notFound(res: Response, resource: string) {
    return res.status(HttpStatus.NOT_FOUND).json({
      success: false,
      error: { code: 'NOT_FOUND', message: `ไม่พบ ${resource}` }
    });
  }

  static serverError(res: Response, message = 'เกิดข้อผิดพลาดภายใน') {
    return res.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      success: false,
      error: { code: 'INTERNAL_ERROR', message }
    });
  }
}
```

---

## 2. API Versioning

### 2.1 URL Path Versioning

```typescript
import { Router } from 'express';

// V1 Router
const v1Router = Router();
v1Router.get('/users', v1UserController.getAll);

// V2 Router (มี features เพิ่ม)
const v2Router = Router();
v2Router.get('/users', v2UserController.getAll);

// Mount versions
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);
```

### 2.2 Header-based Versioning

```typescript
import { Request, Response, NextFunction } from 'express';

function versionMiddleware(req: Request, res: Response, next: NextFunction) {
  const version = req.headers['api-version'] as string ?? '1';
  req.apiVersion = parseInt(version, 10);
  next();
}

// Augment Express Request
declare global {
  namespace Express {
    interface Request {
      apiVersion: number;
    }
  }
}

// Version-aware controller
async function getUserController(req: Request, res: Response) {
  const user = await userService.findById(req.params.id);

  if (req.apiVersion >= 2) {
    // V2: เพิ่ม computed fields
    return res.json({
      data: {
        ...user,
        fullProfile: await userService.getFullProfile(user.id)
      }
    });
  }

  // V1: Basic response
  return res.json({ data: user });
}
```

### 2.3 Version Manager

```typescript
class ApiVersionManager {
  private versions: Map<number, Router> = new Map();
  private deprecatedVersions: Set<number> = new Set();

  registerVersion(version: number, router: Router): this {
    this.versions.set(version, router);
    return this;
  }

  deprecateVersion(version: number, sunsetDate: Date): this {
    this.deprecatedVersions.add(version);
    return this;
  }

  getVersionMiddleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const version = req.apiVersion ?? 1;

      if (this.deprecatedVersions.has(version)) {
        res.setHeader('Deprecation', 'true');
        res.setHeader('Sunset', 'Sat, 01 Jan 2026 00:00:00 GMT');
        res.setHeader(
          'Link',
          `</api/v${version + 1}${req.path}>; rel="successor-version"`
        );
      }

      if (!this.versions.has(version)) {
        return res.status(400).json({ error: `Version ${version} ไม่รองรับ` });
      }

      next();
    };
  }
}
```

---

## 3. Pagination Patterns

### 3.1 Offset-based Pagination

```typescript
interface PaginationParams {
  page: number;
  pageSize: number;
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

function parsePagination(query: Record<string, string>): PaginationParams {
  const page = Math.max(1, parseInt(query.page ?? '1', 10));
  const pageSize = Math.min(100, Math.max(1, parseInt(query.pageSize ?? '20', 10)));
  return { page, pageSize };
}

async function getPaginatedUsers(
  params: PaginationParams
): Promise<PaginatedResponse<User>> {
  const { page, pageSize } = params;
  const offset = (page - 1) * pageSize;

  const [users, total] = await Promise.all([
    userRepo.findAll({ limit: pageSize, offset }),
    userRepo.count()
  ]);

  const totalPages = Math.ceil(total / pageSize);

  return {
    data: users,
    pagination: {
      total,
      page,
      pageSize,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1
    }
  };
}

// Express handler
app.get('/users', async (req: Request, res: Response) => {
  const params = parsePagination(req.query as Record<string, string>);
  const result = await getPaginatedUsers(params);
  res.json(result);
});
```

### 3.2 Cursor-based Pagination

```typescript
interface CursorPaginationParams {
  cursor?: string;
  limit: number;
  direction: 'forward' | 'backward';
}

interface CursorPaginatedResponse<T> {
  data: T[];
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor?: string;
    endCursor?: string;
  };
}

function encodeCursor(id: string, timestamp: Date): string {
  const payload = JSON.stringify({ id, timestamp: timestamp.toISOString() });
  return Buffer.from(payload).toString('base64');
}

function decodeCursor(cursor: string): { id: string; timestamp: Date } {
  const payload = JSON.parse(Buffer.from(cursor, 'base64').toString());
  return { id: payload.id, timestamp: new Date(payload.timestamp) };
}

async function getUsersWithCursor(
  params: CursorPaginationParams
): Promise<CursorPaginatedResponse<User>> {
  const limit = params.limit + 1; // Fetch one extra to check hasMore
  let users: User[] = [];

  if (params.cursor) {
    const { id, timestamp } = decodeCursor(params.cursor);
    users = await userRepo.findAfterCursor(id, timestamp, limit);
  } else {
    users = await userRepo.findAll({ limit });
  }

  const hasNextPage = users.length > params.limit;
  if (hasNextPage) users.pop();

  return {
    data: users,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!params.cursor,
      startCursor: users.length > 0
        ? encodeCursor(users[0].id, users[0].createdAt)
        : undefined,
      endCursor: users.length > 0
        ? encodeCursor(users[users.length - 1].id, users[users.length - 1].createdAt)
        : undefined
    }
  };
}
```

---

## 4. Filtering และ Sorting

### 4.1 Filter Builder

```typescript
interface FilterOperator {
  eq?: unknown;
  ne?: unknown;
  gt?: number | Date;
  gte?: number | Date;
  lt?: number | Date;
  lte?: number | Date;
  in?: unknown[];
  nin?: unknown[];
  contains?: string;
  startsWith?: string;
  endsWith?: string;
}

type FilterQuery<T> = {
  [K in keyof T]?: FilterOperator | T[K];
};

class FilterBuilder<T extends Record<string, unknown>> {
  private conditions: string[] = [];
  private params: unknown[] = [];
  private paramIndex = 1;

  where(field: keyof T, operator: FilterOperator): this {
    const fieldName = String(field);

    if ('eq' in operator) {
      this.conditions.push(`${fieldName} = $${this.paramIndex}`);
      this.params.push(operator.eq);
      this.paramIndex++;
    }
    if ('gt' in operator) {
      this.conditions.push(`${fieldName} > $${this.paramIndex}`);
      this.params.push(operator.gt);
      this.paramIndex++;
    }
    if ('gte' in operator) {
      this.conditions.push(`${fieldName} >= $${this.paramIndex}`);
      this.params.push(operator.gte);
      this.paramIndex++;
    }
    if ('lt' in operator) {
      this.conditions.push(`${fieldName} < $${this.paramIndex}`);
      this.params.push(operator.lt);
      this.paramIndex++;
    }
    if ('lte' in operator) {
      this.conditions.push(`${fieldName} <= $${this.paramIndex}`);
      this.params.push(operator.lte);
      this.paramIndex++;
    }
    if ('in' in operator && Array.isArray(operator.in)) {
      const placeholders = operator.in.map(() => `$${this.paramIndex++}`).join(', ');
      this.conditions.push(`${fieldName} IN (${placeholders})`);
      this.params.push(...operator.in);
    }
    if ('contains' in operator) {
      this.conditions.push(`${fieldName} LIKE $${this.paramIndex}`);
      this.params.push(`%${operator.contains}%`);
      this.paramIndex++;
    }

    return this;
  }

  build(): { where: string; params: unknown[] } {
    return {
      where: this.conditions.length > 0
        ? `WHERE ${this.conditions.join(' AND ')}`
        : '',
      params: this.params
    };
  }
}

// ตัวอย่าง
const filter = new FilterBuilder<User>()
  .where('email', { contains: '@gmail.com' })
  .where('createdAt', { gte: new Date('2024-01-01') })
  .build();
```

### 4.2 Sort Parser

```typescript
interface SortField<T> {
  field: keyof T;
  direction: 'asc' | 'desc';
}

function parseSortParam<T>(
  sortString: string,
  allowedFields: (keyof T)[]
): SortField<T>[] {
  return sortString.split(',').flatMap(part => {
    const trimmed = part.trim();
    const direction: 'asc' | 'desc' = trimmed.startsWith('-') ? 'desc' : 'asc';
    const field = trimmed.replace(/^-/, '') as keyof T;

    if (!allowedFields.includes(field)) {
      return [];
    }

    return [{ field, direction }];
  });
}

// ตัวอย่าง: ?sort=-createdAt,name
// จะได้: [{ field: 'createdAt', direction: 'desc' }, { field: 'name', direction: 'asc' }]

function buildOrderBy<T>(sorts: SortField<T>[]): string {
  if (sorts.length === 0) return '';
  return 'ORDER BY ' + sorts
    .map(s => `${String(s.field)} ${s.direction.toUpperCase()}`)
    .join(', ');
}
```

---

## 5. Field Selection (Sparse Fieldsets)

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

function parseFields(fieldsParam: string): string[] {
  return fieldsParam.split(',').map(f => f.trim()).filter(Boolean);
}

function selectFields<T extends Record<string, unknown>>(
  obj: T,
  fields: string[]
): Partial<T> {
  if (fields.length === 0) return obj;

  return fields.reduce((acc, field) => {
    const keys = field.split('.');
    if (keys.length === 1) {
      if (field in obj) {
        (acc as Record<string, unknown>)[field] = obj[field];
      }
    } else {
      // Handle nested fields like "address.city"
      const [key, ...rest] = keys;
      if (key in obj && typeof obj[key] === 'object') {
        (acc as Record<string, unknown>)[key] = selectFields(
          obj[key] as Record<string, unknown>,
          [rest.join('.')]
        );
      }
    }
    return acc;
  }, {} as Partial<T>);
}

// Express middleware
function fieldSelectionMiddleware(req: Request, res: Response, next: NextFunction) {
  const originalJson = res.json.bind(res);

  res.json = function (body: any) {
    const fieldsParam = req.query.fields as string;
    if (fieldsParam && body?.data) {
      const fields = parseFields(fieldsParam);
      if (Array.isArray(body.data)) {
        body.data = body.data.map((item: any) => selectFields(item, fields));
      } else {
        body.data = selectFields(body.data, fields);
      }
    }
    return originalJson(body);
  };

  next();
}
```

---

## 6. HATEOAS (Hypermedia as the Engine of Application State)

```typescript
interface HateoasLink {
  href: string;
  rel: string;
  method?: string;
  type?: string;
}

interface HateoasResponse<T> {
  data: T;
  _links: Record<string, HateoasLink>;
  _embedded?: Record<string, unknown>;
}

class HateoasBuilder {
  private links: Record<string, HateoasLink> = {};

  self(href: string): this {
    this.links.self = { href, rel: 'self' };
    return this;
  }

  add(rel: string, href: string, method: string = 'GET'): this {
    this.links[rel] = { href, rel, method };
    return this;
  }

  build(): Record<string, HateoasLink> {
    return { ...this.links };
  }
}

function buildUserResponse(user: User, baseUrl: string): HateoasResponse<User> {
  const links = new HateoasBuilder()
    .self(`${baseUrl}/users/${user.id}`)
    .add('update', `${baseUrl}/users/${user.id}`, 'PATCH')
    .add('delete', `${baseUrl}/users/${user.id}`, 'DELETE')
    .add('posts', `${baseUrl}/users/${user.id}/posts`)
    .build();

  return { data: user, _links: links };
}

// ตัวอย่าง Response
// {
//   "data": { "id": "1", "name": "สมชาย", "email": "somchai@example.com" },
//   "_links": {
//     "self": { "href": "/api/users/1", "rel": "self" },
//     "update": { "href": "/api/users/1", "rel": "update", "method": "PATCH" },
//     "posts": { "href": "/api/users/1/posts", "rel": "posts" }
//   }
// }
```

---

## 7. Response Envelope Pattern

```typescript
interface ApiEnvelope<T> {
  success: boolean;
  data?: T;
  error?: ApiError;
  meta?: ApiMeta;
  timestamp: string;
  requestId: string;
}

interface ApiError {
  code: string;
  message: string;
  details?: ValidationErrorDetail[];
  traceId?: string;
}

interface ValidationErrorDetail {
  field: string;
  message: string;
  value?: unknown;
}

interface ApiMeta {
  pagination?: {
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
  };
  version?: string;
  deprecationNotice?: string;
}

class ResponseEnvelope {
  static success<T>(data: T, meta?: ApiMeta, requestId?: string): ApiEnvelope<T> {
    return {
      success: true,
      data,
      meta,
      timestamp: new Date().toISOString(),
      requestId: requestId ?? generateRequestId()
    };
  }

  static error(
    code: string,
    message: string,
    details?: ValidationErrorDetail[],
    requestId?: string
  ): ApiEnvelope<never> {
    return {
      success: false,
      error: { code, message, details },
      timestamp: new Date().toISOString(),
      requestId: requestId ?? generateRequestId()
    };
  }

  static paginated<T>(
    data: T[],
    total: number,
    page: number,
    pageSize: number,
    requestId?: string
  ): ApiEnvelope<T[]> {
    return this.success(data, {
      pagination: {
        total,
        page,
        pageSize,
        totalPages: Math.ceil(total / pageSize)
      }
    }, requestId);
  }
}

function generateRequestId(): string {
  return `req-${Date.now()}-${Math.random().toString(36).substring(2, 9)}`;
}
```

---

## 8. Error Response Standardization

```typescript
// Problem Details (RFC 7807)
interface ProblemDetails {
  type: string;       // URI ที่อธิบาย error
  title: string;      // ชื่อย่อ
  status: number;     // HTTP status code
  detail?: string;    // คำอธิบายละเอียด
  instance?: string;  // URI ของ request ที่เกิด error
  [key: string]: unknown; // Extension fields
}

class ApiError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly code: string,
    message: string,
    public readonly details?: unknown,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = 'ApiError';
  }

  toProblemDetails(instance?: string): ProblemDetails {
    return {
      type: `https://api.example.com/errors/${this.code.toLowerCase()}`,
      title: this.code,
      status: this.statusCode,
      detail: this.message,
      instance,
      ...(this.details ? { details: this.details } : {})
    };
  }
}

class ValidationError extends ApiError {
  constructor(public readonly errors: ValidationErrorDetail[]) {
    super(422, 'VALIDATION_ERROR', 'ข้อมูลไม่ถูกต้อง', errors);
  }
}

class NotFoundError extends ApiError {
  constructor(resource: string, id?: string) {
    super(
      404,
      'NOT_FOUND',
      `ไม่พบ${resource}${id ? ` (id: ${id})` : ''}`
    );
  }
}

class UnauthorizedError extends ApiError {
  constructor(message = 'กรุณาล็อกอินก่อน') {
    super(401, 'UNAUTHORIZED', message);
  }
}

class ForbiddenError extends ApiError {
  constructor(message = 'คุณไม่มีสิทธิ์ดำเนินการนี้') {
    super(403, 'FORBIDDEN', message);
  }
}

// Error Handling Middleware
function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  if (err instanceof ApiError) {
    res.status(err.statusCode).json(
      err.toProblemDetails(req.path)
    );
    return;
  }

  // Unexpected errors
  console.error('Unexpected error:', err);
  res.status(500).json({
    type: 'https://api.example.com/errors/internal-error',
    title: 'INTERNAL_ERROR',
    status: 500,
    detail: process.env.NODE_ENV === 'development' ? err.message : 'เกิดข้อผิดพลาด'
  });
}
```

---

## 9. API Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

// Basic rate limiting
const basicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  limit: 100,
  message: {
    error: 'Too Many Requests',
    message: 'คุณส่ง request มากเกินไป กรุณาลองใหม่ใน 15 นาที'
  },
  standardHeaders: 'draft-7',
  legacyHeaders: false
});

// Tiered rate limiting
function createTieredRateLimiter(tier: 'free' | 'pro' | 'enterprise') {
  const limits = {
    free: { requests: 100, window: 60 * 60 * 1000 },
    pro: { requests: 1000, window: 60 * 60 * 1000 },
    enterprise: { requests: 10000, window: 60 * 60 * 1000 }
  };

  const config = limits[tier];
  return rateLimit({
    windowMs: config.window,
    limit: config.requests,
    keyGenerator: (req: Request) => {
      return req.user?.id ?? req.ip ?? 'anonymous';
    },
    handler: (req: Request, res: Response) => {
      res.status(429).json({
        error: 'RATE_LIMIT_EXCEEDED',
        message: `ถึงขีดจำกัดสำหรับ plan ${tier} แล้ว`,
        retryAfter: res.getHeader('Retry-After')
      });
    }
  });
}

// Custom rate limit per endpoint
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 5, // จำกัด 5 ครั้งต่อ 15 นาทีสำหรับ auth
  message: { error: 'ลองล็อกอินหลายครั้งเกินไป' }
});

app.post('/api/auth/login', authLimiter, loginController);
app.use('/api', basicLimiter);
```

---

## 10. GraphQL Schema Design

### 10.1 Basic Schema

```typescript
import { buildSchema, graphql } from 'graphql';
import { gql } from 'graphql-tag';

// SDL (Schema Definition Language)
const typeDefs = gql`
  scalar DateTime
  scalar JSON

  type User {
    id: ID!
    name: String!
    email: String!
    createdAt: DateTime!
    posts(first: Int, after: String): PostConnection!
    profile: UserProfile
  }

  type UserProfile {
    bio: String
    avatar: String
    location: String
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    tags: [String!]!
    createdAt: DateTime!
    updatedAt: DateTime!
  }

  type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type PostEdge {
    node: Post!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  input CreateUserInput {
    name: String!
    email: String!
    password: String!
    profile: CreateUserProfileInput
  }

  input CreateUserProfileInput {
    bio: String
    avatar: String
    location: String
  }

  input UpdateUserInput {
    name: String
    email: String
    profile: UpdateUserProfileInput
  }

  input UpdateUserProfileInput {
    bio: String
    avatar: String
    location: String
  }

  type Query {
    user(id: ID!): User
    users(
      first: Int = 20
      after: String
      filter: UserFilterInput
      orderBy: UserOrderByInput
    ): UserConnection!
    me: User
  }

  type Mutation {
    createUser(input: CreateUserInput!): CreateUserPayload!
    updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
    deleteUser(id: ID!): DeleteUserPayload!
  }

  type CreateUserPayload {
    user: User
    errors: [UserError!]!
  }

  type UpdateUserPayload {
    user: User
    errors: [UserError!]!
  }

  type DeleteUserPayload {
    deletedId: ID
    errors: [UserError!]!
  }

  type UserError {
    field: [String!]
    message: String!
  }
`;
```

### 10.2 Resolvers

```typescript
interface Context {
  user?: { id: string; role: string };
  dataloaders: DataLoaders;
}

const resolvers = {
  Query: {
    user: async (_: unknown, { id }: { id: string }, ctx: Context) => {
      return userService.findById(id);
    },

    users: async (_: unknown, args: UsersArgs, ctx: Context) => {
      const { first = 20, after, filter, orderBy } = args;
      return userService.findPaginated({ first, after, filter, orderBy });
    },

    me: async (_: unknown, __: unknown, ctx: Context) => {
      if (!ctx.user) throw new Error('กรุณาล็อกอินก่อน');
      return userService.findById(ctx.user.id);
    }
  },

  Mutation: {
    createUser: async (_: unknown, { input }: { input: CreateUserInput }, ctx: Context) => {
      try {
        const user = await userService.create(input);
        return { user, errors: [] };
      } catch (error) {
        return {
          user: null,
          errors: [{ message: (error as Error).message }]
        };
      }
    }
  },

  User: {
    posts: async (parent: User, args: PostsArgs, ctx: Context) => {
      // ใช้ DataLoader เพื่อแก้ N+1 problem
      return ctx.dataloaders.postsByUserId.load(parent.id);
    }
  }
};
```

### 10.3 DataLoader สำหรับแก้ N+1 Problem

```typescript
import DataLoader from 'dataloader';

interface DataLoaders {
  userById: DataLoader<string, User>;
  postsByUserId: DataLoader<string, Post[]>;
}

function createDataLoaders(): DataLoaders {
  return {
    userById: new DataLoader(async (ids: readonly string[]) => {
      const users = await userRepo.findByIds([...ids]);
      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) ?? null);
    }),

    postsByUserId: new DataLoader(async (userIds: readonly string[]) => {
      const posts = await postRepo.findByUserIds([...userIds]);
      const postsByUser = new Map<string, Post[]>();

      for (const post of posts) {
        const existing = postsByUser.get(post.userId) ?? [];
        existing.push(post);
        postsByUser.set(post.userId, existing);
      }

      return userIds.map(id => postsByUser.get(id) ?? []);
    })
  };
}
```

---

## 11. OpenAPI/Swagger Generation

```typescript
import swaggerJsdoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';

const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'API ระบบจัดการผู้ใช้',
      version: '2.0.0',
      description: 'API สำหรับจัดการข้อมูลผู้ใช้และโพสต์',
      contact: {
        name: 'API Support',
        email: 'support@example.com'
      }
    },
    servers: [
      { url: 'https://api.example.com/v2', description: 'Production' },
      { url: 'http://localhost:3000/api/v2', description: 'Development' }
    ],
    components: {
      securitySchemes: {
        BearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        }
      },
      schemas: {
        User: {
          type: 'object',
          required: ['id', 'name', 'email'],
          properties: {
            id: { type: 'string', example: 'usr_abc123' },
            name: { type: 'string', example: 'สมชาย ใจดี' },
            email: { type: 'string', format: 'email', example: 'somchai@example.com' },
            createdAt: { type: 'string', format: 'date-time' }
          }
        },
        Error: {
          type: 'object',
          properties: {
            type: { type: 'string' },
            title: { type: 'string' },
            status: { type: 'integer' },
            detail: { type: 'string' }
          }
        }
      }
    },
    security: [{ BearerAuth: [] }]
  },
  apis: ['./src/routes/**/*.ts']
};

const swaggerSpec = swaggerJsdoc(swaggerOptions);
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));

/**
 * @openapi
 * /users:
 *   get:
 *     summary: ดึงรายการผู้ใช้ทั้งหมด
 *     tags: [Users]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *         description: หน้าที่ต้องการ
 *       - in: query
 *         name: pageSize
 *         schema:
 *           type: integer
 *           default: 20
 *           maximum: 100
 *         description: จำนวนรายการต่อหน้า
 *     responses:
 *       200:
 *         description: สำเร็จ
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 pagination:
 *                   $ref: '#/components/schemas/Pagination'
 *       401:
 *         description: ไม่ได้ล็อกอิน
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
```

---

## 12. API Documentation Best Practices

```typescript
// Type-safe API Client Generation
interface ApiClientConfig {
  baseUrl: string;
  apiKey?: string;
  timeout?: number;
  retries?: number;
}

class ApiClient {
  private baseUrl: string;
  private headers: Record<string, string>;

  constructor(config: ApiClientConfig) {
    this.baseUrl = config.baseUrl;
    this.headers = {
      'Content-Type': 'application/json',
      ...(config.apiKey ? { 'X-API-Key': config.apiKey } : {})
    };
  }

  private async request<T>(
    method: string,
    path: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseUrl}${path}`;
    const response = await fetch(url, {
      method,
      headers: { ...this.headers, ...options.headers },
      ...options
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail ?? `HTTP ${response.status}`);
    }

    return response.json();
  }

  users = {
    list: (params?: UsersListParams) => {
      const query = new URLSearchParams(params as Record<string, string>);
      return this.request<PaginatedResponse<User>>('GET', `/users?${query}`);
    },

    get: (id: string) =>
      this.request<ApiEnvelope<User>>('GET', `/users/${id}`),

    create: (data: CreateUserDTO) =>
      this.request<ApiEnvelope<User>>('POST', '/users', {
        body: JSON.stringify(data)
      }),

    update: (id: string, data: UpdateUserDTO) =>
      this.request<ApiEnvelope<User>>('PATCH', `/users/${id}`, {
        body: JSON.stringify(data)
      }),

    delete: (id: string) =>
      this.request<void>('DELETE', `/users/${id}`)
  };
}

// ตัวอย่างการใช้งาน
const client = new ApiClient({
  baseUrl: 'https://api.example.com/v2',
  apiKey: 'your-api-key'
});

const users = await client.users.list({ page: '1', pageSize: '10' });
const user = await client.users.create({ name: 'สมชาย', email: 'somchai@example.com' });
```

---

## สรุป

ในบทนี้เราได้เรียนรู้รูปแบบการออกแบบ API ที่สำคัญ:

1. **RESTful Design** - Naming conventions, HTTP methods, Status codes
2. **API Versioning** - URL path, Header-based versioning
3. **Pagination** - Offset-based, Cursor-based pagination
4. **Filtering & Sorting** - Filter builder, Sort parser
5. **Field Selection** - Sparse fieldsets สำหรับ optimize responses
6. **HATEOAS** - Hypermedia links ใน responses
7. **Response Envelope** - Standard response format
8. **Error Standardization** - Problem Details (RFC 7807)
9. **Rate Limiting** - Token bucket, Sliding window
10. **GraphQL** - Schema design, Resolvers, DataLoader
11. **OpenAPI/Swagger** - Documentation generation
12. **API Client** - Type-safe client generation
