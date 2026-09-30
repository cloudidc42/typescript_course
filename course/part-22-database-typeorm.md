# ตอนที่ 22: ฐานข้อมูลกับ TypeORM

## บทนำ

TypeORM เป็น Object-Relational Mapper (ORM) ที่รองรับ TypeScript อย่างเต็มที่ ทำให้การทำงานกับฐานข้อมูลเป็นเรื่องง่ายขึ้นด้วยการใช้ decorators และ type safety ในบทนี้เราจะเรียนรู้การใช้งาน TypeORM กับ PostgreSQL ตั้งแต่ต้นจนถึงการสร้าง Blog API ที่สมบูรณ์

---

## 22.1 การติดตั้งและตั้งค่า TypeORM

### การติดตั้ง packages

```bash
# ติดตั้ง TypeORM และ database driver
npm install typeorm reflect-metadata pg
npm install --save-dev @types/pg

# หรือ MySQL
npm install typeorm reflect-metadata mysql2

# หรือ SQLite (สำหรับ development/testing)
npm install typeorm reflect-metadata better-sqlite3
npm install --save-dev @types/better-sqlite3

# ติดตั้ง dependencies เพิ่มเติม
npm install dotenv bcryptjs
npm install --save-dev @types/bcryptjs
```

### package.json

```json
{
  "name": "typeorm-blog-api",
  "version": "1.0.0",
  "description": "Blog API with TypeORM",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "typeorm": "ts-node node_modules/typeorm/cli",
    "migration:generate": "npm run typeorm -- migration:generate -n",
    "migration:run": "npm run typeorm -- migration:run",
    "migration:revert": "npm run typeorm -- migration:revert",
    "migration:show": "npm run typeorm -- migration:show",
    "schema:sync": "npm run typeorm -- schema:sync",
    "schema:drop": "npm run typeorm -- schema:drop",
    "seed": "ts-node src/database/seeds/index.ts"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "dotenv": "^16.0.3",
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "reflect-metadata": "^0.1.13",
    "typeorm": "^0.3.17"
  },
  "devDependencies": {
    "@types/bcryptjs": "^2.4.3",
    "@types/express": "^4.17.17",
    "@types/node": "^20.4.5",
    "@types/pg": "^8.10.2",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.1.6"
  }
}
```

### tsconfig.json สำหรับ TypeORM

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
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "resolveJsonModule": true,
    "sourceMap": true,
    "strictPropertyInitialization": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**หมายเหตุ**: ต้องเปิด `emitDecoratorMetadata` และ `experimentalDecorators` สำหรับ TypeORM

---

## 22.2 การเชื่อมต่อฐานข้อมูล

### src/database/data-source.ts

```typescript
import 'reflect-metadata';
import { DataSource, DataSourceOptions } from 'typeorm';
import { User } from '../entities/User';
import { Post } from '../entities/Post';
import { Comment } from '../entities/Comment';
import { Tag } from '../entities/Tag';
import { Category } from '../entities/Category';

const isDevelopment = process.env.NODE_ENV !== 'production';

const dbConfig: DataSourceOptions = {
  type: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432', 10),
  username: process.env.DB_USERNAME || 'postgres',
  password: process.env.DB_PASSWORD || 'password',
  database: process.env.DB_NAME || 'blog_db',
  
  // Entities
  entities: [User, Post, Comment, Tag, Category],
  
  // Migrations
  migrations: ['dist/database/migrations/*.js'],
  
  // Logging
  logging: isDevelopment ? ['query', 'error'] : ['error'],
  
  // Synchronize (ห้ามใช้ใน production!)
  synchronize: isDevelopment,
  
  // SSL (สำหรับ production)
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: false,
  } : false,
  
  // Connection pool
  poolSize: 10,
  connectTimeoutMS: 10000,
};

export const AppDataSource = new DataSource(dbConfig);

export async function initializeDatabase(): Promise<void> {
  try {
    await AppDataSource.initialize();
    console.log('Database connection established successfully');
  } catch (error) {
    console.error('Failed to connect to database:', error);
    throw error;
  }
}

export async function closeDatabase(): Promise<void> {
  if (AppDataSource.isInitialized) {
    await AppDataSource.destroy();
    console.log('Database connection closed');
  }
}
```

---

## 22.3 Entity Definitions

### src/entities/User.ts

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  BeforeInsert,
  BeforeUpdate,
  Index,
} from 'typeorm';
import bcrypt from 'bcryptjs';
import { Post } from './Post';
import { Comment } from './Comment';

export enum UserRole {
  ADMIN = 'admin',
  AUTHOR = 'author',
  READER = 'reader',
}

@Entity('users')
@Index(['email'], { unique: true })
export class User {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar', length: 100 })
  name!: string;

  @Column({ type: 'varchar', length: 255, unique: true })
  email!: string;

  @Column({ type: 'varchar', length: 255 })
  password!: string;

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.READER,
  })
  role!: UserRole;

  @Column({ type: 'varchar', length: 500, nullable: true })
  bio?: string;

  @Column({ type: 'varchar', length: 255, nullable: true })
  avatar?: string;

  @Column({ type: 'boolean', default: true })
  isActive!: boolean;

  @Column({ type: 'timestamp', nullable: true })
  lastLoginAt?: Date;

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;

  // Relations
  @OneToMany(() => Post, post => post.author)
  posts!: Post[];

  @OneToMany(() => Comment, comment => comment.author)
  comments!: Comment[];

  // Hooks
  @BeforeInsert()
  @BeforeUpdate()
  async hashPassword(): Promise<void> {
    // Hash password เฉพาะเมื่อมีการเปลี่ยนแปลง
    if (this.password && !this.password.startsWith('$2b$')) {
      this.password = await bcrypt.hash(this.password, 12);
    }
  }

  // Methods
  async validatePassword(password: string): Promise<boolean> {
    return bcrypt.compare(password, this.password);
  }

  toJSON(): Omit<User, 'password' | 'hashPassword' | 'validatePassword' | 'toJSON'> {
    const { password, ...rest } = this;
    return rest as Omit<User, 'password' | 'hashPassword' | 'validatePassword' | 'toJSON'>;
  }
}
```

### src/entities/Category.ts

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
} from 'typeorm';
import { Post } from './Post';

@Entity('categories')
export class Category {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar', length: 100, unique: true })
  name!: string;

  @Column({ type: 'varchar', length: 100, unique: true })
  slug!: string;

  @Column({ type: 'text', nullable: true })
  description?: string;

  @Column({ type: 'varchar', length: 255, nullable: true })
  coverImage?: string;

  @Column({ type: 'boolean', default: true })
  isActive!: boolean;

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;

  // Relations
  @OneToMany(() => Post, post => post.category)
  posts!: Post[];
}
```

### src/entities/Tag.ts

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  ManyToMany,
} from 'typeorm';
import { Post } from './Post';

@Entity('tags')
export class Tag {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar', length: 50, unique: true })
  name!: string;

  @Column({ type: 'varchar', length: 50, unique: true })
  slug!: string;

  @Column({ type: 'varchar', length: 7, nullable: true })
  color?: string; // เช่น '#FF5733'

  @CreateDateColumn()
  createdAt!: Date;

  // Relations (ManyToMany กับ Post)
  @ManyToMany(() => Post, post => post.tags)
  posts!: Post[];
}
```

### src/entities/Post.ts

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  OneToMany,
  ManyToMany,
  JoinTable,
  JoinColumn,
  Index,
} from 'typeorm';
import { User } from './User';
import { Category } from './Category';
import { Tag } from './Tag';
import { Comment } from './Comment';

export enum PostStatus {
  DRAFT = 'draft',
  PUBLISHED = 'published',
  ARCHIVED = 'archived',
}

@Entity('posts')
@Index(['slug'], { unique: true })
@Index(['status', 'createdAt'])
export class Post {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar', length: 255 })
  title!: string;

  @Column({ type: 'varchar', length: 255, unique: true })
  slug!: string;

  @Column({ type: 'text' })
  content!: string;

  @Column({ type: 'text', nullable: true })
  excerpt?: string;

  @Column({ type: 'varchar', length: 255, nullable: true })
  coverImage?: string;

  @Column({
    type: 'enum',
    enum: PostStatus,
    default: PostStatus.DRAFT,
  })
  status!: PostStatus;

  @Column({ type: 'int', default: 0 })
  viewCount!: number;

  @Column({ type: 'int', default: 0 })
  likeCount!: number;

  @Column({ type: 'timestamp', nullable: true })
  publishedAt?: Date;

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;

  // Foreign keys
  @Column({ type: 'uuid' })
  authorId!: string;

  @Column({ type: 'uuid', nullable: true })
  categoryId?: string;

  // Relations
  @ManyToOne(() => User, user => user.posts, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'authorId' })
  author!: User;

  @ManyToOne(() => Category, category => category.posts, { nullable: true, onDelete: 'SET NULL' })
  @JoinColumn({ name: 'categoryId' })
  category?: Category;

  @OneToMany(() => Comment, comment => comment.post)
  comments!: Comment[];

  @ManyToMany(() => Tag, tag => tag.posts)
  @JoinTable({
    name: 'post_tags',
    joinColumn: { name: 'postId', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'tagId', referencedColumnName: 'id' },
  })
  tags!: Tag[];
}
```

### src/entities/Comment.ts

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  OneToMany,
  JoinColumn,
} from 'typeorm';
import { User } from './User';
import { Post } from './Post';

@Entity('comments')
export class Comment {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'text' })
  content!: string;

  @Column({ type: 'boolean', default: true })
  isApproved!: boolean;

  @Column({ type: 'int', default: 0 })
  likeCount!: number;

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;

  // Foreign keys
  @Column({ type: 'uuid' })
  authorId!: string;

  @Column({ type: 'uuid' })
  postId!: string;

  @Column({ type: 'uuid', nullable: true })
  parentId?: string;

  // Relations
  @ManyToOne(() => User, user => user.comments, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'authorId' })
  author!: User;

  @ManyToOne(() => Post, post => post.comments, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'postId' })
  post!: Post;

  // Self-referencing relation สำหรับ nested comments
  @ManyToOne(() => Comment, comment => comment.replies, { nullable: true })
  @JoinColumn({ name: 'parentId' })
  parent?: Comment;

  @OneToMany(() => Comment, comment => comment.parent)
  replies!: Comment[];
}
```

---

## 22.4 Repository Pattern

### src/repositories/baseRepository.ts

```typescript
import { Repository, FindOptionsWhere, DeepPartial, FindManyOptions, EntityTarget, DataSource } from 'typeorm';

export abstract class BaseRepository<T extends { id: string }> {
  protected readonly repository: Repository<T>;

  constructor(
    private readonly entity: EntityTarget<T>,
    private readonly dataSource: DataSource
  ) {
    this.repository = dataSource.getRepository(entity);
  }

  async findById(id: string): Promise<T | null> {
    return this.repository.findOne({
      where: { id } as FindOptionsWhere<T>,
    });
  }

  async findAll(options?: FindManyOptions<T>): Promise<T[]> {
    return this.repository.find(options);
  }

  async count(where?: FindOptionsWhere<T>): Promise<number> {
    return this.repository.count({ where });
  }

  async save(entity: DeepPartial<T>): Promise<T> {
    return this.repository.save(entity as DeepPartial<T>);
  }

  async update(id: string, data: Partial<T>): Promise<T | null> {
    await this.repository.update(id, data as any);
    return this.findById(id);
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.repository.delete(id);
    return (result.affected || 0) > 0;
  }

  async exists(where: FindOptionsWhere<T>): Promise<boolean> {
    const count = await this.repository.count({ where });
    return count > 0;
  }
}
```

### src/repositories/userRepository.ts

```typescript
import { AppDataSource } from '../database/data-source';
import { User } from '../entities/User';
import { BaseRepository } from './baseRepository';
import { FindOptionsWhere } from 'typeorm';

export class UserRepository extends BaseRepository<User> {
  constructor() {
    super(User, AppDataSource);
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.repository.findOne({
      where: { email: email.toLowerCase() },
    });
  }

  async findActiveUsers(skip: number, take: number): Promise<[User[], number]> {
    return this.repository.findAndCount({
      where: { isActive: true },
      order: { createdAt: 'DESC' },
      skip,
      take,
    });
  }

  async findWithPosts(id: string): Promise<User | null> {
    return this.repository.findOne({
      where: { id },
      relations: ['posts', 'posts.category', 'posts.tags'],
    });
  }

  async searchUsers(query: string, skip: number, take: number): Promise<[User[], number]> {
    return this.repository
      .createQueryBuilder('user')
      .where('user.name ILIKE :query OR user.email ILIKE :query', {
        query: `%${query}%`,
      })
      .andWhere('user.isActive = true')
      .orderBy('user.createdAt', 'DESC')
      .skip(skip)
      .take(take)
      .getManyAndCount();
  }
}
```

### src/repositories/postRepository.ts

```typescript
import { AppDataSource } from '../database/data-source';
import { Post, PostStatus } from '../entities/Post';
import { BaseRepository } from './baseRepository';

export interface PostFilter {
  status?: PostStatus;
  authorId?: string;
  categoryId?: string;
  tagId?: string;
  search?: string;
  skip?: number;
  take?: number;
}

export class PostRepository extends BaseRepository<Post> {
  constructor() {
    super(Post, AppDataSource);
  }

  async findWithRelations(id: string): Promise<Post | null> {
    return this.repository.findOne({
      where: { id },
      relations: ['author', 'category', 'tags', 'comments', 'comments.author'],
    });
  }

  async findBySlug(slug: string): Promise<Post | null> {
    return this.repository.findOne({
      where: { slug },
      relations: ['author', 'category', 'tags'],
    });
  }

  async findPublishedPosts(filter: PostFilter): Promise<[Post[], number]> {
    const qb = this.repository
      .createQueryBuilder('post')
      .leftJoinAndSelect('post.author', 'author')
      .leftJoinAndSelect('post.category', 'category')
      .leftJoinAndSelect('post.tags', 'tags')
      .where('post.status = :status', { status: PostStatus.PUBLISHED });
    
    if (filter.authorId) {
      qb.andWhere('post.authorId = :authorId', { authorId: filter.authorId });
    }
    
    if (filter.categoryId) {
      qb.andWhere('post.categoryId = :categoryId', { categoryId: filter.categoryId });
    }
    
    if (filter.tagId) {
      qb.andWhere('tags.id = :tagId', { tagId: filter.tagId });
    }
    
    if (filter.search) {
      qb.andWhere(
        '(post.title ILIKE :search OR post.content ILIKE :search)',
        { search: `%${filter.search}%` }
      );
    }
    
    return qb
      .orderBy('post.publishedAt', 'DESC')
      .skip(filter.skip || 0)
      .take(filter.take || 10)
      .getManyAndCount();
  }

  async incrementViewCount(id: string): Promise<void> {
    await this.repository
      .createQueryBuilder()
      .update(Post)
      .set({ viewCount: () => 'viewCount + 1' })
      .where('id = :id', { id })
      .execute();
  }

  async getPopularPosts(limit: number = 10): Promise<Post[]> {
    return this.repository
      .createQueryBuilder('post')
      .leftJoinAndSelect('post.author', 'author')
      .leftJoinAndSelect('post.category', 'category')
      .where('post.status = :status', { status: PostStatus.PUBLISHED })
      .orderBy('post.viewCount', 'DESC')
      .addOrderBy('post.likeCount', 'DESC')
      .take(limit)
      .getMany();
  }
}
```

---

## 22.5 Query Builder

### ตัวอย่างการใช้ Query Builder แบบต่าง ๆ

```typescript
import { AppDataSource } from '../database/data-source';
import { Post, PostStatus } from '../entities/Post';
import { User } from '../entities/User';
import { Comment } from '../entities/Comment';

const postRepo = AppDataSource.getRepository(Post);
const userRepo = AppDataSource.getRepository(User);
const commentRepo = AppDataSource.getRepository(Comment);

// 1. SELECT พร้อม conditions
async function getPublishedPostsByAuthor(authorId: string) {
  return postRepo
    .createQueryBuilder('post')
    .select(['post.id', 'post.title', 'post.slug', 'post.publishedAt'])
    .where('post.authorId = :authorId', { authorId })
    .andWhere('post.status = :status', { status: PostStatus.PUBLISHED })
    .orderBy('post.publishedAt', 'DESC')
    .getMany();
}

// 2. JOIN หลาย tables
async function getPostWithAllRelations(postId: string) {
  return postRepo
    .createQueryBuilder('post')
    .leftJoinAndSelect('post.author', 'author')
    .leftJoinAndSelect('post.category', 'category')
    .leftJoinAndSelect('post.tags', 'tags')
    .leftJoinAndSelect('post.comments', 'comments')
    .leftJoinAndSelect('comments.author', 'commentAuthor')
    .leftJoinAndSelect('comments.replies', 'replies')
    .where('post.id = :postId', { postId })
    .getOne();
}

// 3. Aggregate functions
async function getPostStatsByUser(userId: string) {
  return postRepo
    .createQueryBuilder('post')
    .select('post.status', 'status')
    .addSelect('COUNT(post.id)', 'count')
    .addSelect('SUM(post.viewCount)', 'totalViews')
    .addSelect('SUM(post.likeCount)', 'totalLikes')
    .where('post.authorId = :userId', { userId })
    .groupBy('post.status')
    .getRawMany();
}

// 4. Subquery
async function getUsersWithMostPosts(limit: number = 10) {
  return userRepo
    .createQueryBuilder('user')
    .addSelect(subQuery => {
      return subQuery
        .select('COUNT(post.id)', 'postCount')
        .from(Post, 'post')
        .where('post.authorId = user.id')
        .andWhere('post.status = :status', { status: PostStatus.PUBLISHED });
    }, 'user_postCount')
    .where('user.isActive = true')
    .orderBy('user_postCount', 'DESC')
    .take(limit)
    .getMany();
}

// 5. LIKE search
async function searchPosts(query: string, page: number, limit: number) {
  return postRepo
    .createQueryBuilder('post')
    .leftJoinAndSelect('post.author', 'author')
    .leftJoinAndSelect('post.category', 'category')
    .where('post.status = :status', { status: PostStatus.PUBLISHED })
    .andWhere(
      '(post.title ILIKE :query OR post.content ILIKE :query OR post.excerpt ILIKE :query)',
      { query: `%${query}%` }
    )
    .orderBy('post.publishedAt', 'DESC')
    .skip((page - 1) * limit)
    .take(limit)
    .getManyAndCount();
}

// 6. Update ด้วย Query Builder
async function publishPost(postId: string, authorId: string): Promise<boolean> {
  const result = await postRepo
    .createQueryBuilder()
    .update(Post)
    .set({
      status: PostStatus.PUBLISHED,
      publishedAt: new Date(),
    })
    .where('id = :postId', { postId })
    .andWhere('authorId = :authorId', { authorId })
    .execute();
  
  return (result.affected || 0) > 0;
}

// 7. Delete ด้วย Query Builder
async function deleteOldDrafts(userId: string, beforeDate: Date): Promise<number> {
  const result = await postRepo
    .createQueryBuilder()
    .delete()
    .from(Post)
    .where('authorId = :userId', { userId })
    .andWhere('status = :status', { status: PostStatus.DRAFT })
    .andWhere('createdAt < :beforeDate', { beforeDate })
    .execute();
  
  return result.affected || 0;
}

// 8. Raw query
async function getTagsWithPostCount() {
  return AppDataSource.query(`
    SELECT 
      t.id,
      t.name,
      t.slug,
      COUNT(pt.post_id) as post_count
    FROM tags t
    LEFT JOIN post_tags pt ON t.id = pt.tag_id
    LEFT JOIN posts p ON pt.post_id = p.id AND p.status = 'published'
    GROUP BY t.id, t.name, t.slug
    ORDER BY post_count DESC
  `);
}
```

---

## 22.6 Migrations

### สร้าง Migration

```bash
# สร้าง migration ใหม่
npm run migration:generate -- src/database/migrations/CreateUsersTable

# รัน migrations
npm run migration:run

# ย้อนกลับ migration ล่าสุด
npm run migration:revert

# ดู migration status
npm run migration:show
```

### ตัวอย่าง Migration file

```typescript
// src/database/migrations/1690000000000-CreateBlogTables.ts
import { MigrationInterface, QueryRunner, Table, TableIndex, TableForeignKey } from 'typeorm';

export class CreateBlogTables1690000000000 implements MigrationInterface {
  name = 'CreateBlogTables1690000000000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // สร้าง users table
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()',
          },
          {
            name: 'name',
            type: 'varchar',
            length: '100',
          },
          {
            name: 'email',
            type: 'varchar',
            length: '255',
            isUnique: true,
          },
          {
            name: 'password',
            type: 'varchar',
            length: '255',
          },
          {
            name: 'role',
            type: 'enum',
            enum: ['admin', 'author', 'reader'],
            default: "'reader'",
          },
          {
            name: 'bio',
            type: 'varchar',
            length: '500',
            isNullable: true,
          },
          {
            name: 'avatar',
            type: 'varchar',
            length: '255',
            isNullable: true,
          },
          {
            name: 'is_active',
            type: 'boolean',
            default: true,
          },
          {
            name: 'last_login_at',
            type: 'timestamp',
            isNullable: true,
          },
          {
            name: 'created_at',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP',
          },
          {
            name: 'updated_at',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP',
            onUpdate: 'CURRENT_TIMESTAMP',
          },
        ],
      }),
      true
    );
    
    // สร้าง index บน email
    await queryRunner.createIndex(
      'users',
      new TableIndex({
        name: 'IDX_users_email',
        columnNames: ['email'],
        isUnique: true,
      })
    );
    
    // สร้าง categories table
    await queryRunner.createTable(
      new Table({
        name: 'categories',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()',
          },
          { name: 'name', type: 'varchar', length: '100', isUnique: true },
          { name: 'slug', type: 'varchar', length: '100', isUnique: true },
          { name: 'description', type: 'text', isNullable: true },
          { name: 'cover_image', type: 'varchar', length: '255', isNullable: true },
          { name: 'is_active', type: 'boolean', default: true },
          { name: 'created_at', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
          { name: 'updated_at', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
        ],
      }),
      true
    );
    
    // สร้าง posts table
    await queryRunner.createTable(
      new Table({
        name: 'posts',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()',
          },
          { name: 'title', type: 'varchar', length: '255' },
          { name: 'slug', type: 'varchar', length: '255', isUnique: true },
          { name: 'content', type: 'text' },
          { name: 'excerpt', type: 'text', isNullable: true },
          { name: 'cover_image', type: 'varchar', length: '255', isNullable: true },
          {
            name: 'status',
            type: 'enum',
            enum: ['draft', 'published', 'archived'],
            default: "'draft'",
          },
          { name: 'view_count', type: 'int', default: 0 },
          { name: 'like_count', type: 'int', default: 0 },
          { name: 'published_at', type: 'timestamp', isNullable: true },
          { name: 'author_id', type: 'uuid' },
          { name: 'category_id', type: 'uuid', isNullable: true },
          { name: 'created_at', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
          { name: 'updated_at', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
        ],
      }),
      true
    );
    
    // Foreign keys สำหรับ posts
    await queryRunner.createForeignKey(
      'posts',
      new TableForeignKey({
        name: 'FK_posts_author',
        columnNames: ['author_id'],
        referencedTableName: 'users',
        referencedColumnNames: ['id'],
        onDelete: 'CASCADE',
      })
    );
    
    await queryRunner.createForeignKey(
      'posts',
      new TableForeignKey({
        name: 'FK_posts_category',
        columnNames: ['category_id'],
        referencedTableName: 'categories',
        referencedColumnNames: ['id'],
        onDelete: 'SET NULL',
      })
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('posts', true);
    await queryRunner.dropTable('categories', true);
    await queryRunner.dropTable('users', true);
  }
}
```

---

## 22.7 Transactions

### การใช้ Transactions

```typescript
import { AppDataSource } from '../database/data-source';
import { Post, PostStatus } from '../entities/Post';
import { User } from '../entities/User';

// วิธีที่ 1: ใช้ transaction() method
async function createPostWithStats(
  authorId: string,
  postData: Partial<Post>
): Promise<Post> {
  return AppDataSource.transaction(async transactionalEntityManager => {
    // ตรวจสอบว่า user มีอยู่จริง
    const user = await transactionalEntityManager.findOne(User, {
      where: { id: authorId },
    });
    
    if (!user) {
      throw new Error('User not found');
    }
    
    // สร้าง post
    const post = transactionalEntityManager.create(Post, {
      ...postData,
      authorId,
      status: PostStatus.DRAFT,
    });
    
    await transactionalEntityManager.save(post);
    
    return post;
  });
}

// วิธีที่ 2: Manual transaction
async function transferPostOwnership(
  postId: string,
  fromUserId: string,
  toUserId: string
): Promise<void> {
  const queryRunner = AppDataSource.createQueryRunner();
  
  await queryRunner.connect();
  await queryRunner.startTransaction();
  
  try {
    // ตรวจสอบ post
    const post = await queryRunner.manager.findOne(Post, {
      where: { id: postId, authorId: fromUserId },
    });
    
    if (!post) {
      throw new Error('Post not found or you do not own it');
    }
    
    // ตรวจสอบ target user
    const targetUser = await queryRunner.manager.findOne(User, {
      where: { id: toUserId },
    });
    
    if (!targetUser) {
      throw new Error('Target user not found');
    }
    
    // โอนความเป็นเจ้าของ
    await queryRunner.manager.update(Post, postId, {
      authorId: toUserId,
    });
    
    await queryRunner.commitTransaction();
  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}

// วิธีที่ 3: Transaction Decorator pattern
class PostTransactionService {
  async publishPostWithNotification(
    postId: string,
    authorId: string
  ): Promise<Post> {
    return AppDataSource.transaction(async manager => {
      // 1. Publish post
      const post = await manager.findOne(Post, {
        where: { id: postId, authorId },
        relations: ['author', 'category'],
      });
      
      if (!post) {
        throw new Error('Post not found');
      }
      
      if (post.status === PostStatus.PUBLISHED) {
        throw new Error('Post is already published');
      }
      
      post.status = PostStatus.PUBLISHED;
      post.publishedAt = new Date();
      
      await manager.save(post);
      
      // 2. Update author stats (ในแอพจริงอาจมี table stats แยก)
      await manager
        .createQueryBuilder()
        .update(User)
        .set({ updatedAt: new Date() })
        .where('id = :id', { id: authorId })
        .execute();
      
      return post;
    });
  }
}
```

---

## 22.8 Complete Service Layer

### src/services/postService.ts

```typescript
import { PostRepository, PostFilter } from '../repositories/postRepository';
import { UserRepository } from '../repositories/userRepository';
import { Post, PostStatus } from '../entities/Post';
import { Tag } from '../entities/Tag';
import { AppDataSource } from '../database/data-source';

export interface CreatePostInput {
  title: string;
  content: string;
  excerpt?: string;
  categoryId?: string;
  tagIds?: string[];
  status?: PostStatus;
}

export interface UpdatePostInput {
  title?: string;
  content?: string;
  excerpt?: string;
  categoryId?: string;
  tagIds?: string[];
  status?: PostStatus;
}

function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-')
    .trim();
}

export class PostService {
  private readonly postRepo: PostRepository;
  private readonly userRepo: UserRepository;

  constructor() {
    this.postRepo = new PostRepository();
    this.userRepo = new UserRepository();
  }

  async findPublished(filter: PostFilter) {
    const [posts, total] = await this.postRepo.findPublishedPosts(filter);
    return { posts, total };
  }

  async findById(id: string): Promise<Post | null> {
    return this.postRepo.findWithRelations(id);
  }

  async findBySlug(slug: string): Promise<Post | null> {
    const post = await this.postRepo.findBySlug(slug);
    if (post) {
      await this.postRepo.incrementViewCount(post.id);
    }
    return post;
  }

  async create(authorId: string, input: CreatePostInput): Promise<Post> {
    // ตรวจสอบว่า author มีอยู่
    const author = await this.userRepo.findById(authorId);
    if (!author) throw new Error('Author not found');
    
    // สร้าง unique slug
    let slug = generateSlug(input.title);
    const existingPost = await this.postRepo.findBySlug(slug);
    if (existingPost) {
      slug = `${slug}-${Date.now()}`;
    }
    
    // ดึง tags ถ้ามี
    let tags: Tag[] = [];
    if (input.tagIds && input.tagIds.length > 0) {
      tags = await AppDataSource.getRepository(Tag).findByIds(input.tagIds);
    }
    
    const post = this.postRepo.repository.create({
      title: input.title,
      slug,
      content: input.content,
      excerpt: input.excerpt,
      categoryId: input.categoryId,
      authorId,
      status: input.status || PostStatus.DRAFT,
      tags,
    });
    
    return this.postRepo.repository.save(post);
  }

  async update(
    id: string,
    authorId: string,
    input: UpdatePostInput
  ): Promise<Post | null> {
    const post = await this.postRepo.findById(id);
    
    if (!post) return null;
    if (post.authorId !== authorId) throw new Error('Not authorized to update this post');
    
    // อัพเดต slug ถ้าเปลี่ยน title
    if (input.title && input.title !== post.title) {
      let newSlug = generateSlug(input.title);
      const existingPost = await this.postRepo.findBySlug(newSlug);
      if (existingPost && existingPost.id !== id) {
        newSlug = `${newSlug}-${Date.now()}`;
      }
      post.slug = newSlug;
      post.title = input.title;
    }
    
    // อัพเดต tags
    if (input.tagIds !== undefined) {
      if (input.tagIds.length > 0) {
        post.tags = await AppDataSource.getRepository(Tag).findByIds(input.tagIds);
      } else {
        post.tags = [];
      }
    }
    
    if (input.content !== undefined) post.content = input.content;
    if (input.excerpt !== undefined) post.excerpt = input.excerpt;
    if (input.categoryId !== undefined) post.categoryId = input.categoryId;
    if (input.status !== undefined) {
      post.status = input.status;
      if (input.status === PostStatus.PUBLISHED && !post.publishedAt) {
        post.publishedAt = new Date();
      }
    }
    
    return this.postRepo.repository.save(post);
  }

  async publish(id: string, authorId: string): Promise<Post | null> {
    return AppDataSource.transaction(async manager => {
      const post = await manager.findOne(Post, {
        where: { id, authorId },
      });
      
      if (!post) return null;
      
      post.status = PostStatus.PUBLISHED;
      post.publishedAt = new Date();
      
      return manager.save(post);
    });
  }

  async delete(id: string, authorId: string, isAdmin: boolean = false): Promise<boolean> {
    const post = await this.postRepo.findById(id);
    
    if (!post) return false;
    if (!isAdmin && post.authorId !== authorId) {
      throw new Error('Not authorized to delete this post');
    }
    
    return this.postRepo.delete(id);
  }
}
```

---

## 22.9 Seeding Data

### src/database/seeds/index.ts

```typescript
import 'reflect-metadata';
import { AppDataSource, initializeDatabase, closeDatabase } from '../data-source';
import { User, UserRole } from '../../entities/User';
import { Category } from '../../entities/Category';
import { Tag } from '../../entities/Tag';
import { Post, PostStatus } from '../../entities/Post';
import { Comment } from '../../entities/Comment';

async function seed(): Promise<void> {
  await initializeDatabase();
  
  const userRepo = AppDataSource.getRepository(User);
  const categoryRepo = AppDataSource.getRepository(Category);
  const tagRepo = AppDataSource.getRepository(Tag);
  const postRepo = AppDataSource.getRepository(Post);
  const commentRepo = AppDataSource.getRepository(Comment);
  
  console.log('Seeding database...');
  
  // สร้าง Users
  const admin = userRepo.create({
    name: 'Admin User',
    email: 'admin@example.com',
    password: 'Admin@1234',
    role: UserRole.ADMIN,
  });
  
  const author = userRepo.create({
    name: 'สมชาย นักเขียน',
    email: 'author@example.com',
    password: 'Author@1234',
    role: UserRole.AUTHOR,
    bio: 'นักเขียนที่ชื่นชอบการเขียนบทความด้านเทคโนโลยี',
  });
  
  const reader = userRepo.create({
    name: 'สมหญิง อ่านหนังสือ',
    email: 'reader@example.com',
    password: 'Reader@1234',
    role: UserRole.READER,
  });
  
  await userRepo.save([admin, author, reader]);
  console.log('Users seeded');
  
  // สร้าง Categories
  const categories = categoryRepo.create([
    { name: 'เทคโนโลยี', slug: 'technology', description: 'บทความด้านเทคโนโลยีต่าง ๆ' },
    { name: 'การพัฒนาซอฟต์แวร์', slug: 'software-development', description: 'บทความด้านการพัฒนาซอฟต์แวร์' },
    { name: 'ความปลอดภัย', slug: 'security', description: 'บทความด้าน Cybersecurity' },
    { name: 'ปัญญาประดิษฐ์', slug: 'ai', description: 'บทความด้าน AI และ Machine Learning' },
  ]);
  
  await categoryRepo.save(categories);
  console.log('Categories seeded');
  
  // สร้าง Tags
  const tags = tagRepo.create([
    { name: 'TypeScript', slug: 'typescript', color: '#3178C6' },
    { name: 'Node.js', slug: 'nodejs', color: '#339933' },
    { name: 'React', slug: 'react', color: '#61DAFB' },
    { name: 'PostgreSQL', slug: 'postgresql', color: '#336791' },
    { name: 'Docker', slug: 'docker', color: '#2496ED' },
    { name: 'API', slug: 'api', color: '#FF6B6B' },
  ]);
  
  await tagRepo.save(tags);
  console.log('Tags seeded');
  
  // สร้าง Posts
  const techCategory = categories.find(c => c.slug === 'software-development')!;
  const tsTag = tags.find(t => t.slug === 'typescript')!;
  const nodeTag = tags.find(t => t.slug === 'nodejs')!;
  
  const post1 = postRepo.create({
    title: 'เริ่มต้นใช้งาน TypeScript กับ Node.js',
    slug: 'getting-started-typescript-nodejs',
    content: '# เริ่มต้นใช้งาน TypeScript กับ Node.js\n\nในบทความนี้เราจะมาเรียนรู้...',
    excerpt: 'เรียนรู้การตั้งค่า TypeScript กับ Node.js สำหรับผู้เริ่มต้น',
    status: PostStatus.PUBLISHED,
    publishedAt: new Date(),
    authorId: author.id,
    categoryId: techCategory.id,
    tags: [tsTag, nodeTag],
  });
  
  await postRepo.save([post1]);
  console.log('Posts seeded');
  
  // สร้าง Comments
  const comment1 = commentRepo.create({
    content: 'บทความดีมากเลยครับ ขอบคุณที่แบ่งปัน',
    postId: post1.id,
    authorId: reader.id,
    isApproved: true,
  });
  
  await commentRepo.save([comment1]);
  console.log('Comments seeded');
  
  console.log('Seeding completed!');
  await closeDatabase();
}

seed().catch(error => {
  console.error('Seeding failed:', error);
  process.exit(1);
});
```

---

## 22.10 Controllers สำหรับ Blog API

### src/controllers/postController.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { PostService, CreatePostInput, UpdatePostInput } from '../services/postService';
import { PostStatus } from '../entities/Post';
import { AppError } from '../middleware/errorHandler';

const postService = new PostService();

export class PostController {
  // GET /posts
  async getAll(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const page = parseInt(req.query.page as string || '1', 10);
      const limit = parseInt(req.query.limit as string || '10', 10);
      const skip = (page - 1) * limit;
      
      const { posts, total } = await postService.findPublished({
        skip,
        take: limit,
        search: req.query.search as string,
        categoryId: req.query.categoryId as string,
        tagId: req.query.tagId as string,
      });
      
      res.json({
        success: true,
        data: posts,
        meta: {
          total,
          page,
          limit,
          totalPages: Math.ceil(total / limit),
        },
      });
    } catch (error) {
      next(error);
    }
  }

  // GET /posts/:id
  async getById(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const post = await postService.findById(req.params.id);
      if (!post) throw AppError.notFound('Post not found');
      
      res.json({ success: true, data: post });
    } catch (error) {
      next(error);
    }
  }

  // POST /posts
  async create(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      if (!req.user) throw AppError.unauthorized();
      
      const input: CreatePostInput = {
        title: req.body.title,
        content: req.body.content,
        excerpt: req.body.excerpt,
        categoryId: req.body.categoryId,
        tagIds: req.body.tagIds,
        status: req.body.status,
      };
      
      const post = await postService.create(req.user.id, input);
      res.status(201).json({ success: true, data: post });
    } catch (error) {
      next(error);
    }
  }

  // PUT /posts/:id
  async update(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      if (!req.user) throw AppError.unauthorized();
      
      const post = await postService.update(req.params.id, req.user.id, req.body);
      if (!post) throw AppError.notFound('Post not found');
      
      res.json({ success: true, data: post });
    } catch (error) {
      next(error);
    }
  }

  // POST /posts/:id/publish
  async publish(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      if (!req.user) throw AppError.unauthorized();
      
      const post = await postService.publish(req.params.id, req.user.id);
      if (!post) throw AppError.notFound('Post not found');
      
      res.json({ success: true, data: post, message: 'Post published successfully' });
    } catch (error) {
      next(error);
    }
  }

  // DELETE /posts/:id
  async delete(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      if (!req.user) throw AppError.unauthorized();
      
      const isAdmin = req.user.role === 'admin';
      const deleted = await postService.delete(req.params.id, req.user.id, isAdmin);
      
      if (!deleted) throw AppError.notFound('Post not found');
      
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }
}
```

---

## สรุปบทที่ 22

ในบทนี้เราได้เรียนรู้:

1. **TypeORM Setup** - การติดตั้งและเชื่อมต่อฐานข้อมูล
2. **Entity Definitions** - การกำหนด entities ด้วย decorators
3. **Column Decorators** - ประเภท columns ต่าง ๆ
4. **Relationships** - OneToOne, OneToMany, ManyToOne, ManyToMany
5. **Repository Pattern** - Base repository และ custom repositories
6. **Query Builder** - การสร้าง queries แบบ complex
7. **Migrations** - การจัดการ schema changes
8. **Transactions** - การรับประกัน data consistency
9. **Seeding** - การสร้าง initial data
10. **Complete Blog API** - ตัวอย่าง API ที่สมบูรณ์

ในบทถัดไปเราจะเรียนรู้การใช้งาน Zod สำหรับ validation ที่มี type safety
