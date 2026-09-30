# ตอนที่ 47: Prisma ORM

## บทนำ

Prisma เป็น Next-generation ORM สำหรับ Node.js และ TypeScript ที่มี type safety สูง ทำงานได้กับหลาย database เช่น PostgreSQL, MySQL, SQLite, MongoDB และ SQL Server Prisma แตกต่างจาก ORM อื่นตรงที่ generate TypeScript types โดยอัตโนมัติจาก schema

---

## 47.1 Prisma Setup

### การติดตั้ง

```bash
# ติดตั้ง Prisma
npm install prisma --save-dev
npm install @prisma/client

# หรือด้วย pnpm
pnpm add -D prisma
pnpm add @prisma/client

# Initialize Prisma
npx prisma init

# หรือระบุ database provider
npx prisma init --datasource-provider postgresql
npx prisma init --datasource-provider mysql
npx prisma init --datasource-provider sqlite
```

### โครงสร้าง Project หลังจาก Init

```
project/
├── prisma/
│   ├── schema.prisma    # Prisma schema file
│   ├── migrations/      # Migration files
│   └── seed.ts          # Seed file
├── src/
│   └── lib/
│       └── prisma.ts    # Prisma client singleton
├── .env                 # Environment variables
└── package.json
```

### .env

```env
# Database connection string
DATABASE_URL="postgresql://username:password@localhost:5432/mydb?schema=public"

# สำหรับ MySQL
# DATABASE_URL="mysql://username:password@localhost:3306/mydb"

# สำหรับ SQLite (development)
# DATABASE_URL="file:./dev.db"
```

### Prisma Client Singleton

```typescript
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;

export default prisma;
```

---

## 47.2 Schema Definition

### Prisma Schema ที่สมบูรณ์

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Enums
enum Role {
  USER
  ADMIN
  MODERATOR
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum ProductStatus {
  DRAFT
  ACTIVE
  ARCHIVED
}

// Models
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String
  password      String
  role          Role      @default(USER)
  emailVerified DateTime?
  image         String?
  bio           String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  deletedAt     DateTime? // Soft delete

  // Relations
  profile       Profile?
  posts         Post[]
  orders        Order[]
  reviews       Review[]
  sessions      Session[]
  addresses     Address[]
  
  // Indexes
  @@index([email])
  @@index([createdAt])
  @@map("users")
}

model Profile {
  id          String   @id @default(cuid())
  userId      String   @unique
  bio         String?
  website     String?
  location    String?
  birthDate   DateTime?
  phoneNumber String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  // Relations
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("profiles")
}

model Session {
  id           String   @id @default(cuid())
  userId       String
  token        String   @unique
  expiresAt    DateTime
  ipAddress    String?
  userAgent    String?
  createdAt    DateTime @default(now())

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
  @@index([token])
  @@map("sessions")
}

model Post {
  id          String        @id @default(cuid())
  title       String
  slug        String        @unique
  content     String
  excerpt     String?
  coverImage  String?
  published   Boolean       @default(false)
  publishedAt DateTime?
  authorId    String
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt
  deletedAt   DateTime?

  // Relations
  author      User          @relation(fields: [authorId], references: [id])
  categories  PostCategory[]
  tags        PostTag[]
  comments    Comment[]
  
  @@index([slug])
  @@index([authorId])
  @@index([published, publishedAt])
  @@map("posts")
}

model Category {
  id          String         @id @default(cuid())
  name        String
  slug        String         @unique
  description String?
  parentId    String?
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt

  parent      Category?      @relation("CategoryHierarchy", fields: [parentId], references: [id])
  children    Category[]     @relation("CategoryHierarchy")
  posts       PostCategory[]
  products    Product[]
  
  @@map("categories")
}

model PostCategory {
  postId      String
  categoryId  String

  post        Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  category    Category @relation(fields: [categoryId], references: [id], onDelete: Cascade)

  @@id([postId, categoryId])
  @@map("post_categories")
}

model Tag {
  id    String    @id @default(cuid())
  name  String    @unique
  slug  String    @unique
  posts PostTag[]
  
  @@map("tags")
}

model PostTag {
  postId String
  tagId  String

  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag    Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@map("post_tags")
}

model Comment {
  id        String    @id @default(cuid())
  content   String
  postId    String
  authorId  String
  parentId  String?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime?

  post      Post      @relation(fields: [postId], references: [id], onDelete: Cascade)
  parent    Comment?  @relation("CommentReplies", fields: [parentId], references: [id])
  replies   Comment[] @relation("CommentReplies")
  
  @@index([postId])
  @@map("comments")
}

model Product {
  id          String        @id @default(cuid())
  name        String
  slug        String        @unique
  description String
  price       Decimal       @db.Decimal(10, 2)
  comparePrice Decimal?     @db.Decimal(10, 2)
  cost        Decimal?      @db.Decimal(10, 2)
  sku         String?       @unique
  barcode     String?
  status      ProductStatus @default(DRAFT)
  stock       Int           @default(0)
  weight      Float?
  categoryId  String?
  images      String[]
  tags        String[]
  metadata    Json?
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt
  deletedAt   DateTime?

  category    Category?     @relation(fields: [categoryId], references: [id])
  orderItems  OrderItem[]
  reviews     Review[]
  variants    ProductVariant[]
  
  @@index([slug])
  @@index([status])
  @@index([categoryId])
  @@map("products")
}

model ProductVariant {
  id        String   @id @default(cuid())
  productId String
  name      String
  sku       String?  @unique
  price     Decimal  @db.Decimal(10, 2)
  stock     Int      @default(0)
  options   Json     // e.g. {"color": "red", "size": "M"}
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  product   Product  @relation(fields: [productId], references: [id], onDelete: Cascade)
  
  @@index([productId])
  @@map("product_variants")
}

model Review {
  id        String   @id @default(cuid())
  rating    Int      // 1-5
  title     String?
  content   String?
  productId String
  userId    String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  product   Product  @relation(fields: [productId], references: [id])
  user      User     @relation(fields: [userId], references: [id])
  
  @@unique([productId, userId])
  @@index([productId])
  @@map("reviews")
}

model Order {
  id              String      @id @default(cuid())
  orderNumber     String      @unique @default(cuid())
  userId          String
  status          OrderStatus @default(PENDING)
  subtotal        Decimal     @db.Decimal(10, 2)
  discountAmount  Decimal     @default(0) @db.Decimal(10, 2)
  shippingAmount  Decimal     @default(0) @db.Decimal(10, 2)
  taxAmount       Decimal     @default(0) @db.Decimal(10, 2)
  total           Decimal     @db.Decimal(10, 2)
  notes           String?
  shippingAddress Json?
  billingAddress  Json?
  paymentMethod   String?
  paidAt          DateTime?
  shippedAt       DateTime?
  deliveredAt     DateTime?
  cancelledAt     DateTime?
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  user            User        @relation(fields: [userId], references: [id])
  items           OrderItem[]
  
  @@index([userId])
  @@index([status])
  @@index([orderNumber])
  @@map("orders")
}

model OrderItem {
  id        String  @id @default(cuid())
  orderId   String
  productId String
  quantity  Int
  price     Decimal @db.Decimal(10, 2)
  total     Decimal @db.Decimal(10, 2)

  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product   Product @relation(fields: [productId], references: [id])
  
  @@index([orderId])
  @@map("order_items")
}

model Address {
  id          String   @id @default(cuid())
  userId      String
  name        String
  phone       String
  address1    String
  address2    String?
  city        String
  state       String?
  postalCode  String
  country     String   @default("TH")
  isDefault   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
  @@map("addresses")
}
```

---

## 47.3 Prisma Client CRUD Operations

### การสร้างข้อมูล (Create)

```typescript
// src/lib/user.repository.ts
import prisma from "./prisma";
import { Prisma, User, Role } from "@prisma/client";
import { hash } from "bcryptjs";

// Create single record
export async function createUser(data: {
  name: string;
  email: string;
  password: string;
  role?: Role;
}): Promise<User> {
  const hashedPassword = await hash(data.password, 12);
  
  return prisma.user.create({
    data: {
      name: data.name,
      email: data.email,
      password: hashedPassword,
      role: data.role ?? "USER",
      // Create related profile automatically
      profile: {
        create: {
          bio: null
        }
      }
    },
    // เลือก fields ที่จะ return
    include: {
      profile: true
    }
  });
}

// Create many records
export async function createManyUsers(
  users: Array<{ name: string; email: string; password: string }>
): Promise<Prisma.BatchPayload> {
  const usersWithHashedPasswords = await Promise.all(
    users.map(async user => ({
      ...user,
      password: await hash(user.password, 12)
    }))
  );
  
  return prisma.user.createMany({
    data: usersWithHashedPasswords,
    skipDuplicates: true // ข้าม records ที่มี unique constraint ชน
  });
}

// Upsert - create หรือ update
export async function upsertUser(data: {
  email: string;
  name: string;
  password: string;
}): Promise<User> {
  return prisma.user.upsert({
    where: { email: data.email },
    update: {
      name: data.name,
      updatedAt: new Date()
    },
    create: {
      email: data.email,
      name: data.name,
      password: await hash(data.password, 12)
    }
  });
}
```

### การอ่านข้อมูล (Read)

```typescript
// src/lib/user.repository.ts (เพิ่มเติม)

// FindUnique - ค้นหาด้วย unique field
export async function getUserById(id: string): Promise<User | null> {
  return prisma.user.findUnique({
    where: { id },
    include: {
      profile: true,
      _count: {
        select: {
          posts: true,
          orders: true
        }
      }
    }
  });
}

// FindFirst - ค้นหาตัวแรกที่ตรงเงื่อนไข
export async function getUserByEmail(email: string): Promise<User | null> {
  return prisma.user.findFirst({
    where: {
      email: email.toLowerCase(),
      deletedAt: null // exclude soft deleted users
    }
  });
}

// FindMany - ค้นหาหลาย records
export async function getUsers(params: {
  search?: string;
  role?: Role;
  page?: number;
  pageSize?: number;
  sortBy?: "name" | "email" | "createdAt";
  sortOrder?: "asc" | "desc";
}): Promise<{ users: User[]; total: number }> {
  const {
    search,
    role,
    page = 1,
    pageSize = 10,
    sortBy = "createdAt",
    sortOrder = "desc"
  } = params;
  
  const where: Prisma.UserWhereInput = {
    deletedAt: null,
    ...(search && {
      OR: [
        { name: { contains: search, mode: "insensitive" } },
        { email: { contains: search, mode: "insensitive" } }
      ]
    }),
    ...(role && { role })
  };
  
  const [users, total] = await prisma.$transaction([
    prisma.user.findMany({
      where,
      orderBy: { [sortBy]: sortOrder },
      skip: (page - 1) * pageSize,
      take: pageSize,
      select: {
        id: true,
        name: true,
        email: true,
        role: true,
        createdAt: true,
        updatedAt: true,
        deletedAt: true,
        // ไม่ return password
        _count: {
          select: {
            posts: true,
            orders: true
          }
        }
      }
    }),
    prisma.user.count({ where })
  ]);
  
  return { users: users as unknown as User[], total };
}
```

### การอัปเดตข้อมูล (Update)

```typescript
// src/lib/user.repository.ts (เพิ่มเติม)

// Update single record
export async function updateUser(
  id: string,
  data: Prisma.UserUpdateInput
): Promise<User | null> {
  try {
    return await prisma.user.update({
      where: { id },
      data: {
        ...data,
        updatedAt: new Date()
      }
    });
  } catch (error) {
    if (
      error instanceof Prisma.PrismaClientKnownRequestError &&
      error.code === "P2025"
    ) {
      return null; // Record not found
    }
    throw error;
  }
}

// Update many records
export async function activateUsers(ids: string[]): Promise<Prisma.BatchPayload> {
  return prisma.user.updateMany({
    where: {
      id: { in: ids },
      role: "USER"
    },
    data: {
      emailVerified: new Date()
    }
  });
}

// Update with nested relations
export async function updateUserProfile(
  userId: string,
  profileData: {
    bio?: string;
    website?: string;
    location?: string;
  }
): Promise<User> {
  return prisma.user.update({
    where: { id: userId },
    data: {
      profile: {
        upsert: {
          update: profileData,
          create: profileData
        }
      }
    },
    include: { profile: true }
  });
}
```

### การลบข้อมูล (Delete)

```typescript
// src/lib/user.repository.ts (เพิ่มเติม)

// Hard delete
export async function deleteUser(id: string): Promise<User | null> {
  try {
    return await prisma.user.delete({
      where: { id }
    });
  } catch (error) {
    if (
      error instanceof Prisma.PrismaClientKnownRequestError &&
      error.code === "P2025"
    ) {
      return null;
    }
    throw error;
  }
}

// Soft delete
export async function softDeleteUser(id: string): Promise<User | null> {
  try {
    return await prisma.user.update({
      where: { id, deletedAt: null },
      data: { deletedAt: new Date() }
    });
  } catch (error) {
    if (
      error instanceof Prisma.PrismaClientKnownRequestError &&
      error.code === "P2025"
    ) {
      return null;
    }
    throw error;
  }
}

// Delete many
export async function deleteInactiveUsers(): Promise<Prisma.BatchPayload> {
  const thirtyDaysAgo = new Date();
  thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);
  
  return prisma.user.deleteMany({
    where: {
      createdAt: { lt: thirtyDaysAgo },
      emailVerified: null,
      orders: { none: {} }
    }
  });
}
```

---

## 47.4 Filtering และ Sorting

### Complex Filtering

```typescript
// src/lib/product.repository.ts
import prisma from "./prisma";
import { Prisma, Product, ProductStatus } from "@prisma/client";

interface ProductFilters {
  search?: string;
  categoryId?: string;
  minPrice?: number;
  maxPrice?: number;
  status?: ProductStatus;
  inStock?: boolean;
  tags?: string[];
  sortBy?: "name" | "price" | "createdAt" | "stock";
  sortOrder?: "asc" | "desc";
}

export async function getProducts(
  filters: ProductFilters,
  page = 1,
  pageSize = 20
): Promise<{ products: Product[]; total: number }> {
  const {
    search,
    categoryId,
    minPrice,
    maxPrice,
    status = "ACTIVE",
    inStock,
    tags,
    sortBy = "createdAt",
    sortOrder = "desc"
  } = filters;
  
  const where: Prisma.ProductWhereInput = {
    deletedAt: null,
    status,
    ...(search && {
      OR: [
        { name: { contains: search, mode: "insensitive" } },
        { description: { contains: search, mode: "insensitive" } },
        { sku: { contains: search, mode: "insensitive" } }
      ]
    }),
    ...(categoryId && { categoryId }),
    ...(minPrice !== undefined || maxPrice !== undefined ? {
      price: {
        ...(minPrice !== undefined && { gte: minPrice }),
        ...(maxPrice !== undefined && { lte: maxPrice })
      }
    } : {}),
    ...(inStock === true && { stock: { gt: 0 } }),
    ...(inStock === false && { stock: { equals: 0 } }),
    ...(tags && tags.length > 0 && {
      tags: { hasSome: tags }
    })
  };
  
  const [products, total] = await prisma.$transaction([
    prisma.product.findMany({
      where,
      include: {
        category: {
          select: { id: true, name: true, slug: true }
        },
        _count: {
          select: { reviews: true, orderItems: true }
        }
      },
      orderBy: { [sortBy]: sortOrder },
      skip: (page - 1) * pageSize,
      take: pageSize
    }),
    prisma.product.count({ where })
  ]);
  
  return { products, total };
}

// Full-text search (PostgreSQL)
export async function searchProducts(query: string): Promise<Product[]> {
  return prisma.$queryRaw<Product[]>`
    SELECT * FROM products
    WHERE to_tsvector('thai', name || ' ' || description) 
    @@ plainto_tsquery('thai', ${query})
    AND deleted_at IS NULL
    ORDER BY ts_rank(
      to_tsvector('thai', name || ' ' || description),
      plainto_tsquery('thai', ${query})
    ) DESC
    LIMIT 20
  `;
}
```

---

## 47.5 Pagination

### Cursor-based Pagination

```typescript
// src/lib/pagination.ts
import prisma from "./prisma";
import { Prisma } from "@prisma/client";

interface CursorPaginationParams<T> {
  cursor?: string;
  take?: number;
  where?: T;
}

interface CursorPaginationResult<T> {
  data: T[];
  nextCursor?: string;
  hasMore: boolean;
}

// Cursor-based pagination (ดีกว่า offset สำหรับ data ที่เปลี่ยนแปลงบ่อย)
export async function getUsersWithCursor(
  params: CursorPaginationParams<Prisma.UserWhereInput>
): Promise<CursorPaginationResult<any>> {
  const { cursor, take = 10, where } = params;
  
  const users = await prisma.user.findMany({
    where: {
      ...where,
      deletedAt: null
    },
    take: take + 1, // เอามา 1 extra เพื่อเช็กว่ามีหน้าถัดไปไหม
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1 // ข้าม cursor ปัจจุบัน
    }),
    orderBy: { createdAt: "desc" },
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      createdAt: true
    }
  });
  
  const hasMore = users.length > take;
  const data = hasMore ? users.slice(0, -1) : users;
  const nextCursor = hasMore ? data[data.length - 1]?.id : undefined;
  
  return { data, nextCursor, hasMore };
}

// Offset-based pagination
export async function getUsersWithOffset(
  page: number,
  pageSize: number,
  where?: Prisma.UserWhereInput
): Promise<{
  data: any[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
  };
}> {
  const skip = (page - 1) * pageSize;
  
  const [data, total] = await prisma.$transaction([
    prisma.user.findMany({
      where: { ...where, deletedAt: null },
      skip,
      take: pageSize,
      orderBy: { createdAt: "desc" }
    }),
    prisma.user.count({ where: { ...where, deletedAt: null } })
  ]);
  
  return {
    data,
    pagination: {
      page,
      pageSize,
      total,
      totalPages: Math.ceil(total / pageSize)
    }
  };
}
```

---

## 47.6 Transactions

```typescript
// src/lib/order.service.ts
import prisma from "./prisma";
import { Prisma, Order } from "@prisma/client";

interface CreateOrderData {
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
  shippingAddressId: string;
  notes?: string;
}

// Interactive Transaction - สำหรับ logic ที่ซับซ้อน
export async function createOrder(data: CreateOrderData): Promise<Order> {
  return prisma.$transaction(async (tx) => {
    // 1. ดึงข้อมูล products
    const productIds = data.items.map(item => item.productId);
    const products = await tx.product.findMany({
      where: {
        id: { in: productIds },
        status: "ACTIVE",
        deletedAt: null
      }
    });
    
    if (products.length !== productIds.length) {
      throw new Error("บางสินค้าไม่พร้อมจำหน่าย");
    }
    
    // 2. ตรวจสอบ stock
    for (const item of data.items) {
      const product = products.find(p => p.id === item.productId);
      if (!product || product.stock < item.quantity) {
        throw new Error(`สินค้า ${product?.name ?? item.productId} ไม่เพียงพอ`);
      }
    }
    
    // 3. คำนวณราคา
    const orderItems = data.items.map(item => {
      const product = products.find(p => p.id === item.productId)!;
      const price = Number(product.price);
      return {
        productId: item.productId,
        quantity: item.quantity,
        price,
        total: price * item.quantity
      };
    });
    
    const subtotal = orderItems.reduce((sum, item) => sum + item.total, 0);
    const shippingAmount = subtotal >= 500 ? 0 : 50;
    const total = subtotal + shippingAmount;
    
    // 4. ดึง shipping address
    const address = await tx.address.findUnique({
      where: { id: data.shippingAddressId, userId: data.userId }
    });
    
    if (!address) {
      throw new Error("ที่อยู่จัดส่งไม่ถูกต้อง");
    }
    
    // 5. สร้าง order
    const order = await tx.order.create({
      data: {
        userId: data.userId,
        status: "PENDING",
        subtotal,
        shippingAmount,
        taxAmount: 0,
        discountAmount: 0,
        total,
        notes: data.notes,
        shippingAddress: {
          name: address.name,
          phone: address.phone,
          address1: address.address1,
          address2: address.address2,
          city: address.city,
          postalCode: address.postalCode,
          country: address.country
        },
        items: {
          create: orderItems
        }
      },
      include: {
        items: {
          include: { product: true }
        }
      }
    });
    
    // 6. ลด stock
    await Promise.all(
      data.items.map(item =>
        tx.product.update({
          where: { id: item.productId },
          data: {
            stock: { decrement: item.quantity }
          }
        })
      )
    );
    
    return order;
  }, {
    maxWait: 5000,    // รอ connection สูงสุด 5 วินาที
    timeout: 10000,   // transaction timeout 10 วินาที
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable
  });
}

// Sequential Transactions (อย่างง่าย)
export async function transferFunds(
  fromUserId: string,
  toUserId: string,
  amount: number
): Promise<void> {
  await prisma.$transaction([
    prisma.$executeRaw`
      UPDATE wallets SET balance = balance - ${amount}
      WHERE user_id = ${fromUserId} AND balance >= ${amount}
    `,
    prisma.$executeRaw`
      UPDATE wallets SET balance = balance + ${amount}
      WHERE user_id = ${toUserId}
    `
  ]);
}
```

---

## 47.7 Nested Writes

```typescript
// src/lib/post.repository.ts
import prisma from "./prisma";
import { Post, Prisma } from "@prisma/client";

interface CreatePostData {
  title: string;
  slug: string;
  content: string;
  excerpt?: string;
  authorId: string;
  categoryIds?: string[];
  tagNames?: string[];
  published?: boolean;
}

export async function createPost(data: CreatePostData): Promise<Post> {
  const { categoryIds, tagNames, ...postData } = data;
  
  return prisma.post.create({
    data: {
      ...postData,
      publishedAt: data.published ? new Date() : null,
      // Nested create/connect สำหรับ categories
      ...(categoryIds && categoryIds.length > 0 && {
        categories: {
          create: categoryIds.map(categoryId => ({
            category: { connect: { id: categoryId } }
          }))
        }
      }),
      // Nested create/connect สำหรับ tags (สร้างถ้าไม่มี)
      ...(tagNames && tagNames.length > 0 && {
        tags: {
          create: tagNames.map(name => ({
            tag: {
              connectOrCreate: {
                where: { name },
                create: {
                  name,
                  slug: name.toLowerCase().replace(/\s+/g, "-")
                }
              }
            }
          }))
        }
      })
    },
    include: {
      author: { select: { id: true, name: true, email: true } },
      categories: { include: { category: true } },
      tags: { include: { tag: true } }
    }
  });
}

// Update post กับ nested relations
export async function updatePost(
  id: string,
  data: {
    title?: string;
    content?: string;
    excerpt?: string;
    published?: boolean;
    categoryIds?: string[];
    tagNames?: string[];
  }
): Promise<Post> {
  const { categoryIds, tagNames, published, ...postData } = data;
  
  return prisma.post.update({
    where: { id },
    data: {
      ...postData,
      ...(published !== undefined && {
        published,
        publishedAt: published ? new Date() : null
      }),
      // Replace all categories
      ...(categoryIds !== undefined && {
        categories: {
          deleteMany: {},
          create: categoryIds.map(categoryId => ({
            category: { connect: { id: categoryId } }
          }))
        }
      }),
      // Replace all tags
      ...(tagNames !== undefined && {
        tags: {
          deleteMany: {},
          create: tagNames.map(name => ({
            tag: {
              connectOrCreate: {
                where: { name },
                create: {
                  name,
                  slug: name.toLowerCase().replace(/\s+/g, "-")
                }
              }
            }
          }))
        }
      })
    },
    include: {
      categories: { include: { category: true } },
      tags: { include: { tag: true } }
    }
  });
}
```

---

## 47.8 Migrations

```bash
# สร้าง migration ใหม่
npx prisma migrate dev --name init

# สร้าง migration จาก schema ที่เปลี่ยน
npx prisma migrate dev --name add_user_profile

# Deploy migrations ใน production
npx prisma migrate deploy

# Reset database (development only)
npx prisma migrate reset

# ดู migration status
npx prisma migrate status

# สร้าง migration โดยไม่ apply
npx prisma migrate dev --create-only --name my_migration
```

### Custom Migration

```sql
-- prisma/migrations/20240101000000_add_full_text_search/migration.sql

-- เพิ่ม full text search index สำหรับ PostgreSQL
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX products_name_trgm_idx ON products USING gin(name gin_trgm_ops);
CREATE INDEX products_description_trgm_idx ON products USING gin(description gin_trgm_ops);

-- เพิ่ม trigger สำหรับ updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
   NEW.updated_at = NOW();
   RETURN NEW;
END;
$$ language 'plpgsql';
```

---

## 47.9 Seeding

```typescript
// prisma/seed.ts
import { PrismaClient, Role } from "@prisma/client";
import { hash } from "bcryptjs";

const prisma = new PrismaClient();

async function main() {
  console.log("Starting seed...");
  
  // Seed categories
  const categories = await Promise.all([
    prisma.category.upsert({
      where: { slug: "electronics" },
      update: {},
      create: {
        name: "อิเล็กทรอนิกส์",
        slug: "electronics",
        description: "สินค้าอิเล็กทรอนิกส์และเทคโนโลยี"
      }
    }),
    prisma.category.upsert({
      where: { slug: "clothing" },
      update: {},
      create: {
        name: "เสื้อผ้า",
        slug: "clothing",
        description: "เสื้อผ้าและแฟชั่น"
      }
    }),
    prisma.category.upsert({
      where: { slug: "books" },
      update: {},
      create: {
        name: "หนังสือ",
        slug: "books",
        description: "หนังสือและสื่อการเรียน"
      }
    })
  ]);
  
  console.log(`Created ${categories.length} categories`);
  
  // Seed admin user
  const adminPassword = await hash("Admin@123456", 12);
  const admin = await prisma.user.upsert({
    where: { email: "admin@example.com" },
    update: {},
    create: {
      email: "admin@example.com",
      name: "ผู้ดูแลระบบ",
      password: adminPassword,
      role: Role.ADMIN,
      emailVerified: new Date(),
      profile: {
        create: {
          bio: "ผู้ดูแลระบบหลัก"
        }
      }
    }
  });
  
  console.log(`Admin user: ${admin.email}`);
  
  // Seed products
  const products = await Promise.all([
    prisma.product.upsert({
      where: { slug: "iphone-15" },
      update: {},
      create: {
        name: "iPhone 15",
        slug: "iphone-15",
        description: "iPhone 15 สมาร์ทโฟนรุ่นล่าสุดจาก Apple",
        price: 29900,
        status: "ACTIVE",
        stock: 50,
        categoryId: categories[0].id,
        images: [
          "https://images.unsplash.com/photo-1695048133142-1a20484d2569"
        ],
        tags: ["apple", "smartphone", "ios"],
        sku: "IPHONE-15-128GB"
      }
    }),
    prisma.product.upsert({
      where: { slug: "typescript-handbook" },
      update: {},
      create: {
        name: "TypeScript Handbook",
        slug: "typescript-handbook",
        description: "คู่มือ TypeScript ฉบับสมบูรณ์",
        price: 450,
        status: "ACTIVE",
        stock: 100,
        categoryId: categories[2].id,
        images: [],
        tags: ["typescript", "programming", "book"],
        sku: "BOOK-TS-001"
      }
    })
  ]);
  
  console.log(`Created ${products.length} products`);
  console.log("Seed completed!");
}

main()
  .then(async () => {
    await prisma.$disconnect();
  })
  .catch(async (error) => {
    console.error(error);
    await prisma.$disconnect();
    process.exit(1);
  });
```

### package.json seed script

```json
{
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
}
```

```bash
# รัน seed
npx prisma db seed
```

---

## 47.10 Generated Types

```typescript
// Prisma generates types โดยอัตโนมัติ
import { Prisma, User, Product, Order } from "@prisma/client";

// Prisma.UserCreateInput - type สำหรับ create
const createData: Prisma.UserCreateInput = {
  name: "สมชาย",
  email: "somchai@example.com",
  password: "hashed_password"
};

// Prisma.UserUpdateInput - type สำหรับ update
const updateData: Prisma.UserUpdateInput = {
  name: "สมชาย ใจดี",
  updatedAt: new Date()
};

// Prisma.UserWhereInput - type สำหรับ filter
const whereInput: Prisma.UserWhereInput = {
  email: { contains: "@example.com" },
  role: "ADMIN",
  deletedAt: null
};

// Prisma.UserWhereUniqueInput - type สำหรับ unique where
const uniqueWhere: Prisma.UserWhereUniqueInput = {
  email: "somchai@example.com"
};

// Prisma.UserOrderByWithRelationInput
const orderBy: Prisma.UserOrderByWithRelationInput = {
  createdAt: "desc"
};

// Custom type จาก select
type UserSummary = Prisma.UserGetPayload<{
  select: {
    id: true;
    name: true;
    email: true;
    role: true;
    _count: {
      select: {
        posts: true;
        orders: true;
      };
    };
  };
}>;

// ใช้ type กับ include
type UserWithProfile = Prisma.UserGetPayload<{
  include: {
    profile: true;
    posts: {
      select: {
        id: true;
        title: true;
        published: true;
      };
    };
  };
}>;

// ตัวอย่างการใช้งาน
async function getUserWithPosts(id: string): Promise<UserWithProfile | null> {
  return prisma.user.findUnique({
    where: { id },
    include: {
      profile: true,
      posts: {
        select: {
          id: true,
          title: true,
          published: true
        }
      }
    }
  });
}
```

---

## 47.11 Soft Deletes

```typescript
// src/lib/soft-delete.extension.ts
import { Prisma } from "@prisma/client";

// Prisma Extension สำหรับ soft delete
export const softDeleteExtension = Prisma.defineExtension({
  name: "softDelete",
  model: {
    // เพิ่ม softDelete method ให้ทุก model
    $allModels: {
      async softDelete<T>(
        this: T,
        where: Prisma.Args<T, "update">["where"]
      ): Promise<Prisma.Result<T, { where: typeof where }, "update">> {
        const context = Prisma.getExtensionContext(this);
        return (context as any).update({
          where,
          data: {
            deletedAt: new Date()
          }
        });
      },
      
      async restore<T>(
        this: T,
        where: Prisma.Args<T, "update">["where"]
      ): Promise<Prisma.Result<T, { where: typeof where }, "update">> {
        const context = Prisma.getExtensionContext(this);
        return (context as any).update({
          where,
          data: {
            deletedAt: null
          }
        });
      }
    }
  },
  query: {
    $allModels: {
      // Override findMany เพื่อ exclude soft deleted records
      async findMany({ model, operation, args, query }) {
        if (!args.where) args.where = {};
        args.where = {
          ...args.where,
          deletedAt: (args.where as any).deletedAt ?? null
        };
        return query(args);
      },
      
      async findFirst({ args, query }) {
        if (!args.where) args.where = {};
        args.where = {
          ...args.where,
          deletedAt: (args.where as any).deletedAt ?? null
        };
        return query(args);
      },
      
      async count({ args, query }) {
        if (!args.where) args.where = {};
        args.where = {
          ...args.where,
          deletedAt: (args.where as any).deletedAt ?? null
        };
        return query(args);
      }
    }
  }
});

// การใช้งาน Extension
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient().$extends(softDeleteExtension);

// ตอนนี้ทุก findMany จะ exclude soft deleted records โดยอัตโนมัติ
const users = await prisma.user.findMany(); // ไม่ return users ที่ deletedAt != null

// Soft delete
await prisma.user.softDelete({ id: "user-id" });

// Restore
await prisma.user.restore({ id: "user-id" });

// ถ้าต้องการดึง soft deleted records
const allUsers = await prisma.user.findMany({
  where: {
    deletedAt: { not: null }
  }
});
```

---

## 47.12 Prisma กับ TypeScript Patterns

### Repository Pattern

```typescript
// src/lib/repositories/base.repository.ts
import { Prisma, PrismaClient } from "@prisma/client";

export interface Repository<T, CreateInput, UpdateInput, WhereUniqueInput> {
  findById(id: string): Promise<T | null>;
  findMany(params?: {
    where?: any;
    orderBy?: any;
    skip?: number;
    take?: number;
  }): Promise<T[]>;
  create(data: CreateInput): Promise<T>;
  update(where: WhereUniqueInput, data: UpdateInput): Promise<T | null>;
  delete(where: WhereUniqueInput): Promise<boolean>;
  count(where?: any): Promise<number>;
}

// src/lib/repositories/user.repository.ts
import prisma from "@/lib/prisma";
import { User, Prisma } from "@prisma/client";
import { Repository } from "./base.repository";

type UserWithProfile = Prisma.UserGetPayload<{ include: { profile: true } }>;

export class UserRepository implements Repository<
  UserWithProfile,
  Prisma.UserCreateInput,
  Prisma.UserUpdateInput,
  Prisma.UserWhereUniqueInput
> {
  async findById(id: string): Promise<UserWithProfile | null> {
    return prisma.user.findUnique({
      where: { id, deletedAt: null },
      include: { profile: true }
    });
  }
  
  async findByEmail(email: string): Promise<UserWithProfile | null> {
    return prisma.user.findFirst({
      where: { email, deletedAt: null },
      include: { profile: true }
    });
  }
  
  async findMany(params?: {
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
    skip?: number;
    take?: number;
  }): Promise<UserWithProfile[]> {
    return prisma.user.findMany({
      where: { ...params?.where, deletedAt: null },
      orderBy: params?.orderBy ?? { createdAt: "desc" },
      skip: params?.skip,
      take: params?.take,
      include: { profile: true }
    });
  }
  
  async create(data: Prisma.UserCreateInput): Promise<UserWithProfile> {
    return prisma.user.create({
      data,
      include: { profile: true }
    });
  }
  
  async update(
    where: Prisma.UserWhereUniqueInput,
    data: Prisma.UserUpdateInput
  ): Promise<UserWithProfile | null> {
    try {
      return await prisma.user.update({
        where: { ...where, deletedAt: null } as any,
        data,
        include: { profile: true }
      });
    } catch (error) {
      if (
        error instanceof Prisma.PrismaClientKnownRequestError &&
        error.code === "P2025"
      ) {
        return null;
      }
      throw error;
    }
  }
  
  async delete(where: Prisma.UserWhereUniqueInput): Promise<boolean> {
    try {
      await prisma.user.update({
        where,
        data: { deletedAt: new Date() }
      });
      return true;
    } catch (error) {
      if (
        error instanceof Prisma.PrismaClientKnownRequestError &&
        error.code === "P2025"
      ) {
        return false;
      }
      throw error;
    }
  }
  
  async count(where?: Prisma.UserWhereInput): Promise<number> {
    return prisma.user.count({
      where: { ...where, deletedAt: null }
    });
  }
}

export const userRepository = new UserRepository();
```

### Service Layer

```typescript
// src/services/user.service.ts
import { User, Role } from "@prisma/client";
import { userRepository } from "@/lib/repositories/user.repository";
import { hash, compare } from "bcryptjs";

interface CreateUserDTO {
  name: string;
  email: string;
  password: string;
  role?: Role;
}

interface UpdateUserDTO {
  name?: string;
  bio?: string;
  website?: string;
}

interface LoginDTO {
  email: string;
  password: string;
}

export class UserService {
  async register(dto: CreateUserDTO) {
    // ตรวจสอบ email ซ้ำ
    const existingUser = await userRepository.findByEmail(dto.email);
    if (existingUser) {
      throw new Error("อีเมลนี้ถูกใช้งานแล้ว");
    }
    
    const hashedPassword = await hash(dto.password, 12);
    
    const user = await userRepository.create({
      name: dto.name,
      email: dto.email.toLowerCase(),
      password: hashedPassword,
      role: dto.role ?? "USER",
      profile: { create: {} }
    });
    
    // ไม่ return password
    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }
  
  async login(dto: LoginDTO) {
    const user = await userRepository.findByEmail(dto.email);
    
    if (!user) {
      throw new Error("อีเมลหรือรหัสผ่านไม่ถูกต้อง");
    }
    
    const isValidPassword = await compare(dto.password, user.password);
    
    if (!isValidPassword) {
      throw new Error("อีเมลหรือรหัสผ่านไม่ถูกต้อง");
    }
    
    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }
  
  async updateProfile(userId: string, dto: UpdateUserDTO) {
    const user = await userRepository.findById(userId);
    
    if (!user) {
      throw new Error("ไม่พบผู้ใช้");
    }
    
    return userRepository.update(
      { id: userId },
      {
        ...(dto.name && { name: dto.name }),
        profile: {
          upsert: {
            update: {
              bio: dto.bio,
              website: dto.website
            },
            create: {
              bio: dto.bio,
              website: dto.website
            }
          }
        }
      }
    );
  }
  
  async getUserStats(userId: string) {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: {
        _count: {
          select: {
            posts: { where: { published: true } },
            orders: true
          }
        }
      }
    });
    
    if (!user) throw new Error("ไม่พบผู้ใช้");
    
    const totalSpent = await prisma.order.aggregate({
      where: {
        userId,
        status: { in: ["DELIVERED"] }
      },
      _sum: { total: true }
    });
    
    return {
      publishedPosts: user._count.posts,
      totalOrders: user._count.orders,
      totalSpent: Number(totalSpent._sum.total ?? 0)
    };
  }
}

export const userService = new UserService();
import prisma from "@/lib/prisma";
```

---

## 47.13 Aggregations และ Raw Queries

```typescript
// src/lib/analytics.ts
import prisma from "./prisma";

// Aggregation
export async function getOrderStats() {
  const stats = await prisma.order.aggregate({
    _count: {
      id: true
    },
    _sum: {
      total: true,
      discountAmount: true
    },
    _avg: {
      total: true
    },
    _min: {
      total: true
    },
    _max: {
      total: true
    },
    where: {
      status: { notIn: ["CANCELLED", "REFUNDED"] }
    }
  });
  
  return {
    totalOrders: stats._count.id,
    totalRevenue: Number(stats._sum.total ?? 0),
    averageOrderValue: Number(stats._avg.total ?? 0),
    minOrderValue: Number(stats._min.total ?? 0),
    maxOrderValue: Number(stats._max.total ?? 0)
  };
}

// GroupBy
export async function getOrdersByStatus() {
  const groups = await prisma.order.groupBy({
    by: ["status"],
    _count: { id: true },
    _sum: { total: true },
    orderBy: {
      _count: { id: "desc" }
    }
  });
  
  return groups.map(g => ({
    status: g.status,
    count: g._count.id,
    totalRevenue: Number(g._sum.total ?? 0)
  }));
}

// Raw SQL Query
export async function getTopSellingProducts(limit = 10) {
  const result = await prisma.$queryRaw<Array<{
    product_id: string;
    product_name: string;
    total_sold: bigint;
    revenue: number;
  }>>`
    SELECT 
      p.id as product_id,
      p.name as product_name,
      SUM(oi.quantity) as total_sold,
      SUM(oi.total) as revenue
    FROM order_items oi
    JOIN products p ON p.id = oi.product_id
    JOIN orders o ON o.id = oi.order_id
    WHERE o.status IN ('DELIVERED', 'SHIPPED')
    GROUP BY p.id, p.name
    ORDER BY total_sold DESC
    LIMIT ${limit}
  `;
  
  return result.map(r => ({
    productId: r.product_id,
    productName: r.product_name,
    totalSold: Number(r.total_sold),
    revenue: Number(r.revenue)
  }));
}

// Execute Raw
export async function archiveOldOrders(daysOld: number): Promise<number> {
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - daysOld);
  
  const result = await prisma.$executeRaw`
    UPDATE orders 
    SET status = 'ARCHIVED'
    WHERE status = 'DELIVERED' 
    AND delivered_at < ${cutoffDate}
  `;
  
  return result;
}
```

---

## 47.14 Complete E-Commerce Example

```typescript
// src/app/api/orders/route.ts
import { NextRequest, NextResponse } from "next/server";
import prisma from "@/lib/prisma";
import { z } from "zod";

const createOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().int().positive()
  })).min(1),
  shippingAddressId: z.string(),
  notes: z.string().optional()
});

export async function POST(request: NextRequest) {
  // TODO: Get userId from session
  const userId = "user-id";
  
  const body = await request.json();
  const validation = createOrderSchema.safeParse(body);
  
  if (!validation.success) {
    return NextResponse.json(
      { error: validation.error.flatten() },
      { status: 422 }
    );
  }
  
  try {
    const order = await prisma.$transaction(async (tx) => {
      const productIds = validation.data.items.map(i => i.productId);
      
      const products = await tx.product.findMany({
        where: { id: { in: productIds }, status: "ACTIVE" }
      });
      
      if (products.length !== productIds.length) {
        throw new Error("Some products are unavailable");
      }
      
      const items = validation.data.items.map(item => {
        const product = products.find(p => p.id === item.productId)!;
        const price = Number(product.price);
        return {
          productId: item.productId,
          quantity: item.quantity,
          price,
          total: price * item.quantity
        };
      });
      
      const subtotal = items.reduce((s, i) => s + i.total, 0);
      const shippingAmount = subtotal >= 500 ? 0 : 50;
      const total = subtotal + shippingAmount;
      
      // Create order
      const order = await tx.order.create({
        data: {
          userId,
          status: "PENDING",
          subtotal,
          shippingAmount,
          taxAmount: 0,
          discountAmount: 0,
          total,
          notes: validation.data.notes,
          items: { create: items }
        },
        include: {
          items: { include: { product: true } }
        }
      });
      
      // Decrement stock
      await Promise.all(
        validation.data.items.map(item =>
          tx.product.update({
            where: { id: item.productId },
            data: { stock: { decrement: item.quantity } }
          })
        )
      );
      
      return order;
    });
    
    return NextResponse.json(order, { status: 201 });
  } catch (error) {
    const message = error instanceof Error ? error.message : "Internal error";
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

---

## สรุปบทที่ 47

ในบทนี้เราได้เรียนรู้:

1. **Prisma Setup** - การติดตั้งและตั้งค่า
2. **Schema Definition** - การเขียน Prisma schema
3. **Models & Relations** - One-to-one, One-to-many, Many-to-many
4. **CRUD Operations** - Create, Read, Update, Delete
5. **Filtering & Sorting** - การค้นหาและเรียงข้อมูล
6. **Pagination** - Cursor-based และ Offset-based
7. **Transactions** - Interactive และ Sequential
8. **Nested Writes** - การเขียนข้อมูลซ้อนกัน
9. **Migrations** - การจัดการ database schema changes
10. **Seeding** - การเติมข้อมูลเริ่มต้น
11. **Generated Types** - การใช้ type safety
12. **Soft Deletes** - การลบแบบ soft
13. **Repository Pattern** - การจัดระเบียบโค้ด
14. **Raw Queries** - การใช้ SQL โดยตรง

Prisma ทำให้การทำงานกับ database:
- Type-safe ด้วย generated types
- ง่ายต่อการ maintain
- Migration management ที่สมบูรณ์
- Performance ที่ดีด้วย query optimization
- Developer experience ที่ยอดเยี่ยม
