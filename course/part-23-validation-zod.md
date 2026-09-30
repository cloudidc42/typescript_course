# ตอนที่ 23: Validation ด้วย Zod

## บทนำ

Zod เป็น TypeScript-first schema validation library ที่ช่วยให้เราสามารถกำหนด schema สำหรับ validation และ TypeScript types ในเวลาเดียวกัน ข้อดีสำคัญของ Zod คือไม่ต้องกำหนด interface และ validation rules แยกกัน เพราะ Zod จะ infer TypeScript types จาก schema ให้เองอัตโนมัติ

---

## 23.1 ทำไมต้องใช้ Zod?

### ปัญหาเดิมก่อนใช้ Zod

```typescript
// ก่อนใช้ Zod - ต้องกำหนด interface และ validation แยกกัน
interface CreateUserInput {
  name: string;
  email: string;
  age: number;
  role: 'admin' | 'user';
}

// ต้องเขียน validation เองซึ่งอาจ mismatch กับ interface
function validateCreateUser(data: unknown): CreateUserInput {
  if (typeof data !== 'object' || data === null) {
    throw new Error('Data must be an object');
  }
  
  const obj = data as Record<string, unknown>;
  
  if (typeof obj.name !== 'string') throw new Error('name must be a string');
  if (typeof obj.email !== 'string') throw new Error('email must be a string');
  if (typeof obj.age !== 'number') throw new Error('age must be a number');
  if (obj.role !== 'admin' && obj.role !== 'user') throw new Error('invalid role');
  
  // TypeScript ยังไม่รู้ว่า type นี้ถูกต้อง
  return obj as unknown as CreateUserInput;
}
```

### ด้วย Zod - กำหนดครั้งเดียว ได้ทั้ง validation และ type

```typescript
import { z } from 'zod';

// กำหนด schema ครั้งเดียว
const CreateUserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int().positive(),
  role: z.enum(['admin', 'user']),
});

// Infer type จาก schema อัตโนมัติ
type CreateUserInput = z.infer<typeof CreateUserSchema>;
// เทียบเท่ากับ:
// type CreateUserInput = {
//   name: string;
//   email: string;
//   age: number;
//   role: 'admin' | 'user';
// }

// Validate
function validateCreateUser(data: unknown): CreateUserInput {
  return CreateUserSchema.parse(data); // throw error ถ้า invalid
}
```

---

## 23.2 การติดตั้ง Zod

```bash
npm install zod
```

### package.json

```json
{
  "dependencies": {
    "zod": "^3.22.0",
    "express": "^4.18.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.17",
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## 23.3 Schema Types พื้นฐาน

### String schemas

```typescript
import { z } from 'zod';

// String พื้นฐาน
const nameSchema = z.string();
nameSchema.parse('Alice'); // ✓
nameSchema.parse(123);     // ✗ ZodError

// String validations
const emailSchema = z.string().email('Invalid email format');
const urlSchema = z.string().url('Invalid URL');
const uuidSchema = z.string().uuid('Invalid UUID');

// Length constraints
const usernameSchema = z.string()
  .min(3, 'Username must be at least 3 characters')
  .max(20, 'Username must not exceed 20 characters');

// Pattern matching
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, 
    'Password must contain uppercase, lowercase, and number');

// Trimming และ transformation
const trimmedString = z.string().trim(); // ตัด whitespace หน้าหลัง
const lowercaseEmail = z.string().email().toLowerCase(); // แปลงเป็นตัวเล็ก

// Optional string
const bioSchema = z.string().optional(); // string | undefined
const websiteSchema = z.string().url().nullable(); // string | null
const descriptionSchema = z.string().nullish(); // string | null | undefined

// DateTime string
const dateStringSchema = z.string().datetime('Invalid datetime format');
const dateOnlySchema = z.string().date('Invalid date format');

console.log(emailSchema.safeParse('user@example.com')); // { success: true, data: 'user@example.com' }
console.log(emailSchema.safeParse('invalid-email'));    // { success: false, error: ZodError }
```

### Number schemas

```typescript
import { z } from 'zod';

// Number พื้นฐาน
const ageSchema = z.number()
  .int('Age must be an integer')
  .min(0, 'Age must be non-negative')
  .max(120, 'Age must be realistic');

// Price
const priceSchema = z.number()
  .positive('Price must be positive')
  .multipleOf(0.01, 'Price must have at most 2 decimal places');

// Integer
const countSchema = z.number().int().nonnegative();

// Port number
const portSchema = z.number().int().min(1).max(65535);

// แปลง string เป็น number
const numericStringSchema = z.string().transform(val => parseFloat(val));

// Coerce (แปลงอัตโนมัติ)
const coercedNumber = z.coerce.number(); // แปลง '42' -> 42
coercedNumber.parse('42'); // 42
coercedNumber.parse(42);   // 42
```

### Boolean schemas

```typescript
import { z } from 'zod';

const activeSchema = z.boolean();
activeSchema.parse(true);  // ✓
activeSchema.parse(false); // ✓
activeSchema.parse(1);     // ✗

// Coerce boolean
const coercedBool = z.coerce.boolean();
coercedBool.parse(1);       // true
coercedBool.parse(0);       // false
coercedBool.parse('true');  // true
coercedBool.parse('false'); // false
```

### Date schemas

```typescript
import { z } from 'zod';

// Date object
const dateSchema = z.date();
dateSchema.parse(new Date()); // ✓

// Date with range
const birthDateSchema = z.date()
  .min(new Date('1900-01-01'), 'Date too far in the past')
  .max(new Date(), 'Date cannot be in the future');

// แปลง string เป็น Date
const dateFromStringSchema = z.coerce.date();
dateFromStringSchema.parse('2024-01-15'); // Date object

// ISO string ที่ valid
const isoDateSchema = z.string().pipe(z.coerce.date());
```

---

## 23.4 Object Schemas

### การกำหนด Object schema

```typescript
import { z } from 'zod';

// Object schema พื้นฐาน
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(2).max(100),
  email: z.string().email(),
  age: z.number().int().min(18),
  role: z.enum(['admin', 'moderator', 'user']),
  isActive: z.boolean().default(true),
  bio: z.string().max(500).optional(),
  createdAt: z.date().default(() => new Date()),
});

type User = z.infer<typeof UserSchema>;

// Partial - ทุก field เป็น optional
const UpdateUserSchema = UserSchema.partial();
type UpdateUser = z.infer<typeof UpdateUserSchema>;

// Pick - เลือก fields ที่ต้องการ
const PublicUserSchema = UserSchema.pick({
  id: true,
  name: true,
  email: true,
});
type PublicUser = z.infer<typeof PublicUserSchema>;

// Omit - ตัด fields ออก
const CreateUserSchema = UserSchema.omit({
  id: true,
  createdAt: true,
});
type CreateUser = z.infer<typeof CreateUserSchema>;

// Merge - รวม schemas
const AdminSchema = z.object({
  permissions: z.array(z.string()),
  department: z.string(),
});

const AdminUserSchema = UserSchema.merge(AdminSchema);
type AdminUser = z.infer<typeof AdminUserSchema>;

// Extend - เพิ่ม fields
const ExtendedUserSchema = UserSchema.extend({
  phone: z.string().optional(),
  address: z.string().optional(),
});

// Strip (default) - ตัด fields ที่ไม่ได้กำหนดออก
const parsedUser = UserSchema.parse({
  id: '123e4567-e89b-12d3-a456-426614174000',
  name: 'Alice',
  email: 'alice@example.com',
  age: 25,
  role: 'user',
  isActive: true,
  extraField: 'this will be stripped', // จะถูกตัดออก
});

// Passthrough - ยอมรับ fields เพิ่มเติม
const PassthroughSchema = UserSchema.passthrough();

// Strict - ไม่ยอมรับ fields ที่ไม่ได้กำหนด
const StrictSchema = UserSchema.strict();
```

---

## 23.5 Array Schemas

```typescript
import { z } from 'zod';

// Array พื้นฐาน
const tagsSchema = z.array(z.string());
tagsSchema.parse(['typescript', 'nodejs']); // ✓

// Array with constraints
const emailListSchema = z.array(z.string().email())
  .min(1, 'At least one email required')
  .max(10, 'Maximum 10 emails allowed')
  .nonempty('Email list cannot be empty');

// Array of objects
const PostListSchema = z.array(z.object({
  id: z.string().uuid(),
  title: z.string(),
  publishedAt: z.date().optional(),
}));

// Tuple - array ที่มีความยาวและ type คงที่
const CoordinateSchema = z.tuple([
  z.number(), // latitude
  z.number(), // longitude
]);

type Coordinate = z.infer<typeof CoordinateSchema>;
// [number, number]

const RGBSchema = z.tuple([
  z.number().min(0).max(255), // R
  z.number().min(0).max(255), // G
  z.number().min(0).max(255), // B
]);

// Set
const uniqueTagsSchema = z.set(z.string());

// Array ที่มี unique items (ด้วย refine)
const uniqueEmailsSchema = z.array(z.string().email())
  .refine(
    emails => new Set(emails).size === emails.length,
    'Emails must be unique'
  );
```

---

## 23.6 Union และ Discriminated Union

### Union Schemas

```typescript
import { z } from 'zod';

// Union พื้นฐาน
const stringOrNumberSchema = z.union([z.string(), z.number()]);
stringOrNumberSchema.parse('hello'); // ✓
stringOrNumberSchema.parse(42);      // ✓
stringOrNumberSchema.parse(true);    // ✗

// หรือใช้ .or()
const altSchema = z.string().or(z.number());

// Literal
const successStatus = z.literal('success');
const errorStatus = z.literal('error');

const ApiStatusSchema = z.union([successStatus, errorStatus]);
type ApiStatus = z.infer<typeof ApiStatusSchema>; // 'success' | 'error'
```

### Discriminated Union

```typescript
import { z } from 'zod';

// Discriminated union - มีประสิทธิภาพกว่า union ธรรมดา
const ApiResponseSchema = z.discriminatedUnion('status', [
  z.object({
    status: z.literal('success'),
    data: z.unknown(),
    message: z.string().optional(),
  }),
  z.object({
    status: z.literal('error'),
    error: z.string(),
    code: z.number().optional(),
  }),
  z.object({
    status: z.literal('loading'),
  }),
]);

type ApiResponse = z.infer<typeof ApiResponseSchema>;

// การใช้งาน
const successResponse: ApiResponse = {
  status: 'success',
  data: { id: 1, name: 'Alice' },
};

const errorResponse: ApiResponse = {
  status: 'error',
  error: 'User not found',
  code: 404,
};

// TypeScript จะ narrow type ให้เองเมื่อตรวจสอบ status
function handleResponse(response: ApiResponse) {
  switch (response.status) {
    case 'success':
      console.log('Data:', response.data); // TypeScript รู้ว่ามี .data
      break;
    case 'error':
      console.error('Error:', response.error); // TypeScript รู้ว่ามี .error
      break;
    case 'loading':
      console.log('Loading...');
      break;
  }
}
```

---

## 23.7 Enums

```typescript
import { z } from 'zod';

// Native enum
enum Direction {
  UP = 'UP',
  DOWN = 'DOWN',
  LEFT = 'LEFT',
  RIGHT = 'RIGHT',
}

const DirectionSchema = z.nativeEnum(Direction);
DirectionSchema.parse(Direction.UP); // ✓
DirectionSchema.parse('UP');          // ✓

// Zod enum (แนะนำสำหรับ string literals)
const UserRoleSchema = z.enum(['admin', 'moderator', 'user']);
type UserRole = z.infer<typeof UserRoleSchema>;
// 'admin' | 'moderator' | 'user'

// ดึง values ออกมา
const roles = UserRoleSchema.options; // ['admin', 'moderator', 'user']
```

---

## 23.8 Refinements และ Transforms

### Custom Validation ด้วย refine

```typescript
import { z } from 'zod';

// refine - custom validation
const PasswordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .refine(
    password => /[A-Z]/.test(password),
    'Password must contain at least one uppercase letter'
  )
  .refine(
    password => /[0-9]/.test(password),
    'Password must contain at least one number'
  )
  .refine(
    password => /[^A-Za-z0-9]/.test(password),
    'Password must contain at least one special character'
  );

// superRefine - validation แบบ complex
const PasswordConfirmSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).superRefine((data, ctx) => {
  if (data.password !== data.confirmPassword) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Passwords do not match',
      path: ['confirmPassword'],
    });
  }
});

// เปรียบเทียบ date range
const DateRangeSchema = z.object({
  startDate: z.coerce.date(),
  endDate: z.coerce.date(),
}).refine(
  data => data.endDate > data.startDate,
  {
    message: 'End date must be after start date',
    path: ['endDate'],
  }
);

// ตรวจสอบ unique constraint
const UniqueItemsSchema = z.array(z.string())
  .refine(
    items => new Set(items).size === items.length,
    'All items must be unique'
  );
```

### Transforms

```typescript
import { z } from 'zod';

// transform - แปลงข้อมูล
const TrimmedStringSchema = z.string().transform(s => s.trim());

const EmailSchema = z.string()
  .email()
  .transform(email => email.toLowerCase());

// แปลง string เป็น number
const NumberFromStringSchema = z.string()
  .transform(s => parseInt(s, 10))
  .pipe(z.number().int().positive());

// แปลงข้อมูลแบบซับซ้อน
const CreateUserInputSchema = z.object({
  name: z.string().trim().min(2).max(100),
  email: z.string().email().toLowerCase(),
  birthDate: z.string().transform(str => new Date(str)),
  tags: z.string()
    .optional()
    .transform(str => str ? str.split(',').map(t => t.trim()) : []),
});

type CreateUserInput = z.infer<typeof CreateUserInputSchema>;

// preprocess - แปลงก่อน validate
const BoolFromStringSchema = z.preprocess(
  val => {
    if (typeof val === 'string') {
      if (val === 'true') return true;
      if (val === 'false') return false;
    }
    return val;
  },
  z.boolean()
);

BoolFromStringSchema.parse('true');  // true
BoolFromStringSchema.parse('false'); // false
BoolFromStringSchema.parse(true);    // true
```

---

## 23.9 Error Handling

### safeParse และ parse

```typescript
import { z, ZodError } from 'zod';

const UserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().int().min(18),
});

// parse - throw error ถ้า invalid
try {
  const user = UserSchema.parse({
    name: 'A',           // ✗ too short
    email: 'not-email',  // ✗ invalid email
    age: 15,             // ✗ too young
  });
} catch (error) {
  if (error instanceof ZodError) {
    console.log('Validation errors:');
    error.errors.forEach(err => {
      console.log(`- ${err.path.join('.')}: ${err.message}`);
    });
  }
}

// safeParse - ไม่ throw error แต่ return result object
const result = UserSchema.safeParse({
  name: 'A',
  email: 'not-email',
  age: 15,
});

if (!result.success) {
  console.log('Validation failed:', result.error.errors);
} else {
  console.log('Valid user:', result.data);
}

// แปลง errors เป็น object ที่อ่านง่าย
function formatZodErrors(error: ZodError): Record<string, string[]> {
  const errors: Record<string, string[]> = {};
  
  error.errors.forEach(err => {
    const path = err.path.join('.') || '_root';
    if (!errors[path]) {
      errors[path] = [];
    }
    errors[path].push(err.message);
  });
  
  return errors;
}

const result2 = UserSchema.safeParse({ name: 'A', email: 'bad', age: 15 });
if (!result2.success) {
  const formattedErrors = formatZodErrors(result2.error);
  console.log(formattedErrors);
  // { name: ['...'], email: ['...'], age: ['...'] }
}
```

---

## 23.10 Infer TypeScript Types จาก Zod

```typescript
import { z } from 'zod';

// กำหนด schema ครั้งเดียว
const ProductSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(200),
  description: z.string().max(1000).optional(),
  price: z.number().positive().multipleOf(0.01),
  stock: z.number().int().nonnegative(),
  category: z.enum(['electronics', 'clothing', 'food', 'books']),
  images: z.array(z.string().url()),
  tags: z.array(z.string()).max(10),
  isActive: z.boolean().default(true),
  metadata: z.record(z.string()).optional(),
  createdAt: z.date(),
  updatedAt: z.date(),
});

// Infer types จาก schema
type Product = z.infer<typeof ProductSchema>;
// ได้ type เหมือนกำหนด interface เอง แต่ไม่ต้องเขียนซ้ำ

// สร้าง schemas ที่ derive จาก ProductSchema
const CreateProductSchema = ProductSchema.omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

const UpdateProductSchema = ProductSchema.partial().omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

const ProductResponseSchema = ProductSchema.pick({
  id: true,
  name: true,
  price: true,
  stock: true,
  category: true,
});

// Types ถูก infer อัตโนมัติ
type CreateProductInput = z.infer<typeof CreateProductSchema>;
type UpdateProductInput = z.infer<typeof UpdateProductSchema>;
type ProductResponse = z.infer<typeof ProductResponseSchema>;

// ตัวอย่างการใช้งาน
function createProduct(input: CreateProductInput): Product {
  const now = new Date();
  return {
    id: crypto.randomUUID(),
    ...input,
    createdAt: now,
    updatedAt: now,
  };
}

// Generic helper สำหรับ validate
function validate<T>(schema: z.ZodSchema<T>, data: unknown): T {
  const result = schema.safeParse(data);
  if (!result.success) {
    throw result.error;
  }
  return result.data;
}

const product = validate(CreateProductSchema, {
  name: 'MacBook Pro',
  price: 59900,
  stock: 10,
  category: 'electronics',
  images: ['https://example.com/img.jpg'],
  tags: ['laptop', 'apple'],
});
```

---

## 23.11 Integration กับ Express

### src/middleware/zodValidate.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { z, ZodSchema, ZodError } from 'zod';

type ValidateTarget = 'body' | 'query' | 'params';

function formatZodError(error: ZodError): { field: string; message: string }[] {
  return error.errors.map(err => ({
    field: err.path.join('.') || '_root',
    message: err.message,
  }));
}

// Middleware สำหรับ validate body
export function validateBody<T>(schema: ZodSchema<T>) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const result = schema.safeParse(req.body);
    
    if (!result.success) {
      res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: formatZodError(result.error),
      });
      return;
    }
    
    // แทนที่ req.body ด้วยข้อมูลที่ผ่านการ validate และ transform แล้ว
    req.body = result.data;
    next();
  };
}

// Middleware สำหรับ validate query params
export function validateQuery<T>(schema: ZodSchema<T>) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const result = schema.safeParse(req.query);
    
    if (!result.success) {
      res.status(400).json({
        success: false,
        message: 'Query validation failed',
        errors: formatZodError(result.error),
      });
      return;
    }
    
    // Override query ด้วยข้อมูลที่ถูก parse แล้ว
    (req as any).validatedQuery = result.data;
    next();
  };
}

// Middleware สำหรับ validate route params
export function validateParams<T>(schema: ZodSchema<T>) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const result = schema.safeParse(req.params);
    
    if (!result.success) {
      res.status(400).json({
        success: false,
        message: 'Parameter validation failed',
        errors: formatZodError(result.error),
      });
      return;
    }
    
    next();
  };
}

// Generic middleware ที่ validate หลาย targets
export function validate(schemas: {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const errors: { field: string; message: string }[] = [];
    
    if (schemas.body) {
      const result = schemas.body.safeParse(req.body);
      if (!result.success) {
        errors.push(...formatZodError(result.error).map(e => ({
          ...e,
          field: `body.${e.field}`,
        })));
      } else {
        req.body = result.data;
      }
    }
    
    if (schemas.query) {
      const result = schemas.query.safeParse(req.query);
      if (!result.success) {
        errors.push(...formatZodError(result.error).map(e => ({
          ...e,
          field: `query.${e.field}`,
        })));
      }
    }
    
    if (schemas.params) {
      const result = schemas.params.safeParse(req.params);
      if (!result.success) {
        errors.push(...formatZodError(result.error).map(e => ({
          ...e,
          field: `params.${e.field}`,
        })));
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
```

### src/schemas/userSchemas.ts

```typescript
import { z } from 'zod';

// Password schema ที่ใช้ซ้ำได้
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .max(100, 'Password is too long')
  .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
  .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
  .regex(/[0-9]/, 'Password must contain at least one number');

// Schemas สำหรับ User
export const CreateUserSchema = z.object({
  name: z.string()
    .trim()
    .min(2, 'Name must be at least 2 characters')
    .max(100, 'Name must not exceed 100 characters'),
  email: z.string()
    .email('Invalid email format')
    .toLowerCase()
    .trim(),
  password: passwordSchema,
  role: z.enum(['admin', 'moderator', 'user']).optional().default('user'),
});

export const UpdateUserSchema = z.object({
  name: z.string().trim().min(2).max(100).optional(),
  email: z.string().email().toLowerCase().trim().optional(),
  bio: z.string().max(500).optional(),
  avatar: z.string().url('Invalid avatar URL').optional(),
}).refine(
  data => Object.keys(data).length > 0,
  'At least one field must be provided'
);

export const LoginSchema = z.object({
  email: z.string().email('Invalid email format').toLowerCase().trim(),
  password: z.string().min(1, 'Password is required'),
});

export const ChangePasswordSchema = z.object({
  currentPassword: z.string().min(1, 'Current password is required'),
  newPassword: passwordSchema,
  confirmPassword: z.string(),
}).refine(
  data => data.newPassword === data.confirmPassword,
  {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  }
).refine(
  data => data.currentPassword !== data.newPassword,
  {
    message: 'New password must be different from current password',
    path: ['newPassword'],
  }
);

export const UserQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(10),
  search: z.string().optional(),
  role: z.enum(['admin', 'moderator', 'user']).optional(),
  isActive: z.coerce.boolean().optional(),
  sort: z.enum(['name', 'email', 'createdAt']).optional().default('createdAt'),
  order: z.enum(['asc', 'desc']).optional().default('desc'),
});

// Infer types
export type CreateUserInput = z.infer<typeof CreateUserSchema>;
export type UpdateUserInput = z.infer<typeof UpdateUserSchema>;
export type LoginInput = z.infer<typeof LoginSchema>;
export type ChangePasswordInput = z.infer<typeof ChangePasswordSchema>;
export type UserQuery = z.infer<typeof UserQuerySchema>;
```

### src/schemas/postSchemas.ts

```typescript
import { z } from 'zod';

// Helper สำหรับสร้าง slug
const slugSchema = z.string()
  .regex(/^[a-z0-9-]+$/, 'Slug must contain only lowercase letters, numbers, and hyphens')
  .min(3)
  .max(100);

export const CreatePostSchema = z.object({
  title: z.string()
    .trim()
    .min(5, 'Title must be at least 5 characters')
    .max(255, 'Title must not exceed 255 characters'),
  content: z.string()
    .min(100, 'Content must be at least 100 characters'),
  excerpt: z.string()
    .max(500, 'Excerpt must not exceed 500 characters')
    .optional(),
  coverImage: z.string().url('Invalid cover image URL').optional(),
  categoryId: z.string().uuid('Invalid category ID').optional(),
  tagIds: z.array(z.string().uuid('Invalid tag ID'))
    .max(10, 'Maximum 10 tags allowed')
    .default([]),
  status: z.enum(['draft', 'published']).default('draft'),
});

export const UpdatePostSchema = CreatePostSchema.partial().refine(
  data => Object.keys(data).length > 0,
  'At least one field must be provided'
);

export const PostQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(50).default(10),
  search: z.string().optional(),
  categoryId: z.string().uuid().optional(),
  tagId: z.string().uuid().optional(),
  authorId: z.string().uuid().optional(),
  status: z.enum(['draft', 'published', 'archived']).optional(),
  sort: z.enum(['title', 'createdAt', 'publishedAt', 'viewCount']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

export const IdParamSchema = z.object({
  id: z.string().uuid('Invalid ID format'),
});

// Infer types
export type CreatePostInput = z.infer<typeof CreatePostSchema>;
export type UpdatePostInput = z.infer<typeof UpdatePostSchema>;
export type PostQuery = z.infer<typeof PostQuerySchema>;
export type IdParam = z.infer<typeof IdParamSchema>;
```

---

## 23.12 การใช้งาน Zod กับ Routes

### src/routes/userRoutes.ts พร้อม Zod validation

```typescript
import { Router } from 'express';
import { UserController } from '../controllers/userController';
import { validate, validateBody, validateQuery, validateParams } from '../middleware/zodValidate';
import {
  CreateUserSchema,
  UpdateUserSchema,
  LoginSchema,
  UserQuerySchema,
} from '../schemas/userSchemas';
import { IdParamSchema } from '../schemas/postSchemas';

export function createUserRouter(): Router {
  const router = Router();
  const controller = new UserController();
  
  // GET /users - list users
  router.get(
    '/',
    validateQuery(UserQuerySchema),
    controller.getAll
  );
  
  // GET /users/:id - get user by id
  router.get(
    '/:id',
    validateParams(IdParamSchema),
    controller.getById
  );
  
  // POST /users - create user
  router.post(
    '/',
    validateBody(CreateUserSchema),
    controller.create
  );
  
  // PUT /users/:id - update user
  router.put(
    '/:id',
    validate({
      params: IdParamSchema,
      body: UpdateUserSchema,
    }),
    controller.update
  );
  
  // DELETE /users/:id - delete user
  router.delete(
    '/:id',
    validateParams(IdParamSchema),
    controller.delete
  );
  
  return router;
}
```

---

## 23.13 Form Validation

### ตัวอย่าง Form Validation สำหรับ Frontend

```typescript
import { z } from 'zod';

// Registration form schema
export const RegistrationFormSchema = z.object({
  firstName: z.string()
    .trim()
    .min(1, 'กรุณาระบุชื่อ')
    .max(50, 'ชื่อต้องไม่เกิน 50 ตัวอักษร'),
  lastName: z.string()
    .trim()
    .min(1, 'กรุณาระบุนามสกุล')
    .max(50, 'นามสกุลต้องไม่เกิน 50 ตัวอักษร'),
  email: z.string()
    .min(1, 'กรุณาระบุอีเมล')
    .email('รูปแบบอีเมลไม่ถูกต้อง'),
  phone: z.string()
    .regex(/^(0[6-9][0-9]{8}|0[2-9][0-9]{7})$/, 'รูปแบบเบอร์โทรไม่ถูกต้อง')
    .optional(),
  password: z.string()
    .min(8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    .regex(/[A-Z]/, 'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
    .regex(/[a-z]/, 'ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว')
    .regex(/[0-9]/, 'ต้องมีตัวเลขอย่างน้อย 1 ตัว'),
  confirmPassword: z.string()
    .min(1, 'กรุณายืนยันรหัสผ่าน'),
  birthDate: z.string()
    .refine(
      val => !isNaN(Date.parse(val)),
      'รูปแบบวันเกิดไม่ถูกต้อง'
    )
    .refine(
      val => {
        const birth = new Date(val);
        const today = new Date();
        const age = today.getFullYear() - birth.getFullYear();
        return age >= 13;
      },
      'ต้องมีอายุอย่างน้อย 13 ปี'
    ),
  acceptTerms: z.boolean()
    .refine(val => val === true, 'กรุณายอมรับเงื่อนไขการใช้บริการ'),
}).refine(
  data => data.password === data.confirmPassword,
  {
    message: 'รหัสผ่านไม่ตรงกัน',
    path: ['confirmPassword'],
  }
);

type RegistrationFormData = z.infer<typeof RegistrationFormSchema>;

// ฟังก์ชัน validate form
function validateForm(data: unknown): {
  success: boolean;
  data?: RegistrationFormData;
  errors?: Record<string, string>;
} {
  const result = RegistrationFormSchema.safeParse(data);
  
  if (!result.success) {
    const errors: Record<string, string> = {};
    result.error.errors.forEach(err => {
      const field = err.path.join('.');
      if (!errors[field]) {
        errors[field] = err.message;
      }
    });
    
    return { success: false, errors };
  }
  
  return { success: true, data: result.data };
}

// ตัวอย่างการใช้งาน
const formData = {
  firstName: 'สมชาย',
  lastName: 'ใจดี',
  email: 'somchai@example.com',
  password: 'Password123',
  confirmPassword: 'Password123',
  birthDate: '1990-05-15',
  acceptTerms: true,
};

const validation = validateForm(formData);
if (validation.success) {
  console.log('Form is valid:', validation.data);
} else {
  console.log('Form errors:', validation.errors);
}
```

---

## 23.14 API Request Validation ที่สมบูรณ์

### src/schemas/apiSchemas.ts - Schemas สำหรับ Blog API

```typescript
import { z } from 'zod';

// ========== Common Schemas ==========
export const PaginationSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(10),
});

export const SearchSchema = z.object({
  q: z.string().min(1).max(200).optional(),
});

export const IdSchema = z.object({
  id: z.string().uuid('Invalid ID'),
});

// ========== Auth Schemas ==========
export const RegisterSchema = z.object({
  name: z.string().trim().min(2, 'Name too short').max(100, 'Name too long'),
  email: z.string().email('Invalid email').toLowerCase(),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

export const LoginSchema = z.object({
  email: z.string().email('Invalid email').toLowerCase(),
  password: z.string().min(1, 'Password required'),
});

export const RefreshTokenSchema = z.object({
  refreshToken: z.string().min(1, 'Refresh token required'),
});

// ========== User Schemas ==========
export const UpdateProfileSchema = z.object({
  name: z.string().trim().min(2).max(100).optional(),
  bio: z.string().max(500).optional(),
  avatar: z.string().url().optional(),
}).refine(data => Object.keys(data).length > 0, 'No fields to update');

export const ChangePasswordSchema = z.object({
  currentPassword: z.string().min(1, 'Current password required'),
  newPassword: z.string().min(8, 'New password must be at least 8 characters'),
}).refine(
  data => data.currentPassword !== data.newPassword,
  { message: 'New password must differ from current', path: ['newPassword'] }
);

// ========== Post Schemas ==========
export const CreatePostSchema = z.object({
  title: z.string().trim().min(5, 'Title too short').max(255, 'Title too long'),
  content: z.string().min(10, 'Content too short'),
  excerpt: z.string().max(500).optional(),
  coverImage: z.string().url().optional(),
  categoryId: z.string().uuid().optional(),
  tagIds: z.array(z.string().uuid()).max(10).default([]),
  status: z.enum(['draft', 'published']).default('draft'),
});

export const UpdatePostSchema = z.object({
  title: z.string().trim().min(5).max(255).optional(),
  content: z.string().min(10).optional(),
  excerpt: z.string().max(500).optional().nullable(),
  coverImage: z.string().url().optional().nullable(),
  categoryId: z.string().uuid().optional().nullable(),
  tagIds: z.array(z.string().uuid()).max(10).optional(),
  status: z.enum(['draft', 'published', 'archived']).optional(),
}).refine(data => Object.keys(data).length > 0, 'No fields to update');

export const PostQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(50).default(10),
  q: z.string().max(200).optional(),
  category: z.string().uuid().optional(),
  tag: z.string().uuid().optional(),
  author: z.string().uuid().optional(),
  sort: z.enum(['title', 'publishedAt', 'viewCount', 'likeCount']).default('publishedAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

// ========== Comment Schemas ==========
export const CreateCommentSchema = z.object({
  content: z.string().trim().min(1, 'Comment cannot be empty').max(2000, 'Comment too long'),
  parentId: z.string().uuid().optional(),
});

export const UpdateCommentSchema = z.object({
  content: z.string().trim().min(1).max(2000),
});

// ========== Category Schemas ==========
export const CreateCategorySchema = z.object({
  name: z.string().trim().min(2).max(100),
  slug: z.string().regex(/^[a-z0-9-]+$/).min(2).max(100),
  description: z.string().max(500).optional(),
  coverImage: z.string().url().optional(),
});

// ========== Tag Schemas ==========
export const CreateTagSchema = z.object({
  name: z.string().trim().min(2).max(50),
  slug: z.string().regex(/^[a-z0-9-]+$/).min(2).max(50),
  color: z.string().regex(/^#[0-9A-F]{6}$/i, 'Invalid hex color').optional(),
});

// ========== Export types ==========
export type RegisterInput = z.infer<typeof RegisterSchema>;
export type LoginInput = z.infer<typeof LoginSchema>;
export type UpdateProfileInput = z.infer<typeof UpdateProfileSchema>;
export type ChangePasswordInput = z.infer<typeof ChangePasswordSchema>;
export type CreatePostInput = z.infer<typeof CreatePostSchema>;
export type UpdatePostInput = z.infer<typeof UpdatePostSchema>;
export type PostQuery = z.infer<typeof PostQuerySchema>;
export type CreateCommentInput = z.infer<typeof CreateCommentSchema>;
export type CreateCategoryInput = z.infer<typeof CreateCategorySchema>;
export type CreateTagInput = z.infer<typeof CreateTagSchema>;
```

---

## 23.15 Advanced Patterns

### Schema Composition

```typescript
import { z } from 'zod';

// Base schemas ที่ใช้ซ้ำได้
const TimestampSchema = z.object({
  createdAt: z.date(),
  updatedAt: z.date(),
});

const IdEntitySchema = z.object({
  id: z.string().uuid(),
});

const BaseEntitySchema = IdEntitySchema.merge(TimestampSchema);

// สร้าง schemas ที่ extend จาก base
const UserEntitySchema = BaseEntitySchema.extend({
  name: z.string(),
  email: z.string().email(),
  role: z.enum(['admin', 'user']),
});

const PostEntitySchema = BaseEntitySchema.extend({
  title: z.string(),
  content: z.string(),
  authorId: z.string().uuid(),
});
```

### Conditional Validation

```typescript
import { z } from 'zod';

// Validation แบบ conditional
const ShippingSchema = z.object({
  deliveryMethod: z.enum(['pickup', 'delivery']),
  address: z.string().optional(),
  phone: z.string().optional(),
}).superRefine((data, ctx) => {
  if (data.deliveryMethod === 'delivery') {
    if (!data.address) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Address is required for delivery',
        path: ['address'],
      });
    }
    if (!data.phone) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Phone is required for delivery',
        path: ['phone'],
      });
    }
  }
});

// Conditional type
const PaymentSchema = z.discriminatedUnion('method', [
  z.object({
    method: z.literal('credit_card'),
    cardNumber: z.string().regex(/^\d{16}$/),
    expiryDate: z.string().regex(/^\d{2}\/\d{2}$/),
    cvv: z.string().regex(/^\d{3,4}$/),
  }),
  z.object({
    method: z.literal('bank_transfer'),
    bankName: z.string(),
    accountNumber: z.string(),
  }),
  z.object({
    method: z.literal('promptpay'),
    promptpayId: z.string().regex(/^\d{10,13}$/),
  }),
]);

type Payment = z.infer<typeof PaymentSchema>;
```

### Recursive Schema

```typescript
import { z } from 'zod';

// Recursive schema สำหรับ nested comments
type Comment = {
  id: string;
  content: string;
  author: string;
  replies: Comment[];
};

const CommentSchema: z.ZodType<Comment> = z.lazy(() =>
  z.object({
    id: z.string().uuid(),
    content: z.string(),
    author: z.string(),
    replies: z.array(CommentSchema),
  })
);

// Recursive schema สำหรับ folder structure
type TreeNode = {
  name: string;
  type: 'file' | 'folder';
  children?: TreeNode[];
};

const TreeNodeSchema: z.ZodType<TreeNode> = z.lazy(() =>
  z.object({
    name: z.string(),
    type: z.enum(['file', 'folder']),
    children: z.array(TreeNodeSchema).optional(),
  })
);
```

---

## สรุปบทที่ 23

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Zod** - ปัญหาของการกำหนด types และ validation แยกกัน
2. **Basic Schema Types** - string, number, boolean, date schemas
3. **Object Schemas** - partial, pick, omit, merge, extend
4. **Array Schemas** - array, tuple, set
5. **Union Schemas** - union, discriminated union, enum
6. **Refinements** - custom validation ด้วย refine และ superRefine
7. **Transforms** - การแปลงข้อมูลด้วย transform และ preprocess
8. **Error Handling** - parse vs safeParse, format errors
9. **Type Inference** - z.infer ดึง TypeScript type จาก schema
10. **Express Integration** - middleware สำหรับ validate request
11. **Form Validation** - validation สำหรับ HTML forms
12. **API Validation** - schemas สำหรับ REST API ที่สมบูรณ์
13. **Advanced Patterns** - composition, conditional, recursive schemas

Zod ช่วยให้โค้ด TypeScript ของเรามีความปลอดภัยมากขึ้นด้วยการรับประกันว่าข้อมูลที่เข้ามาใน application มีรูปแบบที่ถูกต้องตามที่คาดหวัง
