# ส่วนที่ 74: รูปแบบการออกแบบฐานข้อมูลใน TypeScript

## บทนำ

การออกแบบ layer ฐานข้อมูลที่ดีเป็นสิ่งสำคัญในการพัฒนาแอปพลิเคชัน ในบทนี้เราจะเรียนรู้รูปแบบต่างๆ สำหรับการทำงานกับฐานข้อมูลใน TypeScript

---

## 1. Active Record vs Data Mapper

### 1.1 Active Record Pattern

Active Record รวมตรรกะทางธุรกิจและการเข้าถึงข้อมูลไว้ใน model เดียวกัน

```typescript
// Active Record style (เช่น TypeORM's @Entity)
class UserActiveRecord {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  private db: Database;

  constructor(db: Database) {
    this.db = db;
  }

  static async find(db: Database, id: string): Promise<UserActiveRecord | null> {
    const row = await db.query('SELECT * FROM users WHERE id = $1', [id]);
    if (!row) return null;
    const user = new UserActiveRecord(db);
    Object.assign(user, row);
    return user;
  }

  static async findAll(db: Database): Promise<UserActiveRecord[]> {
    const rows = await db.query('SELECT * FROM users');
    return rows.map((row: any) => {
      const user = new UserActiveRecord(db);
      Object.assign(user, row);
      return user;
    });
  }

  async save(): Promise<void> {
    if (this.id) {
      await this.db.query(
        'UPDATE users SET name = $1, email = $2 WHERE id = $3',
        [this.name, this.email, this.id]
      );
    } else {
      this.id = generateId();
      this.createdAt = new Date();
      await this.db.query(
        'INSERT INTO users (id, name, email, created_at) VALUES ($1, $2, $3, $4)',
        [this.id, this.name, this.email, this.createdAt]
      );
    }
  }

  async delete(): Promise<void> {
    await this.db.query('DELETE FROM users WHERE id = $1', [this.id]);
  }

  // Business Logic ใน model
  async changeEmail(newEmail: string): Promise<void> {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(newEmail)) {
      throw new Error('อีเมลไม่ถูกต้อง');
    }
    this.email = newEmail;
    await this.save();
  }
}
```

### 1.2 Data Mapper Pattern

Data Mapper แยก domain objects ออกจากฐานข้อมูล

```typescript
// Domain Object - ไม่รู้เรื่องฐานข้อมูล
class User {
  constructor(
    public readonly id: string,
    public name: string,
    public email: string,
    public readonly createdAt: Date = new Date()
  ) {}

  changeEmail(newEmail: string): void {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(newEmail)) {
      throw new Error('อีเมลไม่ถูกต้อง');
    }
    this.email = newEmail;
  }

  isAdmin(): boolean {
    return this.email.endsWith('@admin.example.com');
  }
}

// Data Mapper - รับผิดชอบการแปลงระหว่าง domain object และ database
class UserMapper {
  toDomain(raw: Record<string, unknown>): User {
    return new User(
      raw.id as string,
      raw.name as string,
      raw.email as string,
      new Date(raw.created_at as string)
    );
  }

  toPersistence(user: User): Record<string, unknown> {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
      created_at: user.createdAt.toISOString()
    };
  }
}

// Repository ที่ใช้ Data Mapper
class UserDataMapperRepository {
  private mapper = new UserMapper();

  constructor(private db: Database) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.db.queryOne('SELECT * FROM users WHERE id = $1', [id]);
    return row ? this.mapper.toDomain(row) : null;
  }

  async save(user: User): Promise<void> {
    const data = this.mapper.toPersistence(user);
    await this.db.query(
      `INSERT INTO users (id, name, email, created_at)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (id) DO UPDATE
       SET name = $2, email = $3`,
      [data.id, data.name, data.email, data.created_at]
    );
  }
}
```

---

## 2. Repository Pattern (Full Implementation)

### 2.1 Generic Repository

```typescript
interface Repository<T, ID> {
  findById(id: ID): Promise<T | null>;
  findAll(options?: QueryOptions<T>): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: ID): Promise<void>;
  exists(id: ID): Promise<boolean>;
  count(filter?: Partial<T>): Promise<number>;
}

interface QueryOptions<T> {
  where?: Partial<T>;
  orderBy?: { field: keyof T; direction: 'ASC' | 'DESC' }[];
  limit?: number;
  offset?: number;
  include?: string[];
}

abstract class BaseRepository<T extends { id: string }, DB = any> implements Repository<T, string> {
  constructor(
    protected db: DB,
    protected tableName: string
  ) {}

  abstract findById(id: string): Promise<T | null>;
  abstract findAll(options?: QueryOptions<T>): Promise<T[]>;
  abstract save(entity: T): Promise<T>;

  async delete(id: string): Promise<void> {
    await (this.db as any).query(
      `DELETE FROM ${this.tableName} WHERE id = $1`,
      [id]
    );
  }

  async exists(id: string): Promise<boolean> {
    const result = await (this.db as any).queryOne(
      `SELECT 1 FROM ${this.tableName} WHERE id = $1`,
      [id]
    );
    return !!result;
  }

  async count(filter?: Record<string, unknown>): Promise<number> {
    let query = `SELECT COUNT(*) FROM ${this.tableName}`;
    const params: unknown[] = [];

    if (filter && Object.keys(filter).length > 0) {
      const conditions = Object.entries(filter).map(([key, value], i) => {
        params.push(value);
        return `${key} = $${i + 1}`;
      });
      query += ` WHERE ${conditions.join(' AND ')}`;
    }

    const result = await (this.db as any).queryOne(query, params);
    return parseInt(result.count, 10);
  }

  protected buildWhereClause(
    filter: Record<string, unknown>,
    startIndex: number = 1
  ): { clause: string; params: unknown[] } {
    const conditions: string[] = [];
    const params: unknown[] = [];
    let index = startIndex;

    for (const [key, value] of Object.entries(filter)) {
      if (value === null) {
        conditions.push(`${key} IS NULL`);
      } else if (Array.isArray(value)) {
        const placeholders = value.map(() => `$${index++}`).join(', ');
        conditions.push(`${key} IN (${placeholders})`);
        params.push(...value);
      } else {
        conditions.push(`${key} = $${index++}`);
        params.push(value);
      }
    }

    return {
      clause: conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '',
      params
    };
  }
}

// Concrete Implementation
class PostgresUserRepository extends BaseRepository<User> {
  private mapper = new UserMapper();

  async findById(id: string): Promise<User | null> {
    const row = await this.db.queryOne(
      `SELECT * FROM ${this.tableName} WHERE id = $1`,
      [id]
    );
    return row ? this.mapper.toDomain(row) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const row = await this.db.queryOne(
      `SELECT * FROM ${this.tableName} WHERE email = $1`,
      [email]
    );
    return row ? this.mapper.toDomain(row) : null;
  }

  async findAll(options: QueryOptions<User> = {}): Promise<User[]> {
    let query = `SELECT * FROM ${this.tableName}`;
    const params: unknown[] = [];

    if (options.where) {
      const { clause, params: whereParams } = this.buildWhereClause(
        options.where as Record<string, unknown>
      );
      query += ` ${clause}`;
      params.push(...whereParams);
    }

    if (options.orderBy?.length) {
      const orders = options.orderBy
        .map(o => `${String(o.field)} ${o.direction}`)
        .join(', ');
      query += ` ORDER BY ${orders}`;
    }

    if (options.limit) {
      query += ` LIMIT $${params.length + 1}`;
      params.push(options.limit);
    }

    if (options.offset) {
      query += ` OFFSET $${params.length + 1}`;
      params.push(options.offset);
    }

    const rows = await this.db.query(query, params);
    return rows.map((row: any) => this.mapper.toDomain(row));
  }

  async save(user: User): Promise<User> {
    const data = this.mapper.toPersistence(user);
    await this.db.query(
      `INSERT INTO ${this.tableName} (id, name, email, created_at)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (id) DO UPDATE
       SET name = EXCLUDED.name, email = EXCLUDED.email`,
      [data.id, data.name, data.email, data.created_at]
    );
    return user;
  }
}
```

---

## 3. Query Object Pattern

```typescript
interface QueryObject<T> {
  execute(db: Database): Promise<T[]>;
  count(db: Database): Promise<number>;
}

abstract class BaseQuery<T> implements QueryObject<T> {
  protected conditions: string[] = [];
  protected params: unknown[] = [];
  protected orderClauses: string[] = [];
  protected limitValue?: number;
  protected offsetValue?: number;

  abstract getBaseQuery(): string;
  abstract mapRow(row: Record<string, unknown>): T;

  protected addCondition(condition: string, ...values: unknown[]): void {
    this.conditions.push(condition);
    this.params.push(...values);
  }

  orderBy(field: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.orderClauses.push(`${field} ${direction}`);
    return this;
  }

  limit(n: number): this {
    this.limitValue = n;
    return this;
  }

  offset(n: number): this {
    this.offsetValue = n;
    return this;
  }

  protected buildQuery(): { sql: string; params: unknown[] } {
    let sql = this.getBaseQuery();
    const params = [...this.params];

    if (this.conditions.length > 0) {
      sql += ` WHERE ${this.conditions.join(' AND ')}`;
    }
    if (this.orderClauses.length > 0) {
      sql += ` ORDER BY ${this.orderClauses.join(', ')}`;
    }
    if (this.limitValue !== undefined) {
      sql += ` LIMIT $${params.length + 1}`;
      params.push(this.limitValue);
    }
    if (this.offsetValue !== undefined) {
      sql += ` OFFSET $${params.length + 1}`;
      params.push(this.offsetValue);
    }

    return { sql, params };
  }

  async execute(db: Database): Promise<T[]> {
    const { sql, params } = this.buildQuery();
    const rows = await db.query(sql, params);
    return rows.map((row: Record<string, unknown>) => this.mapRow(row));
  }

  async count(db: Database): Promise<number> {
    const { sql: baseSql, params } = this.buildQuery();
    const countSql = `SELECT COUNT(*) FROM (${baseSql}) AS subquery`;
    const result = await db.queryOne(countSql, params);
    return parseInt(result.count, 10);
  }
}

class UserQuery extends BaseQuery<User> {
  private mapper = new UserMapper();

  getBaseQuery(): string {
    return 'SELECT * FROM users';
  }

  mapRow(row: Record<string, unknown>): User {
    return this.mapper.toDomain(row);
  }

  withEmail(email: string): this {
    this.addCondition('email = $' + (this.params.length + 1), email);
    return this;
  }

  withNameContaining(name: string): this {
    this.addCondition('name ILIKE $' + (this.params.length + 1), `%${name}%`);
    return this;
  }

  createdAfter(date: Date): this {
    this.addCondition('created_at > $' + (this.params.length + 1), date);
    return this;
  }

  createdBefore(date: Date): this {
    this.addCondition('created_at < $' + (this.params.length + 1), date);
    return this;
  }
}

// ตัวอย่าง
const users = await new UserQuery()
  .withNameContaining('สมชาย')
  .createdAfter(new Date('2024-01-01'))
  .orderBy('created_at', 'DESC')
  .limit(10)
  .execute(db);
```

---

## 4. Soft Delete Pattern

```typescript
interface SoftDeletable {
  deletedAt: Date | null;
  deletedBy: string | null;
}

abstract class SoftDeleteRepository<T extends { id: string } & SoftDeletable> {
  constructor(protected db: Database, protected tableName: string) {}

  async softDelete(id: string, deletedBy: string): Promise<void> {
    await this.db.query(
      `UPDATE ${this.tableName}
       SET deleted_at = NOW(), deleted_by = $1
       WHERE id = $2 AND deleted_at IS NULL`,
      [deletedBy, id]
    );
  }

  async restore(id: string): Promise<void> {
    await this.db.query(
      `UPDATE ${this.tableName}
       SET deleted_at = NULL, deleted_by = NULL
       WHERE id = $1`,
      [id]
    );
  }

  async findById(id: string, includeDeleted = false): Promise<T | null> {
    let query = `SELECT * FROM ${this.tableName} WHERE id = $1`;
    if (!includeDeleted) {
      query += ' AND deleted_at IS NULL';
    }
    return this.db.queryOne(query, [id]);
  }

  async findAll(includeDeleted = false): Promise<T[]> {
    let query = `SELECT * FROM ${this.tableName}`;
    if (!includeDeleted) {
      query += ' WHERE deleted_at IS NULL';
    }
    return this.db.query(query);
  }

  async findDeleted(): Promise<T[]> {
    return this.db.query(
      `SELECT * FROM ${this.tableName} WHERE deleted_at IS NOT NULL`
    );
  }

  async permanentDelete(id: string): Promise<void> {
    await this.db.query(
      `DELETE FROM ${this.tableName} WHERE id = $1`,
      [id]
    );
  }
}

// SQL สำหรับ Soft Delete
const createTableSQL = `
  CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP,
    deleted_by VARCHAR(255)
  );

  -- Index สำหรับ query ที่ filter deleted records
  CREATE INDEX idx_users_deleted_at ON users (deleted_at) WHERE deleted_at IS NULL;
`;
```

---

## 5. Audit Trail Pattern

```typescript
interface AuditEntry {
  id: string;
  tableName: string;
  recordId: string;
  action: 'INSERT' | 'UPDATE' | 'DELETE';
  oldData: Record<string, unknown> | null;
  newData: Record<string, unknown> | null;
  changedFields: string[];
  performedBy: string;
  performedAt: Date;
  ipAddress?: string;
  userAgent?: string;
}

class AuditRepository {
  constructor(private db: Database) {}

  async log(entry: Omit<AuditEntry, 'id'>): Promise<void> {
    await this.db.query(
      `INSERT INTO audit_log (
        id, table_name, record_id, action,
        old_data, new_data, changed_fields,
        performed_by, performed_at, ip_address, user_agent
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)`,
      [
        generateId(),
        entry.tableName,
        entry.recordId,
        entry.action,
        entry.oldData ? JSON.stringify(entry.oldData) : null,
        entry.newData ? JSON.stringify(entry.newData) : null,
        JSON.stringify(entry.changedFields),
        entry.performedBy,
        entry.performedAt,
        entry.ipAddress ?? null,
        entry.userAgent ?? null
      ]
    );
  }

  async findByRecord(tableName: string, recordId: string): Promise<AuditEntry[]> {
    return this.db.query(
      `SELECT * FROM audit_log
       WHERE table_name = $1 AND record_id = $2
       ORDER BY performed_at DESC`,
      [tableName, recordId]
    );
  }

  async findByUser(userId: string, limit = 50): Promise<AuditEntry[]> {
    return this.db.query(
      `SELECT * FROM audit_log
       WHERE performed_by = $1
       ORDER BY performed_at DESC
       LIMIT $2`,
      [userId, limit]
    );
  }
}

// Audit-aware Repository
class AuditableUserRepository extends PostgresUserRepository {
  constructor(
    db: Database,
    private auditRepo: AuditRepository,
    private currentUserId: string
  ) {
    super(db, 'users');
  }

  async save(user: User): Promise<User> {
    const existing = await this.findById(user.id);
    const result = await super.save(user);

    // บันทึก audit log
    const changedFields = existing
      ? this.getChangedFields(existing, user)
      : Object.keys(user);

    await this.auditRepo.log({
      tableName: 'users',
      recordId: user.id,
      action: existing ? 'UPDATE' : 'INSERT',
      oldData: existing ? this.userToRecord(existing) : null,
      newData: this.userToRecord(user),
      changedFields,
      performedBy: this.currentUserId,
      performedAt: new Date()
    });

    return result;
  }

  async delete(id: string): Promise<void> {
    const existing = await this.findById(id);
    await super.delete(id);

    if (existing) {
      await this.auditRepo.log({
        tableName: 'users',
        recordId: id,
        action: 'DELETE',
        oldData: this.userToRecord(existing),
        newData: null,
        changedFields: [],
        performedBy: this.currentUserId,
        performedAt: new Date()
      });
    }
  }

  private getChangedFields(original: User, updated: User): string[] {
    return (['name', 'email'] as (keyof User)[]).filter(
      field => original[field] !== updated[field]
    ) as string[];
  }

  private userToRecord(user: User): Record<string, unknown> {
    return { id: user.id, name: user.name, email: user.email, createdAt: user.createdAt };
  }
}
```

---

## 6. Multi-tenancy Pattern

### 6.1 Schema-per-tenant

```typescript
class SchemaPerTenantRepository {
  constructor(private db: Database) {}

  private getSchemaName(tenantId: string): string {
    // Sanitize tenant ID to prevent SQL injection
    if (!/^[a-z0-9_]+$/.test(tenantId)) {
      throw new Error(`Invalid tenant ID: ${tenantId}`);
    }
    return `tenant_${tenantId}`;
  }

  async createTenantSchema(tenantId: string): Promise<void> {
    const schema = this.getSchemaName(tenantId);
    await this.db.query(`CREATE SCHEMA IF NOT EXISTS ${schema}`);
    await this.db.query(`
      CREATE TABLE IF NOT EXISTS ${schema}.users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
      )
    `);
  }

  async findUsersByTenant(tenantId: string): Promise<User[]> {
    const schema = this.getSchemaName(tenantId);
    return this.db.query(`SELECT * FROM ${schema}.users`);
  }
}

// Row-level security approach
class RowLevelTenancyRepository {
  async findUsersByTenant(tenantId: string): Promise<User[]> {
    return this.db.query(
      'SELECT * FROM users WHERE tenant_id = $1',
      [tenantId]
    );
  }

  async createUser(tenantId: string, userData: CreateUserDTO): Promise<User> {
    return this.db.queryOne(
      `INSERT INTO users (id, tenant_id, name, email, created_at)
       VALUES ($1, $2, $3, $4, NOW())
       RETURNING *`,
      [generateId(), tenantId, userData.name, userData.email]
    );
  }
}

// Middleware สำหรับ inject tenant context
function tenantMiddleware(req: Request, res: Response, next: NextFunction) {
  const tenantId = req.headers['x-tenant-id'] as string
    ?? req.user?.tenantId;

  if (!tenantId) {
    return res.status(400).json({ error: 'ต้องระบุ tenant ID' });
  }

  req.tenantId = tenantId;
  next();
}
```

---

## 7. Connection Pooling

```typescript
import { Pool, PoolClient } from 'pg';

interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  min: number;
  max: number;
  idleTimeoutMillis: number;
  connectionTimeoutMillis: number;
  ssl?: boolean;
}

class DatabasePool {
  private pool: Pool;

  constructor(config: DatabaseConfig) {
    this.pool = new Pool({
      host: config.host,
      port: config.port,
      database: config.database,
      user: config.user,
      password: config.password,
      min: config.min,
      max: config.max,
      idleTimeoutMillis: config.idleTimeoutMillis,
      connectionTimeoutMillis: config.connectionTimeoutMillis,
      ssl: config.ssl ? { rejectUnauthorized: false } : false
    });

    this.pool.on('error', (err) => {
      console.error('Unexpected pool error:', err);
    });
  }

  async query<T = unknown>(sql: string, params?: unknown[]): Promise<T[]> {
    const result = await this.pool.query(sql, params);
    return result.rows as T[];
  }

  async queryOne<T = unknown>(sql: string, params?: unknown[]): Promise<T | null> {
    const result = await this.pool.query(sql, params);
    return result.rows[0] ?? null;
  }

  async withClient<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    const client = await this.pool.connect();
    try {
      return await fn(client);
    } finally {
      client.release();
    }
  }

  async withTransaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    return this.withClient(async (client) => {
      await client.query('BEGIN');
      try {
        const result = await fn(client);
        await client.query('COMMIT');
        return result;
      } catch (error) {
        await client.query('ROLLBACK');
        throw error;
      }
    });
  }

  async getPoolStatus() {
    return {
      total: this.pool.totalCount,
      idle: this.pool.idleCount,
      waiting: this.pool.waitingCount
    };
  }

  async close(): Promise<void> {
    await this.pool.end();
  }
}

// ตัวอย่าง pool configuration
const dbPool = new DatabasePool({
  host: process.env.DB_HOST ?? 'localhost',
  port: parseInt(process.env.DB_PORT ?? '5432'),
  database: process.env.DB_NAME ?? 'myapp',
  user: process.env.DB_USER ?? 'postgres',
  password: process.env.DB_PASSWORD ?? '',
  min: 2,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  ssl: process.env.NODE_ENV === 'production'
});
```

---

## 8. Transaction Management

```typescript
type TransactionFunction<T> = (tx: TransactionContext) => Promise<T>;

interface TransactionContext {
  query<T>(sql: string, params?: unknown[]): Promise<T[]>;
  queryOne<T>(sql: string, params?: unknown[]): Promise<T | null>;
}

class TransactionManager {
  constructor(private pool: DatabasePool) {}

  async run<T>(fn: TransactionFunction<T>): Promise<T> {
    return this.pool.withTransaction(async (client) => {
      const ctx: TransactionContext = {
        query: (sql, params) => client.query(sql, params).then(r => r.rows),
        queryOne: (sql, params) => client.query(sql, params).then(r => r.rows[0] ?? null)
      };
      return fn(ctx);
    });
  }

  async runSerializable<T>(fn: TransactionFunction<T>): Promise<T> {
    return this.pool.withClient(async (client) => {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      try {
        const ctx: TransactionContext = {
          query: (sql, params) => client.query(sql, params).then(r => r.rows),
          queryOne: (sql, params) => client.query(sql, params).then(r => r.rows[0] ?? null)
        };
        const result = await fn(ctx);
        await client.query('COMMIT');
        return result;
      } catch (error) {
        await client.query('ROLLBACK');
        throw error;
      }
    });
  }
}

// ตัวอย่าง - โอนเงิน (ต้องทำใน transaction)
async function transferMoney(
  txManager: TransactionManager,
  fromAccountId: string,
  toAccountId: string,
  amount: number
): Promise<void> {
  await txManager.run(async (tx) => {
    // ตรวจสอบยอดเงิน
    const fromAccount = await tx.queryOne<{ balance: number }>(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
      [fromAccountId]
    );

    if (!fromAccount || fromAccount.balance < amount) {
      throw new Error('ยอดเงินไม่เพียงพอ');
    }

    // หักเงินจาก account ต้นทาง
    await tx.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromAccountId]
    );

    // เพิ่มเงินใน account ปลายทาง
    await tx.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toAccountId]
    );

    // บันทึก transaction log
    await tx.query(
      `INSERT INTO transactions (id, from_account, to_account, amount, created_at)
       VALUES ($1, $2, $3, $4, NOW())`,
      [generateId(), fromAccountId, toAccountId, amount]
    );
  });
}
```

---

## 9. Read Replicas

```typescript
type QueryMode = 'read' | 'write';

class ReadWriteSplitDatabase {
  private writePool: DatabasePool;
  private readPools: DatabasePool[];
  private readPoolIndex = 0;

  constructor(writeConfig: DatabaseConfig, readConfigs: DatabaseConfig[]) {
    this.writePool = new DatabasePool(writeConfig);
    this.readPools = readConfigs.map(config => new DatabasePool(config));
  }

  private getReadPool(): DatabasePool {
    // Round-robin load balancing
    const pool = this.readPools[this.readPoolIndex];
    this.readPoolIndex = (this.readPoolIndex + 1) % this.readPools.length;
    return pool;
  }

  async query<T>(sql: string, params?: unknown[], mode: QueryMode = 'read'): Promise<T[]> {
    const pool = mode === 'write' ? this.writePool : this.getReadPool();
    return pool.query<T>(sql, params);
  }

  async withTransaction<T>(fn: (tx: TransactionContext) => Promise<T>): Promise<T> {
    return this.writePool.withTransaction(async (client) => {
      const ctx: TransactionContext = {
        query: (sql, params) => client.query(sql, params).then(r => r.rows),
        queryOne: (sql, params) => client.query(sql, params).then(r => r.rows[0] ?? null)
      };
      return fn(ctx);
    });
  }
}
```

---

## 10. Specification Pattern สำหรับ Queries

```typescript
interface DatabaseSpecification<T> {
  toQuery(): { where: string; params: unknown[] };
  and(other: DatabaseSpecification<T>): DatabaseSpecification<T>;
  or(other: DatabaseSpecification<T>): DatabaseSpecification<T>;
}

abstract class SqlSpecification<T> implements DatabaseSpecification<T> {
  abstract toQuery(): { where: string; params: unknown[] };

  and(other: DatabaseSpecification<T>): DatabaseSpecification<T> {
    return new AndSqlSpec(this, other);
  }

  or(other: DatabaseSpecification<T>): DatabaseSpecification<T> {
    return new OrSqlSpec(this, other);
  }
}

class AndSqlSpec<T> extends SqlSpecification<T> {
  constructor(
    private left: DatabaseSpecification<T>,
    private right: DatabaseSpecification<T>
  ) {
    super();
  }

  toQuery(): { where: string; params: unknown[] } {
    const leftQuery = this.left.toQuery();
    const rightQuery = this.right.toQuery();

    // Adjust parameter indices
    let rightWhere = rightQuery.where;
    const leftParamCount = leftQuery.params.length;

    // Re-index parameters in right clause
    rightWhere = rightWhere.replace(/\$(\d+)/g, (_, n) => `$${parseInt(n) + leftParamCount}`);

    return {
      where: `(${leftQuery.where}) AND (${rightWhere})`,
      params: [...leftQuery.params, ...rightQuery.params]
    };
  }
}

class OrSqlSpec<T> extends SqlSpecification<T> {
  constructor(
    private left: DatabaseSpecification<T>,
    private right: DatabaseSpecification<T>
  ) {
    super();
  }

  toQuery(): { where: string; params: unknown[] } {
    const leftQuery = this.left.toQuery();
    const rightQuery = this.right.toQuery();

    let rightWhere = rightQuery.where;
    const leftParamCount = leftQuery.params.length;
    rightWhere = rightWhere.replace(/\$(\d+)/g, (_, n) => `$${parseInt(n) + leftParamCount}`);

    return {
      where: `(${leftQuery.where}) OR (${rightWhere})`,
      params: [...leftQuery.params, ...rightQuery.params]
    };
  }
}

// Concrete specifications
class ActiveUsersSpec extends SqlSpecification<User> {
  toQuery(): { where: string; params: unknown[] } {
    return {
      where: 'deleted_at IS NULL',
      params: []
    };
  }
}

class UserEmailDomainSpec extends SqlSpecification<User> {
  constructor(private domain: string) {
    super();
  }

  toQuery(): { where: string; params: unknown[] } {
    return {
      where: 'email LIKE $1',
      params: [`%@${this.domain}`]
    };
  }
}

class UserCreatedInPeriodSpec extends SqlSpecification<User> {
  constructor(private from: Date, private to: Date) {
    super();
  }

  toQuery(): { where: string; params: unknown[] } {
    return {
      where: 'created_at BETWEEN $1 AND $2',
      params: [this.from, this.to]
    };
  }
}

// ตัวอย่าง
const spec = new ActiveUsersSpec()
  .and(new UserEmailDomainSpec('gmail.com'))
  .and(new UserCreatedInPeriodSpec(new Date('2024-01-01'), new Date('2024-12-31')));

const { where, params } = spec.toQuery();
const users = await db.query(
  `SELECT * FROM users WHERE ${where}`,
  params
);
```

---

## 11. Sharding Considerations

```typescript
type ShardKey = string | number;

interface ShardConfig {
  id: number;
  host: string;
  port: number;
  database: string;
  min: ShardKey;
  max: ShardKey;
}

class ShardRouter {
  private shards: Map<number, DatabasePool> = new Map();
  private shardConfigs: ShardConfig[];

  constructor(shardConfigs: ShardConfig[]) {
    this.shardConfigs = shardConfigs;
    for (const config of shardConfigs) {
      this.shards.set(config.id, new DatabasePool({
        host: config.host,
        port: config.port,
        database: config.database,
        user: process.env.DB_USER ?? 'postgres',
        password: process.env.DB_PASSWORD ?? '',
        min: 1,
        max: 5,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000
      }));
    }
  }

  getShardForKey(key: ShardKey): DatabasePool {
    const config = this.shardConfigs.find(
      c => key >= c.min && key < c.max
    );
    if (!config) throw new Error(`No shard found for key: ${key}`);
    return this.shards.get(config.id)!;
  }

  getShardByHash(key: string): DatabasePool {
    let hash = 0;
    for (const char of key) {
      hash = (hash * 31 + char.charCodeAt(0)) % this.shardConfigs.length;
    }
    const config = this.shardConfigs[Math.abs(hash)];
    return this.shards.get(config.id)!;
  }

  async queryAllShards<T>(sql: string, params?: unknown[]): Promise<T[]> {
    const results = await Promise.all(
      Array.from(this.shards.values()).map(pool => pool.query<T>(sql, params))
    );
    return results.flat();
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้รูปแบบการออกแบบฐานข้อมูลที่สำคัญ:

1. **Active Record vs Data Mapper** - สองแนวทางหลักในการเชื่อมต่อกับฐานข้อมูล
2. **Repository Pattern** - Generic repository สำหรับ CRUD operations
3. **Query Object Pattern** - สร้าง queries แบบ type-safe และ composable
4. **Soft Delete** - การลบข้อมูลแบบไม่จริง
5. **Audit Trail** - บันทึกประวัติการเปลี่ยนแปลง
6. **Multi-tenancy** - จัดการหลาย tenants ในระบบเดียว
7. **Connection Pooling** - จัดการ connections อย่างมีประสิทธิภาพ
8. **Transaction Management** - จัดการ transactions และ isolation levels
9. **Read Replicas** - แยก read/write workloads
10. **SQL Specification** - Composable query specifications
11. **Sharding** - แบ่งข้อมูลออกหลาย databases
