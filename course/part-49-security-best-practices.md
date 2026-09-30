# ส่วนที่ 49: Security Best Practices ใน TypeScript

## บทนำ

การรักษาความปลอดภัยของแอปพลิเคชัน TypeScript เป็นสิ่งที่ต้องให้ความสำคัญสูงสุด บทนี้จะครอบคลุมแนวทางปฏิบัติที่ดีที่สุดด้านความปลอดภัย ตั้งแต่การตรวจสอบข้อมูลนำเข้า การป้องกัน SQL Injection, XSS, CSRF และอื่นๆ อีกมากมาย

---

## 1. Input Validation and Sanitization (การตรวจสอบและทำความสะอาดข้อมูลนำเข้า)

### 1.1 การใช้ Zod สำหรับ Runtime Validation

```typescript
import { z } from 'zod';

// ✅ กำหนด schema สำหรับ input validation
const UserSchema = z.object({
  username: z
    .string()
    .min(3, 'Username ต้องมีอย่างน้อย 3 ตัวอักษร')
    .max(50, 'Username ต้องไม่เกิน 50 ตัวอักษร')
    .regex(/^[a-zA-Z0-9_-]+$/, 'Username ต้องใช้ตัวอักษรและตัวเลขเท่านั้น'),
  
  email: z
    .string()
    .email('รูปแบบ email ไม่ถูกต้อง')
    .toLowerCase(),
  
  password: z
    .string()
    .min(8, 'Password ต้องมีอย่างน้อย 8 ตัวอักษร')
    .regex(/[A-Z]/, 'Password ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
    .regex(/[0-9]/, 'Password ต้องมีตัวเลขอย่างน้อย 1 ตัว')
    .regex(/[!@#$%^&*]/, 'Password ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว'),
  
  age: z
    .number()
    .int('Age ต้องเป็นจำนวนเต็ม')
    .min(0, 'Age ต้องไม่ติดลบ')
    .max(150, 'Age ไม่สมเหตุสมผล'),
  
  website: z
    .string()
    .url('รูปแบบ URL ไม่ถูกต้อง')
    .optional(),
});

type UserInput = z.infer<typeof UserSchema>;

// ✅ Middleware สำหรับ validate request body
import { Request, Response, NextFunction } from 'express';

function validateBody<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.body);
    
    if (!result.success) {
      res.status(400).json({
        error: 'Validation failed',
        details: result.error.errors.map(err => ({
          field: err.path.join('.'),
          message: err.message,
        })),
      });
      return;
    }
    
    req.body = result.data;
    next();
  };
}

// ✅ Route ที่มีการ validate
import express from 'express';
const router = express.Router();

router.post('/users', validateBody(UserSchema), async (req: Request, res: Response) => {
  const userData: UserInput = req.body; // Type-safe ที่นี่
  // ดำเนินการสร้าง user...
  res.status(201).json({ message: 'User created successfully' });
});
```

### 1.2 HTML Sanitization

```typescript
import DOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';

// ✅ Sanitize HTML input เพื่อป้องกัน XSS
class HTMLSanitizer {
  private purify: DOMPurify.DOMPurifyI;

  constructor() {
    const window = new JSDOM('').window as unknown as Window;
    this.purify = DOMPurify(window);
  }

  sanitize(dirty: string, options?: DOMPurify.Config): string {
    return this.purify.sanitize(dirty, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li'],
      ALLOWED_ATTR: ['href', 'title', 'class'],
      ...options,
    });
  }

  sanitizeStrict(dirty: string): string {
    return this.purify.sanitize(dirty, {
      ALLOWED_TAGS: [],
      ALLOWED_ATTR: [],
    });
  }

  stripTags(html: string): string {
    return this.purify.sanitize(html, {
      ALLOWED_TAGS: [],
      ALLOWED_ATTR: [],
    }).replace(/\s+/g, ' ').trim();
  }
}

// ✅ String sanitization utilities
class StringSanitizer {
  static escapeHtml(str: string): string {
    const htmlEntities: Record<string, string> = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#39;',
      '/': '&#x2F;',
    };
    return str.replace(/[&<>"'/]/g, char => htmlEntities[char] ?? char);
  }

  static escapeSql(str: string): string {
    return str.replace(/[\0\n\r\b\t\\'"\x1a]/g, (char) => {
      switch (char) {
        case '\0': return '\\0';
        case '\n': return '\\n';
        case '\r': return '\\r';
        case '\b': return '\\b';
        case '\t': return '\\t';
        case '\x1a': return '\\Z';
        case "'": return "''";
        case '"': return '\\"';
        case '\\': return '\\\\';
        default: return char;
      }
    });
  }

  static sanitizeFilename(filename: string): string {
    return filename
      .replace(/[^a-zA-Z0-9._-]/g, '_')
      .replace(/_{2,}/g, '_')
      .substring(0, 255);
  }

  static sanitizeUrl(url: string): string | null {
    try {
      const parsed = new URL(url);
      // อนุญาตเฉพาะ http และ https
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        return null;
      }
      return parsed.toString();
    } catch {
      return null;
    }
  }
}
```

---

## 2. SQL Injection Prevention (การป้องกัน SQL Injection)

### 2.1 Parameterized Queries

```typescript
import { Pool, PoolClient } from 'pg';

// ✅ ใช้ parameterized queries เสมอ
class UserRepository {
  constructor(private pool: Pool) {}

  // ❌ อย่าทำแบบนี้! Vulnerable to SQL Injection
  async findUserUnsafe(username: string): Promise<unknown> {
    const query = `SELECT * FROM users WHERE username = '${username}'`;
    // หากส่ง username = "admin' OR '1'='1" จะ bypass authentication!
    return this.pool.query(query);
  }

  // ✅ ใช้ parameterized queries
  async findUser(username: string): Promise<UserRecord | null> {
    const result = await this.pool.query<UserRecord>(
      'SELECT id, username, email, role FROM users WHERE username = $1 AND active = true',
      [username] // parameter แยกจาก query
    );
    return result.rows[0] ?? null;
  }

  async findUsers(filters: UserFilters): Promise<UserRecord[]> {
    const params: unknown[] = [];
    const conditions: string[] = [];

    if (filters.username) {
      params.push(`%${filters.username}%`);
      conditions.push(`username ILIKE $${params.length}`);
    }

    if (filters.email) {
      params.push(filters.email);
      conditions.push(`email = $${params.length}`);
    }

    if (filters.role) {
      params.push(filters.role);
      conditions.push(`role = $${params.length}`);
    }

    const whereClause = conditions.length > 0
      ? `WHERE ${conditions.join(' AND ')}`
      : '';

    const result = await this.pool.query<UserRecord>(
      `SELECT id, username, email, role FROM users ${whereClause} ORDER BY created_at DESC`,
      params
    );

    return result.rows;
  }

  async createUser(data: CreateUserData): Promise<UserRecord> {
    const result = await this.pool.query<UserRecord>(
      `INSERT INTO users (username, email, password_hash, role)
       VALUES ($1, $2, $3, $4)
       RETURNING id, username, email, role, created_at`,
      [data.username, data.email, data.passwordHash, data.role ?? 'user']
    );
    return result.rows[0];
  }

  async updateUser(
    id: number,
    updates: Partial<Pick<UserRecord, 'email' | 'role'>>
  ): Promise<UserRecord | null> {
    const setClause: string[] = [];
    const params: unknown[] = [];

    if (updates.email !== undefined) {
      params.push(updates.email);
      setClause.push(`email = $${params.length}`);
    }

    if (updates.role !== undefined) {
      params.push(updates.role);
      setClause.push(`role = $${params.length}`);
    }

    if (setClause.length === 0) return null;

    params.push(id);
    const result = await this.pool.query<UserRecord>(
      `UPDATE users SET ${setClause.join(', ')}, updated_at = NOW()
       WHERE id = $${params.length}
       RETURNING id, username, email, role`,
      params
    );

    return result.rows[0] ?? null;
  }
}

interface UserRecord {
  id: number;
  username: string;
  email: string;
  role: string;
  created_at?: Date;
}

interface UserFilters {
  username?: string;
  email?: string;
  role?: string;
}

interface CreateUserData {
  username: string;
  email: string;
  passwordHash: string;
  role?: string;
}
```

### 2.2 ORM สำหรับ SQL Injection Prevention

```typescript
import { DataSource, Entity, Column, PrimaryGeneratedColumn, Repository } from 'typeorm';

// ✅ TypeORM entities ที่ type-safe
@Entity('users')
class UserEntity {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column({ unique: true, length: 50 })
  username!: string;

  @Column({ unique: true })
  email!: string;

  @Column({ name: 'password_hash' })
  passwordHash!: string;

  @Column({ default: 'user' })
  role!: string;

  @Column({ default: true })
  active!: boolean;
}

// ✅ Repository pattern ที่ปลอดภัย
class SecureUserService {
  constructor(private userRepo: Repository<UserEntity>) {}

  async findByUsername(username: string): Promise<UserEntity | null> {
    // TypeORM ใช้ parameterized queries อัตโนมัติ
    return this.userRepo.findOne({ where: { username, active: true } });
  }

  async searchUsers(searchTerm: string): Promise<UserEntity[]> {
    return this.userRepo
      .createQueryBuilder('user')
      .where('user.username ILIKE :search OR user.email ILIKE :search', {
        search: `%${searchTerm}%`, // TypeORM escape ให้อัตโนมัติ
      })
      .andWhere('user.active = :active', { active: true })
      .orderBy('user.username', 'ASC')
      .limit(50)
      .getMany();
  }
}
```

---

## 3. XSS Prevention (การป้องกัน Cross-Site Scripting)

### 3.1 Content Security Policy

```typescript
import helmet from 'helmet';
import express from 'express';

const app = express();

// ✅ ตั้งค่า Content Security Policy ที่เข้มงวด
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: [
          "'self'",
          // ถ้าใช้ nonce สำหรับ inline scripts
          (req, res) => `'nonce-${(res as express.Response).locals.nonce}'`,
        ],
        styleSrc: ["'self'", "'unsafe-inline'"], // เฉพาะ styles
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: ["'self'", 'https://api.example.com'],
        fontSrc: ["'self'", 'https://fonts.gstatic.com'],
        objectSrc: ["'none'"],
        mediaSrc: ["'self'"],
        frameSrc: ["'none'"],
        frameAncestors: ["'none'"],
        formAction: ["'self'"],
        upgradeInsecureRequests: [],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  })
);

// ✅ Nonce generation สำหรับ inline scripts
import crypto from 'crypto';

function generateNonce(): string {
  return crypto.randomBytes(16).toString('base64');
}

app.use((req, res, next) => {
  res.locals.nonce = generateNonce();
  next();
});
```

### 3.2 Output Encoding

```typescript
// ✅ Template engine ที่ auto-escape HTML
import Handlebars from 'handlebars';

// Handlebars escape HTML อัตโนมัติด้วย {{ }}
// ใช้ {{{ }}} เฉพาะเมื่อต้องการ raw HTML ที่ผ่านการ sanitize แล้วเท่านั้น

const template = Handlebars.compile(`
  <div class="user-profile">
    <h1>{{username}}</h1>  <!-- Auto-escaped -->
    <p>{{bio}}</p>          <!-- Auto-escaped -->
    {{{safeHtml}}}          <!-- NOT escaped - ต้องผ่าน sanitize แล้ว -->
  </div>
`);

interface ProfileData {
  username: string;
  bio: string;
  safeHtml: string; // ต้องผ่าน DOMPurify แล้ว
}

// ✅ React ก็ auto-escape ให้อัตโนมัติ (ตัวอย่าง JSX)
// เฉพาะ dangerouslySetInnerHTML เท่านั้นที่ไม่ escape
// ต้องใช้ DOMPurify ก่อนเสมอ

function UserProfile({ username, bio }: { username: string; bio: string }) {
  // username และ bio จะถูก escape อัตโนมัติโดย React
  return `<div><h1>${escapeHtml(username)}</h1><p>${escapeHtml(bio)}</p></div>`;
}

function escapeHtml(str: string): string {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}
```

---

## 4. CSRF Protection (การป้องกัน Cross-Site Request Forgery)

### 4.1 CSRF Token Implementation

```typescript
import crypto from 'crypto';
import { Request, Response, NextFunction } from 'express';

// ✅ CSRF Token Manager
class CSRFProtection {
  private readonly tokenLength: number;
  private readonly headerName: string;
  private readonly cookieName: string;

  constructor(options: {
    tokenLength?: number;
    headerName?: string;
    cookieName?: string;
  } = {}) {
    this.tokenLength = options.tokenLength ?? 32;
    this.headerName = options.headerName ?? 'X-CSRF-Token';
    this.cookieName = options.cookieName ?? '_csrf';
  }

  generateToken(): string {
    return crypto.randomBytes(this.tokenLength).toString('hex');
  }

  middleware() {
    return (req: Request, res: Response, next: NextFunction): void => {
      // GET, HEAD, OPTIONS ไม่ต้องตรวจสอบ
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        // Generate token ใหม่ถ้ายังไม่มี
        if (!req.cookies[this.cookieName]) {
          const token = this.generateToken();
          res.cookie(this.cookieName, token, {
            httpOnly: false, // ต้องเข้าถึงได้จาก JavaScript สำหรับ AJAX
            secure: process.env.NODE_ENV === 'production',
            sameSite: 'strict',
            maxAge: 3600000, // 1 hour
          });
          res.locals.csrfToken = token;
        } else {
          res.locals.csrfToken = req.cookies[this.cookieName];
        }
        next();
        return;
      }

      // ตรวจสอบ CSRF token สำหรับ state-changing requests
      const cookieToken = req.cookies[this.cookieName];
      const headerToken = req.headers[this.headerName.toLowerCase()] as string;
      const bodyToken = req.body?._csrf;

      const submittedToken = headerToken ?? bodyToken;

      if (!cookieToken || !submittedToken) {
        res.status(403).json({ error: 'CSRF token missing' });
        return;
      }

      // Constant-time comparison เพื่อป้องกัน timing attacks
      if (!this.safeCompare(cookieToken, submittedToken)) {
        res.status(403).json({ error: 'CSRF token invalid' });
        return;
      }

      next();
    };
  }

  private safeCompare(a: string, b: string): boolean {
    if (a.length !== b.length) return false;
    
    let result = 0;
    for (let i = 0; i < a.length; i++) {
      result |= a.charCodeAt(i) ^ b.charCodeAt(i);
    }
    return result === 0;
  }
}

const csrf = new CSRFProtection();
app.use(csrf.middleware());

// ✅ ใน Frontend - ส่ง CSRF token ใน header
async function apiRequest<T>(
  url: string,
  options: RequestInit = {}
): Promise<T> {
  const csrfToken = getCookie('_csrf');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-Token': csrfToken ?? '',
      ...options.headers,
    },
    credentials: 'same-origin', // ส่ง cookies
  });
  
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  
  return response.json();
}

function getCookie(name: string): string | null {
  const match = document.cookie.match(
    new RegExp('(^| )' + name + '=([^;]+)')
  );
  return match ? decodeURIComponent(match[2]) : null;
}
```

---

## 5. Authentication Security (ความปลอดภัยของ Authentication)

### 5.1 Secure Password Handling

```typescript
import bcrypt from 'bcrypt';
import { z } from 'zod';

// ✅ Password hashing ที่ปลอดภัย
class PasswordService {
  private readonly saltRounds: number;
  
  constructor(saltRounds = 12) {
    this.saltRounds = saltRounds;
  }

  async hash(password: string): Promise<string> {
    return bcrypt.hash(password, this.saltRounds);
  }

  async verify(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }

  validateStrength(password: string): {
    isValid: boolean;
    score: number;
    feedback: string[];
  } {
    const feedback: string[] = [];
    let score = 0;

    if (password.length >= 8) score++;
    else feedback.push('Password ควรมีอย่างน้อย 8 ตัวอักษร');

    if (password.length >= 12) score++;
    
    if (/[a-z]/.test(password)) score++;
    else feedback.push('ควรมีตัวพิมพ์เล็ก');
    
    if (/[A-Z]/.test(password)) score++;
    else feedback.push('ควรมีตัวพิมพ์ใหญ่');
    
    if (/[0-9]/.test(password)) score++;
    else feedback.push('ควรมีตัวเลข');
    
    if (/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)) score++;
    else feedback.push('ควรมีอักขระพิเศษ');

    // ตรวจสอบ common passwords
    const commonPasswords = ['password', '123456', 'qwerty', 'admin'];
    if (commonPasswords.some(p => password.toLowerCase().includes(p))) {
      score = Math.max(0, score - 2);
      feedback.push('ไม่ควรใช้ password ที่คาดเดาได้ง่าย');
    }

    return {
      isValid: score >= 4,
      score: Math.min(score, 6),
      feedback,
    };
  }
}

// ✅ Account lockout protection
class AccountLockoutService {
  private failedAttempts = new Map<string, { count: number; lockedUntil?: Date }>();
  private readonly maxAttempts: number;
  private readonly lockoutDuration: number; // milliseconds

  constructor(maxAttempts = 5, lockoutDurationMs = 15 * 60 * 1000) {
    this.maxAttempts = maxAttempts;
    this.lockoutDuration = lockoutDurationMs;
  }

  isLocked(identifier: string): boolean {
    const record = this.failedAttempts.get(identifier);
    if (!record?.lockedUntil) return false;
    
    if (record.lockedUntil <= new Date()) {
      // Lockout หมดแล้ว reset
      this.failedAttempts.delete(identifier);
      return false;
    }
    return true;
  }

  recordFailure(identifier: string): void {
    const record = this.failedAttempts.get(identifier) ?? { count: 0 };
    record.count++;
    
    if (record.count >= this.maxAttempts) {
      record.lockedUntil = new Date(Date.now() + this.lockoutDuration);
    }
    
    this.failedAttempts.set(identifier, record);
  }

  recordSuccess(identifier: string): void {
    this.failedAttempts.delete(identifier);
  }

  getRemainingTime(identifier: string): number {
    const record = this.failedAttempts.get(identifier);
    if (!record?.lockedUntil) return 0;
    return Math.max(0, record.lockedUntil.getTime() - Date.now());
  }
}
```

---

## 6. JWT Security (ความปลอดภัยของ JWT)

### 6.1 Secure JWT Implementation

```typescript
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

interface TokenPayload {
  sub: string;      // user ID
  email: string;
  role: string;
  jti: string;      // JWT ID สำหรับ revocation
  iat?: number;
  exp?: number;
}

interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

// ✅ Secure JWT Service
class JWTService {
  private readonly accessTokenSecret: string;
  private readonly refreshTokenSecret: string;
  private readonly accessTokenExpiry: string;
  private readonly refreshTokenExpiry: string;
  private revokedTokens = new Set<string>(); // ในการผลิตจริงควรใช้ Redis

  constructor() {
    // ✅ โหลด secrets จาก environment variables เท่านั้น
    const accessSecret = process.env.JWT_ACCESS_SECRET;
    const refreshSecret = process.env.JWT_REFRESH_SECRET;
    
    if (!accessSecret || !refreshSecret) {
      throw new Error('JWT secrets must be configured in environment variables');
    }
    
    if (accessSecret.length < 32 || refreshSecret.length < 32) {
      throw new Error('JWT secrets must be at least 32 characters');
    }
    
    this.accessTokenSecret = accessSecret;
    this.refreshTokenSecret = refreshSecret;
    this.accessTokenExpiry = process.env.JWT_ACCESS_EXPIRY ?? '15m';
    this.refreshTokenExpiry = process.env.JWT_REFRESH_EXPIRY ?? '7d';
  }

  generateTokenPair(userId: string, email: string, role: string): TokenPair {
    const jti = crypto.randomUUID();
    
    const payload: Omit<TokenPayload, 'iat' | 'exp'> = {
      sub: userId,
      email,
      role,
      jti,
    };

    const accessToken = jwt.sign(payload, this.accessTokenSecret, {
      expiresIn: this.accessTokenExpiry,
      algorithm: 'HS256',
    });

    const refreshToken = jwt.sign(
      { ...payload, type: 'refresh' },
      this.refreshTokenSecret,
      {
        expiresIn: this.refreshTokenExpiry,
        algorithm: 'HS256',
      }
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: 15 * 60, // 15 minutes in seconds
    };
  }

  verifyAccessToken(token: string): TokenPayload {
    const decoded = jwt.verify(token, this.accessTokenSecret) as TokenPayload;
    
    if (this.revokedTokens.has(decoded.jti)) {
      throw new Error('Token has been revoked');
    }
    
    return decoded;
  }

  verifyRefreshToken(token: string): TokenPayload {
    const decoded = jwt.verify(token, this.refreshTokenSecret) as TokenPayload & { type: string };
    
    if (decoded.type !== 'refresh') {
      throw new Error('Invalid token type');
    }
    
    if (this.revokedTokens.has(decoded.jti)) {
      throw new Error('Token has been revoked');
    }
    
    return decoded;
  }

  revokeToken(jti: string): void {
    this.revokedTokens.add(jti);
    // ในการผลิตจริง: บันทึกลง Redis ด้วย TTL เท่ากับ token expiry
  }
}

// ✅ Authentication Middleware
function authMiddleware(jwtService: JWTService) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader?.startsWith('Bearer ')) {
      res.status(401).json({ error: 'No token provided' });
      return;
    }
    
    const token = authHeader.substring(7);
    
    try {
      const payload = jwtService.verifyAccessToken(token);
      req.user = payload; // เพิ่ม user ใน request
      next();
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
      } else if (error instanceof jwt.JsonWebTokenError) {
        res.status(401).json({ error: 'Invalid token' });
      } else {
        res.status(401).json({ error: 'Authentication failed' });
      }
    }
  };
}

// Type augmentation สำหรับ Express
declare global {
  namespace Express {
    interface Request {
      user?: TokenPayload;
    }
  }
}
```

---

## 7. Rate Limiting (การจำกัดอัตราการร้องขอ)

### 7.1 Rate Limiter Implementation

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { Redis } from 'ioredis';

// ✅ Rate limiter configurations
const redis = new Redis(process.env.REDIS_URL ?? 'redis://localhost:6379');

// General rate limit สำหรับทุก API
export const generalRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // จำกัด 100 requests ต่อ window
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many requests',
      message: 'กรุณารอสักครู่แล้วลองใหม่อีกครั้ง',
      retryAfter: Math.ceil((res.getHeader('X-RateLimit-Reset') as number - Date.now()) / 1000),
    });
  },
});

// Rate limit เข้มงวดสำหรับ Auth endpoints
export const authRateLimit = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10, // จำกัด 10 attempts ต่อชั่วโมง
  skipSuccessfulRequests: true, // นับเฉพาะ failed requests
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  keyGenerator: (req) => {
    // ใช้ IP + username เพื่อ rate limit ที่แม่นยำขึ้น
    return `${req.ip}:${req.body?.email ?? 'unknown'}`;
  },
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many authentication attempts',
      message: 'บัญชีของคุณถูกล็อคชั่วคราว กรุณาลองใหม่ใน 1 ชั่วโมง',
    });
  },
});

// ✅ Custom rate limiter แบบ sliding window
class SlidingWindowRateLimiter {
  private redis: Redis;
  private prefix: string;

  constructor(redis: Redis, prefix = 'rate_limit:') {
    this.redis = redis;
    this.prefix = prefix;
  }

  async isAllowed(
    key: string,
    maxRequests: number,
    windowMs: number
  ): Promise<{ allowed: boolean; remaining: number; resetAt: Date }> {
    const now = Date.now();
    const windowStart = now - windowMs;
    const redisKey = `${this.prefix}${key}`;

    // ใช้ Redis pipeline เพื่อ atomic operations
    const pipeline = this.redis.pipeline();
    
    // ลบ entries เก่า
    pipeline.zremrangebyscore(redisKey, '-inf', windowStart);
    // เพิ่ม request ปัจจุบัน
    pipeline.zadd(redisKey, now, `${now}-${Math.random()}`);
    // นับจำนวน requests ใน window
    pipeline.zcard(redisKey);
    // Set expiry
    pipeline.expire(redisKey, Math.ceil(windowMs / 1000));
    
    const results = await pipeline.exec();
    const count = results?.[2]?.[1] as number ?? 0;
    
    const allowed = count <= maxRequests;
    const remaining = Math.max(0, maxRequests - count);
    const resetAt = new Date(now + windowMs);
    
    return { allowed, remaining, resetAt };
  }
}
```

---

## 8. CORS Configuration (การตั้งค่า CORS)

### 8.1 Secure CORS Setup

```typescript
import cors from 'cors';

// ✅ CORS configuration ที่ปลอดภัย
const allowedOrigins = new Set([
  'https://www.example.com',
  'https://app.example.com',
  process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : null,
].filter(Boolean) as string[]);

const corsOptions: cors.CorsOptions = {
  origin: (origin, callback) => {
    // อนุญาต requests ที่ไม่มี origin (เช่น server-to-server)
    if (!origin) {
      callback(null, true);
      return;
    }
    
    if (allowedOrigins.has(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`CORS policy: Origin ${origin} is not allowed`));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'X-CSRF-Token',
    'X-Request-ID',
  ],
  exposedHeaders: [
    'X-Total-Count',
    'X-Page-Size',
    'X-Current-Page',
  ],
  credentials: true,
  maxAge: 86400, // Preflight cache 24 hours
  optionsSuccessStatus: 204,
};

app.use(cors(corsOptions));
```

---

## 9. Helmet.js Security Headers

### 9.1 การตั้งค่า Helmet อย่างสมบูรณ์

```typescript
import helmet from 'helmet';

// ✅ Helmet configuration ที่ครอบคลุม
app.use(
  helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        baseUri: ["'self'"],
        fontSrc: ["'self'", 'https:', 'data:'],
        formAction: ["'self'"],
        frameAncestors: ["'self'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        objectSrc: ["'none'"],
        scriptSrc: ["'self'"],
        scriptSrcAttr: ["'none'"],
        styleSrc: ["'self'", 'https:', "'unsafe-inline'"],
        upgradeInsecureRequests: [],
      },
    },
    
    // HTTP Strict Transport Security
    hsts: {
      maxAge: 63072000, // 2 years
      includeSubDomains: true,
      preload: true,
    },
    
    // Prevent clickjacking
    frameguard: {
      action: 'deny',
    },
    
    // Prevent MIME type sniffing
    noSniff: true,
    
    // X-Powered-By header removal
    hidePoweredBy: true,
    
    // XSS filter
    xssFilter: true,
    
    // Referrer Policy
    referrerPolicy: {
      policy: ['no-referrer', 'strict-origin-when-cross-origin'],
    },
    
    // Permissions Policy
    permittedCrossDomainPolicies: {
      permittedPolicies: 'none',
    },
  })
);

// ✅ Permissions-Policy header
app.use((req, res, next) => {
  res.setHeader(
    'Permissions-Policy',
    [
      'camera=()',
      'geolocation=()',
      'microphone=()',
      'payment=()',
      'usb=()',
      'fullscreen=(self)',
    ].join(', ')
  );
  next();
});
```

---

## 10. Environment Variable Security (ความปลอดภัยของ Environment Variables)

### 10.1 Secure Configuration Management

```typescript
import { z } from 'zod';
import dotenv from 'dotenv';

// โหลด .env ก่อน
dotenv.config();

// ✅ Validate environment variables ด้วย Zod
const EnvSchema = z.object({
  // Server
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.string().regex(/^\d+$/).transform(Number).default('3000'),
  HOST: z.string().default('0.0.0.0'),
  
  // Database
  DATABASE_URL: z.string().url('DATABASE_URL ต้องเป็น valid URL'),
  DATABASE_POOL_SIZE: z.string().regex(/^\d+$/).transform(Number).default('10'),
  
  // JWT
  JWT_ACCESS_SECRET: z.string().min(32, 'JWT_ACCESS_SECRET ต้องมีอย่างน้อย 32 ตัวอักษร'),
  JWT_REFRESH_SECRET: z.string().min(32, 'JWT_REFRESH_SECRET ต้องมีอย่างน้อย 32 ตัวอักษร'),
  JWT_ACCESS_EXPIRY: z.string().default('15m'),
  JWT_REFRESH_EXPIRY: z.string().default('7d'),
  
  // Redis
  REDIS_URL: z.string().url().default('redis://localhost:6379'),
  
  // External Services
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.string().regex(/^\d+$/).transform(Number).optional(),
  SMTP_USER: z.string().email().optional(),
  SMTP_PASS: z.string().optional(),
  
  // Encryption
  ENCRYPTION_KEY: z.string().length(64, 'ENCRYPTION_KEY ต้องเป็น 32 bytes hex string'),
  
  // CORS
  ALLOWED_ORIGINS: z.string().default('http://localhost:3000'),
});

type Env = z.infer<typeof EnvSchema>;

class ConfigService {
  private readonly env: Env;

  constructor() {
    const result = EnvSchema.safeParse(process.env);
    
    if (!result.success) {
      console.error('❌ Invalid environment variables:');
      result.error.errors.forEach(err => {
        console.error(`  ${err.path.join('.')}: ${err.message}`);
      });
      process.exit(1);
    }
    
    this.env = result.data;
  }

  get<K extends keyof Env>(key: K): Env[K] {
    return this.env[key];
  }

  get allowedOrigins(): string[] {
    return this.env.ALLOWED_ORIGINS.split(',').map(o => o.trim());
  }

  get isDevelopment(): boolean {
    return this.env.NODE_ENV === 'development';
  }

  get isProduction(): boolean {
    return this.env.NODE_ENV === 'production';
  }
}

export const config = new ConfigService();
```

---

## 11. Secret Management (การจัดการ Secrets)

### 11.1 Encryption Service

```typescript
import crypto from 'crypto';

// ✅ AES-256-GCM encryption service
class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly key: Buffer;
  private readonly ivLength = 16;
  private readonly tagLength = 16;
  private readonly saltLength = 32;

  constructor(key: string) {
    if (key.length !== 64) {
      throw new Error('Encryption key must be a 32-byte hex string (64 characters)');
    }
    this.key = Buffer.from(key, 'hex');
  }

  encrypt(plaintext: string): string {
    const iv = crypto.randomBytes(this.ivLength);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv) as crypto.CipherGCM;
    
    const encrypted = Buffer.concat([
      cipher.update(plaintext, 'utf8'),
      cipher.final(),
    ]);
    
    const tag = cipher.getAuthTag();
    
    // format: iv:tag:encrypted (all hex)
    return [
      iv.toString('hex'),
      tag.toString('hex'),
      encrypted.toString('hex'),
    ].join(':');
  }

  decrypt(ciphertext: string): string {
    const [ivHex, tagHex, encryptedHex] = ciphertext.split(':');
    
    if (!ivHex || !tagHex || !encryptedHex) {
      throw new Error('Invalid ciphertext format');
    }
    
    const iv = Buffer.from(ivHex, 'hex');
    const tag = Buffer.from(tagHex, 'hex');
    const encrypted = Buffer.from(encryptedHex, 'hex');
    
    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv) as crypto.DecipherGCM;
    decipher.setAuthTag(tag);
    
    return decipher.update(encrypted) + decipher.final('utf8');
  }

  hashPassword(password: string): string {
    const salt = crypto.randomBytes(this.saltLength);
    const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
    return `${salt.toString('hex')}:${hash.toString('hex')}`;
  }

  verifyPassword(password: string, storedHash: string): boolean {
    const [saltHex, hashHex] = storedHash.split(':');
    
    if (!saltHex || !hashHex) return false;
    
    const salt = Buffer.from(saltHex, 'hex');
    const storedHashBuffer = Buffer.from(hashHex, 'hex');
    const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
    
    return crypto.timingSafeEqual(storedHashBuffer, hash);
  }
}
```

---

## 12. OWASP Top 10 สำหรับ TypeScript

### 12.1 Broken Access Control Prevention

```typescript
// ✅ Role-Based Access Control (RBAC)
type Permission = 
  | 'user:read'
  | 'user:write'
  | 'user:delete'
  | 'admin:access'
  | 'report:generate';

type Role = 'guest' | 'user' | 'moderator' | 'admin';

const rolePermissions: Record<Role, Permission[]> = {
  guest: ['user:read'],
  user: ['user:read', 'user:write'],
  moderator: ['user:read', 'user:write', 'user:delete'],
  admin: ['user:read', 'user:write', 'user:delete', 'admin:access', 'report:generate'],
};

class RBACService {
  hasPermission(role: Role, permission: Permission): boolean {
    return rolePermissions[role]?.includes(permission) ?? false;
  }

  requirePermission(permission: Permission) {
    return (req: Request, res: Response, next: NextFunction): void => {
      const userRole = req.user?.role as Role;
      
      if (!userRole || !this.hasPermission(userRole, permission)) {
        res.status(403).json({
          error: 'Forbidden',
          message: 'คุณไม่มีสิทธิ์ในการดำเนินการนี้',
        });
        return;
      }
      
      next();
    };
  }

  requireOwnership(getResourceUserId: (req: Request) => Promise<string | null>) {
    return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
      const userId = req.user?.sub;
      const resourceUserId = await getResourceUserId(req);
      
      if (!userId || (userId !== resourceUserId && req.user?.role !== 'admin')) {
        res.status(403).json({
          error: 'Forbidden',
          message: 'คุณสามารถแก้ไขได้เฉพาะข้อมูลของตัวเองเท่านั้น',
        });
        return;
      }
      
      next();
    };
  }
}

// ✅ Path traversal prevention
function securePath(basePath: string, userInput: string): string | null {
  const path = require('path');
  
  // Normalize และตรวจสอบ path
  const normalizedBase = path.resolve(basePath);
  const requestedPath = path.resolve(basePath, userInput);
  
  // ตรวจสอบว่า path อยู่ใน base directory
  if (!requestedPath.startsWith(normalizedBase + path.sep) && 
      requestedPath !== normalizedBase) {
    return null; // Path traversal detected!
  }
  
  return requestedPath;
}
```

### 12.2 Security Misconfiguration Prevention

```typescript
// ✅ Security headers audit
function auditSecurityHeaders(headers: Record<string, string>): {
  missing: string[];
  warnings: string[];
  pass: string[];
} {
  const requiredHeaders = [
    'X-Content-Type-Options',
    'X-Frame-Options',
    'X-XSS-Protection',
    'Strict-Transport-Security',
    'Content-Security-Policy',
    'Referrer-Policy',
  ];
  
  const missing: string[] = [];
  const warnings: string[] = [];
  const pass: string[] = [];
  
  for (const header of requiredHeaders) {
    const lowerHeader = header.toLowerCase();
    
    if (!headers[lowerHeader]) {
      missing.push(header);
    } else {
      pass.push(header);
    }
  }
  
  // Check specific values
  const hsts = headers['strict-transport-security'];
  if (hsts) {
    if (!hsts.includes('includeSubDomains')) {
      warnings.push('HSTS should include includeSubDomains');
    }
    if (!hsts.includes('max-age')) {
      warnings.push('HSTS should include max-age');
    }
  }
  
  return { missing, warnings, pass };
}
```

---

## 13. Security Testing (การทดสอบความปลอดภัย)

### 13.1 Security Test Examples

```typescript
import { describe, it, expect, beforeEach } from '@jest/globals';

// ✅ Security unit tests
describe('Security Tests', () => {
  describe('SQL Injection Prevention', () => {
    const maliciousInputs = [
      "'; DROP TABLE users; --",
      "1' OR '1'='1",
      "admin'--",
      "1; EXEC xp_cmdshell('dir')--",
    ];
    
    it('should reject SQL injection in username field', async () => {
      for (const input of maliciousInputs) {
        const schema = z.object({
          username: z.string().regex(/^[a-zA-Z0-9_-]+$/),
        });
        
        const result = schema.safeParse({ username: input });
        expect(result.success).toBe(false);
      }
    });
  });
  
  describe('XSS Prevention', () => {
    const xssPayloads = [
      '<script>alert("xss")</script>',
      '"><img src=x onerror=alert(1)>',
      "javascript:alert('xss')",
      '<a href="javascript:void(0)" onclick="alert(1)">click</a>',
    ];
    
    it('should escape HTML in user-generated content', () => {
      for (const payload of xssPayloads) {
        const escaped = StringSanitizer.escapeHtml(payload);
        expect(escaped).not.toContain('<script>');
        expect(escaped).not.toContain('javascript:');
        expect(escaped).toContain('&lt;');
      }
    });
  });
  
  describe('Authentication', () => {
    let passwordService: PasswordService;
    
    beforeEach(() => {
      passwordService = new PasswordService(12);
    });
    
    it('should hash passwords securely', async () => {
      const password = 'SecurePass123!';
      const hash = await passwordService.hash(password);
      
      expect(hash).not.toBe(password);
      expect(hash.length).toBeGreaterThan(50);
    });
    
    it('should verify correct passwords', async () => {
      const password = 'SecurePass123!';
      const hash = await passwordService.hash(password);
      
      const isValid = await passwordService.verify(password, hash);
      expect(isValid).toBe(true);
    });
    
    it('should reject incorrect passwords', async () => {
      const password = 'SecurePass123!';
      const hash = await passwordService.hash(password);
      
      const isValid = await passwordService.verify('WrongPassword!', hash);
      expect(isValid).toBe(false);
    });
  });
  
  describe('Path Traversal Prevention', () => {
    it('should prevent directory traversal attacks', () => {
      const basePath = '/var/app/uploads';
      const attacks = [
        '../../../etc/passwd',
        '..\\..\\windows\\system32',
        '%2e%2e%2f%2e%2e%2f',
      ];
      
      for (const attack of attacks) {
        const result = securePath(basePath, attack);
        expect(result).toBeNull();
      }
    });
    
    it('should allow valid paths', () => {
      const basePath = '/var/app/uploads';
      const validPath = securePath(basePath, 'user-123/avatar.png');
      expect(validPath).not.toBeNull();
    });
  });
});
```

---

## 14. Audit Dependencies (การตรวจสอบ Dependencies)

### 14.1 Automated Security Auditing

```typescript
// scripts/security-audit.ts
import { execSync } from 'child_process';
import fs from 'fs';
import path from 'path';

interface AuditResult {
  vulnerabilities: Vulnerability[];
  summary: {
    critical: number;
    high: number;
    moderate: number;
    low: number;
    info: number;
  };
}

interface Vulnerability {
  name: string;
  severity: 'critical' | 'high' | 'moderate' | 'low' | 'info';
  description: string;
  fixAvailable: boolean;
  fixVersion?: string;
}

async function runSecurityAudit(): Promise<void> {
  console.log('🔍 Running security audit...\n');
  
  try {
    // npm audit
    const auditOutput = execSync('npm audit --json', {
      encoding: 'utf8',
      cwd: process.cwd(),
    });
    
    const auditData = JSON.parse(auditOutput) as {
      vulnerabilities: Record<string, {
        severity: string;
        via: Array<string | { title: string; url: string }>;
        fixAvailable: boolean | { version: string };
      }>;
      metadata: {
        vulnerabilities: {
          critical: number;
          high: number;
          moderate: number;
          low: number;
          info: number;
          total: number;
        };
      };
    };
    
    const { metadata } = auditData;
    
    console.log('=== Security Audit Summary ===');
    console.log(`Critical: ${metadata.vulnerabilities.critical}`);
    console.log(`High: ${metadata.vulnerabilities.high}`);
    console.log(`Moderate: ${metadata.vulnerabilities.moderate}`);
    console.log(`Low: ${metadata.vulnerabilities.low}`);
    console.log(`Total: ${metadata.vulnerabilities.total}\n`);
    
    if (metadata.vulnerabilities.critical > 0 || metadata.vulnerabilities.high > 0) {
      console.error('❌ Critical or High vulnerabilities found! Build should fail.');
      process.exit(1);
    }
    
    if (metadata.vulnerabilities.moderate > 0) {
      console.warn('⚠️ Moderate vulnerabilities found. Please review.');
    }
    
    console.log('✅ No critical or high vulnerabilities found.');
    
  } catch (error) {
    // npm audit exits with non-zero if vulnerabilities found
    console.error('Security audit failed:', error);
    process.exit(1);
  }
}

// package.json scripts ที่ควรเพิ่ม
const packageJsonAdditions = {
  scripts: {
    'security:audit': 'npm audit',
    'security:fix': 'npm audit fix',
    'security:check': 'ts-node scripts/security-audit.ts',
  },
};

// GitHub Actions workflow
const githubWorkflow = `
name: Security Audit

on:
  schedule:
    - cron: '0 6 * * 1'  # ทุกวันจันทร์ 6:00 AM UTC
  push:
    paths:
      - 'package.json'
      - 'package-lock.json'

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run security audit
        run: npm audit --audit-level=high
        
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: \${{ secrets.SNYK_TOKEN }}
`;

console.log('GitHub Actions workflow for security:');
console.log(githubWorkflow);

runSecurityAudit().catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **Input Validation** - ใช้ Zod สำหรับ runtime validation, HTML sanitization
2. **SQL Injection Prevention** - Parameterized queries, ORM usage
3. **XSS Prevention** - Content Security Policy, Output encoding
4. **CSRF Protection** - CSRF tokens, SameSite cookies
5. **Authentication Security** - Secure password hashing, Account lockout
6. **JWT Security** - Secure token generation, Token revocation
7. **Rate Limiting** - Express rate limit, Sliding window limiter
8. **CORS Configuration** - Whitelist origins, Proper headers
9. **Helmet.js** - Security headers, CSP configuration
10. **Environment Variables** - Zod validation, Secret management
11. **Encryption** - AES-256-GCM, Password hashing
12. **OWASP Top 10** - Access control, Path traversal prevention
13. **Security Testing** - Unit tests for security functions
14. **Dependency Auditing** - Automated audit scripts

ความปลอดภัยต้องเป็น mindset ไม่ใช่แค่ feature ควรรวม security practices เข้าไปในทุกขั้นตอนของการพัฒนา

---

*หัวข้อถัดไป: Part 50 - TypeScript Compiler API*
