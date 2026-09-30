# ตอนที่ 80: Real-world Project - SaaS Platform (แพลตฟอร์ม SaaS จริง)

## บทนำ

ในตอนนี้เราจะสร้าง SaaS (Software as a Service) Platform ที่สมบูรณ์ตั้งแต่ต้นจนจบ โดยใช้ TypeScript ทั้งหมด โปรเจกต์นี้จะครอบคลุม:

- Multi-tenant architecture
- User management และ authentication
- Subscription billing ด้วย Stripe
- Organization และ Workspace management
- Feature flags
- Usage limits
- API keys management
- Webhooks
- Email notifications

---

## 1. โครงสร้างโปรเจกต์

```
saas-platform/
├── src/
│   ├── core/
│   │   ├── types.ts
│   │   ├── errors.ts
│   │   ├── result.ts
│   │   └── events.ts
│   ├── domain/
│   │   ├── user/
│   │   │   ├── User.ts
│   │   │   ├── UserRepository.ts
│   │   │   └── UserService.ts
│   │   ├── organization/
│   │   │   ├── Organization.ts
│   │   │   ├── OrgRepository.ts
│   │   │   └── OrgService.ts
│   │   ├── subscription/
│   │   │   ├── Subscription.ts
│   │   │   ├── Plan.ts
│   │   │   └── BillingService.ts
│   │   ├── feature-flags/
│   │   │   └── FeatureFlag.ts
│   │   ├── api-keys/
│   │   │   └── ApiKey.ts
│   │   └── webhooks/
│   │       └── Webhook.ts
│   ├── infrastructure/
│   │   ├── database/
│   │   │   ├── Database.ts
│   │   │   └── migrations/
│   │   ├── email/
│   │   │   └── EmailService.ts
│   │   ├── stripe/
│   │   │   └── StripeClient.ts
│   │   └── cache/
│   │       └── CacheService.ts
│   └── api/
│       ├── middleware/
│       │   ├── auth.ts
│       │   ├── rateLimit.ts
│       │   └── validation.ts
│       └── routes/
│           ├── users.ts
│           ├── organizations.ts
│           ├── subscriptions.ts
│           └── webhooks.ts
├── prisma/
│   └── schema.prisma
├── package.json
└── tsconfig.json
```

---

## 2. Core Types และ Domain Models

### ไฟล์: src/core/types.ts

```typescript
// ============================================
// Core Types สำหรับ SaaS Platform
// ============================================

// Branded Types
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

export type UserId = Brand<string, "UserId">;
export type OrgId = Brand<string, "OrgId">;
export type WorkspaceId = Brand<string, "WorkspaceId">;
export type PlanId = Brand<string, "PlanId">;
export type SubscriptionId = Brand<string, "SubscriptionId">;
export type ApiKeyId = Brand<string, "ApiKeyId">;
export type WebhookId = Brand<string, "WebhookId">;
export type FeatureFlagId = Brand<string, "FeatureFlagId">;
export type InvitationId = Brand<string, "InvitationId">;
export type AuditLogId = Brand<string, "AuditLogId">;

// Type constructors
export const UserId = (id: string): UserId => id as UserId;
export const OrgId = (id: string): OrgId => id as OrgId;
export const WorkspaceId = (id: string): WorkspaceId => id as WorkspaceId;
export const PlanId = (id: string): PlanId => id as PlanId;
export const SubscriptionId = (id: string): SubscriptionId => id as SubscriptionId;
export const ApiKeyId = (id: string): ApiKeyId => id as ApiKeyId;
export const WebhookId = (id: string): WebhookId => id as WebhookId;

// Role types
export type UserRole = "owner" | "admin" | "member" | "viewer";
export type SystemRole = "superadmin" | "support" | "user";

// Permission types
export type Permission =
  | "org:read"
  | "org:write"
  | "org:delete"
  | "member:read"
  | "member:invite"
  | "member:remove"
  | "billing:read"
  | "billing:write"
  | "apikey:read"
  | "apikey:create"
  | "apikey:delete"
  | "webhook:read"
  | "webhook:create"
  | "webhook:delete"
  | "feature:read"
  | "feature:write";

// Role permissions mapping
export const ROLE_PERMISSIONS: Record<UserRole, Permission[]> = {
  owner: [
    "org:read", "org:write", "org:delete",
    "member:read", "member:invite", "member:remove",
    "billing:read", "billing:write",
    "apikey:read", "apikey:create", "apikey:delete",
    "webhook:read", "webhook:create", "webhook:delete",
    "feature:read", "feature:write",
  ],
  admin: [
    "org:read", "org:write",
    "member:read", "member:invite", "member:remove",
    "billing:read",
    "apikey:read", "apikey:create", "apikey:delete",
    "webhook:read", "webhook:create", "webhook:delete",
    "feature:read", "feature:write",
  ],
  member: [
    "org:read",
    "member:read",
    "apikey:read", "apikey:create",
    "webhook:read",
    "feature:read",
  ],
  viewer: [
    "org:read",
    "member:read",
    "feature:read",
  ],
};

// Pagination
export interface PaginationOptions {
  page: number;
  limit: number;
  sortBy?: string;
  sortOrder?: "asc" | "desc";
}

export interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
  hasNextPage: boolean;
  hasPrevPage: boolean;
}

export function createPaginatedResult<T>(
  data: T[],
  total: number,
  options: PaginationOptions
): PaginatedResult<T> {
  const totalPages = Math.ceil(total / options.limit);
  return {
    data,
    total,
    page: options.page,
    limit: options.limit,
    totalPages,
    hasNextPage: options.page < totalPages,
    hasPrevPage: options.page > 1,
  };
}
```

### ไฟล์: src/core/result.ts

```typescript
// Result type สำหรับ error handling
export type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

export function ok<T>(data: T): Result<T> {
  return { success: true, data };
}

export function err<E = Error>(error: E): Result<never, E> {
  return { success: false, error };
}

export function isOk<T, E>(result: Result<T, E>): result is { success: true; data: T } {
  return result.success;
}

export function isErr<T, E>(result: Result<T, E>): result is { success: false; error: E } {
  return !result.success;
}

export async function tryAsync<T>(
  fn: () => Promise<T>
): Promise<Result<T, Error>> {
  try {
    const data = await fn();
    return ok(data);
  } catch (e) {
    return err(e instanceof Error ? e : new Error(String(e)));
  }
}

export function mapResult<T, U, E>(
  result: Result<T, E>,
  fn: (data: T) => U
): Result<U, E> {
  if (result.success) return ok(fn(result.data));
  return result;
}

export async function chainResult<T, U, E>(
  result: Result<T, E>,
  fn: (data: T) => Promise<Result<U, E>>
): Promise<Result<U, E>> {
  if (!result.success) return result;
  return fn(result.data);
}
```

### ไฟล์: src/core/errors.ts

```typescript
// Domain-specific errors
export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 400,
    public readonly context?: Record<string, unknown>
  ) {
    super(message);
    this.name = "AppError";
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`ไม่พบ ${resource} ที่มี ID: ${id}`, "NOT_FOUND", 404, { resource, id });
    this.name = "NotFoundError";
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = "ไม่ได้รับอนุญาต") {
    super(message, "UNAUTHORIZED", 401);
    this.name = "UnauthorizedError";
  }
}

export class ForbiddenError extends AppError {
  constructor(action: string) {
    super(`ไม่มีสิทธิ์ทำ: ${action}`, "FORBIDDEN", 403, { action });
    this.name = "ForbiddenError";
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Record<string, string>
  ) {
    super(message, "VALIDATION_ERROR", 400, { fields });
    this.name = "ValidationError";
  }
}

export class ConflictError extends AppError {
  constructor(resource: string, field: string) {
    super(`${resource} ที่มี ${field} นี้มีอยู่แล้ว`, "CONFLICT", 409, {
      resource,
      field,
    });
    this.name = "ConflictError";
  }
}

export class PaymentError extends AppError {
  constructor(message: string, public readonly stripeCode?: string) {
    super(message, "PAYMENT_ERROR", 402, { stripeCode });
    this.name = "PaymentError";
  }
}

export class UsageLimitError extends AppError {
  constructor(
    resource: string,
    current: number,
    limit: number
  ) {
    super(
      `คุณใช้ ${resource} เกินกว่าที่กำหนด (ใช้ ${current}/${limit})`,
      "USAGE_LIMIT_EXCEEDED",
      429,
      { resource, current, limit }
    );
    this.name = "UsageLimitError";
  }
}
```

---

## 3. Database Schema (Prisma)

### ไฟล์: prisma/schema.prisma

```prisma
// This is your Prisma schema file
// ใช้ PostgreSQL สำหรับ production

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ============================================
// Users
// ============================================
model User {
  id                String    @id @default(cuid())
  email             String    @unique
  emailVerified     Boolean   @default(false)
  emailVerifiedAt   DateTime?
  name              String
  avatarUrl         String?
  passwordHash      String?
  systemRole        String    @default("user")
  isActive          Boolean   @default(true)
  lastLoginAt       DateTime?
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt

  // Relations
  memberships       OrgMember[]
  sessions          Session[]
  ownedOrgs         Organization[]  @relation("OrgOwner")
  apiKeys           ApiKey[]
  auditLogs         AuditLog[]
  notifications     Notification[]

  @@index([email])
}

model Session {
  id          String   @id @default(cuid())
  userId      String
  token       String   @unique
  userAgent   String?
  ipAddress   String?
  expiresAt   DateTime
  createdAt   DateTime @default(now())

  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([token])
}

// ============================================
// Organizations
// ============================================
model Organization {
  id              String    @id @default(cuid())
  name            String
  slug            String    @unique
  description     String?
  logoUrl         String?
  website         String?
  ownerId         String
  planId          String?
  isActive        Boolean   @default(true)
  settings        Json      @default("{}")
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  // Relations
  owner           User      @relation("OrgOwner", fields: [ownerId], references: [id])
  plan            Plan?     @relation(fields: [planId], references: [id])
  members         OrgMember[]
  workspaces      Workspace[]
  subscription    Subscription?
  invitations     Invitation[]
  apiKeys         ApiKey[]
  webhooks        Webhook[]
  featureFlags    OrgFeatureFlag[]
  usageRecords    UsageRecord[]
  auditLogs       AuditLog[]

  @@index([slug])
  @@index([ownerId])
}

model OrgMember {
  id            String   @id @default(cuid())
  orgId         String
  userId        String
  role          String   @default("member")
  joinedAt      DateTime @default(now())
  invitedBy     String?
  updatedAt     DateTime @updatedAt

  organization  Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)
  user          User         @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([orgId, userId])
  @@index([orgId])
  @@index([userId])
}

model Invitation {
  id            String    @id @default(cuid())
  orgId         String
  email         String
  role          String    @default("member")
  token         String    @unique
  invitedBy     String
  expiresAt     DateTime
  acceptedAt    DateTime?
  createdAt     DateTime  @default(now())

  organization  Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)

  @@index([orgId])
  @@index([email])
  @@index([token])
}

// ============================================
// Workspaces
// ============================================
model Workspace {
  id          String   @id @default(cuid())
  orgId       String
  name        String
  slug        String
  description String?
  settings    Json     @default("{}")
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  organization Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)

  @@unique([orgId, slug])
  @@index([orgId])
}

// ============================================
// Plans & Subscriptions
// ============================================
model Plan {
  id              String   @id @default(cuid())
  name            String
  displayName     String
  description     String?
  price           Float    @default(0)
  currency        String   @default("USD")
  interval        String   @default("month")
  stripePriceId   String?  @unique
  features        Json     @default("{}")
  limits          Json     @default("{}")
  isActive        Boolean  @default(true)
  isPublic        Boolean  @default(true)
  sortOrder       Int      @default(0)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  organizations   Organization[]
  subscriptions   Subscription[]
}

model Subscription {
  id                    String    @id @default(cuid())
  orgId                 String    @unique
  planId                String
  stripeSubscriptionId  String?   @unique
  stripeCustomerId      String?
  status                String    @default("active")
  currentPeriodStart    DateTime
  currentPeriodEnd      DateTime
  cancelAtPeriodEnd     Boolean   @default(false)
  canceledAt            DateTime?
  trialStart            DateTime?
  trialEnd              DateTime?
  metadata              Json      @default("{}")
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt

  organization          Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)
  plan                  Plan         @relation(fields: [planId], references: [id])
  invoices              Invoice[]

  @@index([orgId])
  @@index([stripeSubscriptionId])
}

model Invoice {
  id              String   @id @default(cuid())
  subscriptionId  String
  stripeInvoiceId String?  @unique
  amount          Float
  currency        String   @default("USD")
  status          String   @default("pending")
  paidAt          DateTime?
  dueDate         DateTime?
  downloadUrl     String?
  metadata        Json     @default("{}")
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  subscription    Subscription @relation(fields: [subscriptionId], references: [id])

  @@index([subscriptionId])
}

// ============================================
// Usage Tracking
// ============================================
model UsageRecord {
  id          String   @id @default(cuid())
  orgId       String
  resource    String
  quantity    Int      @default(1)
  metadata    Json     @default("{}")
  recordedAt  DateTime @default(now())

  organization Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)

  @@index([orgId, resource])
  @@index([recordedAt])
}

// ============================================
// API Keys
// ============================================
model ApiKey {
  id          String    @id @default(cuid())
  orgId       String
  userId      String
  name        String
  keyHash     String    @unique
  keyPrefix   String
  scopes      String[]
  expiresAt   DateTime?
  lastUsedAt  DateTime?
  usageCount  Int       @default(0)
  isActive    Boolean   @default(true)
  metadata    Json      @default("{}")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  organization Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)
  user         User          @relation(fields: [userId], references: [id])

  @@index([orgId])
  @@index([keyHash])
}

// ============================================
// Webhooks
// ============================================
model Webhook {
  id          String    @id @default(cuid())
  orgId       String
  name        String
  url         String
  secret      String
  events      String[]
  isActive    Boolean   @default(true)
  metadata    Json      @default("{}")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  organization Organization  @relation(fields: [orgId], references: [id], onDelete: Cascade)
  deliveries   WebhookDelivery[]

  @@index([orgId])
}

model WebhookDelivery {
  id            String   @id @default(cuid())
  webhookId     String
  event         String
  payload       Json
  response      String?
  statusCode    Int?
  success       Boolean  @default(false)
  attemptCount  Int      @default(0)
  nextRetryAt   DateTime?
  deliveredAt   DateTime?
  createdAt     DateTime @default(now())

  webhook       Webhook  @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId])
  @@index([createdAt])
}

// ============================================
// Feature Flags
// ============================================
model FeatureFlag {
  id          String   @id @default(cuid())
  key         String   @unique
  name        String
  description String?
  type        String   @default("boolean")
  defaultValue Json    @default("false")
  isActive    Boolean  @default(true)
  metadata    Json     @default("{}")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  orgOverrides OrgFeatureFlag[]
}

model OrgFeatureFlag {
  id            String   @id @default(cuid())
  orgId         String
  featureFlagId String
  value         Json
  reason        String?
  setBy         String?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  organization  Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)
  featureFlag   FeatureFlag  @relation(fields: [featureFlagId], references: [id], onDelete: Cascade)

  @@unique([orgId, featureFlagId])
}

// ============================================
// Notifications
// ============================================
model Notification {
  id          String    @id @default(cuid())
  userId      String
  type        String
  title       String
  message     String
  data        Json      @default("{}")
  readAt      DateTime?
  createdAt   DateTime  @default(now())

  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([readAt])
}

// ============================================
// Audit Logs
// ============================================
model AuditLog {
  id          String   @id @default(cuid())
  orgId       String?
  userId      String?
  action      String
  resource    String
  resourceId  String?
  before      Json?
  after       Json?
  ipAddress   String?
  userAgent   String?
  metadata    Json     @default("{}")
  createdAt   DateTime @default(now())

  organization Organization? @relation(fields: [orgId], references: [id])
  user         User?          @relation(fields: [userId], references: [id])

  @@index([orgId])
  @@index([userId])
  @@index([action])
  @@index([createdAt])
}
```

---

## 4. User Domain

### ไฟล์: src/domain/user/User.ts

```typescript
import { UserId, UserRole, SystemRole } from "../../core/types";

export interface User {
  id: UserId;
  email: string;
  emailVerified: boolean;
  name: string;
  avatarUrl?: string;
  systemRole: SystemRole;
  isActive: boolean;
  lastLoginAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateUserInput {
  email: string;
  name: string;
  password?: string;
  avatarUrl?: string;
}

export interface UpdateUserInput {
  name?: string;
  avatarUrl?: string;
  isActive?: boolean;
}

export interface UserWithOrgs extends User {
  memberships: Array<{
    orgId: string;
    orgName: string;
    role: UserRole;
    joinedAt: Date;
  }>;
}
```

### ไฟล์: src/domain/user/UserService.ts

```typescript
import * as bcrypt from "bcrypt";
import * as jwt from "jsonwebtoken";
import { v4 as uuidv4 } from "uuid";
import { User, CreateUserInput, UpdateUserInput } from "./User";
import { UserId } from "../../core/types";
import { Result, ok, err } from "../../core/result";
import {
  NotFoundError,
  ConflictError,
  UnauthorizedError,
  ValidationError,
} from "../../core/errors";
import { PrismaClient } from "@prisma/client";
import { EmailService } from "../../infrastructure/email/EmailService";
import { AuditService } from "../audit/AuditService";

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

export interface LoginInput {
  email: string;
  password: string;
  ipAddress?: string;
  userAgent?: string;
}

export class UserService {
  private readonly BCRYPT_ROUNDS = 12;
  private readonly JWT_SECRET: string;
  private readonly JWT_EXPIRES_IN = "1h";
  private readonly REFRESH_TOKEN_EXPIRES_IN = "30d";

  constructor(
    private readonly db: PrismaClient,
    private readonly emailService: EmailService,
    private readonly auditService: AuditService
  ) {
    this.JWT_SECRET = process.env.JWT_SECRET ?? "default-secret";
  }

  // ============================================
  // Authentication
  // ============================================

  async register(input: CreateUserInput): Promise<Result<User>> {
    // Check email uniqueness
    const existing = await this.db.user.findUnique({
      where: { email: input.email.toLowerCase() },
    });

    if (existing) {
      return err(new ConflictError("User", "email"));
    }

    // Validate
    const validationErrors: Record<string, string> = {};
    if (!input.email.includes("@")) {
      validationErrors.email = "รูปแบบ email ไม่ถูกต้อง";
    }
    if (!input.name || input.name.length < 2) {
      validationErrors.name = "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร";
    }
    if (input.password && input.password.length < 8) {
      validationErrors.password = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร";
    }

    if (Object.keys(validationErrors).length > 0) {
      return err(new ValidationError("ข้อมูลไม่ถูกต้อง", validationErrors));
    }

    // Hash password
    const passwordHash = input.password
      ? await bcrypt.hash(input.password, this.BCRYPT_ROUNDS)
      : undefined;

    // Create user
    const user = await this.db.user.create({
      data: {
        email: input.email.toLowerCase(),
        name: input.name,
        avatarUrl: input.avatarUrl,
        passwordHash,
        emailVerified: false,
      },
    });

    // Send verification email
    await this.sendVerificationEmail(user.id, user.email);

    // Audit log
    await this.auditService.log({
      userId: user.id,
      action: "user.register",
      resource: "user",
      resourceId: user.id,
      after: { email: user.email, name: user.name },
    });

    return ok(this.mapToUser(user));
  }

  async login(input: LoginInput): Promise<Result<{ user: User; tokens: AuthTokens }>> {
    const user = await this.db.user.findUnique({
      where: { email: input.email.toLowerCase() },
    });

    if (!user || !user.isActive) {
      return err(new UnauthorizedError("อีเมลหรือรหัสผ่านไม่ถูกต้อง"));
    }

    if (!user.passwordHash) {
      return err(new UnauthorizedError("บัญชีนี้ใช้การเข้าสู่ระบบผ่าน OAuth"));
    }

    const passwordValid = await bcrypt.compare(input.password, user.passwordHash);
    if (!passwordValid) {
      return err(new UnauthorizedError("อีเมลหรือรหัสผ่านไม่ถูกต้อง"));
    }

    // Create session
    const tokens = this.generateTokens(user.id);
    await this.db.session.create({
      data: {
        userId: user.id,
        token: tokens.refreshToken,
        userAgent: input.userAgent,
        ipAddress: input.ipAddress,
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      },
    });

    // Update last login
    await this.db.user.update({
      where: { id: user.id },
      data: { lastLoginAt: new Date() },
    });

    await this.auditService.log({
      userId: user.id,
      action: "user.login",
      resource: "user",
      resourceId: user.id,
      metadata: {
        ipAddress: input.ipAddress,
        userAgent: input.userAgent,
      },
    });

    return ok({
      user: this.mapToUser(user),
      tokens,
    });
  }

  async refreshToken(refreshToken: string): Promise<Result<AuthTokens>> {
    const session = await this.db.session.findUnique({
      where: { token: refreshToken },
      include: { user: true },
    });

    if (!session || session.expiresAt < new Date()) {
      return err(new UnauthorizedError("Refresh token ไม่ถูกต้องหรือหมดอายุ"));
    }

    if (!session.user.isActive) {
      return err(new UnauthorizedError("บัญชีถูกปิดใช้งาน"));
    }

    // Rotate refresh token
    const newTokens = this.generateTokens(session.userId);

    await this.db.session.update({
      where: { id: session.id },
      data: {
        token: newTokens.refreshToken,
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      },
    });

    return ok(newTokens);
  }

  async logout(userId: string, refreshToken: string): Promise<void> {
    await this.db.session.deleteMany({
      where: {
        userId,
        token: refreshToken,
      },
    });
  }

  async verifyEmail(token: string): Promise<Result<void>> {
    // In practice, token would be stored and verified
    const user = await this.db.user.findFirst({
      where: {
        emailVerified: false,
        // In real implementation: emailVerificationToken: token
      },
    });

    if (!user) {
      return err(new Error("Token ไม่ถูกต้องหรือหมดอายุ"));
    }

    await this.db.user.update({
      where: { id: user.id },
      data: {
        emailVerified: true,
        emailVerifiedAt: new Date(),
      },
    });

    return ok(undefined);
  }

  async forgotPassword(email: string): Promise<Result<void>> {
    const user = await this.db.user.findUnique({
      where: { email: email.toLowerCase() },
    });

    // Don't reveal if user exists
    if (!user) return ok(undefined);

    const resetToken = uuidv4();
    // Store reset token (in real implementation)

    await this.emailService.send({
      to: email,
      subject: "รีเซ็ตรหัสผ่าน",
      template: "password-reset",
      data: {
        name: user.name,
        resetUrl: `${process.env.APP_URL}/reset-password?token=${resetToken}`,
        expiresIn: "1 ชั่วโมง",
      },
    });

    return ok(undefined);
  }

  async resetPassword(
    token: string,
    newPassword: string
  ): Promise<Result<void>> {
    if (newPassword.length < 8) {
      return err(new ValidationError("ข้อมูลไม่ถูกต้อง", {
        password: "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร",
      }));
    }

    // In real implementation: verify token and find user
    const userId = "user-from-token";

    const passwordHash = await bcrypt.hash(newPassword, this.BCRYPT_ROUNDS);
    await this.db.user.update({
      where: { id: userId },
      data: { passwordHash },
    });

    // Invalidate all sessions
    await this.db.session.deleteMany({ where: { userId } });

    return ok(undefined);
  }

  // ============================================
  // User Management
  // ============================================

  async getUser(userId: string): Promise<Result<User>> {
    const user = await this.db.user.findUnique({
      where: { id: userId },
    });

    if (!user) {
      return err(new NotFoundError("User", userId));
    }

    return ok(this.mapToUser(user));
  }

  async updateUser(
    userId: string,
    input: UpdateUserInput,
    requesterId: string
  ): Promise<Result<User>> {
    const user = await this.db.user.findUnique({ where: { id: userId } });

    if (!user) {
      return err(new NotFoundError("User", userId));
    }

    const updated = await this.db.user.update({
      where: { id: userId },
      data: {
        ...input,
        updatedAt: new Date(),
      },
    });

    await this.auditService.log({
      userId: requesterId,
      action: "user.update",
      resource: "user",
      resourceId: userId,
      before: { name: user.name, avatarUrl: user.avatarUrl },
      after: input,
    });

    return ok(this.mapToUser(updated));
  }

  async changePassword(
    userId: string,
    currentPassword: string,
    newPassword: string
  ): Promise<Result<void>> {
    const user = await this.db.user.findUnique({ where: { id: userId } });

    if (!user?.passwordHash) {
      return err(new Error("ไม่พบรหัสผ่าน"));
    }

    const valid = await bcrypt.compare(currentPassword, user.passwordHash);
    if (!valid) {
      return err(new UnauthorizedError("รหัสผ่านปัจจุบันไม่ถูกต้อง"));
    }

    if (newPassword.length < 8) {
      return err(new ValidationError("ข้อมูลไม่ถูกต้อง", {
        newPassword: "รหัสผ่านใหม่ต้องมีอย่างน้อย 8 ตัวอักษร",
      }));
    }

    const newHash = await bcrypt.hash(newPassword, this.BCRYPT_ROUNDS);
    await this.db.user.update({
      where: { id: userId },
      data: { passwordHash: newHash },
    });

    return ok(undefined);
  }

  // ============================================
  // Private helpers
  // ============================================

  private generateTokens(userId: string): AuthTokens {
    const accessToken = jwt.sign({ sub: userId }, this.JWT_SECRET, {
      expiresIn: this.JWT_EXPIRES_IN,
    });

    const refreshToken = jwt.sign(
      { sub: userId, type: "refresh" },
      this.JWT_SECRET,
      { expiresIn: this.REFRESH_TOKEN_EXPIRES_IN }
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: 3600,
    };
  }

  verifyAccessToken(token: string): { userId: string } | null {
    try {
      const payload = jwt.verify(token, this.JWT_SECRET) as { sub: string };
      return { userId: payload.sub };
    } catch {
      return null;
    }
  }

  private async sendVerificationEmail(
    userId: string,
    email: string
  ): Promise<void> {
    const token = uuidv4();
    await this.emailService.send({
      to: email,
      subject: "ยืนยันอีเมลของคุณ",
      template: "email-verification",
      data: {
        verifyUrl: `${process.env.APP_URL}/verify-email?token=${token}`,
        expiresIn: "24 ชั่วโมง",
      },
    });
  }

  private mapToUser(user: any): User {
    return {
      id: UserId(user.id),
      email: user.email,
      emailVerified: user.emailVerified,
      name: user.name,
      avatarUrl: user.avatarUrl ?? undefined,
      systemRole: user.systemRole as any,
      isActive: user.isActive,
      lastLoginAt: user.lastLoginAt ?? undefined,
      createdAt: user.createdAt,
      updatedAt: user.updatedAt,
    };
  }
}
```

---

## 5. Organization Domain

### ไฟล์: src/domain/organization/OrgService.ts

```typescript
import { PrismaClient } from "@prisma/client";
import { Organization, CreateOrgInput, UpdateOrgInput } from "./Organization";
import { OrgId, UserId, UserRole, Permission, ROLE_PERMISSIONS } from "../../core/types";
import { Result, ok, err } from "../../core/result";
import {
  NotFoundError,
  ConflictError,
  ForbiddenError,
  UsageLimitError,
} from "../../core/errors";
import { EmailService } from "../../infrastructure/email/EmailService";
import { PlanService } from "../subscription/PlanService";
import { AuditService } from "../audit/AuditService";

export class OrgService {
  constructor(
    private readonly db: PrismaClient,
    private readonly emailService: EmailService,
    private readonly planService: PlanService,
    private readonly auditService: AuditService
  ) {}

  // ============================================
  // Organization CRUD
  // ============================================

  async createOrg(
    ownerId: string,
    input: CreateOrgInput
  ): Promise<Result<Organization>> {
    // Check slug uniqueness
    const existing = await this.db.organization.findUnique({
      where: { slug: input.slug },
    });

    if (existing) {
      return err(new ConflictError("Organization", "slug"));
    }

    const org = await this.db.organization.create({
      data: {
        name: input.name,
        slug: input.slug,
        description: input.description,
        ownerId,
      },
    });

    // Add owner as member
    await this.db.orgMember.create({
      data: {
        orgId: org.id,
        userId: ownerId,
        role: "owner",
      },
    });

    // Assign free plan
    await this.planService.assignFreePlan(org.id);

    await this.auditService.log({
      userId: ownerId,
      orgId: org.id,
      action: "org.create",
      resource: "organization",
      resourceId: org.id,
      after: { name: org.name, slug: org.slug },
    });

    return ok(this.mapToOrg(org));
  }

  async updateOrg(
    orgId: string,
    requesterId: string,
    input: UpdateOrgInput
  ): Promise<Result<Organization>> {
    const canEdit = await this.hasPermission(requesterId, orgId, "org:write");
    if (!canEdit) {
      return err(new ForbiddenError("แก้ไของค์กร"));
    }

    const org = await this.db.organization.update({
      where: { id: orgId },
      data: input,
    });

    return ok(this.mapToOrg(org));
  }

  async deleteOrg(
    orgId: string,
    requesterId: string
  ): Promise<Result<void>> {
    const canDelete = await this.hasPermission(requesterId, orgId, "org:delete");
    if (!canDelete) {
      return err(new ForbiddenError("ลบองค์กร"));
    }

    await this.db.organization.update({
      where: { id: orgId },
      data: { isActive: false },
    });

    return ok(undefined);
  }

  // ============================================
  // Member Management
  // ============================================

  async inviteMember(
    orgId: string,
    requesterId: string,
    email: string,
    role: UserRole
  ): Promise<Result<void>> {
    const canInvite = await this.hasPermission(requesterId, orgId, "member:invite");
    if (!canInvite) {
      return err(new ForbiddenError("เชิญสมาชิก"));
    }

    // Check plan limits
    const memberCount = await this.db.orgMember.count({ where: { orgId } });
    const limit = await this.planService.getMemberLimit(orgId);

    if (memberCount >= limit) {
      return err(new UsageLimitError("สมาชิก", memberCount, limit));
    }

    // Check if already member
    const user = await this.db.user.findUnique({ where: { email } });
    if (user) {
      const existingMember = await this.db.orgMember.findUnique({
        where: { orgId_userId: { orgId, userId: user.id } },
      });
      if (existingMember) {
        return err(new ConflictError("Member", "email"));
      }
    }

    // Create invitation
    const token = `inv_${Math.random().toString(36).substr(2, 32)}`;
    const org = await this.db.organization.findUnique({ where: { id: orgId } });
    const inviter = await this.db.user.findUnique({ where: { id: requesterId } });

    await this.db.invitation.create({
      data: {
        orgId,
        email,
        role,
        token,
        invitedBy: requesterId,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      },
    });

    // Send invitation email
    await this.emailService.send({
      to: email,
      subject: `คุณได้รับคำเชิญให้เข้าร่วม ${org?.name}`,
      template: "org-invitation",
      data: {
        orgName: org?.name,
        inviterName: inviter?.name,
        role,
        acceptUrl: `${process.env.APP_URL}/invitations/accept?token=${token}`,
        expiresIn: "7 วัน",
      },
    });

    return ok(undefined);
  }

  async acceptInvitation(
    token: string,
    userId: string
  ): Promise<Result<Organization>> {
    const invitation = await this.db.invitation.findUnique({
      where: { token },
      include: { organization: true },
    });

    if (!invitation || invitation.expiresAt < new Date()) {
      return err(new Error("คำเชิญไม่ถูกต้องหรือหมดอายุ"));
    }

    if (invitation.acceptedAt) {
      return err(new Error("คำเชิญนี้ถูกใช้งานแล้ว"));
    }

    // Add member
    await this.db.orgMember.create({
      data: {
        orgId: invitation.orgId,
        userId,
        role: invitation.role,
        invitedBy: invitation.invitedBy,
      },
    });

    // Mark invitation as accepted
    await this.db.invitation.update({
      where: { id: invitation.id },
      data: { acceptedAt: new Date() },
    });

    return ok(this.mapToOrg(invitation.organization));
  }

  async removeMember(
    orgId: string,
    requesterId: string,
    targetUserId: string
  ): Promise<Result<void>> {
    const canRemove = await this.hasPermission(requesterId, orgId, "member:remove");
    if (!canRemove) {
      return err(new ForbiddenError("ลบสมาชิก"));
    }

    // Cannot remove owner
    const targetMember = await this.db.orgMember.findUnique({
      where: { orgId_userId: { orgId, userId: targetUserId } },
    });

    if (!targetMember) {
      return err(new NotFoundError("Member", targetUserId));
    }

    if (targetMember.role === "owner") {
      return err(new ForbiddenError("ลบ owner ขององค์กร"));
    }

    await this.db.orgMember.delete({
      where: { orgId_userId: { orgId, userId: targetUserId } },
    });

    return ok(undefined);
  }

  async updateMemberRole(
    orgId: string,
    requesterId: string,
    targetUserId: string,
    newRole: UserRole
  ): Promise<Result<void>> {
    const canManage = await this.hasPermission(requesterId, orgId, "member:invite");
    if (!canManage) {
      return err(new ForbiddenError("เปลี่ยน role สมาชิก"));
    }

    await this.db.orgMember.update({
      where: { orgId_userId: { orgId, userId: targetUserId } },
      data: { role: newRole },
    });

    return ok(undefined);
  }

  // ============================================
  // Permission Checking
  // ============================================

  async hasPermission(
    userId: string,
    orgId: string,
    permission: Permission
  ): Promise<boolean> {
    const member = await this.db.orgMember.findUnique({
      where: { orgId_userId: { orgId, userId } },
    });

    if (!member) return false;

    const permissions = ROLE_PERMISSIONS[member.role as UserRole] ?? [];
    return permissions.includes(permission);
  }

  async getMemberRole(
    userId: string,
    orgId: string
  ): Promise<UserRole | null> {
    const member = await this.db.orgMember.findUnique({
      where: { orgId_userId: { orgId, userId } },
    });

    return (member?.role as UserRole) ?? null;
  }

  // ============================================
  // Private helpers
  // ============================================

  private mapToOrg(org: any): Organization {
    return {
      id: OrgId(org.id),
      name: org.name,
      slug: org.slug,
      description: org.description ?? undefined,
      logoUrl: org.logoUrl ?? undefined,
      ownerId: UserId(org.ownerId),
      isActive: org.isActive,
      settings: org.settings,
      createdAt: org.createdAt,
      updatedAt: org.updatedAt,
    };
  }
}
```

---

## 6. Subscription & Billing

### ไฟล์: src/domain/subscription/BillingService.ts

```typescript
import Stripe from "stripe";
import { PrismaClient } from "@prisma/client";
import { Result, ok, err } from "../../core/result";
import { PaymentError, NotFoundError } from "../../core/errors";
import { EmailService } from "../../infrastructure/email/EmailService";
import { AuditService } from "../audit/AuditService";

export interface PlanDetails {
  id: string;
  name: string;
  displayName: string;
  price: number;
  currency: string;
  interval: "month" | "year";
  features: Record<string, boolean | number | string>;
  limits: {
    members: number;
    workspaces: number;
    apiKeys: number;
    webhooks: number;
    storageGB: number;
    requestsPerMonth: number;
  };
}

export interface SubscriptionDetails {
  id: string;
  orgId: string;
  planId: string;
  status: "active" | "inactive" | "past_due" | "cancelled" | "trialing";
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  cancelAtPeriodEnd: boolean;
  trialEnd?: Date;
}

export class BillingService {
  private stripe: Stripe;

  constructor(
    private readonly db: PrismaClient,
    private readonly emailService: EmailService,
    private readonly auditService: AuditService
  ) {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY ?? "", {
      apiVersion: "2023-10-16",
    });
  }

  // ============================================
  // Plan Management
  // ============================================

  async getPlans(): Promise<PlanDetails[]> {
    const plans = await this.db.plan.findMany({
      where: { isActive: true, isPublic: true },
      orderBy: { sortOrder: "asc" },
    });

    return plans.map((p) => ({
      id: p.id,
      name: p.name,
      displayName: p.displayName,
      price: p.price,
      currency: p.currency,
      interval: p.interval as "month" | "year",
      features: p.features as Record<string, boolean | number | string>,
      limits: p.limits as any,
    }));
  }

  async getPlan(planId: string): Promise<Result<PlanDetails>> {
    const plan = await this.db.plan.findUnique({ where: { id: planId } });

    if (!plan) {
      return err(new NotFoundError("Plan", planId));
    }

    return ok({
      id: plan.id,
      name: plan.name,
      displayName: plan.displayName,
      price: plan.price,
      currency: plan.currency,
      interval: plan.interval as "month" | "year",
      features: plan.features as Record<string, boolean | number | string>,
      limits: plan.limits as any,
    });
  }

  // ============================================
  // Subscription Management
  // ============================================

  async subscribe(
    orgId: string,
    planId: string,
    paymentMethodId: string
  ): Promise<Result<SubscriptionDetails>> {
    const [org, plan] = await Promise.all([
      this.db.organization.findUnique({
        where: { id: orgId },
        include: { subscription: true },
      }),
      this.db.plan.findUnique({ where: { id: planId } }),
    ]);

    if (!org) return err(new NotFoundError("Organization", orgId));
    if (!plan) return err(new NotFoundError("Plan", planId));

    try {
      let customerId = org.subscription?.stripeCustomerId;

      // Create Stripe customer if not exists
      if (!customerId) {
        const owner = await this.db.user.findUnique({
          where: { id: org.ownerId },
        });

        const customer = await this.stripe.customers.create({
          email: owner?.email,
          name: org.name,
          metadata: { orgId },
        });

        customerId = customer.id;
      }

      // Attach payment method
      await this.stripe.paymentMethods.attach(paymentMethodId, {
        customer: customerId,
      });

      // Create subscription
      const stripeSubscription = await this.stripe.subscriptions.create({
        customer: customerId,
        items: [{ price: plan.stripePriceId! }],
        payment_settings: {
          payment_method_types: ["card"],
          save_default_payment_method: "on_subscription",
        },
        expand: ["latest_invoice.payment_intent"],
      });

      // Save subscription
      const subscription = await this.db.subscription.upsert({
        where: { orgId },
        create: {
          orgId,
          planId,
          stripeSubscriptionId: stripeSubscription.id,
          stripeCustomerId: customerId,
          status: stripeSubscription.status,
          currentPeriodStart: new Date(stripeSubscription.current_period_start * 1000),
          currentPeriodEnd: new Date(stripeSubscription.current_period_end * 1000),
        },
        update: {
          planId,
          stripeSubscriptionId: stripeSubscription.id,
          stripeCustomerId: customerId,
          status: stripeSubscription.status,
          currentPeriodStart: new Date(stripeSubscription.current_period_start * 1000),
          currentPeriodEnd: new Date(stripeSubscription.current_period_end * 1000),
        },
      });

      // Update org plan
      await this.db.organization.update({
        where: { id: orgId },
        data: { planId },
      });

      // Send confirmation email
      const owner = await this.db.user.findUnique({
        where: { id: org.ownerId },
      });

      if (owner) {
        await this.emailService.send({
          to: owner.email,
          subject: `ยืนยันการสมัคร ${plan.displayName}`,
          template: "subscription-confirmed",
          data: {
            name: owner.name,
            planName: plan.displayName,
            price: `${plan.price} ${plan.currency.toUpperCase()}/${plan.interval === "month" ? "เดือน" : "ปี"}`,
            nextBillingDate: new Date(stripeSubscription.current_period_end * 1000).toLocaleDateString("th-TH"),
          },
        });
      }

      return ok(this.mapToSubscription(subscription));
    } catch (error) {
      if (error instanceof Stripe.errors.StripeError) {
        return err(new PaymentError(error.message, error.code ?? undefined));
      }
      throw error;
    }
  }

  async cancelSubscription(
    orgId: string,
    immediately = false
  ): Promise<Result<void>> {
    const subscription = await this.db.subscription.findUnique({
      where: { orgId },
    });

    if (!subscription?.stripeSubscriptionId) {
      return err(new NotFoundError("Subscription", orgId));
    }

    try {
      if (immediately) {
        await this.stripe.subscriptions.cancel(subscription.stripeSubscriptionId);
        await this.db.subscription.update({
          where: { orgId },
          data: { status: "cancelled", canceledAt: new Date() },
        });
      } else {
        await this.stripe.subscriptions.update(
          subscription.stripeSubscriptionId,
          { cancel_at_period_end: true }
        );
        await this.db.subscription.update({
          where: { orgId },
          data: { cancelAtPeriodEnd: true },
        });
      }

      return ok(undefined);
    } catch (error) {
      if (error instanceof Stripe.errors.StripeError) {
        return err(new PaymentError(error.message));
      }
      throw error;
    }
  }

  async handleStripeWebhook(
    payload: string,
    signature: string
  ): Promise<Result<void>> {
    let event: Stripe.Event;

    try {
      event = this.stripe.webhooks.constructEvent(
        payload,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET ?? ""
      );
    } catch (error) {
      return err(new Error("Stripe webhook signature ไม่ถูกต้อง"));
    }

    switch (event.type) {
      case "customer.subscription.updated": {
        const subscription = event.data.object as Stripe.Subscription;
        await this.db.subscription.updateMany({
          where: { stripeSubscriptionId: subscription.id },
          data: {
            status: subscription.status,
            currentPeriodStart: new Date(subscription.current_period_start * 1000),
            currentPeriodEnd: new Date(subscription.current_period_end * 1000),
            cancelAtPeriodEnd: subscription.cancel_at_period_end,
          },
        });
        break;
      }

      case "customer.subscription.deleted": {
        const subscription = event.data.object as Stripe.Subscription;
        await this.db.subscription.updateMany({
          where: { stripeSubscriptionId: subscription.id },
          data: {
            status: "cancelled",
            canceledAt: new Date(),
          },
        });
        break;
      }

      case "invoice.payment_succeeded": {
        const invoice = event.data.object as Stripe.Invoice;
        if (invoice.subscription) {
          await this.db.invoice.create({
            data: {
              subscriptionId: (
                await this.db.subscription.findUnique({
                  where: { stripeSubscriptionId: invoice.subscription as string },
                })
              )?.id ?? "",
              stripeInvoiceId: invoice.id,
              amount: invoice.amount_paid / 100,
              currency: invoice.currency.toUpperCase(),
              status: "paid",
              paidAt: new Date(),
              downloadUrl: invoice.hosted_invoice_url ?? undefined,
            },
          });
        }
        break;
      }

      case "invoice.payment_failed": {
        const invoice = event.data.object as Stripe.Invoice;
        const sub = await this.db.subscription.findUnique({
          where: { stripeSubscriptionId: invoice.subscription as string },
          include: { organization: { include: { owner: true } } },
        });

        if (sub?.organization.owner.email) {
          await this.emailService.send({
            to: sub.organization.owner.email,
            subject: "การชำระเงินล้มเหลว",
            template: "payment-failed",
            data: {
              name: sub.organization.owner.name,
              amount: `${invoice.amount_due / 100} ${invoice.currency.toUpperCase()}`,
              updatePaymentUrl: `${process.env.APP_URL}/billing`,
            },
          });
        }
        break;
      }
    }

    return ok(undefined);
  }

  // ============================================
  // Usage Limits
  // ============================================

  async checkUsageLimit(
    orgId: string,
    resource: string
  ): Promise<{ allowed: boolean; current: number; limit: number }> {
    const org = await this.db.organization.findUnique({
      where: { id: orgId },
      include: { plan: true },
    });

    if (!org?.plan) {
      return { allowed: true, current: 0, limit: Infinity };
    }

    const limits = org.plan.limits as Record<string, number>;
    const limit = limits[resource] ?? Infinity;

    let current = 0;

    switch (resource) {
      case "members":
        current = await this.db.orgMember.count({ where: { orgId } });
        break;
      case "workspaces":
        current = await this.db.workspace.count({ where: { orgId } });
        break;
      case "apiKeys":
        current = await this.db.apiKey.count({ where: { orgId, isActive: true } });
        break;
      case "webhooks":
        current = await this.db.webhook.count({ where: { orgId, isActive: true } });
        break;
      default:
        // Check usage records for other resources
        const startOfMonth = new Date();
        startOfMonth.setDate(1);
        startOfMonth.setHours(0, 0, 0, 0);

        const usage = await this.db.usageRecord.aggregate({
          where: {
            orgId,
            resource,
            recordedAt: { gte: startOfMonth },
          },
          _sum: { quantity: true },
        });

        current = usage._sum.quantity ?? 0;
    }

    return {
      allowed: current < limit,
      current,
      limit,
    };
  }

  private mapToSubscription(sub: any): SubscriptionDetails {
    return {
      id: sub.id,
      orgId: sub.orgId,
      planId: sub.planId,
      status: sub.status,
      currentPeriodStart: sub.currentPeriodStart,
      currentPeriodEnd: sub.currentPeriodEnd,
      cancelAtPeriodEnd: sub.cancelAtPeriodEnd,
      trialEnd: sub.trialEnd ?? undefined,
    };
  }
}
```

---

## 7. API Keys Management

### ไฟล์: src/domain/api-keys/ApiKeyService.ts

```typescript
import * as crypto from "crypto";
import { PrismaClient } from "@prisma/client";
import { Result, ok, err } from "../../core/result";
import { NotFoundError, ForbiddenError } from "../../core/errors";

export interface ApiKey {
  id: string;
  orgId: string;
  userId: string;
  name: string;
  keyPrefix: string;
  scopes: string[];
  expiresAt?: Date;
  lastUsedAt?: Date;
  usageCount: number;
  isActive: boolean;
  createdAt: Date;
}

export interface CreateApiKeyInput {
  name: string;
  scopes: string[];
  expiresAt?: Date;
}

export interface ApiKeyWithSecret extends ApiKey {
  secret: string; // แสดงครั้งเดียวเมื่อสร้าง
}

export const AVAILABLE_SCOPES = [
  "read:users",
  "write:users",
  "read:org",
  "write:org",
  "read:data",
  "write:data",
  "read:analytics",
  "webhooks:write",
] as const;

export type ApiKeyScope = (typeof AVAILABLE_SCOPES)[number];

export class ApiKeyService {
  constructor(private readonly db: PrismaClient) {}

  async createApiKey(
    orgId: string,
    userId: string,
    input: CreateApiKeyInput
  ): Promise<Result<ApiKeyWithSecret>> {
    // Validate scopes
    const invalidScopes = input.scopes.filter(
      (s) => !AVAILABLE_SCOPES.includes(s as ApiKeyScope)
    );

    if (invalidScopes.length > 0) {
      return err(new Error(`Scopes ไม่ถูกต้อง: ${invalidScopes.join(", ")}`));
    }

    // Generate API key
    const rawKey = `sk_live_${crypto.randomBytes(32).toString("hex")}`;
    const prefix = rawKey.substring(0, 15);
    const keyHash = crypto.createHash("sha256").update(rawKey).digest("hex");

    const apiKey = await this.db.apiKey.create({
      data: {
        orgId,
        userId,
        name: input.name,
        keyHash,
        keyPrefix: prefix,
        scopes: input.scopes,
        expiresAt: input.expiresAt,
      },
    });

    return ok({
      id: apiKey.id,
      orgId: apiKey.orgId,
      userId: apiKey.userId,
      name: apiKey.name,
      keyPrefix: apiKey.keyPrefix,
      scopes: apiKey.scopes,
      expiresAt: apiKey.expiresAt ?? undefined,
      lastUsedAt: apiKey.lastUsedAt ?? undefined,
      usageCount: apiKey.usageCount,
      isActive: apiKey.isActive,
      createdAt: apiKey.createdAt,
      secret: rawKey, // แสดงครั้งเดียว!
    });
  }

  async validateApiKey(
    rawKey: string
  ): Promise<{ valid: boolean; orgId?: string; scopes?: string[] }> {
    const keyHash = crypto.createHash("sha256").update(rawKey).digest("hex");

    const apiKey = await this.db.apiKey.findUnique({
      where: { keyHash },
    });

    if (!apiKey || !apiKey.isActive) {
      return { valid: false };
    }

    if (apiKey.expiresAt && apiKey.expiresAt < new Date()) {
      return { valid: false };
    }

    // Update usage stats
    await this.db.apiKey.update({
      where: { id: apiKey.id },
      data: {
        lastUsedAt: new Date(),
        usageCount: { increment: 1 },
      },
    });

    return {
      valid: true,
      orgId: apiKey.orgId,
      scopes: apiKey.scopes,
    };
  }

  async listApiKeys(orgId: string): Promise<ApiKey[]> {
    const keys = await this.db.apiKey.findMany({
      where: { orgId },
      orderBy: { createdAt: "desc" },
    });

    return keys.map((k) => ({
      id: k.id,
      orgId: k.orgId,
      userId: k.userId,
      name: k.name,
      keyPrefix: k.keyPrefix,
      scopes: k.scopes,
      expiresAt: k.expiresAt ?? undefined,
      lastUsedAt: k.lastUsedAt ?? undefined,
      usageCount: k.usageCount,
      isActive: k.isActive,
      createdAt: k.createdAt,
    }));
  }

  async revokeApiKey(
    keyId: string,
    orgId: string
  ): Promise<Result<void>> {
    const key = await this.db.apiKey.findFirst({
      where: { id: keyId, orgId },
    });

    if (!key) {
      return err(new NotFoundError("API Key", keyId));
    }

    await this.db.apiKey.update({
      where: { id: keyId },
      data: { isActive: false },
    });

    return ok(undefined);
  }

  async rotateApiKey(
    keyId: string,
    orgId: string
  ): Promise<Result<ApiKeyWithSecret>> {
    const key = await this.db.apiKey.findFirst({
      where: { id: keyId, orgId },
    });

    if (!key) {
      return err(new NotFoundError("API Key", keyId));
    }

    // Revoke old key
    await this.db.apiKey.update({
      where: { id: keyId },
      data: { isActive: false },
    });

    // Create new key
    return this.createApiKey(orgId, key.userId, {
      name: `${key.name} (rotated)`,
      scopes: key.scopes,
      expiresAt: key.expiresAt ?? undefined,
    });
  }
}
```

---

## 8. Webhooks System

### ไฟล์: src/domain/webhooks/WebhookService.ts

```typescript
import * as crypto from "crypto";
import { PrismaClient } from "@prisma/client";
import { Result, ok, err } from "../../core/result";
import { NotFoundError } from "../../core/errors";

export type WebhookEventType =
  | "user.created"
  | "user.updated"
  | "user.deleted"
  | "member.added"
  | "member.removed"
  | "subscription.created"
  | "subscription.updated"
  | "subscription.cancelled"
  | "invoice.created"
  | "invoice.paid";

export interface WebhookPayload {
  id: string;
  type: WebhookEventType;
  orgId: string;
  data: Record<string, unknown>;
  timestamp: string;
  version: "1.0";
}

export interface CreateWebhookInput {
  name: string;
  url: string;
  events: WebhookEventType[];
}

export class WebhookService {
  constructor(private readonly db: PrismaClient) {}

  async createWebhook(
    orgId: string,
    input: CreateWebhookInput
  ): Promise<Result<{ id: string; secret: string }>> {
    // Validate URL
    try {
      new URL(input.url);
    } catch {
      return err(new Error("URL ไม่ถูกต้อง"));
    }

    const secret = `whsec_${crypto.randomBytes(32).toString("hex")}`;

    const webhook = await this.db.webhook.create({
      data: {
        orgId,
        name: input.name,
        url: input.url,
        secret,
        events: input.events,
      },
    });

    return ok({ id: webhook.id, secret });
  }

  async deliver(
    orgId: string,
    eventType: WebhookEventType,
    data: Record<string, unknown>
  ): Promise<void> {
    const webhooks = await this.db.webhook.findMany({
      where: {
        orgId,
        isActive: true,
        events: { has: eventType },
      },
    });

    const payload: WebhookPayload = {
      id: `evt_${crypto.randomBytes(16).toString("hex")}`,
      type: eventType,
      orgId,
      data,
      timestamp: new Date().toISOString(),
      version: "1.0",
    };

    await Promise.allSettled(
      webhooks.map((webhook) =>
        this.deliverToWebhook(webhook, payload)
      )
    );
  }

  private async deliverToWebhook(
    webhook: any,
    payload: WebhookPayload
  ): Promise<void> {
    const payloadStr = JSON.stringify(payload);
    const signature = this.generateSignature(payloadStr, webhook.secret);

    const delivery = await this.db.webhookDelivery.create({
      data: {
        webhookId: webhook.id,
        event: payload.type,
        payload: payload as any,
        attemptCount: 1,
      },
    });

    try {
      const response = await this.sendRequest(
        webhook.url,
        payloadStr,
        signature
      );

      await this.db.webhookDelivery.update({
        where: { id: delivery.id },
        data: {
          statusCode: response.status,
          response: response.body,
          success: response.status >= 200 && response.status < 300,
          deliveredAt: new Date(),
        },
      });

      console.log(
        `Webhook delivered: ${webhook.url} - ${response.status}`
      );
    } catch (error) {
      await this.db.webhookDelivery.update({
        where: { id: delivery.id },
        data: {
          response: (error as Error).message,
          success: false,
          nextRetryAt: new Date(Date.now() + 5 * 60 * 1000), // retry ใน 5 นาที
        },
      });
    }
  }

  private generateSignature(payload: string, secret: string): string {
    const timestamp = Math.floor(Date.now() / 1000);
    const signedPayload = `${timestamp}.${payload}`;
    const signature = crypto
      .createHmac("sha256", secret)
      .update(signedPayload)
      .digest("hex");

    return `t=${timestamp},v1=${signature}`;
  }

  private async sendRequest(
    url: string,
    payload: string,
    signature: string
  ): Promise<{ status: number; body: string }> {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 30000);

    try {
      const response = await fetch(url, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Webhook-Signature": signature,
          "X-Webhook-Event": "saas-platform",
        },
        body: payload,
        signal: controller.signal,
      });

      const body = await response.text();
      return { status: response.status, body };
    } finally {
      clearTimeout(timeout);
    }
  }

  async retryFailedDeliveries(): Promise<void> {
    const failedDeliveries = await this.db.webhookDelivery.findMany({
      where: {
        success: false,
        attemptCount: { lt: 5 },
        nextRetryAt: { lte: new Date() },
      },
      include: { webhook: true },
    });

    for (const delivery of failedDeliveries) {
      if (!delivery.webhook.isActive) continue;

      const payload = delivery.payload as WebhookPayload;
      const signature = this.generateSignature(
        JSON.stringify(payload),
        delivery.webhook.secret
      );

      try {
        const response = await this.sendRequest(
          delivery.webhook.url,
          JSON.stringify(payload),
          signature
        );

        await this.db.webhookDelivery.update({
          where: { id: delivery.id },
          data: {
            statusCode: response.status,
            response: response.body,
            success: response.status >= 200 && response.status < 300,
            attemptCount: { increment: 1 },
            deliveredAt:
              response.status >= 200 && response.status < 300
                ? new Date()
                : undefined,
            nextRetryAt:
              response.status >= 200 && response.status < 300
                ? null
                : new Date(Date.now() + Math.pow(delivery.attemptCount + 1, 2) * 60 * 1000),
          },
        });
      } catch (error) {
        await this.db.webhookDelivery.update({
          where: { id: delivery.id },
          data: {
            attemptCount: { increment: 1 },
            nextRetryAt: new Date(
              Date.now() + Math.pow(delivery.attemptCount + 1, 2) * 60 * 1000
            ),
          },
        });
      }
    }
  }
}
```

---

## 9. Feature Flags

### ไฟล์: src/domain/feature-flags/FeatureFlagService.ts

```typescript
import { PrismaClient } from "@prisma/client";
import { Result, ok, err } from "../../core/result";

export type FlagValue = boolean | number | string | string[];

export interface FeatureFlag {
  key: string;
  name: string;
  description?: string;
  type: "boolean" | "number" | "string" | "list";
  defaultValue: FlagValue;
  isActive: boolean;
}

export class FeatureFlagService {
  private cache = new Map<string, { value: FlagValue; expiresAt: number }>();
  private readonly CACHE_TTL = 5 * 60 * 1000; // 5 minutes

  constructor(private readonly db: PrismaClient) {}

  async getFlag(
    key: string,
    orgId?: string
  ): Promise<FlagValue> {
    const cacheKey = `${key}:${orgId ?? "global"}`;
    const cached = this.cache.get(cacheKey);

    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }

    const flag = await this.db.featureFlag.findUnique({
      where: { key },
      include: orgId
        ? {
            orgOverrides: {
              where: { orgId },
            },
          }
        : undefined,
    });

    if (!flag || !flag.isActive) {
      return false;
    }

    const orgOverride = orgId ? (flag as any).orgOverrides?.[0] : null;
    const value = orgOverride ? orgOverride.value : flag.defaultValue;

    this.cache.set(cacheKey, {
      value: value as FlagValue,
      expiresAt: Date.now() + this.CACHE_TTL,
    });

    return value as FlagValue;
  }

  async isEnabled(key: string, orgId?: string): Promise<boolean> {
    const value = await this.getFlag(key, orgId);
    return Boolean(value);
  }

  async getNumber(
    key: string,
    defaultValue: number,
    orgId?: string
  ): Promise<number> {
    const value = await this.getFlag(key, orgId);
    return typeof value === "number" ? value : defaultValue;
  }

  async getString(
    key: string,
    defaultValue: string,
    orgId?: string
  ): Promise<string> {
    const value = await this.getFlag(key, orgId);
    return typeof value === "string" ? value : defaultValue;
  }

  async setOrgOverride(
    orgId: string,
    key: string,
    value: FlagValue,
    setBy?: string
  ): Promise<Result<void>> {
    const flag = await this.db.featureFlag.findUnique({ where: { key } });

    if (!flag) {
      return err(new Error(`Feature flag "${key}" ไม่พบ`));
    }

    await this.db.orgFeatureFlag.upsert({
      where: {
        orgId_featureFlagId: { orgId, featureFlagId: flag.id },
      },
      create: {
        orgId,
        featureFlagId: flag.id,
        value: value as any,
        setBy,
      },
      update: {
        value: value as any,
        setBy,
      },
    });

    // Clear cache
    this.cache.delete(`${key}:${orgId}`);

    return ok(undefined);
  }

  async removeOrgOverride(orgId: string, key: string): Promise<Result<void>> {
    const flag = await this.db.featureFlag.findUnique({ where: { key } });

    if (!flag) {
      return err(new Error(`Feature flag "${key}" ไม่พบ`));
    }

    await this.db.orgFeatureFlag.delete({
      where: {
        orgId_featureFlagId: { orgId, featureFlagId: flag.id },
      },
    });

    this.cache.delete(`${key}:${orgId}`);
    return ok(undefined);
  }

  async createFlag(flag: {
    key: string;
    name: string;
    description?: string;
    type: FeatureFlag["type"];
    defaultValue: FlagValue;
  }): Promise<Result<FeatureFlag>> {
    const existing = await this.db.featureFlag.findUnique({
      where: { key: flag.key },
    });

    if (existing) {
      return err(new Error(`Feature flag "${flag.key}" มีอยู่แล้ว`));
    }

    const created = await this.db.featureFlag.create({
      data: {
        key: flag.key,
        name: flag.name,
        description: flag.description,
        type: flag.type,
        defaultValue: flag.defaultValue as any,
      },
    });

    return ok({
      key: created.key,
      name: created.name,
      description: created.description ?? undefined,
      type: created.type as FeatureFlag["type"],
      defaultValue: created.defaultValue as FlagValue,
      isActive: created.isActive,
    });
  }

  // Batch check multiple flags
  async getFlags(
    keys: string[],
    orgId?: string
  ): Promise<Record<string, FlagValue>> {
    const results: Record<string, FlagValue> = {};

    await Promise.all(
      keys.map(async (key) => {
        results[key] = await this.getFlag(key, orgId);
      })
    );

    return results;
  }
}
```

---

## 10. API Middleware และ Routes

### ไฟล์: src/api/middleware/auth.ts

```typescript
import { Request, Response, NextFunction } from "express";
import { UserService } from "../../domain/user/UserService";
import { ApiKeyService } from "../../domain/api-keys/ApiKeyService";
import { UnauthorizedError, ForbiddenError } from "../../core/errors";
import { Permission } from "../../core/types";

// Extend Express Request
declare global {
  namespace Express {
    interface Request {
      userId?: string;
      orgId?: string;
      scopes?: string[];
      authType?: "jwt" | "apikey";
    }
  }
}

export function authMiddleware(
  userService: UserService,
  apiKeyService: ApiKeyService
) {
  return async (
    req: Request,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const authHeader = req.headers.authorization;

      if (!authHeader) {
        res.status(401).json({ error: "ต้องการ Authorization header" });
        return;
      }

      if (authHeader.startsWith("Bearer ")) {
        // JWT authentication
        const token = authHeader.substring(7);
        const payload = userService.verifyAccessToken(token);

        if (!payload) {
          res.status(401).json({ error: "Token ไม่ถูกต้อง" });
          return;
        }

        req.userId = payload.userId;
        req.authType = "jwt";
      } else if (authHeader.startsWith("ApiKey ")) {
        // API Key authentication
        const apiKey = authHeader.substring(7);
        const result = await apiKeyService.validateApiKey(apiKey);

        if (!result.valid) {
          res.status(401).json({ error: "API Key ไม่ถูกต้อง" });
          return;
        }

        req.orgId = result.orgId;
        req.scopes = result.scopes;
        req.authType = "apikey";
      } else {
        res.status(401).json({ error: "รูปแบบ Authorization ไม่ถูกต้อง" });
        return;
      }

      next();
    } catch (error) {
      next(error);
    }
  };
}

// Permission middleware
export function requirePermission(
  permission: Permission,
  orgService: { hasPermission: (userId: string, orgId: string, permission: Permission) => Promise<boolean> }
) {
  return async (
    req: Request,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    try {
      const orgId = req.params.orgId ?? req.orgId;

      if (!orgId) {
        res.status(403).json({ error: "ต้องระบุ Organization" });
        return;
      }

      if (req.authType === "apikey") {
        // Check API key scopes
        // Convert permission to scope
        const hasScope = req.scopes?.some((scope) =>
          scope.includes(permission.split(":")[0])
        );

        if (!hasScope) {
          res.status(403).json({ error: `ไม่มีสิทธิ์ทำ: ${permission}` });
          return;
        }

        req.orgId = orgId;
        next();
        return;
      }

      if (!req.userId) {
        res.status(401).json({ error: "ต้องเข้าสู่ระบบก่อน" });
        return;
      }

      const hasPermission = await orgService.hasPermission(
        req.userId,
        orgId,
        permission
      );

      if (!hasPermission) {
        res.status(403).json({ error: `ไม่มีสิทธิ์ทำ: ${permission}` });
        return;
      }

      req.orgId = orgId;
      next();
    } catch (error) {
      next(error);
    }
  };
}
```

### ไฟล์: src/api/routes/users.ts

```typescript
import { Router, Request, Response, NextFunction } from "express";
import { UserService } from "../../domain/user/UserService";
import { authMiddleware } from "../middleware/auth";
import { ApiKeyService } from "../../domain/api-keys/ApiKeyService";

export function createUserRouter(
  userService: UserService,
  apiKeyService: ApiKeyService
): Router {
  const router = Router();
  const auth = authMiddleware(userService, apiKeyService);

  // POST /auth/register
  router.post("/auth/register", async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await userService.register(req.body);

      if (!result.success) {
        const err = result.error as any;
        return res
          .status(err.statusCode ?? 400)
          .json({ error: err.message, fields: err.fields });
      }

      res.status(201).json({
        message: "สร้างบัญชีสำเร็จ กรุณายืนยัน email",
        user: result.data,
      });
    } catch (error) {
      next(error);
    }
  });

  // POST /auth/login
  router.post("/auth/login", async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await userService.login({
        ...req.body,
        ipAddress: req.ip,
        userAgent: req.headers["user-agent"],
      });

      if (!result.success) {
        return res.status(401).json({ error: (result.error as any).message });
      }

      res.json(result.data);
    } catch (error) {
      next(error);
    }
  });

  // POST /auth/refresh
  router.post("/auth/refresh", async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { refreshToken } = req.body;
      const result = await userService.refreshToken(refreshToken);

      if (!result.success) {
        return res.status(401).json({ error: (result.error as any).message });
      }

      res.json(result.data);
    } catch (error) {
      next(error);
    }
  });

  // POST /auth/logout
  router.post("/auth/logout", auth, async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { refreshToken } = req.body;
      await userService.logout(req.userId!, refreshToken);
      res.json({ message: "ออกจากระบบสำเร็จ" });
    } catch (error) {
      next(error);
    }
  });

  // GET /users/me
  router.get("/users/me", auth, async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await userService.getUser(req.userId!);

      if (!result.success) {
        return res.status(404).json({ error: (result.error as any).message });
      }

      res.json(result.data);
    } catch (error) {
      next(error);
    }
  });

  // PATCH /users/me
  router.patch("/users/me", auth, async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await userService.updateUser(
        req.userId!,
        req.body,
        req.userId!
      );

      if (!result.success) {
        return res.status(400).json({ error: (result.error as any).message });
      }

      res.json(result.data);
    } catch (error) {
      next(error);
    }
  });

  // POST /users/me/change-password
  router.post("/users/me/change-password", auth, async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { currentPassword, newPassword } = req.body;
      const result = await userService.changePassword(
        req.userId!,
        currentPassword,
        newPassword
      );

      if (!result.success) {
        return res.status(400).json({ error: (result.error as any).message });
      }

      res.json({ message: "เปลี่ยนรหัสผ่านสำเร็จ" });
    } catch (error) {
      next(error);
    }
  });

  return router;
}
```

---

## 11. Email Notifications

### ไฟล์: src/infrastructure/email/EmailService.ts

```typescript
import nodemailer from "nodemailer";
import handlebars from "handlebars";
import * as fs from "fs";
import * as path from "path";

export interface EmailOptions {
  to: string | string[];
  subject: string;
  template: string;
  data: Record<string, unknown>;
  from?: string;
  cc?: string[];
  bcc?: string[];
  attachments?: Array<{
    filename: string;
    content: Buffer | string;
    contentType?: string;
  }>;
}

export class EmailService {
  private transporter: nodemailer.Transporter;
  private templatesDir: string;
  private compiledTemplates = new Map<string, HandlebarsTemplateDelegate>();

  constructor() {
    this.transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST ?? "smtp.example.com",
      port: parseInt(process.env.SMTP_PORT ?? "587"),
      secure: process.env.SMTP_SECURE === "true",
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    });

    this.templatesDir = path.join(__dirname, "templates");

    // Register Handlebars helpers
    handlebars.registerHelper("formatDate", (date: Date) => {
      return new Intl.DateTimeFormat("th-TH", {
        year: "numeric",
        month: "long",
        day: "numeric",
      }).format(date);
    });

    handlebars.registerHelper("formatMoney", (amount: number, currency: string) => {
      return new Intl.NumberFormat("th-TH", {
        style: "currency",
        currency: currency ?? "THB",
      }).format(amount);
    });
  }

  async send(options: EmailOptions): Promise<void> {
    const html = await this.renderTemplate(options.template, options.data);

    await this.transporter.sendMail({
      from: options.from ?? `"${process.env.APP_NAME ?? "SaaS Platform"}" <${process.env.SMTP_FROM ?? "noreply@example.com"}>`,
      to: Array.isArray(options.to) ? options.to.join(", ") : options.to,
      cc: options.cc,
      bcc: options.bcc,
      subject: options.subject,
      html,
      attachments: options.attachments,
    });
  }

  async sendBulk(
    recipients: Array<{ email: string; data: Record<string, unknown> }>,
    subject: string,
    template: string
  ): Promise<void> {
    // ส่งทีละ batch เพื่อไม่ให้ล้น queue
    const batchSize = 10;

    for (let i = 0; i < recipients.length; i += batchSize) {
      const batch = recipients.slice(i, i + batchSize);

      await Promise.allSettled(
        batch.map((recipient) =>
          this.send({
            to: recipient.email,
            subject,
            template,
            data: recipient.data,
          })
        )
      );

      // Small delay between batches
      if (i + batchSize < recipients.length) {
        await new Promise((resolve) => setTimeout(resolve, 100));
      }
    }
  }

  private async renderTemplate(
    templateName: string,
    data: Record<string, unknown>
  ): Promise<string> {
    let compiled = this.compiledTemplates.get(templateName);

    if (!compiled) {
      const templatePath = path.join(
        this.templatesDir,
        `${templateName}.hbs`
      );

      let templateContent: string;

      try {
        templateContent = fs.readFileSync(templatePath, "utf-8");
      } catch {
        // Fallback template
        templateContent = this.getDefaultTemplate(templateName, data);
      }

      compiled = handlebars.compile(templateContent);
      this.compiledTemplates.set(templateName, compiled);
    }

    return compiled(data);
  }

  private getDefaultTemplate(
    templateName: string,
    data: Record<string, unknown>
  ): string {
    // Default email templates
    const templates: Record<string, string> = {
      "email-verification": `
        <h1>ยืนยัน Email ของคุณ</h1>
        <p>คลิกลิงก์ด้านล่างเพื่อยืนยัน email</p>
        <a href="{{verifyUrl}}">ยืนยัน Email</a>
        <p>ลิงก์นี้จะหมดอายุใน {{expiresIn}}</p>
      `,
      "password-reset": `
        <h1>รีเซ็ตรหัสผ่าน</h1>
        <p>สวัสดีคุณ {{name}}</p>
        <p>คลิกลิงก์ด้านล่างเพื่อรีเซ็ตรหัสผ่าน</p>
        <a href="{{resetUrl}}">รีเซ็ตรหัสผ่าน</a>
        <p>ลิงก์นี้จะหมดอายุใน {{expiresIn}}</p>
      `,
      "org-invitation": `
        <h1>คุณได้รับคำเชิญ</h1>
        <p>{{inviterName}} เชิญคุณเข้าร่วม {{orgName}} ในฐานะ {{role}}</p>
        <a href="{{acceptUrl}}">รับคำเชิญ</a>
        <p>คำเชิญนี้จะหมดอายุใน {{expiresIn}}</p>
      `,
      "subscription-confirmed": `
        <h1>ยืนยันการสมัครสมาชิก</h1>
        <p>สวัสดีคุณ {{name}}</p>
        <p>คุณสมัคร {{planName}} เรียบร้อยแล้ว</p>
        <p>ราคา: {{price}}</p>
        <p>ต่ออายุครั้งถัดไป: {{nextBillingDate}}</p>
      `,
      "payment-failed": `
        <h1>การชำระเงินล้มเหลว</h1>
        <p>สวัสดีคุณ {{name}}</p>
        <p>การชำระเงินจำนวน {{amount}} ล้มเหลว</p>
        <p>กรุณาอัปเดตข้อมูลการชำระเงินของคุณ</p>
        <a href="{{updatePaymentUrl}}">อัปเดตการชำระเงิน</a>
      `,
    };

    return templates[templateName] ?? `<p>${JSON.stringify(data)}</p>`;
  }
}
```

---

## 12. Application Bootstrap

### ไฟล์: src/app.ts

```typescript
import express, { Express, Request, Response, NextFunction } from "express";
import cors from "cors";
import helmet from "helmet";
import compression from "compression";
import { PrismaClient } from "@prisma/client";
import { UserService } from "./domain/user/UserService";
import { OrgService } from "./domain/organization/OrgService";
import { BillingService } from "./domain/subscription/BillingService";
import { ApiKeyService } from "./domain/api-keys/ApiKeyService";
import { WebhookService } from "./domain/webhooks/WebhookService";
import { FeatureFlagService } from "./domain/feature-flags/FeatureFlagService";
import { EmailService } from "./infrastructure/email/EmailService";
import { AuditService } from "./domain/audit/AuditService";
import { createUserRouter } from "./api/routes/users";
import { authMiddleware } from "./api/middleware/auth";
import { AppError } from "./core/errors";

export class Application {
  private app: Express;
  private db: PrismaClient;

  // Services
  private userService!: UserService;
  private orgService!: OrgService;
  private billingService!: BillingService;
  private apiKeyService!: ApiKeyService;
  private webhookService!: WebhookService;
  private featureFlagService!: FeatureFlagService;
  private emailService!: EmailService;
  private auditService!: AuditService;

  constructor() {
    this.app = express();
    this.db = new PrismaClient({
      log: process.env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
    });
  }

  async initialize(): Promise<void> {
    // Initialize services
    this.emailService = new EmailService();
    this.auditService = new AuditService(this.db);

    this.userService = new UserService(
      this.db,
      this.emailService,
      this.auditService
    );

    this.apiKeyService = new ApiKeyService(this.db);
    this.webhookService = new WebhookService(this.db);
    this.featureFlagService = new FeatureFlagService(this.db);

    this.billingService = new BillingService(
      this.db,
      this.emailService,
      this.auditService
    );

    // Setup middleware
    this.setupMiddleware();

    // Setup routes
    this.setupRoutes();

    // Error handling
    this.setupErrorHandling();

    // Connect to database
    await this.db.$connect();
    console.log("เชื่อมต่อฐานข้อมูลสำเร็จ");
  }

  private setupMiddleware(): void {
    // Security headers
    this.app.use(helmet());

    // CORS
    this.app.use(cors({
      origin: process.env.ALLOWED_ORIGINS?.split(",") ?? "*",
      methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
      allowedHeaders: ["Content-Type", "Authorization"],
      credentials: true,
    }));

    // Compression
    this.app.use(compression());

    // Body parsing
    this.app.use(express.json({ limit: "10mb" }));
    this.app.use(express.urlencoded({ extended: true }));

    // Request logging
    this.app.use((req: Request, _res: Response, next: NextFunction) => {
      console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
      next();
    });
  }

  private setupRoutes(): void {
    const apiV1 = express.Router();

    // Health check
    apiV1.get("/health", (_req: Request, res: Response) => {
      res.json({
        status: "ok",
        timestamp: new Date().toISOString(),
        version: process.env.npm_package_version ?? "1.0.0",
      });
    });

    // Auth & User routes
    apiV1.use("/", createUserRouter(this.userService, this.apiKeyService));

    // Stripe webhook (ต้องรับ raw body)
    this.app.post(
      "/webhooks/stripe",
      express.raw({ type: "application/json" }),
      async (req: Request, res: Response, next: NextFunction) => {
        const signature = req.headers["stripe-signature"] as string;

        const result = await this.billingService.handleStripeWebhook(
          req.body.toString(),
          signature
        );

        if (!result.success) {
          return res.status(400).json({ error: (result.error as any).message });
        }

        res.json({ received: true });
      }
    );

    this.app.use("/api/v1", apiV1);
  }

  private setupErrorHandling(): void {
    // 404 handler
    this.app.use((_req: Request, res: Response) => {
      res.status(404).json({ error: "ไม่พบ endpoint ที่ต้องการ" });
    });

    // Error handler
    this.app.use(
      (error: Error, _req: Request, res: Response, _next: NextFunction) => {
        console.error("Error:", error);

        if (error instanceof AppError) {
          return res.status(error.statusCode).json({
            error: error.message,
            code: error.code,
            context: error.context,
          });
        }

        res.status(500).json({
          error: "เกิดข้อผิดพลาดภายในระบบ",
          code: "INTERNAL_ERROR",
        });
      }
    );
  }

  async start(port: number = 3000): Promise<void> {
    await this.initialize();

    this.app.listen(port, () => {
      console.log(`🚀 SaaS Platform ทำงานที่ port ${port}`);
      console.log(`   สภาพแวดล้อม: ${process.env.NODE_ENV ?? "development"}`);
    });

    // Graceful shutdown
    process.on("SIGTERM", async () => {
      console.log("ได้รับ SIGTERM กำลังปิดระบบ...");
      await this.db.$disconnect();
      process.exit(0);
    });
  }
}

// Start application
const app = new Application();
app.start(parseInt(process.env.PORT ?? "3000")).catch((error) => {
  console.error("ไม่สามารถเริ่ม application:", error);
  process.exit(1);
});
```

---

## สรุป

SaaS Platform ที่เราสร้างนี้ประกอบด้วย:

### Features หลัก
1. **User Management** - register, login, JWT/refresh tokens, email verification, password reset
2. **Organization Management** - multi-tenant, member roles, invitations
3. **Subscription Billing** - Stripe integration, plan management, usage limits
4. **API Keys** - สร้าง, rotate, revoke, scope-based authorization
5. **Webhooks** - real-time events, retry logic, signature verification
6. **Feature Flags** - per-org overrides, caching
7. **Email Notifications** - template-based, bulk sending
8. **Audit Logging** - ตรวจสอบ activity ทุก action

### Architecture Patterns
- **Domain-Driven Design** - แยก domain logic อย่างชัดเจน
- **Result Type** - error handling ที่ปลอดภัย
- **Branded Types** - type-safe IDs
- **Repository Pattern** - abstract database access
- **Service Layer** - business logic
- **Middleware Pattern** - auth, validation, rate limiting

### Security Features
- Password hashing ด้วย bcrypt
- JWT tokens with refresh rotation
- API Key hashing (SHA-256)
- Webhook signature verification
- Input validation
- Audit logging

---

*จบตอนที่ 80 - Real-world Project: SaaS Platform*
