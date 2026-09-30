# Part 57: Real-world Project - E-commerce API (โปรเจกต์จริง: E-commerce API)

## บทนำ

ในบทนี้เราจะสร้าง E-commerce API ที่สมบูรณ์ด้วย TypeScript, Node.js, Express และ PostgreSQL โดยครอบคลุมฟีเจอร์หลักทั้งหมดตั้งแต่การจัดการสินค้า ระบบตะกร้า การสั่งซื้อ จนถึงการชำระเงิน

## โครงสร้างโปรเจกต์ (Project Structure)

```
ecommerce-api/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── redis.ts
│   │   └── environment.ts
│   ├── types/
│   │   ├── models.ts
│   │   ├── requests.ts
│   │   ├── responses.ts
│   │   └── common.ts
│   ├── models/
│   │   ├── User.ts
│   │   ├── Product.ts
│   │   ├── Category.ts
│   │   ├── Order.ts
│   │   ├── OrderItem.ts
│   │   ├── Cart.ts
│   │   ├── CartItem.ts
│   │   ├── Review.ts
│   │   └── Payment.ts
│   ├── repositories/
│   │   ├── UserRepository.ts
│   │   ├── ProductRepository.ts
│   │   ├── OrderRepository.ts
│   │   └── CartRepository.ts
│   ├── services/
│   │   ├── AuthService.ts
│   │   ├── ProductService.ts
│   │   ├── CartService.ts
│   │   ├── OrderService.ts
│   │   ├── PaymentService.ts
│   │   ├── SearchService.ts
│   │   └── FileUploadService.ts
│   ├── controllers/
│   │   ├── AuthController.ts
│   │   ├── ProductController.ts
│   │   ├── CartController.ts
│   │   ├── OrderController.ts
│   │   └── ReviewController.ts
│   ├── middlewares/
│   │   ├── auth.ts
│   │   ├── validation.ts
│   │   ├── errorHandler.ts
│   │   └── rateLimiter.ts
│   ├── routes/
│   │   ├── auth.ts
│   │   ├── products.ts
│   │   ├── cart.ts
│   │   ├── orders.ts
│   │   └── reviews.ts
│   └── index.ts
├── prisma/
│   └── schema.prisma
├── package.json
└── tsconfig.json
```

---

## ไฟล์ Types หลัก (Core Types)

### src/types/models.ts

```typescript
// ========================
// User Types
// ========================
export interface User {
  id: string;
  email: string;
  passwordHash: string;
  firstName: string;
  lastName: string;
  phone?: string;
  role: UserRole;
  isEmailVerified: boolean;
  isActive: boolean;
  addresses: Address[];
  createdAt: Date;
  updatedAt: Date;
}

export enum UserRole {
  CUSTOMER = "CUSTOMER",
  ADMIN = "ADMIN",
  VENDOR = "VENDOR"
}

export interface Address {
  id: string;
  userId: string;
  label: string; // "บ้าน", "ที่ทำงาน"
  recipientName: string;
  phone: string;
  addressLine1: string;
  addressLine2?: string;
  district: string;
  province: string;
  postalCode: string;
  country: string;
  isDefault: boolean;
}

// ========================
// Product Types
// ========================
export interface Product {
  id: string;
  sku: string;
  name: string;
  description: string;
  price: number;
  compareAtPrice?: number; // ราคาก่อนลด
  costPrice: number;
  categoryId: string;
  category?: Category;
  vendorId?: string;
  images: ProductImage[];
  variants: ProductVariant[];
  attributes: ProductAttribute[];
  inventory: Inventory;
  isActive: boolean;
  isFeatured: boolean;
  tags: string[];
  weight?: number; // kg
  dimensions?: ProductDimensions;
  rating: number;
  reviewCount: number;
  soldCount: number;
  createdAt: Date;
  updatedAt: Date;
}

export interface ProductImage {
  id: string;
  productId: string;
  url: string;
  altText?: string;
  sortOrder: number;
  isMain: boolean;
}

export interface ProductVariant {
  id: string;
  productId: string;
  sku: string;
  name: string;
  options: VariantOption[]; // { color: "red", size: "L" }
  price: number;
  compareAtPrice?: number;
  inventory: number;
  images: string[];
  isActive: boolean;
}

export interface VariantOption {
  name: string;  // "color", "size"
  value: string; // "red", "L"
}

export interface ProductAttribute {
  name: string;
  value: string;
}

export interface ProductDimensions {
  length: number;
  width: number;
  height: number;
  unit: "cm" | "inch";
}

export interface Inventory {
  quantity: number;
  reservedQuantity: number;
  availableQuantity: number; // quantity - reservedQuantity
  lowStockThreshold: number;
  trackInventory: boolean;
  allowBackorder: boolean;
}

// ========================
// Category Types
// ========================
export interface Category {
  id: string;
  name: string;
  slug: string;
  description?: string;
  imageUrl?: string;
  parentId?: string;
  parent?: Category;
  children?: Category[];
  isActive: boolean;
  sortOrder: number;
  createdAt: Date;
  updatedAt: Date;
}

// ========================
// Cart Types
// ========================
export interface Cart {
  id: string;
  userId?: string;
  sessionId?: string;
  items: CartItem[];
  couponCode?: string;
  discount: number;
  subtotal: number;
  shipping: number;
  tax: number;
  total: number;
  expiresAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export interface CartItem {
  id: string;
  cartId: string;
  productId: string;
  product?: Product;
  variantId?: string;
  variant?: ProductVariant;
  quantity: number;
  price: number; // ราคาตอนเพิ่มเข้าตะกร้า
  total: number;
  notes?: string;
}

// ========================
// Order Types
// ========================
export interface Order {
  id: string;
  orderNumber: string;
  userId: string;
  user?: User;
  items: OrderItem[];
  status: OrderStatus;
  paymentStatus: PaymentStatus;
  shippingAddress: Address;
  billingAddress?: Address;
  subtotal: number;
  discount: number;
  shippingFee: number;
  tax: number;
  total: number;
  couponCode?: string;
  notes?: string;
  trackingNumber?: string;
  shippingProvider?: string;
  estimatedDelivery?: Date;
  deliveredAt?: Date;
  cancelledAt?: Date;
  cancelReason?: string;
  refundAmount?: number;
  refundedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export enum OrderStatus {
  PENDING = "PENDING",
  CONFIRMED = "CONFIRMED",
  PROCESSING = "PROCESSING",
  SHIPPED = "SHIPPED",
  DELIVERED = "DELIVERED",
  CANCELLED = "CANCELLED",
  REFUNDED = "REFUNDED"
}

export enum PaymentStatus {
  PENDING = "PENDING",
  PAID = "PAID",
  FAILED = "FAILED",
  REFUNDED = "REFUNDED",
  PARTIALLY_REFUNDED = "PARTIALLY_REFUNDED"
}

export interface OrderItem {
  id: string;
  orderId: string;
  productId: string;
  product?: Product;
  variantId?: string;
  variant?: ProductVariant;
  quantity: number;
  price: number;
  total: number;
  productSnapshot: ProductSnapshot; // ข้อมูล product ตอนสั่ง
}

export interface ProductSnapshot {
  name: string;
  sku: string;
  price: number;
  imageUrl?: string;
  options?: VariantOption[];
}

// ========================
// Payment Types
// ========================
export interface Payment {
  id: string;
  orderId: string;
  order?: Order;
  amount: number;
  currency: string;
  method: PaymentMethod;
  status: PaymentStatus;
  provider: string; // "stripe", "promptpay", "omise"
  providerTransactionId?: string;
  providerData?: Record<string, unknown>;
  failureReason?: string;
  refunds: Refund[];
  paidAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export enum PaymentMethod {
  CREDIT_CARD = "CREDIT_CARD",
  DEBIT_CARD = "DEBIT_CARD",
  BANK_TRANSFER = "BANK_TRANSFER",
  PROMPTPAY = "PROMPTPAY",
  COD = "COD" // Cash on Delivery
}

export interface Refund {
  id: string;
  paymentId: string;
  amount: number;
  reason: string;
  status: "PENDING" | "PROCESSED" | "FAILED";
  processedAt?: Date;
  createdAt: Date;
}

// ========================
// Review Types
// ========================
export interface Review {
  id: string;
  productId: string;
  product?: Product;
  userId: string;
  user?: User;
  orderId?: string;
  rating: number; // 1-5
  title?: string;
  body: string;
  images?: string[];
  isVerifiedPurchase: boolean;
  isApproved: boolean;
  helpfulCount: number;
  createdAt: Date;
  updatedAt: Date;
}

// ========================
// Coupon Types
// ========================
export interface Coupon {
  id: string;
  code: string;
  description: string;
  type: CouponType;
  value: number; // amount หรือ percentage
  minOrderAmount?: number;
  maxDiscountAmount?: number;
  usageLimit?: number;
  usedCount: number;
  userLimit?: number; // limit ต่อ user
  isActive: boolean;
  startsAt?: Date;
  expiresAt?: Date;
  applicableCategories?: string[];
  applicableProducts?: string[];
  createdAt: Date;
}

export enum CouponType {
  FIXED = "FIXED",
  PERCENTAGE = "PERCENTAGE",
  FREE_SHIPPING = "FREE_SHIPPING"
}
```

---

### src/types/requests.ts

```typescript
// ========================
// Auth Requests
// ========================
export interface RegisterRequest {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  phone?: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface RefreshTokenRequest {
  refreshToken: string;
}

export interface ForgotPasswordRequest {
  email: string;
}

export interface ResetPasswordRequest {
  token: string;
  newPassword: string;
}

export interface UpdateProfileRequest {
  firstName?: string;
  lastName?: string;
  phone?: string;
}

export interface ChangePasswordRequest {
  currentPassword: string;
  newPassword: string;
}

// ========================
// Product Requests
// ========================
export interface CreateProductRequest {
  sku: string;
  name: string;
  description: string;
  price: number;
  compareAtPrice?: number;
  costPrice: number;
  categoryId: string;
  tags?: string[];
  weight?: number;
  dimensions?: {
    length: number;
    width: number;
    height: number;
    unit: "cm" | "inch";
  };
}

export interface UpdateProductRequest extends Partial<CreateProductRequest> {
  isActive?: boolean;
  isFeatured?: boolean;
}

export interface CreateVariantRequest {
  sku: string;
  name: string;
  options: Array<{ name: string; value: string }>;
  price: number;
  compareAtPrice?: number;
  inventory: number;
}

export interface ProductSearchRequest {
  query?: string;
  categoryId?: string;
  minPrice?: number;
  maxPrice?: number;
  tags?: string[];
  sortBy?: "price_asc" | "price_desc" | "rating" | "newest" | "popular";
  page?: number;
  limit?: number;
  inStock?: boolean;
}

// ========================
// Cart Requests
// ========================
export interface AddToCartRequest {
  productId: string;
  variantId?: string;
  quantity: number;
  notes?: string;
}

export interface UpdateCartItemRequest {
  quantity: number;
  notes?: string;
}

export interface ApplyCouponRequest {
  couponCode: string;
}

// ========================
// Order Requests
// ========================
export interface CreateOrderRequest {
  cartId: string;
  shippingAddressId: string;
  billingAddressId?: string;
  paymentMethod: string;
  notes?: string;
}

export interface UpdateOrderStatusRequest {
  status: import("./models").OrderStatus;
  notes?: string;
  trackingNumber?: string;
  shippingProvider?: string;
}

// ========================
// Review Requests
// ========================
export interface CreateReviewRequest {
  productId: string;
  orderId?: string;
  rating: number;
  title?: string;
  body: string;
}

// ========================
// Address Requests
// ========================
export interface CreateAddressRequest {
  label: string;
  recipientName: string;
  phone: string;
  addressLine1: string;
  addressLine2?: string;
  district: string;
  province: string;
  postalCode: string;
  country: string;
  isDefault?: boolean;
}
```

---

### src/types/responses.ts

```typescript
// ========================
// Common Response Types
// ========================
export interface ApiResponse<T = void> {
  success: boolean;
  data?: T;
  message?: string;
  errors?: ValidationError[];
}

export interface PaginatedResponse<T> extends ApiResponse<T[]> {
  meta: PaginationMeta;
}

export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}

export interface ValidationError {
  field: string;
  message: string;
}

// ========================
// Auth Responses
// ========================
export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

export interface LoginResponse extends ApiResponse<AuthTokens> {
  user: UserProfile;
}

export interface UserProfile {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  phone?: string;
  role: string;
  isEmailVerified: boolean;
  avatar?: string;
}

// ========================
// Product Responses
// ========================
export interface ProductListItem {
  id: string;
  name: string;
  slug: string;
  price: number;
  compareAtPrice?: number;
  mainImage?: string;
  rating: number;
  reviewCount: number;
  isInStock: boolean;
  category: {
    id: string;
    name: string;
  };
}

export interface ProductDetail extends ProductListItem {
  sku: string;
  description: string;
  images: Array<{
    url: string;
    altText?: string;
    isMain: boolean;
  }>;
  variants: Array<{
    id: string;
    name: string;
    options: Array<{ name: string; value: string }>;
    price: number;
    inventory: number;
  }>;
  attributes: Array<{ name: string; value: string }>;
  tags: string[];
  weight?: number;
  dimensions?: object;
}

// ========================
// Order Responses
// ========================
export interface OrderSummary {
  id: string;
  orderNumber: string;
  status: string;
  paymentStatus: string;
  total: number;
  itemCount: number;
  createdAt: Date;
}
```

---

## การตั้งค่า Environment

### src/config/environment.ts

```typescript
import { z } from "zod";

const envSchema = z.object({
  // App
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.string().default("3000"),
  API_BASE_URL: z.string().url(),
  
  // Database
  DATABASE_URL: z.string(),
  
  // Redis
  REDIS_URL: z.string().default("redis://localhost:6379"),
  
  // JWT
  JWT_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default("15m"),
  JWT_REFRESH_EXPIRES_IN: z.string().default("7d"),
  
  // Email
  SMTP_HOST: z.string(),
  SMTP_PORT: z.string().default("587"),
  SMTP_USER: z.string(),
  SMTP_PASS: z.string(),
  FROM_EMAIL: z.string().email(),
  
  // Storage
  STORAGE_TYPE: z.enum(["local", "s3"]).default("local"),
  AWS_ACCESS_KEY_ID: z.string().optional(),
  AWS_SECRET_ACCESS_KEY: z.string().optional(),
  AWS_REGION: z.string().optional(),
  AWS_S3_BUCKET: z.string().optional(),
  
  // Payment
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  
  // Rate Limiting
  RATE_LIMIT_WINDOW: z.string().default("15"),
  RATE_LIMIT_MAX: z.string().default("100"),
});

function validateEnv() {
  const result = envSchema.safeParse(process.env);
  
  if (!result.success) {
    console.error("Invalid environment variables:");
    console.error(result.error.format());
    process.exit(1);
  }
  
  return result.data;
}

export const env = validateEnv();
export type Env = typeof env;
```

---

## Database Configuration

### src/config/database.ts

```typescript
import { PrismaClient } from "@prisma/client";
import { env } from "./environment";

declare global {
  var prisma: PrismaClient | undefined;
}

function createPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: env.NODE_ENV === "development"
      ? ["query", "error", "warn"]
      : ["error"],
    errorFormat: env.NODE_ENV === "development" ? "pretty" : "minimal",
  });
}

export const prisma = globalThis.prisma ?? createPrismaClient();

if (env.NODE_ENV !== "production") {
  globalThis.prisma = prisma;
}

// Graceful shutdown
process.on("beforeExit", async () => {
  await prisma.$disconnect();
});
```

---

## Authentication Service

### src/services/AuthService.ts

```typescript
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
import crypto from "crypto";
import { prisma } from "../config/database";
import { env } from "../config/environment";
import type {
  RegisterRequest,
  LoginRequest,
  RefreshTokenRequest
} from "../types/requests";
import type { AuthTokens, LoginResponse, UserProfile } from "../types/responses";
import type { User } from "../types/models";

export class AuthError extends Error {
  constructor(
    message: string,
    public statusCode: number = 400,
    public code: string = "AUTH_ERROR"
  ) {
    super(message);
    this.name = "AuthError";
  }
}

export interface TokenPayload {
  userId: string;
  email: string;
  role: string;
  type: "access" | "refresh";
}

export class AuthService {
  private readonly SALT_ROUNDS = 12;
  private readonly REFRESH_TOKEN_EXPIRY = 7 * 24 * 60 * 60 * 1000; // 7 days
  
  async register(data: RegisterRequest): Promise<LoginResponse> {
    // ตรวจสอบ email ซ้ำ
    const existingUser = await prisma.user.findUnique({
      where: { email: data.email.toLowerCase() }
    });
    
    if (existingUser) {
      throw new AuthError("Email นี้ถูกใช้งานแล้ว", 409, "EMAIL_TAKEN");
    }
    
    // Hash password
    const passwordHash = await bcrypt.hash(data.password, this.SALT_ROUNDS);
    
    // สร้าง email verification token
    const emailVerificationToken = crypto.randomBytes(32).toString("hex");
    
    // สร้าง user
    const user = await prisma.user.create({
      data: {
        email: data.email.toLowerCase(),
        passwordHash,
        firstName: data.firstName,
        lastName: data.lastName,
        phone: data.phone,
        emailVerificationToken,
        role: "CUSTOMER"
      }
    });
    
    // ส่ง verification email (async)
    this.sendVerificationEmail(user.email, emailVerificationToken).catch(err => {
      console.error("Failed to send verification email:", err);
    });
    
    // Generate tokens
    const tokens = this.generateTokens(user);
    await this.saveRefreshToken(user.id, tokens.refreshToken);
    
    return {
      success: true,
      data: tokens,
      user: this.mapToUserProfile(user),
      message: "ลงทะเบียนสำเร็จ กรุณาตรวจสอบ email เพื่อยืนยัน"
    };
  }
  
  async login(data: LoginRequest): Promise<LoginResponse> {
    const user = await prisma.user.findUnique({
      where: { email: data.email.toLowerCase() }
    });
    
    if (!user || !user.isActive) {
      throw new AuthError("Email หรือรหัสผ่านไม่ถูกต้อง", 401, "INVALID_CREDENTIALS");
    }
    
    const isPasswordValid = await bcrypt.compare(data.password, user.passwordHash);
    
    if (!isPasswordValid) {
      // บันทึก failed attempt
      await this.recordFailedLogin(user.id);
      throw new AuthError("Email หรือรหัสผ่านไม่ถูกต้อง", 401, "INVALID_CREDENTIALS");
    }
    
    // Reset failed attempts
    await prisma.user.update({
      where: { id: user.id },
      data: {
        failedLoginAttempts: 0,
        lastLoginAt: new Date()
      }
    });
    
    const tokens = this.generateTokens(user);
    await this.saveRefreshToken(user.id, tokens.refreshToken);
    
    return {
      success: true,
      data: tokens,
      user: this.mapToUserProfile(user)
    };
  }
  
  async refreshToken(data: RefreshTokenRequest): Promise<AuthTokens> {
    let payload: TokenPayload;
    
    try {
      payload = jwt.verify(
        data.refreshToken,
        env.JWT_REFRESH_SECRET
      ) as TokenPayload;
    } catch {
      throw new AuthError("Refresh token ไม่ถูกต้อง", 401, "INVALID_TOKEN");
    }
    
    if (payload.type !== "refresh") {
      throw new AuthError("Token type ไม่ถูกต้อง", 401, "INVALID_TOKEN");
    }
    
    // ตรวจสอบว่า refresh token ยังใช้งานได้
    const savedToken = await prisma.refreshToken.findFirst({
      where: {
        userId: payload.userId,
        token: data.refreshToken,
        isRevoked: false,
        expiresAt: { gt: new Date() }
      }
    });
    
    if (!savedToken) {
      throw new AuthError("Refresh token หมดอายุหรือถูกเพิกถอน", 401, "TOKEN_EXPIRED");
    }
    
    const user = await prisma.user.findUnique({
      where: { id: payload.userId }
    });
    
    if (!user || !user.isActive) {
      throw new AuthError("บัญชีผู้ใช้ไม่ถูกต้อง", 401, "INVALID_USER");
    }
    
    // Rotate refresh token
    await prisma.refreshToken.update({
      where: { id: savedToken.id },
      data: { isRevoked: true }
    });
    
    const tokens = this.generateTokens(user);
    await this.saveRefreshToken(user.id, tokens.refreshToken);
    
    return tokens;
  }
  
  async logout(userId: string, refreshToken?: string): Promise<void> {
    if (refreshToken) {
      await prisma.refreshToken.updateMany({
        where: { userId, token: refreshToken },
        data: { isRevoked: true }
      });
    } else {
      // Logout from all devices
      await prisma.refreshToken.updateMany({
        where: { userId },
        data: { isRevoked: true }
      });
    }
  }
  
  async verifyEmail(token: string): Promise<void> {
    const user = await prisma.user.findFirst({
      where: { emailVerificationToken: token }
    });
    
    if (!user) {
      throw new AuthError("Token ไม่ถูกต้อง", 400, "INVALID_TOKEN");
    }
    
    await prisma.user.update({
      where: { id: user.id },
      data: {
        isEmailVerified: true,
        emailVerificationToken: null
      }
    });
  }
  
  async forgotPassword(email: string): Promise<void> {
    const user = await prisma.user.findUnique({
      where: { email: email.toLowerCase() }
    });
    
    // ไม่บอกว่าพบหรือไม่พบ (security)
    if (!user) return;
    
    const resetToken = crypto.randomBytes(32).toString("hex");
    const resetTokenHash = crypto
      .createHash("sha256")
      .update(resetToken)
      .digest("hex");
    
    await prisma.user.update({
      where: { id: user.id },
      data: {
        passwordResetToken: resetTokenHash,
        passwordResetExpires: new Date(Date.now() + 60 * 60 * 1000) // 1 hour
      }
    });
    
    await this.sendPasswordResetEmail(user.email, resetToken);
  }
  
  async resetPassword(token: string, newPassword: string): Promise<void> {
    const tokenHash = crypto
      .createHash("sha256")
      .update(token)
      .digest("hex");
    
    const user = await prisma.user.findFirst({
      where: {
        passwordResetToken: tokenHash,
        passwordResetExpires: { gt: new Date() }
      }
    });
    
    if (!user) {
      throw new AuthError("Token ไม่ถูกต้องหรือหมดอายุ", 400, "INVALID_TOKEN");
    }
    
    const passwordHash = await bcrypt.hash(newPassword, this.SALT_ROUNDS);
    
    await prisma.user.update({
      where: { id: user.id },
      data: {
        passwordHash,
        passwordResetToken: null,
        passwordResetExpires: null
      }
    });
    
    // Revoke all refresh tokens
    await this.logout(user.id);
  }
  
  generateTokens(user: { id: string; email: string; role: string }): AuthTokens {
    const accessPayload: TokenPayload = {
      userId: user.id,
      email: user.email,
      role: user.role,
      type: "access"
    };
    
    const refreshPayload: TokenPayload = {
      userId: user.id,
      email: user.email,
      role: user.role,
      type: "refresh"
    };
    
    const accessToken = jwt.sign(accessPayload, env.JWT_SECRET, {
      expiresIn: env.JWT_EXPIRES_IN as any
    });
    
    const refreshToken = jwt.sign(refreshPayload, env.JWT_REFRESH_SECRET, {
      expiresIn: env.JWT_REFRESH_EXPIRES_IN as any
    });
    
    return {
      accessToken,
      refreshToken,
      expiresIn: 15 * 60 // 15 minutes in seconds
    };
  }
  
  verifyAccessToken(token: string): TokenPayload {
    try {
      return jwt.verify(token, env.JWT_SECRET) as TokenPayload;
    } catch {
      throw new AuthError("Access token ไม่ถูกต้อง", 401, "INVALID_TOKEN");
    }
  }
  
  private async saveRefreshToken(userId: string, token: string): Promise<void> {
    await prisma.refreshToken.create({
      data: {
        userId,
        token,
        expiresAt: new Date(Date.now() + this.REFRESH_TOKEN_EXPIRY)
      }
    });
  }
  
  private async recordFailedLogin(userId: string): Promise<void> {
    const user = await prisma.user.findUnique({ where: { id: userId } });
    if (!user) return;
    
    const attempts = (user.failedLoginAttempts || 0) + 1;
    const shouldLock = attempts >= 5;
    
    await prisma.user.update({
      where: { id: userId },
      data: {
        failedLoginAttempts: attempts,
        isActive: shouldLock ? false : undefined,
        lockedUntil: shouldLock
          ? new Date(Date.now() + 30 * 60 * 1000) // 30 minutes
          : undefined
      }
    });
  }
  
  private async sendVerificationEmail(email: string, token: string): Promise<void> {
    // Implementation with nodemailer
    console.log(`Send verification email to ${email} with token ${token}`);
  }
  
  private async sendPasswordResetEmail(email: string, token: string): Promise<void> {
    console.log(`Send password reset email to ${email} with token ${token}`);
  }
  
  private mapToUserProfile(user: any): UserProfile {
    return {
      id: user.id,
      email: user.email,
      firstName: user.firstName,
      lastName: user.lastName,
      phone: user.phone,
      role: user.role,
      isEmailVerified: user.isEmailVerified
    };
  }
}

export const authService = new AuthService();
```

---

## Product Service

### src/services/ProductService.ts

```typescript
import { prisma } from "../config/database";
import type {
  CreateProductRequest,
  UpdateProductRequest,
  ProductSearchRequest
} from "../types/requests";
import type {
  ProductDetail,
  ProductListItem,
  PaginatedResponse
} from "../types/responses";

export class ProductNotFoundError extends Error {
  constructor(id: string) {
    super(`Product ${id} not found`);
    this.name = "ProductNotFoundError";
  }
}

export class ProductService {
  async createProduct(data: CreateProductRequest, vendorId?: string): Promise<ProductDetail> {
    const slug = this.generateSlug(data.name);
    
    const product = await prisma.product.create({
      data: {
        sku: data.sku,
        name: data.name,
        slug,
        description: data.description,
        price: data.price,
        compareAtPrice: data.compareAtPrice,
        costPrice: data.costPrice,
        categoryId: data.categoryId,
        vendorId,
        tags: data.tags || [],
        weight: data.weight,
        dimensions: data.dimensions ? JSON.stringify(data.dimensions) : undefined,
        inventory: {
          create: {
            quantity: 0,
            reservedQuantity: 0,
            lowStockThreshold: 5,
            trackInventory: true,
            allowBackorder: false
          }
        }
      },
      include: {
        category: true,
        images: true,
        variants: true,
        inventory: true
      }
    });
    
    return this.mapToProductDetail(product);
  }
  
  async updateProduct(
    id: string,
    data: UpdateProductRequest
  ): Promise<ProductDetail> {
    const product = await prisma.product.findUnique({ where: { id } });
    
    if (!product) {
      throw new ProductNotFoundError(id);
    }
    
    const updated = await prisma.product.update({
      where: { id },
      data: {
        ...data,
        updatedAt: new Date()
      },
      include: {
        category: true,
        images: true,
        variants: true,
        inventory: true
      }
    });
    
    return this.mapToProductDetail(updated);
  }
  
  async getProductById(id: string): Promise<ProductDetail> {
    const product = await prisma.product.findUnique({
      where: { id, isActive: true },
      include: {
        category: true,
        images: { orderBy: { sortOrder: "asc" } },
        variants: { where: { isActive: true } },
        inventory: true
      }
    });
    
    if (!product) {
      throw new ProductNotFoundError(id);
    }
    
    return this.mapToProductDetail(product);
  }
  
  async searchProducts(
    params: ProductSearchRequest
  ): Promise<PaginatedResponse<ProductListItem>> {
    const {
      query,
      categoryId,
      minPrice,
      maxPrice,
      tags,
      sortBy = "newest",
      page = 1,
      limit = 20,
      inStock
    } = params;
    
    const where: any = {
      isActive: true,
    };
    
    if (query) {
      where.OR = [
        { name: { contains: query, mode: "insensitive" } },
        { description: { contains: query, mode: "insensitive" } },
        { sku: { contains: query, mode: "insensitive" } }
      ];
    }
    
    if (categoryId) {
      // Include subcategories
      const categoryIds = await this.getCategoryAndSubIds(categoryId);
      where.categoryId = { in: categoryIds };
    }
    
    if (minPrice !== undefined || maxPrice !== undefined) {
      where.price = {};
      if (minPrice !== undefined) where.price.gte = minPrice;
      if (maxPrice !== undefined) where.price.lte = maxPrice;
    }
    
    if (tags && tags.length > 0) {
      where.tags = { hasSome: tags };
    }
    
    if (inStock) {
      where.inventory = {
        availableQuantity: { gt: 0 }
      };
    }
    
    // Build orderBy
    const orderBy = this.buildOrderBy(sortBy);
    
    const [total, products] = await Promise.all([
      prisma.product.count({ where }),
      prisma.product.findMany({
        where,
        orderBy,
        skip: (page - 1) * limit,
        take: limit,
        include: {
          category: true,
          images: {
            where: { isMain: true },
            take: 1
          },
          inventory: true
        }
      })
    ]);
    
    return {
      success: true,
      data: products.map(this.mapToProductListItem),
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page * limit < total,
        hasPrev: page > 1
      }
    };
  }
  
  async updateInventory(
    productId: string,
    quantity: number,
    operation: "add" | "subtract" | "set"
  ): Promise<void> {
    const inventory = await prisma.inventory.findUnique({
      where: { productId }
    });
    
    if (!inventory) {
      throw new Error(`Inventory for product ${productId} not found`);
    }
    
    let newQuantity: number;
    
    switch (operation) {
      case "add":
        newQuantity = inventory.quantity + quantity;
        break;
      case "subtract":
        newQuantity = inventory.quantity - quantity;
        if (newQuantity < 0 && !inventory.allowBackorder) {
          throw new Error("Insufficient stock");
        }
        break;
      case "set":
        newQuantity = quantity;
        break;
    }
    
    await prisma.inventory.update({
      where: { productId },
      data: {
        quantity: newQuantity!,
        availableQuantity: newQuantity! - inventory.reservedQuantity
      }
    });
  }
  
  async addProductImages(
    productId: string,
    images: Array<{ url: string; altText?: string; isMain?: boolean }>
  ): Promise<void> {
    const existingCount = await prisma.productImage.count({ where: { productId } });
    
    await prisma.productImage.createMany({
      data: images.map((img, index) => ({
        productId,
        url: img.url,
        altText: img.altText,
        sortOrder: existingCount + index,
        isMain: img.isMain || existingCount === 0 && index === 0
      }))
    });
    
    // ถ้ามีรูป main ใหม่ ให้ unset รูปเดิม
    const mainImage = images.find(img => img.isMain);
    if (mainImage) {
      await prisma.productImage.updateMany({
        where: { productId, url: { not: mainImage.url } },
        data: { isMain: false }
      });
    }
  }
  
  private async getCategoryAndSubIds(categoryId: string): Promise<string[]> {
    const ids = [categoryId];
    
    const children = await prisma.category.findMany({
      where: { parentId: categoryId },
      select: { id: true }
    });
    
    for (const child of children) {
      const subIds = await this.getCategoryAndSubIds(child.id);
      ids.push(...subIds);
    }
    
    return ids;
  }
  
  private buildOrderBy(sortBy: string): any {
    switch (sortBy) {
      case "price_asc": return { price: "asc" };
      case "price_desc": return { price: "desc" };
      case "rating": return { rating: "desc" };
      case "popular": return { soldCount: "desc" };
      case "newest":
      default:
        return { createdAt: "desc" };
    }
  }
  
  private generateSlug(name: string): string {
    return name
      .toLowerCase()
      .replace(/[^\w\s-]/g, "")
      .replace(/\s+/g, "-")
      .replace(/-+/g, "-")
      .trim();
  }
  
  private mapToProductDetail(product: any): ProductDetail {
    const mainImage = product.images?.find((img: any) => img.isMain)?.url;
    
    return {
      id: product.id,
      name: product.name,
      slug: product.slug,
      sku: product.sku,
      description: product.description,
      price: product.price,
      compareAtPrice: product.compareAtPrice,
      mainImage,
      images: product.images || [],
      rating: product.rating || 0,
      reviewCount: product.reviewCount || 0,
      isInStock: (product.inventory?.availableQuantity || 0) > 0,
      category: {
        id: product.category?.id,
        name: product.category?.name
      },
      variants: product.variants || [],
      attributes: product.attributes || [],
      tags: product.tags || [],
      weight: product.weight,
      dimensions: product.dimensions
    };
  }
  
  private mapToProductListItem(product: any): ProductListItem {
    const mainImage = product.images?.[0]?.url;
    
    return {
      id: product.id,
      name: product.name,
      slug: product.slug,
      price: product.price,
      compareAtPrice: product.compareAtPrice,
      mainImage,
      rating: product.rating || 0,
      reviewCount: product.reviewCount || 0,
      isInStock: (product.inventory?.availableQuantity || 0) > 0,
      category: {
        id: product.category?.id,
        name: product.category?.name
      }
    };
  }
}

export const productService = new ProductService();
```

---

## Cart Service

### src/services/CartService.ts

```typescript
import { prisma } from "../config/database";
import type { AddToCartRequest, UpdateCartItemRequest } from "../types/requests";
import type { Cart } from "../types/models";

export class CartError extends Error {
  constructor(message: string, public statusCode: number = 400) {
    super(message);
    this.name = "CartError";
  }
}

export class CartService {
  async getOrCreateCart(userId?: string, sessionId?: string): Promise<Cart> {
    if (!userId && !sessionId) {
      throw new CartError("ต้องระบุ userId หรือ sessionId");
    }
    
    const where = userId ? { userId } : { sessionId };
    
    let cart = await prisma.cart.findFirst({
      where: { ...where, expiresAt: { gt: new Date() } },
      include: {
        items: {
          include: {
            product: {
              include: {
                images: { where: { isMain: true }, take: 1 },
                inventory: true
              }
            },
            variant: true
          }
        }
      }
    });
    
    if (!cart) {
      cart = await prisma.cart.create({
        data: {
          userId,
          sessionId,
          expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
        },
        include: {
          items: {
            include: {
              product: {
                include: {
                  images: { where: { isMain: true }, take: 1 },
                  inventory: true
                }
              },
              variant: true
            }
          }
        }
      });
    }
    
    return this.calculateCartTotals(cart);
  }
  
  async addItem(
    cartId: string,
    data: AddToCartRequest
  ): Promise<Cart> {
    // ตรวจสอบ product
    const product = await prisma.product.findUnique({
      where: { id: data.productId, isActive: true },
      include: {
        inventory: true,
        variants: data.variantId
          ? { where: { id: data.variantId } }
          : undefined
      }
    });
    
    if (!product) {
      throw new CartError("ไม่พบสินค้า", 404);
    }
    
    // ตรวจสอบ stock
    const availableStock = product.inventory?.availableQuantity || 0;
    if (availableStock < data.quantity && !product.inventory?.allowBackorder) {
      throw new CartError(`สินค้าคงเหลือไม่เพียงพอ (มี ${availableStock} ชิ้น)`);
    }
    
    const price = data.variantId
      ? product.variants?.[0]?.price ?? product.price
      : product.price;
    
    // ตรวจสอบว่ามีสินค้านี้ในตะกร้าแล้วหรือไม่
    const existingItem = await prisma.cartItem.findFirst({
      where: {
        cartId,
        productId: data.productId,
        variantId: data.variantId || null
      }
    });
    
    if (existingItem) {
      const newQuantity = existingItem.quantity + data.quantity;
      
      if (newQuantity > availableStock && !product.inventory?.allowBackorder) {
        throw new CartError(`ไม่สามารถเพิ่มได้ มีสินค้าในตะกร้าแล้ว ${existingItem.quantity} ชิ้น`);
      }
      
      await prisma.cartItem.update({
        where: { id: existingItem.id },
        data: {
          quantity: newQuantity,
          total: price * newQuantity,
          notes: data.notes
        }
      });
    } else {
      await prisma.cartItem.create({
        data: {
          cartId,
          productId: data.productId,
          variantId: data.variantId,
          quantity: data.quantity,
          price,
          total: price * data.quantity,
          notes: data.notes
        }
      });
    }
    
    return this.getCart(cartId);
  }
  
  async updateItem(
    cartId: string,
    itemId: string,
    data: UpdateCartItemRequest
  ): Promise<Cart> {
    const item = await prisma.cartItem.findFirst({
      where: { id: itemId, cartId },
      include: {
        product: { include: { inventory: true } }
      }
    });
    
    if (!item) {
      throw new CartError("ไม่พบรายการในตะกร้า", 404);
    }
    
    if (data.quantity === 0) {
      return this.removeItem(cartId, itemId);
    }
    
    const stock = item.product?.inventory?.availableQuantity || 0;
    if (data.quantity > stock && !item.product?.inventory?.allowBackorder) {
      throw new CartError(`สินค้าคงเหลือไม่เพียงพอ (มี ${stock} ชิ้น)`);
    }
    
    await prisma.cartItem.update({
      where: { id: itemId },
      data: {
        quantity: data.quantity,
        total: item.price * data.quantity,
        notes: data.notes
      }
    });
    
    return this.getCart(cartId);
  }
  
  async removeItem(cartId: string, itemId: string): Promise<Cart> {
    await prisma.cartItem.deleteMany({
      where: { id: itemId, cartId }
    });
    
    return this.getCart(cartId);
  }
  
  async clearCart(cartId: string): Promise<void> {
    await prisma.cartItem.deleteMany({ where: { cartId } });
    await prisma.cart.update({
      where: { id: cartId },
      data: { couponCode: null, discount: 0 }
    });
  }
  
  async applyCoupon(cartId: string, couponCode: string): Promise<Cart> {
    const cart = await this.getCart(cartId);
    
    const coupon = await prisma.coupon.findUnique({
      where: { code: couponCode.toUpperCase(), isActive: true }
    });
    
    if (!coupon) {
      throw new CartError("โค้ดส่วนลดไม่ถูกต้อง", 404);
    }
    
    // ตรวจสอบวันหมดอายุ
    if (coupon.expiresAt && coupon.expiresAt < new Date()) {
      throw new CartError("โค้ดส่วนลดหมดอายุแล้ว");
    }
    
    // ตรวจสอบจำนวนการใช้งาน
    if (coupon.usageLimit && coupon.usedCount >= coupon.usageLimit) {
      throw new CartError("โค้ดส่วนลดถูกใช้งานเต็มจำนวนแล้ว");
    }
    
    // ตรวจสอบยอดขั้นต่ำ
    if (coupon.minOrderAmount && cart.subtotal < coupon.minOrderAmount) {
      throw new CartError(
        `ต้องสั่งซื้อขั้นต่ำ ฿${coupon.minOrderAmount.toLocaleString()}`
      );
    }
    
    // คำนวณส่วนลด
    let discount = 0;
    switch (coupon.type) {
      case "FIXED":
        discount = Math.min(coupon.value, cart.subtotal);
        break;
      case "PERCENTAGE":
        discount = (cart.subtotal * coupon.value) / 100;
        if (coupon.maxDiscountAmount) {
          discount = Math.min(discount, coupon.maxDiscountAmount);
        }
        break;
      case "FREE_SHIPPING":
        discount = cart.shipping;
        break;
    }
    
    await prisma.cart.update({
      where: { id: cartId },
      data: { couponCode: coupon.code, discount }
    });
    
    return this.getCart(cartId);
  }
  
  async mergeGuestCart(guestCartId: string, userId: string): Promise<Cart> {
    const guestCart = await prisma.cart.findUnique({
      where: { id: guestCartId },
      include: { items: true }
    });
    
    if (!guestCart) return this.getOrCreateCart(userId);
    
    const userCart = await this.getOrCreateCart(userId);
    
    // Merge items
    for (const item of guestCart.items) {
      await this.addItem(userCart.id, {
        productId: item.productId,
        variantId: item.variantId || undefined,
        quantity: item.quantity,
        notes: item.notes || undefined
      }).catch(err => {
        // Skip items with errors (out of stock, etc.)
        console.warn(`Failed to merge cart item: ${err.message}`);
      });
    }
    
    // Delete guest cart
    await prisma.cart.delete({ where: { id: guestCartId } });
    
    return this.getCart(userCart.id);
  }
  
  async getCart(cartId: string): Promise<Cart> {
    const cart = await prisma.cart.findUnique({
      where: { id: cartId },
      include: {
        items: {
          include: {
            product: {
              include: {
                images: { where: { isMain: true }, take: 1 },
                inventory: true
              }
            },
            variant: true
          }
        }
      }
    });
    
    if (!cart) throw new CartError("ไม่พบตะกร้าสินค้า", 404);
    
    return this.calculateCartTotals(cart);
  }
  
  private calculateCartTotals(cart: any): Cart {
    const subtotal = cart.items.reduce(
      (sum: number, item: any) => sum + item.total,
      0
    );
    
    const shipping = this.calculateShipping(subtotal, cart.items);
    const tax = (subtotal - (cart.discount || 0)) * 0.07; // VAT 7%
    const total = subtotal - (cart.discount || 0) + shipping + tax;
    
    return {
      ...cart,
      subtotal,
      shipping,
      tax,
      total: Math.max(0, total)
    };
  }
  
  private calculateShipping(subtotal: number, items: any[]): number {
    // Free shipping over 1000 THB
    if (subtotal >= 1000) return 0;
    
    // Calculate based on weight
    const totalWeight = items.reduce(
      (sum, item) => sum + (item.product?.weight || 0.1) * item.quantity,
      0
    );
    
    // Base rate: 50 THB for first kg, 20 THB per additional kg
    const baseRate = 50;
    const additionalRate = Math.max(0, totalWeight - 1) * 20;
    
    return baseRate + additionalRate;
  }
}

export const cartService = new CartService();
```

---

## Order Service

### src/services/OrderService.ts

```typescript
import { prisma } from "../config/database";
import { cartService } from "./CartService";
import type { CreateOrderRequest, UpdateOrderStatusRequest } from "../types/requests";
import type { Order, OrderStatus } from "../types/models";

export class OrderError extends Error {
  constructor(
    message: string,
    public statusCode: number = 400,
    public code: string = "ORDER_ERROR"
  ) {
    super(message);
    this.name = "OrderError";
  }
}

export class OrderService {
  async createOrder(userId: string, data: CreateOrderRequest): Promise<Order> {
    // Get cart
    const cart = await cartService.getCart(data.cartId);
    
    if (cart.userId !== userId) {
      throw new OrderError("ไม่มีสิทธิ์เข้าถึงตะกร้าสินค้านี้", 403);
    }
    
    if (cart.items.length === 0) {
      throw new OrderError("ตะกร้าสินค้าว่างเปล่า");
    }
    
    // Validate shipping address
    const shippingAddress = await prisma.address.findFirst({
      where: { id: data.shippingAddressId, userId }
    });
    
    if (!shippingAddress) {
      throw new OrderError("ไม่พบที่อยู่จัดส่ง", 404);
    }
    
    // Check stock for all items
    await this.validateStock(cart.items);
    
    // Generate order number
    const orderNumber = await this.generateOrderNumber();
    
    // Create order in transaction
    const order = await prisma.$transaction(async (tx) => {
      // Create order
      const newOrder = await tx.order.create({
        data: {
          orderNumber,
          userId,
          status: "PENDING",
          paymentStatus: "PENDING",
          shippingAddressSnapshot: JSON.stringify(shippingAddress),
          subtotal: cart.subtotal,
          discount: cart.discount,
          shippingFee: cart.shipping,
          tax: cart.tax,
          total: cart.total,
          couponCode: cart.couponCode,
          notes: data.notes,
          items: {
            create: cart.items.map(item => ({
              productId: item.productId,
              variantId: item.variantId,
              quantity: item.quantity,
              price: item.price,
              total: item.total,
              productSnapshot: JSON.stringify({
                name: item.product?.name,
                sku: item.product?.sku || item.variant?.sku,
                price: item.price,
                imageUrl: item.product?.images?.[0]?.url,
                options: item.variant?.options
              })
            }))
          }
        },
        include: {
          items: true,
          user: true
        }
      });
      
      // Reserve inventory
      for (const item of cart.items) {
        await tx.inventory.update({
          where: { productId: item.productId },
          data: {
            reservedQuantity: { increment: item.quantity },
            availableQuantity: { decrement: item.quantity }
          }
        });
      }
      
      // Update coupon usage
      if (cart.couponCode) {
        await tx.coupon.update({
          where: { code: cart.couponCode },
          data: { usedCount: { increment: 1 } }
        });
      }
      
      // Clear cart
      await tx.cartItem.deleteMany({ where: { cartId: cart.id } });
      
      return newOrder;
    });
    
    // Send order confirmation email (async)
    this.sendOrderConfirmation(order).catch(console.error);
    
    return order as unknown as Order;
  }
  
  async getOrderById(orderId: string, userId?: string): Promise<Order> {
    const where: any = { id: orderId };
    if (userId) where.userId = userId;
    
    const order = await prisma.order.findFirst({
      where,
      include: {
        items: {
          include: {
            product: {
              include: { images: { where: { isMain: true }, take: 1 } }
            }
          }
        },
        user: true,
        payment: true
      }
    });
    
    if (!order) {
      throw new OrderError("ไม่พบคำสั่งซื้อ", 404);
    }
    
    return order as unknown as Order;
  }
  
  async getUserOrders(
    userId: string,
    page: number = 1,
    limit: number = 10,
    status?: OrderStatus
  ) {
    const where: any = { userId };
    if (status) where.status = status;
    
    const [total, orders] = await Promise.all([
      prisma.order.count({ where }),
      prisma.order.findMany({
        where,
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          items: {
            take: 3, // Show first 3 items
            include: {
              product: {
                include: { images: { where: { isMain: true }, take: 1 } }
              }
            }
          }
        }
      })
    ]);
    
    return {
      orders,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async updateOrderStatus(
    orderId: string,
    data: UpdateOrderStatusRequest
  ): Promise<Order> {
    const order = await prisma.order.findUnique({ where: { id: orderId } });
    
    if (!order) {
      throw new OrderError("ไม่พบคำสั่งซื้อ", 404);
    }
    
    // Validate status transition
    this.validateStatusTransition(order.status as OrderStatus, data.status);
    
    const updated = await prisma.order.update({
      where: { id: orderId },
      data: {
        status: data.status,
        trackingNumber: data.trackingNumber,
        shippingProvider: data.shippingProvider,
        deliveredAt: data.status === "DELIVERED" ? new Date() : undefined,
        cancelledAt: data.status === "CANCELLED" ? new Date() : undefined,
        cancelReason: data.status === "CANCELLED" ? data.notes : undefined
      },
      include: { items: true, user: true }
    });
    
    // Release inventory on cancel
    if (data.status === "CANCELLED") {
      await this.releaseInventory(orderId);
    }
    
    // Send status update notification
    this.sendStatusUpdateNotification(updated).catch(console.error);
    
    return updated as unknown as Order;
  }
  
  async cancelOrder(orderId: string, userId: string, reason: string): Promise<Order> {
    const order = await this.getOrderById(orderId, userId);
    
    if (!["PENDING", "CONFIRMED"].includes(order.status as string)) {
      throw new OrderError("ไม่สามารถยกเลิกคำสั่งซื้อที่กำลังดำเนินการได้");
    }
    
    return this.updateOrderStatus(orderId, {
      status: "CANCELLED" as OrderStatus,
      notes: reason
    });
  }
  
  private async validateStock(items: any[]): Promise<void> {
    for (const item of items) {
      const inventory = await prisma.inventory.findUnique({
        where: { productId: item.productId }
      });
      
      if (!inventory) continue;
      
      const available = inventory.availableQuantity;
      
      if (available < item.quantity && !inventory.allowBackorder) {
        const product = await prisma.product.findUnique({
          where: { id: item.productId },
          select: { name: true }
        });
        
        throw new OrderError(
          `สินค้า "${product?.name}" มีสต็อกไม่เพียงพอ (มี ${available} ชิ้น)`
        );
      }
    }
  }
  
  private async releaseInventory(orderId: string): Promise<void> {
    const items = await prisma.orderItem.findMany({ where: { orderId } });
    
    for (const item of items) {
      await prisma.inventory.update({
        where: { productId: item.productId },
        data: {
          reservedQuantity: { decrement: item.quantity },
          availableQuantity: { increment: item.quantity }
        }
      });
    }
  }
  
  private validateStatusTransition(
    current: OrderStatus,
    next: OrderStatus
  ): void {
    const validTransitions: Record<string, string[]> = {
      PENDING: ["CONFIRMED", "CANCELLED"],
      CONFIRMED: ["PROCESSING", "CANCELLED"],
      PROCESSING: ["SHIPPED", "CANCELLED"],
      SHIPPED: ["DELIVERED"],
      DELIVERED: ["REFUNDED"],
      CANCELLED: [],
      REFUNDED: []
    };
    
    if (!validTransitions[current]?.includes(next)) {
      throw new OrderError(
        `ไม่สามารถเปลี่ยนสถานะจาก ${current} เป็น ${next} ได้`
      );
    }
  }
  
  private async generateOrderNumber(): Promise<string> {
    const date = new Date();
    const year = date.getFullYear().toString().slice(-2);
    const month = String(date.getMonth() + 1).padStart(2, "0");
    const day = String(date.getDate()).padStart(2, "0");
    
    const count = await prisma.order.count({
      where: {
        createdAt: {
          gte: new Date(date.setHours(0, 0, 0, 0)),
          lt: new Date(date.setHours(23, 59, 59, 999))
        }
      }
    });
    
    const sequence = String(count + 1).padStart(4, "0");
    return `ORD${year}${month}${day}${sequence}`;
  }
  
  private async sendOrderConfirmation(order: any): Promise<void> {
    console.log(`Sending order confirmation for order ${order.orderNumber}`);
  }
  
  private async sendStatusUpdateNotification(order: any): Promise<void> {
    console.log(`Sending status update for order ${order.orderNumber}: ${order.status}`);
  }
}

export const orderService = new OrderService();
```

---

## Controllers และ Routes

### src/controllers/AuthController.ts

```typescript
import { Request, Response, NextFunction } from "express";
import { authService } from "../services/AuthService";
import type {
  RegisterRequest,
  LoginRequest,
  RefreshTokenRequest
} from "../types/requests";

export class AuthController {
  async register(req: Request<{}, {}, RegisterRequest>, res: Response, next: NextFunction) {
    try {
      const result = await authService.register(req.body);
      res.status(201).json(result);
    } catch (error) {
      next(error);
    }
  }
  
  async login(req: Request<{}, {}, LoginRequest>, res: Response, next: NextFunction) {
    try {
      const result = await authService.login(req.body);
      
      // Set HTTP-only cookie for refresh token
      res.cookie("refreshToken", result.data?.refreshToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === "production",
        sameSite: "strict",
        maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
      });
      
      res.json({
        ...result,
        data: {
          accessToken: result.data?.accessToken,
          expiresIn: result.data?.expiresIn
        }
      });
    } catch (error) {
      next(error);
    }
  }
  
  async refreshToken(req: Request, res: Response, next: NextFunction) {
    try {
      const refreshToken =
        req.cookies.refreshToken ||
        req.body.refreshToken;
      
      if (!refreshToken) {
        return res.status(401).json({ success: false, message: "กรุณาเข้าสู่ระบบใหม่" });
      }
      
      const tokens = await authService.refreshToken({ refreshToken });
      
      res.cookie("refreshToken", tokens.refreshToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === "production",
        sameSite: "strict",
        maxAge: 7 * 24 * 60 * 60 * 1000
      });
      
      res.json({
        success: true,
        data: {
          accessToken: tokens.accessToken,
          expiresIn: tokens.expiresIn
        }
      });
    } catch (error) {
      next(error);
    }
  }
  
  async logout(req: Request, res: Response, next: NextFunction) {
    try {
      const refreshToken = req.cookies.refreshToken;
      await authService.logout(req.user!.userId, refreshToken);
      
      res.clearCookie("refreshToken");
      res.json({ success: true, message: "ออกจากระบบสำเร็จ" });
    } catch (error) {
      next(error);
    }
  }
  
  async verifyEmail(req: Request, res: Response, next: NextFunction) {
    try {
      const { token } = req.params;
      await authService.verifyEmail(token);
      res.json({ success: true, message: "ยืนยัน email สำเร็จ" });
    } catch (error) {
      next(error);
    }
  }
}

export const authController = new AuthController();
```

---

### src/middlewares/auth.ts

```typescript
import { Request, Response, NextFunction } from "express";
import { authService } from "../services/AuthService";

// Extend Express Request type
declare global {
  namespace Express {
    interface Request {
      user?: {
        userId: string;
        email: string;
        role: string;
      };
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith("Bearer ")) {
    return res.status(401).json({
      success: false,
      message: "กรุณาเข้าสู่ระบบ"
    });
  }
  
  const token = authHeader.slice(7);
  
  try {
    const payload = authService.verifyAccessToken(token);
    req.user = {
      userId: payload.userId,
      email: payload.email,
      role: payload.role
    };
    next();
  } catch {
    res.status(401).json({
      success: false,
      message: "Token ไม่ถูกต้องหรือหมดอายุ"
    });
  }
}

export function optionalAuth(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (authHeader?.startsWith("Bearer ")) {
    const token = authHeader.slice(7);
    try {
      const payload = authService.verifyAccessToken(token);
      req.user = {
        userId: payload.userId,
        email: payload.email,
        role: payload.role
      };
    } catch {
      // ignore
    }
  }
  
  next();
}

export function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ success: false, message: "กรุณาเข้าสู่ระบบ" });
    }
    
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: "ไม่มีสิทธิ์ดำเนินการ"
      });
    }
    
    next();
  };
}
```

---

### src/routes/products.ts

```typescript
import { Router } from "express";
import multer from "multer";
import { productService } from "../services/ProductService";
import { authenticate, authorize } from "../middlewares/auth";

const router = Router();
const upload = multer({ dest: "uploads/" });

// Public routes
router.get("/", async (req, res, next) => {
  try {
    const result = await productService.searchProducts({
      query: req.query.q as string,
      categoryId: req.query.categoryId as string,
      minPrice: req.query.minPrice ? Number(req.query.minPrice) : undefined,
      maxPrice: req.query.maxPrice ? Number(req.query.maxPrice) : undefined,
      sortBy: req.query.sortBy as any,
      page: Number(req.query.page) || 1,
      limit: Number(req.query.limit) || 20,
      inStock: req.query.inStock === "true"
    });
    res.json(result);
  } catch (error) {
    next(error);
  }
});

router.get("/:id", async (req, res, next) => {
  try {
    const product = await productService.getProductById(req.params.id);
    res.json({ success: true, data: product });
  } catch (error) {
    next(error);
  }
});

// Admin routes
router.post(
  "/",
  authenticate,
  authorize("ADMIN", "VENDOR"),
  async (req, res, next) => {
    try {
      const product = await productService.createProduct(req.body, req.user!.userId);
      res.status(201).json({ success: true, data: product });
    } catch (error) {
      next(error);
    }
  }
);

router.put(
  "/:id",
  authenticate,
  authorize("ADMIN", "VENDOR"),
  async (req, res, next) => {
    try {
      const product = await productService.updateProduct(req.params.id, req.body);
      res.json({ success: true, data: product });
    } catch (error) {
      next(error);
    }
  }
);

router.post(
  "/:id/images",
  authenticate,
  authorize("ADMIN", "VENDOR"),
  upload.array("images", 10),
  async (req, res, next) => {
    try {
      const files = req.files as Express.Multer.File[];
      const images = files.map(file => ({
        url: `/uploads/${file.filename}`,
        altText: req.body.altText
      }));
      
      await productService.addProductImages(req.params.id, images);
      res.json({ success: true, message: "อัปโหลดรูปภาพสำเร็จ" });
    } catch (error) {
      next(error);
    }
  }
);

export default router;
```

---

## Payment Service

### src/services/PaymentService.ts

```typescript
import Stripe from "stripe";
import { prisma } from "../config/database";
import { env } from "../config/environment";
import type { Order } from "../types/models";

export class PaymentService {
  private stripe: Stripe;
  
  constructor() {
    this.stripe = new Stripe(env.STRIPE_SECRET_KEY || "", {
      apiVersion: "2023-10-16"
    });
  }
  
  async createPaymentIntent(orderId: string): Promise<{
    clientSecret: string;
    paymentIntentId: string;
  }> {
    const order = await prisma.order.findUnique({
      where: { id: orderId },
      include: { user: true }
    });
    
    if (!order) throw new Error("ไม่พบคำสั่งซื้อ");
    
    const paymentIntent = await this.stripe.paymentIntents.create({
      amount: Math.round(order.total * 100), // Stripe ใช้ satang
      currency: "thb",
      metadata: {
        orderId: order.id,
        orderNumber: order.orderNumber,
        userId: order.userId
      },
      receipt_email: order.user?.email
    });
    
    // บันทึก payment record
    await prisma.payment.create({
      data: {
        orderId,
        amount: order.total,
        currency: "THB",
        method: "CREDIT_CARD",
        status: "PENDING",
        provider: "stripe",
        providerTransactionId: paymentIntent.id
      }
    });
    
    return {
      clientSecret: paymentIntent.client_secret!,
      paymentIntentId: paymentIntent.id
    };
  }
  
  async handleStripeWebhook(
    payload: Buffer,
    signature: string
  ): Promise<void> {
    let event: Stripe.Event;
    
    try {
      event = this.stripe.webhooks.constructEvent(
        payload,
        signature,
        env.STRIPE_WEBHOOK_SECRET || ""
      );
    } catch (err) {
      throw new Error(`Webhook signature verification failed: ${err}`);
    }
    
    switch (event.type) {
      case "payment_intent.succeeded":
        await this.handlePaymentSuccess(event.data.object as Stripe.PaymentIntent);
        break;
        
      case "payment_intent.payment_failed":
        await this.handlePaymentFailed(event.data.object as Stripe.PaymentIntent);
        break;
        
      case "charge.refunded":
        await this.handleRefund(event.data.object as Stripe.Charge);
        break;
    }
  }
  
  private async handlePaymentSuccess(
    paymentIntent: Stripe.PaymentIntent
  ): Promise<void> {
    const { orderId } = paymentIntent.metadata;
    
    await prisma.$transaction(async (tx) => {
      await tx.payment.update({
        where: { providerTransactionId: paymentIntent.id },
        data: {
          status: "PAID",
          paidAt: new Date()
        }
      });
      
      await tx.order.update({
        where: { id: orderId },
        data: {
          paymentStatus: "PAID",
          status: "CONFIRMED"
        }
      });
      
      // Confirm inventory deduction
      const items = await tx.orderItem.findMany({ where: { orderId } });
      
      for (const item of items) {
        await tx.inventory.update({
          where: { productId: item.productId },
          data: {
            quantity: { decrement: item.quantity },
            reservedQuantity: { decrement: item.quantity },
            soldCount: { increment: item.quantity }
          }
        });
        
        // Update product sold count
        await tx.product.update({
          where: { id: item.productId },
          data: { soldCount: { increment: item.quantity } }
        });
      }
    });
  }
  
  private async handlePaymentFailed(
    paymentIntent: Stripe.PaymentIntent
  ): Promise<void> {
    const { orderId } = paymentIntent.metadata;
    
    await prisma.$transaction([
      prisma.payment.update({
        where: { providerTransactionId: paymentIntent.id },
        data: {
          status: "FAILED",
          failureReason: paymentIntent.last_payment_error?.message
        }
      }),
      prisma.order.update({
        where: { id: orderId },
        data: { paymentStatus: "FAILED" }
      })
    ]);
  }
  
  private async handleRefund(charge: Stripe.Charge): Promise<void> {
    const payment = await prisma.payment.findFirst({
      where: { providerTransactionId: charge.payment_intent as string }
    });
    
    if (!payment) return;
    
    const refundAmount = charge.amount_refunded / 100;
    const isFullRefund = charge.refunded;
    
    await prisma.$transaction([
      prisma.payment.update({
        where: { id: payment.id },
        data: {
          status: isFullRefund ? "REFUNDED" : "PARTIALLY_REFUNDED"
        }
      }),
      prisma.order.update({
        where: { id: payment.orderId },
        data: {
          paymentStatus: isFullRefund ? "REFUNDED" : "PARTIALLY_REFUNDED",
          refundAmount,
          refundedAt: new Date()
        }
      })
    ]);
  }
}

export const paymentService = new PaymentService();
```

---

## Error Handler Middleware

### src/middlewares/errorHandler.ts

```typescript
import { Request, Response, NextFunction } from "express";
import { AuthError } from "../services/AuthService";
import { CartError } from "../services/CartService";
import { OrderError } from "../services/OrderService";
import { ProductNotFoundError } from "../services/ProductService";
import { ZodError } from "zod";

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  console.error(`[Error] ${req.method} ${req.path}:`, {
    message: error.message,
    stack: process.env.NODE_ENV === "development" ? error.stack : undefined
  });
  
  // Auth errors
  if (error instanceof AuthError) {
    res.status(error.statusCode).json({
      success: false,
      message: error.message,
      code: error.code
    });
    return;
  }
  
  // Cart errors
  if (error instanceof CartError) {
    res.status(error.statusCode).json({
      success: false,
      message: error.message
    });
    return;
  }
  
  // Order errors
  if (error instanceof OrderError) {
    res.status(error.statusCode).json({
      success: false,
      message: error.message,
      code: error.code
    });
    return;
  }
  
  // Not found errors
  if (error instanceof ProductNotFoundError) {
    res.status(404).json({
      success: false,
      message: error.message
    });
    return;
  }
  
  // Validation errors
  if (error instanceof ZodError) {
    res.status(400).json({
      success: false,
      message: "ข้อมูลไม่ถูกต้อง",
      errors: error.errors.map(e => ({
        field: e.path.join("."),
        message: e.message
      }))
    });
    return;
  }
  
  // Default server error
  res.status(500).json({
    success: false,
    message: process.env.NODE_ENV === "production"
      ? "เกิดข้อผิดพลาดภายในเซิร์ฟเวอร์"
      : error.message
  });
}
```

---

## Main Application

### src/index.ts

```typescript
import express from "express";
import cors from "cors";
import helmet from "helmet";
import cookieParser from "cookie-parser";
import rateLimit from "express-rate-limit";

import { env } from "./config/environment";
import { errorHandler } from "./middlewares/errorHandler";
import authRoutes from "./routes/auth";
import productRoutes from "./routes/products";
import cartRoutes from "./routes/cart";
import orderRoutes from "./routes/orders";

const app = express();

// Security middlewares
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(",") || "*",
  credentials: true
}));

// Rate limiting
app.use(rateLimit({
  windowMs: Number(env.RATE_LIMIT_WINDOW) * 60 * 1000,
  max: Number(env.RATE_LIMIT_MAX),
  message: {
    success: false,
    message: "คุณส่งคำขอมากเกินไป กรุณาลองใหม่อีกครั้ง"
  }
}));

// Body parsing
app.use(express.json({ limit: "10mb" }));
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

// Static files
app.use("/uploads", express.static("uploads"));

// Routes
app.use("/api/auth", authRoutes);
app.use("/api/products", productRoutes);
app.use("/api/cart", cartRoutes);
app.use("/api/orders", orderRoutes);

// Health check
app.get("/health", (req, res) => {
  res.json({
    status: "ok",
    timestamp: new Date().toISOString(),
    environment: env.NODE_ENV
  });
});

// Error handler
app.use(errorHandler);

const PORT = Number(env.PORT) || 3000;

app.listen(PORT, () => {
  console.log(`🚀 E-commerce API running on port ${PORT}`);
  console.log(`📘 Environment: ${env.NODE_ENV}`);
});

export default app;
```

---

## Prisma Schema

### prisma/schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id                     String    @id @default(cuid())
  email                  String    @unique
  passwordHash           String
  firstName              String
  lastName               String
  phone                  String?
  role                   String    @default("CUSTOMER")
  isEmailVerified        Boolean   @default(false)
  isActive               Boolean   @default(true)
  emailVerificationToken String?
  passwordResetToken     String?
  passwordResetExpires   DateTime?
  failedLoginAttempts    Int       @default(0)
  lockedUntil            DateTime?
  lastLoginAt            DateTime?
  
  addresses    Address[]
  orders       Order[]
  cart         Cart[]
  reviews      Review[]
  refreshTokens RefreshToken[]
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([email])
}

model Product {
  id              String    @id @default(cuid())
  sku             String    @unique
  name            String
  slug            String    @unique
  description     String
  price           Float
  compareAtPrice  Float?
  costPrice       Float
  categoryId      String
  vendorId        String?
  tags            String[]
  weight          Float?
  dimensions      String?   // JSON
  rating          Float     @default(0)
  reviewCount     Int       @default(0)
  soldCount       Int       @default(0)
  isActive        Boolean   @default(true)
  isFeatured      Boolean   @default(false)
  
  category    Category     @relation(fields: [categoryId], references: [id])
  images      ProductImage[]
  variants    ProductVariant[]
  inventory   Inventory?
  orderItems  OrderItem[]
  cartItems   CartItem[]
  reviews     Review[]
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([categoryId])
  @@index([slug])
}

model Inventory {
  id                 String  @id @default(cuid())
  productId          String  @unique
  quantity           Int     @default(0)
  reservedQuantity   Int     @default(0)
  availableQuantity  Int     @default(0)
  lowStockThreshold  Int     @default(5)
  trackInventory     Boolean @default(true)
  allowBackorder     Boolean @default(false)
  soldCount          Int     @default(0)
  
  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)
}

model Order {
  id                      String    @id @default(cuid())
  orderNumber             String    @unique
  userId                  String
  status                  String    @default("PENDING")
  paymentStatus           String    @default("PENDING")
  shippingAddressSnapshot String    // JSON
  subtotal                Float
  discount                Float     @default(0)
  shippingFee             Float     @default(0)
  tax                     Float     @default(0)
  total                   Float
  couponCode              String?
  notes                   String?
  trackingNumber          String?
  shippingProvider        String?
  estimatedDelivery       DateTime?
  deliveredAt             DateTime?
  cancelledAt             DateTime?
  cancelReason            String?
  refundAmount            Float?
  refundedAt              DateTime?
  
  user    User        @relation(fields: [userId], references: [id])
  items   OrderItem[]
  payment Payment?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([userId])
  @@index([orderNumber])
  @@index([status])
}
```

---

## สรุป

โปรเจกต์ E-commerce API นี้ครอบคลุม:

1. **Authentication**: Register, Login, JWT tokens, refresh tokens, email verification
2. **Product Management**: CRUD, search, inventory management, image upload
3. **Shopping Cart**: Add/remove items, apply coupons, calculate totals
4. **Order Processing**: Create orders, track status, inventory management
5. **Payment**: Stripe integration, webhooks, refunds
6. **Security**: Rate limiting, helmet, CORS, input validation

Technology Stack:
- Node.js + Express + TypeScript
- PostgreSQL + Prisma ORM
- Redis (caching)
- Stripe (payment)
- JWT (authentication)
- Zod (validation)

---

*จบ Part 57 - Real-world E-commerce API*
