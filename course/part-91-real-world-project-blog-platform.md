# Part 91: โปรเจกต์จริง - Blog Platform ด้วย TypeScript

## บทนำ

ในบทนี้เราจะสร้าง Blog Platform ที่ครบครันและพร้อมใช้งานจริง โดยใช้ TypeScript ทั้ง Backend (NestJS) และ Frontend (Next.js) ครอบคลุมฟีเจอร์หลักที่ Blog Platform ควรมี

## สถาปัตยกรรมโปรเจกต์

```
blog-platform/
├── backend/                    # NestJS API
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── users/
│   │   │   ├── posts/
│   │   │   ├── categories/
│   │   │   ├── tags/
│   │   │   ├── comments/
│   │   │   ├── likes/
│   │   │   ├── bookmarks/
│   │   │   ├── search/
│   │   │   ├── uploads/
│   │   │   └── rss/
│   │   ├── common/
│   │   │   ├── decorators/
│   │   │   ├── filters/
│   │   │   ├── guards/
│   │   │   ├── interceptors/
│   │   │   └── pipes/
│   │   ├── config/
│   │   └── database/
│   └── test/
├── frontend/                   # Next.js App
│   ├── app/
│   ├── components/
│   ├── hooks/
│   ├── lib/
│   └── types/
└── shared/                     # Shared Types
    └── types/
```

## ส่วนที่ 1: การตั้งค่าโปรเจกต์

### 1.1 Backend - NestJS Setup

```bash
# สร้าง NestJS project
npm i -g @nestjs/cli
nest new blog-backend
cd blog-backend

# ติดตั้ง dependencies
npm install @nestjs/typeorm typeorm pg
npm install @nestjs/jwt passport-jwt @nestjs/passport
npm install @nestjs/config
npm install @nestjs/swagger swagger-ui-express
npm install bcryptjs
npm install slugify
npm install marked marked-sanitizer-html
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install rss
npm install class-validator class-transformer
npm install @nestjs/throttler

# Dev dependencies
npm install -D @types/bcryptjs @types/passport-jwt
npm install -D @types/multer
```

### 1.2 Frontend - Next.js Setup

```bash
# สร้าง Next.js project
npx create-next-app@latest blog-frontend --typescript --tailwind --app
cd blog-frontend

# ติดตั้ง dependencies
npm install @tanstack/react-query axios
npm install next-auth
npm install react-hook-form @hookform/resolvers zod
npm install @tiptap/react @tiptap/pm @tiptap/starter-kit
npm install react-markdown remark-gfm
npm install date-fns
npm install lucide-react
```

### 1.3 Shared Types

```typescript
// shared/types/index.ts

export interface User {
  id: string;
  email: string;
  username: string;
  displayName: string;
  bio?: string;
  avatar?: string;
  role: UserRole;
  createdAt: Date;
  updatedAt: Date;
}

export enum UserRole {
  ADMIN = 'admin',
  AUTHOR = 'author',
  READER = 'reader',
}

export interface Post {
  id: string;
  title: string;
  slug: string;
  excerpt: string;
  content: string;
  coverImage?: string;
  status: PostStatus;
  author: User;
  category: Category;
  tags: Tag[];
  likesCount: number;
  commentsCount: number;
  viewsCount: number;
  readingTime: number;
  publishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export enum PostStatus {
  DRAFT = 'draft',
  PUBLISHED = 'published',
  ARCHIVED = 'archived',
}

export interface Category {
  id: string;
  name: string;
  slug: string;
  description?: string;
  postsCount: number;
}

export interface Tag {
  id: string;
  name: string;
  slug: string;
  postsCount: number;
}

export interface Comment {
  id: string;
  content: string;
  author: User;
  post: Post;
  parentId?: string;
  replies?: Comment[];
  likesCount: number;
  createdAt: Date;
  updatedAt: Date;
}

export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNextPage: boolean;
  hasPrevPage: boolean;
}

export interface PaginatedResponse<T> {
  data: T[];
  meta: PaginationMeta;
}
```

## ส่วนที่ 2: Backend - NestJS Implementation

### 2.1 Database Configuration

```typescript
// backend/src/config/database.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import { ConfigService } from '@nestjs/config';

export const getDatabaseConfig = (
  configService: ConfigService,
): TypeOrmModuleOptions => ({
  type: 'postgres',
  host: configService.get('DB_HOST', 'localhost'),
  port: configService.get<number>('DB_PORT', 5432),
  username: configService.get('DB_USERNAME', 'postgres'),
  password: configService.get('DB_PASSWORD', 'password'),
  database: configService.get('DB_NAME', 'blog_platform'),
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  migrations: [__dirname + '/../database/migrations/*{.ts,.js}'],
  synchronize: configService.get('NODE_ENV') !== 'production',
  logging: configService.get('NODE_ENV') === 'development',
  ssl:
    configService.get('NODE_ENV') === 'production'
      ? { rejectUnauthorized: false }
      : false,
});
```

### 2.2 User Entity

```typescript
// backend/src/modules/users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  BeforeInsert,
  BeforeUpdate,
} from 'typeorm';
import * as bcrypt from 'bcryptjs';
import { Post } from '../../posts/entities/post.entity';
import { Comment } from '../../comments/entities/comment.entity';
import { Like } from '../../likes/entities/like.entity';
import { Bookmark } from '../../bookmarks/entities/bookmark.entity';

export enum UserRole {
  ADMIN = 'admin',
  AUTHOR = 'author',
  READER = 'reader',
}

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column({ unique: true })
  username: string;

  @Column()
  displayName: string;

  @Column({ select: false })
  password: string;

  @Column({ nullable: true, type: 'text' })
  bio: string;

  @Column({ nullable: true })
  avatar: string;

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.READER,
  })
  role: UserRole;

  @Column({ default: true })
  isActive: boolean;

  @Column({ nullable: true })
  refreshToken: string;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @OneToMany(() => Comment, (comment) => comment.author)
  comments: Comment[];

  @OneToMany(() => Like, (like) => like.user)
  likes: Like[];

  @OneToMany(() => Bookmark, (bookmark) => bookmark.user)
  bookmarks: Bookmark[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @BeforeInsert()
  @BeforeUpdate()
  async hashPassword() {
    if (this.password) {
      this.password = await bcrypt.hash(this.password, 12);
    }
  }

  async comparePassword(attempt: string): Promise<boolean> {
    return bcrypt.compare(attempt, this.password);
  }
}
```

### 2.3 Post Entity

```typescript
// backend/src/modules/posts/entities/post.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  ManyToMany,
  JoinTable,
  OneToMany,
  JoinColumn,
  Index,
  BeforeInsert,
  BeforeUpdate,
} from 'typeorm';
import { User } from '../../users/entities/user.entity';
import { Category } from '../../categories/entities/category.entity';
import { Tag } from '../../tags/entities/tag.entity';
import { Comment } from '../../comments/entities/comment.entity';
import { Like } from '../../likes/entities/like.entity';
import { Bookmark } from '../../bookmarks/entities/bookmark.entity';
import slugify from 'slugify';

export enum PostStatus {
  DRAFT = 'draft',
  PUBLISHED = 'published',
  ARCHIVED = 'archived',
}

@Entity('posts')
@Index(['slug'], { unique: true })
@Index(['status', 'publishedAt'])
export class Post {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  title: string;

  @Column({ unique: true })
  slug: string;

  @Column({ type: 'text' })
  excerpt: string;

  @Column({ type: 'text' })
  content: string;

  @Column({ nullable: true })
  coverImage: string;

  @Column({
    type: 'enum',
    enum: PostStatus,
    default: PostStatus.DRAFT,
  })
  status: PostStatus;

  @Column({ default: 0 })
  viewsCount: number;

  @Column({ default: 0 })
  likesCount: number;

  @Column({ default: 0 })
  commentsCount: number;

  @Column({ default: 0 })
  readingTime: number;

  @Column({ nullable: true })
  publishedAt: Date;

  @Column({ nullable: true })
  metaTitle: string;

  @Column({ nullable: true, type: 'text' })
  metaDescription: string;

  @ManyToOne(() => User, (user) => user.posts, { eager: true })
  @JoinColumn()
  author: User;

  @ManyToOne(() => Category, (category) => category.posts, { eager: true })
  @JoinColumn()
  category: Category;

  @ManyToMany(() => Tag, (tag) => tag.posts, { eager: true })
  @JoinTable({
    name: 'post_tags',
    joinColumn: { name: 'post_id' },
    inverseJoinColumn: { name: 'tag_id' },
  })
  tags: Tag[];

  @OneToMany(() => Comment, (comment) => comment.post)
  comments: Comment[];

  @OneToMany(() => Like, (like) => like.post)
  likes: Like[];

  @OneToMany(() => Bookmark, (bookmark) => bookmark.post)
  bookmarks: Bookmark[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @BeforeInsert()
  async generateSlug() {
    if (!this.slug) {
      this.slug = await this.createUniqueSlug(this.title);
    }
    this.calculateReadingTime();
  }

  @BeforeUpdate()
  async onUpdate() {
    this.calculateReadingTime();
  }

  private async createUniqueSlug(title: string): Promise<string> {
    return slugify(title, {
      lower: true,
      strict: true,
      locale: 'th',
    });
  }

  private calculateReadingTime() {
    const wordsPerMinute = 200;
    const wordCount = this.content?.split(/\s+/).length || 0;
    this.readingTime = Math.ceil(wordCount / wordsPerMinute);
  }
}
```

### 2.4 Category Entity

```typescript
// backend/src/modules/categories/entities/category.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  BeforeInsert,
} from 'typeorm';
import { Post } from '../../posts/entities/post.entity';
import slugify from 'slugify';

@Entity('categories')
export class Category {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  name: string;

  @Column({ unique: true })
  slug: string;

  @Column({ nullable: true, type: 'text' })
  description: string;

  @Column({ nullable: true })
  image: string;

  @OneToMany(() => Post, (post) => post.category)
  posts: Post[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @BeforeInsert()
  generateSlug() {
    if (!this.slug) {
      this.slug = slugify(this.name, { lower: true, strict: true });
    }
  }
}
```

### 2.5 Tag Entity

```typescript
// backend/src/modules/tags/entities/tag.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  ManyToMany,
  BeforeInsert,
} from 'typeorm';
import { Post } from '../../posts/entities/post.entity';
import slugify from 'slugify';

@Entity('tags')
export class Tag {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  name: string;

  @Column({ unique: true })
  slug: string;

  @ManyToMany(() => Post, (post) => post.tags)
  posts: Post[];

  @CreateDateColumn()
  createdAt: Date;

  @BeforeInsert()
  generateSlug() {
    if (!this.slug) {
      this.slug = slugify(this.name, { lower: true, strict: true });
    }
  }
}
```

### 2.6 Comment Entity

```typescript
// backend/src/modules/comments/entities/comment.entity.ts
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
import { User } from '../../users/entities/user.entity';
import { Post } from '../../posts/entities/post.entity';

@Entity('comments')
export class Comment {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'text' })
  content: string;

  @Column({ default: 0 })
  likesCount: number;

  @Column({ default: false })
  isEdited: boolean;

  @ManyToOne(() => User, (user) => user.comments, { eager: true })
  @JoinColumn()
  author: User;

  @ManyToOne(() => Post, (post) => post.comments)
  @JoinColumn()
  post: Post;

  @ManyToOne(() => Comment, (comment) => comment.replies, { nullable: true })
  parent: Comment;

  @OneToMany(() => Comment, (comment) => comment.parent)
  replies: Comment[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### 2.7 Authentication Module

```typescript
// backend/src/modules/auth/auth.service.ts
import {
  Injectable,
  UnauthorizedException,
  ConflictException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import { User, UserRole } from '../users/entities/user.entity';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

interface JwtPayload {
  sub: string;
  email: string;
  role: UserRole;
}

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

  async register(registerDto: RegisterDto): Promise<TokenPair> {
    const existingUser = await this.userRepository.findOne({
      where: [
        { email: registerDto.email },
        { username: registerDto.username },
      ],
    });

    if (existingUser) {
      throw new ConflictException(
        existingUser.email === registerDto.email
          ? 'อีเมลนี้ถูกใช้งานแล้ว'
          : 'ชื่อผู้ใช้นี้ถูกใช้งานแล้ว',
      );
    }

    const user = this.userRepository.create({
      ...registerDto,
      role: UserRole.READER,
    });

    await this.userRepository.save(user);
    return this.generateTokens(user);
  }

  async login(loginDto: LoginDto): Promise<TokenPair> {
    const user = await this.userRepository.findOne({
      where: { email: loginDto.email },
      select: ['id', 'email', 'username', 'password', 'role', 'isActive'],
    });

    if (!user || !user.isActive) {
      throw new UnauthorizedException('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
    }

    const isPasswordValid = await user.comparePassword(loginDto.password);
    if (!isPasswordValid) {
      throw new UnauthorizedException('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
    }

    return this.generateTokens(user);
  }

  async refreshTokens(userId: string, refreshToken: string): Promise<TokenPair> {
    const user = await this.userRepository.findOne({
      where: { id: userId },
      select: ['id', 'email', 'role', 'refreshToken'],
    });

    if (!user || user.refreshToken !== refreshToken) {
      throw new UnauthorizedException('Refresh token ไม่ถูกต้อง');
    }

    return this.generateTokens(user);
  }

  async logout(userId: string): Promise<void> {
    await this.userRepository.update(userId, { refreshToken: null });
  }

  private async generateTokens(user: User): Promise<TokenPair> {
    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      role: user.role,
    };

    const [accessToken, refreshToken] = await Promise.all([
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_SECRET'),
        expiresIn: '15m',
      }),
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
        expiresIn: '7d',
      }),
    ]);

    await this.userRepository.update(user.id, { refreshToken });

    return { accessToken, refreshToken };
  }
}
```

### 2.8 Posts Service

```typescript
// backend/src/modules/posts/posts.service.ts
import {
  Injectable,
  NotFoundException,
  ForbiddenException,
} from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, SelectQueryBuilder } from 'typeorm';
import { Post, PostStatus } from './entities/post.entity';
import { CreatePostDto } from './dto/create-post.dto';
import { UpdatePostDto } from './dto/update-post.dto';
import { GetPostsQueryDto } from './dto/get-posts-query.dto';
import { User, UserRole } from '../users/entities/user.entity';
import { Tag } from '../tags/entities/tag.entity';
import { Category } from '../categories/entities/category.entity';
import { PaginatedResponse } from '../../../shared/types';
import { marked } from 'marked';
import sanitizeHtml from 'sanitize-html';
import slugify from 'slugify';

@Injectable()
export class PostsService {
  constructor(
    @InjectRepository(Post)
    private readonly postRepository: Repository<Post>,
    @InjectRepository(Tag)
    private readonly tagRepository: Repository<Tag>,
    @InjectRepository(Category)
    private readonly categoryRepository: Repository<Category>,
  ) {}

  async findAll(query: GetPostsQueryDto): Promise<PaginatedResponse<Post>> {
    const { page = 1, limit = 10, status, categorySlug, tagSlug, authorId, search } = query;

    const qb = this.postRepository
      .createQueryBuilder('post')
      .leftJoinAndSelect('post.author', 'author')
      .leftJoinAndSelect('post.category', 'category')
      .leftJoinAndSelect('post.tags', 'tag');

    if (status) {
      qb.andWhere('post.status = :status', { status });
    } else {
      qb.andWhere('post.status = :status', { status: PostStatus.PUBLISHED });
    }

    if (categorySlug) {
      qb.andWhere('category.slug = :categorySlug', { categorySlug });
    }

    if (tagSlug) {
      qb.andWhere('tag.slug = :tagSlug', { tagSlug });
    }

    if (authorId) {
      qb.andWhere('author.id = :authorId', { authorId });
    }

    if (search) {
      qb.andWhere(
        '(post.title ILIKE :search OR post.excerpt ILIKE :search)',
        { search: `%${search}%` },
      );
    }

    qb.orderBy('post.publishedAt', 'DESC')
      .skip((page - 1) * limit)
      .take(limit);

    const [data, total] = await qb.getManyAndCount();

    return {
      data,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNextPage: page < Math.ceil(total / limit),
        hasPrevPage: page > 1,
      },
    };
  }

  async findBySlug(slug: string): Promise<Post> {
    const post = await this.postRepository.findOne({
      where: { slug, status: PostStatus.PUBLISHED },
      relations: ['author', 'category', 'tags'],
    });

    if (!post) {
      throw new NotFoundException(`ไม่พบบทความ: ${slug}`);
    }

    // เพิ่มจำนวนการดู
    await this.postRepository.increment({ id: post.id }, 'viewsCount', 1);
    post.viewsCount += 1;

    return post;
  }

  async create(createPostDto: CreatePostDto, author: User): Promise<Post> {
    const category = await this.categoryRepository.findOne({
      where: { id: createPostDto.categoryId },
    });

    if (!category) {
      throw new NotFoundException('ไม่พบหมวดหมู่');
    }

    const tags = await Promise.all(
      (createPostDto.tags || []).map((tagName) =>
        this.findOrCreateTag(tagName),
      ),
    );

    // แปลง Markdown เป็น HTML และ sanitize
    const htmlContent = this.processMarkdown(createPostDto.content);

    const slug = await this.generateUniqueSlug(createPostDto.title);

    const post = this.postRepository.create({
      ...createPostDto,
      content: htmlContent,
      slug,
      author,
      category,
      tags,
      publishedAt:
        createPostDto.status === PostStatus.PUBLISHED ? new Date() : null,
    });

    return this.postRepository.save(post);
  }

  async update(
    id: string,
    updatePostDto: UpdatePostDto,
    currentUser: User,
  ): Promise<Post> {
    const post = await this.postRepository.findOne({
      where: { id },
      relations: ['author'],
    });

    if (!post) {
      throw new NotFoundException('ไม่พบบทความ');
    }

    if (
      post.author.id !== currentUser.id &&
      currentUser.role !== UserRole.ADMIN
    ) {
      throw new ForbiddenException('ไม่มีสิทธิ์แก้ไขบทความนี้');
    }

    if (updatePostDto.content) {
      updatePostDto.content = this.processMarkdown(updatePostDto.content);
    }

    if (
      updatePostDto.status === PostStatus.PUBLISHED &&
      post.status !== PostStatus.PUBLISHED
    ) {
      post.publishedAt = new Date();
    }

    if (updatePostDto.tags) {
      const tags = await Promise.all(
        updatePostDto.tags.map((tagName) => this.findOrCreateTag(tagName)),
      );
      post.tags = tags;
    }

    Object.assign(post, updatePostDto);
    return this.postRepository.save(post);
  }

  async delete(id: string, currentUser: User): Promise<void> {
    const post = await this.postRepository.findOne({
      where: { id },
      relations: ['author'],
    });

    if (!post) {
      throw new NotFoundException('ไม่พบบทความ');
    }

    if (
      post.author.id !== currentUser.id &&
      currentUser.role !== UserRole.ADMIN
    ) {
      throw new ForbiddenException('ไม่มีสิทธิ์ลบบทความนี้');
    }

    await this.postRepository.softDelete(id);
  }

  private processMarkdown(content: string): string {
    const htmlContent = marked(content) as string;
    return sanitizeHtml(htmlContent, {
      allowedTags: [
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'p', 'br', 'strong', 'em', 'u', 's',
        'ul', 'ol', 'li',
        'blockquote', 'pre', 'code',
        'a', 'img',
        'table', 'thead', 'tbody', 'tr', 'th', 'td',
        'div', 'span',
      ],
      allowedAttributes: {
        a: ['href', 'title', 'target', 'rel'],
        img: ['src', 'alt', 'title', 'width', 'height'],
        code: ['class'],
        pre: ['class'],
        '*': ['class'],
      },
    });
  }

  private async findOrCreateTag(name: string): Promise<Tag> {
    const slug = slugify(name, { lower: true, strict: true });
    let tag = await this.tagRepository.findOne({ where: { slug } });

    if (!tag) {
      tag = this.tagRepository.create({ name, slug });
      await this.tagRepository.save(tag);
    }

    return tag;
  }

  private async generateUniqueSlug(title: string): Promise<string> {
    let slug = slugify(title, { lower: true, strict: true, locale: 'th' });
    let counter = 0;

    while (true) {
      const checkSlug = counter > 0 ? `${slug}-${counter}` : slug;
      const existing = await this.postRepository.findOne({
        where: { slug: checkSlug },
      });

      if (!existing) {
        return checkSlug;
      }
      counter++;
    }
  }
}
```

### 2.9 Search Service (Full-text Search)

```typescript
// backend/src/modules/search/search.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Post, PostStatus } from '../posts/entities/post.entity';

interface SearchResult {
  posts: Post[];
  total: number;
  query: string;
}

@Injectable()
export class SearchService {
  constructor(
    @InjectRepository(Post)
    private readonly postRepository: Repository<Post>,
  ) {}

  async search(query: string, page = 1, limit = 10): Promise<SearchResult> {
    const searchTerms = query.trim().split(/\s+/).join(' & ');

    const qb = this.postRepository
      .createQueryBuilder('post')
      .leftJoinAndSelect('post.author', 'author')
      .leftJoinAndSelect('post.category', 'category')
      .leftJoinAndSelect('post.tags', 'tags')
      .where('post.status = :status', { status: PostStatus.PUBLISHED })
      .andWhere(
        `to_tsvector('english', post.title || ' ' || post.excerpt || ' ' || post.content) @@ to_tsquery('english', :query)`,
        { query: searchTerms },
      )
      .addSelect(
        `ts_rank(to_tsvector('english', post.title || ' ' || post.excerpt || ' ' || post.content), to_tsquery('english', :query))`,
        'rank',
      )
      .orderBy('rank', 'DESC')
      .skip((page - 1) * limit)
      .take(limit);

    const [posts, total] = await qb.getManyAndCount();

    return { posts, total, query };
  }

  async suggest(query: string): Promise<string[]> {
    const results = await this.postRepository
      .createQueryBuilder('post')
      .select(['post.title'])
      .where('post.status = :status', { status: PostStatus.PUBLISHED })
      .andWhere('post.title ILIKE :query', { query: `%${query}%` })
      .limit(5)
      .getMany();

    return results.map((post) => post.title);
  }
}
```

### 2.10 Image Upload Service (S3)

```typescript
// backend/src/modules/uploads/uploads.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import {
  S3Client,
  PutObjectCommand,
  DeleteObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import * as crypto from 'crypto';
import * as path from 'path';

interface UploadResult {
  url: string;
  key: string;
  bucket: string;
}

@Injectable()
export class UploadsService {
  private readonly s3Client: S3Client;
  private readonly bucketName: string;

  constructor(private readonly configService: ConfigService) {
    this.s3Client = new S3Client({
      region: configService.get('AWS_REGION', 'ap-southeast-1'),
      credentials: {
        accessKeyId: configService.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: configService.get('AWS_SECRET_ACCESS_KEY'),
      },
    });
    this.bucketName = configService.get('AWS_S3_BUCKET');
  }

  async uploadFile(
    file: Express.Multer.File,
    folder = 'uploads',
  ): Promise<UploadResult> {
    const allowedMimeTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
    
    if (!allowedMimeTypes.includes(file.mimetype)) {
      throw new BadRequestException('ประเภทไฟล์ไม่ถูกต้อง');
    }

    const maxSize = 5 * 1024 * 1024; // 5MB
    if (file.size > maxSize) {
      throw new BadRequestException('ขนาดไฟล์ใหญ่เกินไป (สูงสุด 5MB)');
    }

    const fileExtension = path.extname(file.originalname);
    const fileName = `${crypto.randomBytes(16).toString('hex')}${fileExtension}`;
    const key = `${folder}/${fileName}`;

    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
      ACL: 'public-read',
      Metadata: {
        originalName: file.originalname,
      },
    });

    await this.s3Client.send(command);

    const url = `https://${this.bucketName}.s3.${this.configService.get('AWS_REGION')}.amazonaws.com/${key}`;

    return { url, key, bucket: this.bucketName };
  }

  async deleteFile(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    await this.s3Client.send(command);
  }

  async getPresignedUrl(key: string, expiresIn = 3600): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    return getSignedUrl(this.s3Client, command, { expiresIn });
  }
}
```

### 2.11 RSS Feed Service

```typescript
// backend/src/modules/rss/rss.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import * as RSS from 'rss';
import { Post, PostStatus } from '../posts/entities/post.entity';

@Injectable()
export class RssService {
  constructor(
    @InjectRepository(Post)
    private readonly postRepository: Repository<Post>,
    private readonly configService: ConfigService,
  ) {}

  async generateFeed(categorySlug?: string): Promise<string> {
    const siteUrl = this.configService.get('SITE_URL', 'https://myblog.com');
    const siteName = this.configService.get('SITE_NAME', 'My Blog');

    const feed = new RSS({
      title: siteName,
      description: `${siteName} - บทความล่าสุด`,
      feed_url: `${siteUrl}/rss${categorySlug ? `/${categorySlug}` : ''}`,
      site_url: siteUrl,
      language: 'th',
      pubDate: new Date().toUTCString(),
      ttl: 60,
    });

    const qb = this.postRepository
      .createQueryBuilder('post')
      .leftJoinAndSelect('post.author', 'author')
      .leftJoinAndSelect('post.category', 'category')
      .where('post.status = :status', { status: PostStatus.PUBLISHED });

    if (categorySlug) {
      qb.andWhere('category.slug = :categorySlug', { categorySlug });
    }

    const posts = await qb
      .orderBy('post.publishedAt', 'DESC')
      .take(20)
      .getMany();

    posts.forEach((post) => {
      feed.item({
        title: post.title,
        description: post.excerpt,
        url: `${siteUrl}/posts/${post.slug}`,
        author: post.author.displayName,
        date: post.publishedAt,
        categories: [post.category.name],
      });
    });

    return feed.xml({ indent: true });
  }
}
```

### 2.12 Likes Service

```typescript
// backend/src/modules/likes/likes.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Like } from './entities/like.entity';
import { Post } from '../posts/entities/post.entity';
import { Comment } from '../comments/entities/comment.entity';
import { User } from '../users/entities/user.entity';

interface LikeResult {
  liked: boolean;
  likesCount: number;
}

@Injectable()
export class LikesService {
  constructor(
    @InjectRepository(Like)
    private readonly likeRepository: Repository<Like>,
    @InjectRepository(Post)
    private readonly postRepository: Repository<Post>,
    @InjectRepository(Comment)
    private readonly commentRepository: Repository<Comment>,
  ) {}

  async togglePostLike(postId: string, user: User): Promise<LikeResult> {
    const existingLike = await this.likeRepository.findOne({
      where: { user: { id: user.id }, post: { id: postId } },
    });

    if (existingLike) {
      await this.likeRepository.remove(existingLike);
      await this.postRepository.decrement({ id: postId }, 'likesCount', 1);
      const post = await this.postRepository.findOne({ where: { id: postId } });
      return { liked: false, likesCount: post.likesCount };
    } else {
      const like = this.likeRepository.create({
        user,
        post: { id: postId } as Post,
      });
      await this.likeRepository.save(like);
      await this.postRepository.increment({ id: postId }, 'likesCount', 1);
      const post = await this.postRepository.findOne({ where: { id: postId } });
      return { liked: true, likesCount: post.likesCount };
    }
  }

  async hasUserLikedPost(postId: string, userId: string): Promise<boolean> {
    const like = await this.likeRepository.findOne({
      where: { user: { id: userId }, post: { id: postId } },
    });
    return !!like;
  }
}
```

### 2.13 Bookmarks Service

```typescript
// backend/src/modules/bookmarks/bookmarks.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Bookmark } from './entities/bookmark.entity';
import { Post } from '../posts/entities/post.entity';
import { User } from '../users/entities/user.entity';
import { PaginatedResponse } from '../../../shared/types';

@Injectable()
export class BookmarksService {
  constructor(
    @InjectRepository(Bookmark)
    private readonly bookmarkRepository: Repository<Bookmark>,
  ) {}

  async toggleBookmark(postId: string, user: User): Promise<boolean> {
    const existingBookmark = await this.bookmarkRepository.findOne({
      where: { user: { id: user.id }, post: { id: postId } },
    });

    if (existingBookmark) {
      await this.bookmarkRepository.remove(existingBookmark);
      return false;
    } else {
      const bookmark = this.bookmarkRepository.create({
        user,
        post: { id: postId } as Post,
      });
      await this.bookmarkRepository.save(bookmark);
      return true;
    }
  }

  async getUserBookmarks(
    userId: string,
    page = 1,
    limit = 10,
  ): Promise<PaginatedResponse<Post>> {
    const [bookmarks, total] = await this.bookmarkRepository.findAndCount({
      where: { user: { id: userId } },
      relations: ['post', 'post.author', 'post.category', 'post.tags'],
      skip: (page - 1) * limit,
      take: limit,
      order: { createdAt: 'DESC' },
    });

    const posts = bookmarks.map((b) => b.post);

    return {
      data: posts,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNextPage: page < Math.ceil(total / limit),
        hasPrevPage: page > 1,
      },
    };
  }
}
```

## ส่วนที่ 3: DTO Validation

### 3.1 Create Post DTO

```typescript
// backend/src/modules/posts/dto/create-post.dto.ts
import {
  IsString,
  IsNotEmpty,
  IsEnum,
  IsOptional,
  IsArray,
  MinLength,
  MaxLength,
  IsUUID,
} from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { PostStatus } from '../entities/post.entity';

export class CreatePostDto {
  @ApiProperty({ description: 'หัวข้อบทความ' })
  @IsString()
  @IsNotEmpty()
  @MinLength(5)
  @MaxLength(200)
  title: string;

  @ApiProperty({ description: 'เนื้อหาย่อ' })
  @IsString()
  @IsNotEmpty()
  @MaxLength(500)
  excerpt: string;

  @ApiProperty({ description: 'เนื้อหาบทความ (Markdown)' })
  @IsString()
  @IsNotEmpty()
  @MinLength(50)
  content: string;

  @ApiPropertyOptional({ description: 'URL รูปภาพหน้าปก' })
  @IsString()
  @IsOptional()
  coverImage?: string;

  @ApiProperty({ description: 'สถานะบทความ', enum: PostStatus })
  @IsEnum(PostStatus)
  status: PostStatus;

  @ApiProperty({ description: 'ID ของหมวดหมู่' })
  @IsUUID()
  categoryId: string;

  @ApiPropertyOptional({ description: 'รายชื่อแท็ก', type: [String] })
  @IsArray()
  @IsString({ each: true })
  @IsOptional()
  tags?: string[];

  @ApiPropertyOptional({ description: 'Meta Title สำหรับ SEO' })
  @IsString()
  @IsOptional()
  @MaxLength(60)
  metaTitle?: string;

  @ApiPropertyOptional({ description: 'Meta Description สำหรับ SEO' })
  @IsString()
  @IsOptional()
  @MaxLength(160)
  metaDescription?: string;
}
```

### 3.2 Guard และ Decorator

```typescript
// backend/src/common/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    return super.canActivate(context);
  }
}

// backend/src/common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';
import { UserRole } from '../../modules/users/entities/user.entity';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.role === role);
  }
}

// backend/src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { UserRole } from '../../modules/users/entities/user.entity';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: UserRole[]) => SetMetadata(ROLES_KEY, roles);

// backend/src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);
```

## ส่วนที่ 4: Frontend - Next.js Implementation

### 4.1 API Client Configuration

```typescript
// frontend/lib/api.ts
import axios, { AxiosError, AxiosInstance } from 'axios';
import { getSession } from 'next-auth/react';

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

export const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor - เพิ่ม auth token
apiClient.interceptors.request.use(async (config) => {
  const session = await getSession();
  if (session?.accessToken) {
    config.headers.Authorization = `Bearer ${session.accessToken}`;
  }
  return config;
});

// Response interceptor - จัดการ errors
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    if (error.response?.status === 401) {
      // Handle token refresh หรือ redirect to login
      window.location.href = '/auth/login';
    }
    return Promise.reject(error);
  },
);

// Type-safe API functions
export const postsApi = {
  getAll: (params?: Record<string, unknown>) =>
    apiClient.get('/posts', { params }),

  getBySlug: (slug: string) =>
    apiClient.get(`/posts/${slug}`),

  create: (data: FormData | Record<string, unknown>) =>
    apiClient.post('/posts', data),

  update: (id: string, data: Partial<Record<string, unknown>>) =>
    apiClient.patch(`/posts/${id}`, data),

  delete: (id: string) =>
    apiClient.delete(`/posts/${id}`),

  like: (id: string) =>
    apiClient.post(`/likes/posts/${id}`),

  bookmark: (id: string) =>
    apiClient.post(`/bookmarks/posts/${id}`),
};

export const searchApi = {
  search: (query: string, page?: number) =>
    apiClient.get('/search', { params: { query, page } }),

  suggest: (query: string) =>
    apiClient.get('/search/suggest', { params: { query } }),
};
```

### 4.2 Post List Component

```tsx
// frontend/components/PostCard.tsx
import Link from 'next/link';
import Image from 'next/image';
import { formatDistanceToNow } from 'date-fns';
import { th } from 'date-fns/locale';
import { Heart, MessageCircle, Bookmark, Clock } from 'lucide-react';
import type { Post } from '@/types';

interface PostCardProps {
  post: Post;
  onLike?: (postId: string) => void;
  onBookmark?: (postId: string) => void;
  isLiked?: boolean;
  isBookmarked?: boolean;
}

export function PostCard({
  post,
  onLike,
  onBookmark,
  isLiked = false,
  isBookmarked = false,
}: PostCardProps) {
  return (
    <article className="bg-white rounded-xl shadow-sm hover:shadow-md transition-shadow duration-200 overflow-hidden">
      {post.coverImage && (
        <Link href={`/posts/${post.slug}`}>
          <div className="relative h-48 overflow-hidden">
            <Image
              src={post.coverImage}
              alt={post.title}
              fill
              className="object-cover hover:scale-105 transition-transform duration-300"
            />
          </div>
        </Link>
      )}

      <div className="p-5">
        {/* Category */}
        <Link
          href={`/categories/${post.category.slug}`}
          className="text-xs font-semibold text-blue-600 uppercase tracking-wide hover:text-blue-800"
        >
          {post.category.name}
        </Link>

        {/* Title */}
        <h2 className="mt-2 text-xl font-bold text-gray-900 hover:text-blue-600">
          <Link href={`/posts/${post.slug}`}>{post.title}</Link>
        </h2>

        {/* Excerpt */}
        <p className="mt-2 text-gray-600 text-sm line-clamp-3">{post.excerpt}</p>

        {/* Tags */}
        <div className="mt-3 flex flex-wrap gap-1">
          {post.tags.slice(0, 3).map((tag) => (
            <Link
              key={tag.id}
              href={`/tags/${tag.slug}`}
              className="px-2 py-1 bg-gray-100 text-gray-600 text-xs rounded-full hover:bg-gray-200"
            >
              #{tag.name}
            </Link>
          ))}
        </div>

        {/* Meta */}
        <div className="mt-4 flex items-center justify-between">
          <div className="flex items-center gap-2">
            <Image
              src={post.author.avatar || '/default-avatar.png'}
              alt={post.author.displayName}
              width={24}
              height={24}
              className="rounded-full"
            />
            <span className="text-sm text-gray-600">{post.author.displayName}</span>
            <span className="text-gray-400">·</span>
            <span className="text-sm text-gray-500">
              {formatDistanceToNow(new Date(post.publishedAt), {
                addSuffix: true,
                locale: th,
              })}
            </span>
          </div>

          <div className="flex items-center gap-1 text-gray-500">
            <Clock size={14} />
            <span className="text-xs">{post.readingTime} นาที</span>
          </div>
        </div>

        {/* Actions */}
        <div className="mt-3 flex items-center gap-4 border-t pt-3">
          <button
            onClick={() => onLike?.(post.id)}
            className={`flex items-center gap-1 text-sm transition-colors ${
              isLiked ? 'text-red-500' : 'text-gray-500 hover:text-red-500'
            }`}
          >
            <Heart size={16} fill={isLiked ? 'currentColor' : 'none'} />
            <span>{post.likesCount}</span>
          </button>

          <Link
            href={`/posts/${post.slug}#comments`}
            className="flex items-center gap-1 text-sm text-gray-500 hover:text-blue-500"
          >
            <MessageCircle size={16} />
            <span>{post.commentsCount}</span>
          </Link>

          <button
            onClick={() => onBookmark?.(post.id)}
            className={`flex items-center gap-1 text-sm ml-auto transition-colors ${
              isBookmarked ? 'text-yellow-500' : 'text-gray-500 hover:text-yellow-500'
            }`}
          >
            <Bookmark size={16} fill={isBookmarked ? 'currentColor' : 'none'} />
          </button>
        </div>
      </div>
    </article>
  );
}
```

### 4.3 Post Editor Component

```tsx
// frontend/components/PostEditor.tsx
'use client';

import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postsApi } from '@/lib/api';
import type { CreatePostInput } from '@/types';

const postSchema = z.object({
  title: z.string().min(5, 'หัวข้อต้องมีอย่างน้อย 5 ตัวอักษร').max(200),
  excerpt: z.string().min(10, 'เนื้อหาย่อต้องมีอย่างน้อย 10 ตัวอักษร').max(500),
  content: z.string().min(50, 'เนื้อหาต้องมีอย่างน้อย 50 ตัวอักษร'),
  categoryId: z.string().uuid('กรุณาเลือกหมวดหมู่'),
  tags: z.array(z.string()).optional(),
  status: z.enum(['draft', 'published']),
  metaTitle: z.string().max(60).optional(),
  metaDescription: z.string().max(160).optional(),
});

type PostFormData = z.infer<typeof postSchema>;

interface PostEditorProps {
  initialData?: Partial<PostFormData>;
  postId?: string;
  onSuccess?: () => void;
}

export function PostEditor({ initialData, postId, onSuccess }: PostEditorProps) {
  const queryClient = useQueryClient();

  const {
    register,
    control,
    handleSubmit,
    setValue,
    formState: { errors },
  } = useForm<PostFormData>({
    resolver: zodResolver(postSchema),
    defaultValues: {
      status: 'draft',
      tags: [],
      ...initialData,
    },
  });

  const editor = useEditor({
    extensions: [StarterKit],
    content: initialData?.content || '',
    onUpdate: ({ editor }) => {
      setValue('content', editor.getText());
    },
  });

  const createMutation = useMutation({
    mutationFn: (data: PostFormData) => postsApi.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
      onSuccess?.();
    },
  });

  const updateMutation = useMutation({
    mutationFn: (data: PostFormData) => postsApi.update(postId!, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
      queryClient.invalidateQueries({ queryKey: ['post', postId] });
      onSuccess?.();
    },
  });

  const onSubmit = (data: PostFormData) => {
    if (postId) {
      updateMutation.mutate(data);
    } else {
      createMutation.mutate(data);
    }
  };

  const isPending = createMutation.isPending || updateMutation.isPending;

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      {/* Title */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          หัวข้อบทความ
        </label>
        <input
          {...register('title')}
          type="text"
          className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          placeholder="หัวข้อบทความ..."
        />
        {errors.title && (
          <p className="mt-1 text-sm text-red-600">{errors.title.message}</p>
        )}
      </div>

      {/* Excerpt */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          เนื้อหาย่อ
        </label>
        <textarea
          {...register('excerpt')}
          rows={3}
          className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          placeholder="สรุปเนื้อหาบทความ..."
        />
        {errors.excerpt && (
          <p className="mt-1 text-sm text-red-600">{errors.excerpt.message}</p>
        )}
      </div>

      {/* Content Editor */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          เนื้อหาบทความ
        </label>
        <div className="border border-gray-300 rounded-lg overflow-hidden">
          <EditorContent
            editor={editor}
            className="min-h-[400px] p-4 prose max-w-none"
          />
        </div>
        {errors.content && (
          <p className="mt-1 text-sm text-red-600">{errors.content.message}</p>
        )}
      </div>

      {/* Actions */}
      <div className="flex gap-3">
        <button
          type="submit"
          onClick={() => setValue('status', 'draft')}
          disabled={isPending}
          className="px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700 disabled:opacity-50"
        >
          {isPending ? 'กำลังบันทึก...' : 'บันทึกแบบร่าง'}
        </button>
        <button
          type="submit"
          onClick={() => setValue('status', 'published')}
          disabled={isPending}
          className="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
        >
          {isPending ? 'กำลังเผยแพร่...' : 'เผยแพร่บทความ'}
        </button>
      </div>
    </form>
  );
}
```

### 4.4 Search Component

```tsx
// frontend/components/SearchBar.tsx
'use client';

import { useState, useCallback, useRef, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { Search, X } from 'lucide-react';
import { searchApi } from '@/lib/api';
import { useDebounce } from '@/hooks/useDebounce';

export function SearchBar() {
  const router = useRouter();
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [isOpen, setIsOpen] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery.length >= 2) {
      setIsLoading(true);
      searchApi
        .suggest(debouncedQuery)
        .then((res) => {
          setSuggestions(res.data);
          setIsOpen(true);
        })
        .finally(() => setIsLoading(false));
    } else {
      setSuggestions([]);
      setIsOpen(false);
    }
  }, [debouncedQuery]);

  const handleSearch = useCallback(
    (searchQuery: string) => {
      if (searchQuery.trim()) {
        router.push(`/search?q=${encodeURIComponent(searchQuery.trim())}`);
        setIsOpen(false);
        setQuery('');
      }
    },
    [router],
  );

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      handleSearch(query);
    }
    if (e.key === 'Escape') {
      setIsOpen(false);
    }
  };

  return (
    <div className="relative">
      <div className="flex items-center bg-gray-100 rounded-full px-4 py-2 gap-2">
        <Search size={18} className="text-gray-500" />
        <input
          ref={inputRef}
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyDown={handleKeyDown}
          type="text"
          placeholder="ค้นหาบทความ..."
          className="bg-transparent outline-none text-sm flex-1"
        />
        {query && (
          <button onClick={() => setQuery('')}>
            <X size={16} className="text-gray-500 hover:text-gray-700" />
          </button>
        )}
      </div>

      {isOpen && suggestions.length > 0 && (
        <div className="absolute top-full mt-1 left-0 right-0 bg-white rounded-lg shadow-lg border z-50">
          {suggestions.map((suggestion, index) => (
            <button
              key={index}
              onClick={() => handleSearch(suggestion)}
              className="w-full px-4 py-2 text-left text-sm hover:bg-gray-50 flex items-center gap-2"
            >
              <Search size={14} className="text-gray-400" />
              {suggestion}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

### 4.5 SEO Component

```tsx
// frontend/components/SEO.tsx
import Head from 'next/head';

interface SEOProps {
  title: string;
  description: string;
  image?: string;
  url?: string;
  type?: 'website' | 'article';
  publishedAt?: string;
  author?: string;
}

export function SEO({
  title,
  description,
  image,
  url,
  type = 'website',
  publishedAt,
  author,
}: SEOProps) {
  const siteUrl = process.env.NEXT_PUBLIC_SITE_URL || 'https://myblog.com';
  const fullUrl = url ? `${siteUrl}${url}` : siteUrl;
  const fullTitle = `${title} | My Blog`;

  return (
    <Head>
      <title>{fullTitle}</title>
      <meta name="description" content={description} />

      {/* Open Graph */}
      <meta property="og:title" content={fullTitle} />
      <meta property="og:description" content={description} />
      <meta property="og:url" content={fullUrl} />
      <meta property="og:type" content={type} />
      {image && <meta property="og:image" content={image} />}

      {/* Twitter */}
      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={fullTitle} />
      <meta name="twitter:description" content={description} />
      {image && <meta name="twitter:image" content={image} />}

      {/* Article */}
      {type === 'article' && publishedAt && (
        <meta property="article:published_time" content={publishedAt} />
      )}
      {type === 'article' && author && (
        <meta property="article:author" content={author} />
      )}

      {/* Canonical */}
      <link rel="canonical" href={fullUrl} />
    </Head>
  );
}
```

### 4.6 Custom Hooks

```typescript
// frontend/hooks/usePosts.ts
import { useQuery, useMutation, useQueryClient, useInfiniteQuery } from '@tanstack/react-query';
import { postsApi } from '@/lib/api';
import type { Post, PaginatedResponse } from '@/types';

export const postKeys = {
  all: ['posts'] as const,
  lists: () => [...postKeys.all, 'list'] as const,
  list: (params: Record<string, unknown>) => [...postKeys.lists(), params] as const,
  details: () => [...postKeys.all, 'detail'] as const,
  detail: (slug: string) => [...postKeys.details(), slug] as const,
};

export function usePosts(params?: Record<string, unknown>) {
  return useQuery({
    queryKey: postKeys.list(params || {}),
    queryFn: () => postsApi.getAll(params).then((res) => res.data as PaginatedResponse<Post>),
  });
}

export function usePost(slug: string) {
  return useQuery({
    queryKey: postKeys.detail(slug),
    queryFn: () => postsApi.getBySlug(slug).then((res) => res.data as Post),
    enabled: !!slug,
  });
}

export function useInfinitePosts(params?: Record<string, unknown>) {
  return useInfiniteQuery({
    queryKey: [...postKeys.lists(), 'infinite', params],
    queryFn: ({ pageParam = 1 }) =>
      postsApi.getAll({ ...params, page: pageParam }).then((res) => res.data as PaginatedResponse<Post>),
    initialPageParam: 1,
    getNextPageParam: (lastPage) =>
      lastPage.meta.hasNextPage ? lastPage.meta.page + 1 : undefined,
  });
}

export function useToggleLike() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (postId: string) => postsApi.like(postId).then((res) => res.data),
    onSuccess: (data, postId) => {
      queryClient.invalidateQueries({ queryKey: postKeys.all });
    },
  });
}

export function useToggleBookmark() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (postId: string) => postsApi.bookmark(postId).then((res) => res.data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.all });
      queryClient.invalidateQueries({ queryKey: ['bookmarks'] });
    },
  });
}
```

### 4.7 Blog Post Page

```tsx
// frontend/app/posts/[slug]/page.tsx
import { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { apiClient } from '@/lib/api';
import { PostDetail } from '@/components/PostDetail';
import { Comments } from '@/components/Comments';
import { RelatedPosts } from '@/components/RelatedPosts';
import type { Post } from '@/types';

interface Props {
  params: { slug: string };
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  try {
    const { data: post } = await apiClient.get<Post>(`/posts/${params.slug}`);

    return {
      title: post.metaTitle || post.title,
      description: post.metaDescription || post.excerpt,
      openGraph: {
        title: post.title,
        description: post.excerpt,
        images: post.coverImage ? [{ url: post.coverImage }] : [],
        type: 'article',
        publishedTime: post.publishedAt?.toString(),
        authors: [post.author.displayName],
      },
    };
  } catch {
    return { title: 'ไม่พบบทความ' };
  }
}

export default async function PostPage({ params }: Props) {
  let post: Post;

  try {
    const { data } = await apiClient.get<Post>(`/posts/${params.slug}`);
    post = data;
  } catch {
    notFound();
  }

  return (
    <main className="max-w-4xl mx-auto px-4 py-8">
      <PostDetail post={post} />
      <Comments postId={post.id} />
      <RelatedPosts
        categorySlug={post.category.slug}
        excludeSlug={post.slug}
      />
    </main>
  );
}
```

### 4.8 RSS Feed Route Handler

```typescript
// frontend/app/rss/route.ts
import { NextResponse } from 'next/server';
import { apiClient } from '@/lib/api';

export async function GET() {
  try {
    const { data: rssXml } = await apiClient.get('/rss');

    return new NextResponse(rssXml, {
      headers: {
        'Content-Type': 'application/xml; charset=utf-8',
        'Cache-Control': 'public, max-age=3600, s-maxage=3600',
      },
    });
  } catch {
    return new NextResponse('Error generating RSS feed', { status: 500 });
  }
}
```

## ส่วนที่ 5: Database Migration

### 5.1 Initial Migration

```typescript
// backend/src/database/migrations/001-initial-schema.ts
import { MigrationInterface, QueryRunner } from 'typeorm';

export class InitialSchema001 implements MigrationInterface {
  name = 'InitialSchema001';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // Users table
    await queryRunner.query(`
      CREATE TYPE "user_role_enum" AS ENUM ('admin', 'author', 'reader');
      
      CREATE TABLE "users" (
        "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
        "email" character varying NOT NULL,
        "username" character varying NOT NULL,
        "displayName" character varying NOT NULL,
        "password" character varying NOT NULL,
        "bio" text,
        "avatar" character varying,
        "role" "user_role_enum" NOT NULL DEFAULT 'reader',
        "isActive" boolean NOT NULL DEFAULT true,
        "refreshToken" character varying,
        "createdAt" TIMESTAMP NOT NULL DEFAULT now(),
        "updatedAt" TIMESTAMP NOT NULL DEFAULT now(),
        CONSTRAINT "UQ_users_email" UNIQUE ("email"),
        CONSTRAINT "UQ_users_username" UNIQUE ("username"),
        CONSTRAINT "PK_users" PRIMARY KEY ("id")
      );
    `);

    // Full-text search index
    await queryRunner.query(`
      CREATE INDEX "IDX_posts_fts" ON "posts" 
      USING GIN (to_tsvector('english', title || ' ' || excerpt || ' ' || content));
    `);

    // Slug indexes
    await queryRunner.query(`
      CREATE UNIQUE INDEX "IDX_posts_slug" ON "posts" ("slug");
      CREATE UNIQUE INDEX "IDX_categories_slug" ON "categories" ("slug");
      CREATE UNIQUE INDEX "IDX_tags_slug" ON "tags" ("slug");
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE IF EXISTS "users"`);
    await queryRunner.query(`DROP TYPE IF EXISTS "user_role_enum"`);
  }
}
```

## ส่วนที่ 6: Testing

### 6.1 Unit Tests - Posts Service

```typescript
// backend/test/posts.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { PostsService } from '../src/modules/posts/posts.service';
import { Post, PostStatus } from '../src/modules/posts/entities/post.entity';
import { Tag } from '../src/modules/tags/entities/tag.entity';
import { Category } from '../src/modules/categories/entities/category.entity';
import { NotFoundException } from '@nestjs/common';

const mockPostRepository = () => ({
  create: jest.fn(),
  save: jest.fn(),
  findOne: jest.fn(),
  findAndCount: jest.fn(),
  createQueryBuilder: jest.fn(),
  increment: jest.fn(),
  softDelete: jest.fn(),
});

describe('PostsService', () => {
  let service: PostsService;
  let postRepository: jest.Mocked<Repository<Post>>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        PostsService,
        {
          provide: getRepositoryToken(Post),
          useFactory: mockPostRepository,
        },
        {
          provide: getRepositoryToken(Tag),
          useFactory: mockPostRepository,
        },
        {
          provide: getRepositoryToken(Category),
          useFactory: mockPostRepository,
        },
      ],
    }).compile();

    service = module.get<PostsService>(PostsService);
    postRepository = module.get(getRepositoryToken(Post));
  });

  describe('findBySlug', () => {
    it('should return post when found', async () => {
      const mockPost = {
        id: '1',
        slug: 'test-post',
        status: PostStatus.PUBLISHED,
        viewsCount: 0,
      } as Post;

      postRepository.findOne.mockResolvedValue(mockPost);
      postRepository.increment.mockResolvedValue(undefined);

      const result = await service.findBySlug('test-post');

      expect(result).toBeDefined();
      expect(result.viewsCount).toBe(1);
    });

    it('should throw NotFoundException when post not found', async () => {
      postRepository.findOne.mockResolvedValue(null);

      await expect(service.findBySlug('nonexistent')).rejects.toThrow(
        NotFoundException,
      );
    });
  });
});
```

### 6.2 E2E Tests

```typescript
// backend/test/posts.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Posts (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    await app.init();

    // Get auth token
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });

    authToken = loginRes.body.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  describe('GET /posts', () => {
    it('should return paginated posts', async () => {
      const res = await request(app.getHttpServer())
        .get('/posts')
        .expect(200);

      expect(res.body.data).toBeInstanceOf(Array);
      expect(res.body.meta).toBeDefined();
      expect(res.body.meta.page).toBe(1);
    });
  });

  describe('POST /posts', () => {
    it('should create post when authenticated', async () => {
      const postData = {
        title: 'บทความทดสอบ TypeScript',
        excerpt: 'นี่คือบทความทดสอบสำหรับ TypeScript',
        content: 'เนื้อหาบทความทดสอบที่มีความยาวมากกว่า 50 ตัวอักษร...',
        categoryId: 'category-uuid',
        status: 'draft',
      };

      const res = await request(app.getHttpServer())
        .post('/posts')
        .set('Authorization', `Bearer ${authToken}`)
        .send(postData)
        .expect(201);

      expect(res.body.title).toBe(postData.title);
      expect(res.body.slug).toBeDefined();
    });
  });
});
```

## ส่วนที่ 7: Docker Configuration

### 7.1 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: blog_user
      POSTGRES_PASSWORD: blog_password
      POSTGRES_DB: blog_platform
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U blog_user -d blog_platform"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "3001:3001"
    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USERNAME: blog_user
      DB_PASSWORD: blog_password
      DB_NAME: blog_platform
      JWT_SECRET: ${JWT_SECRET}
      JWT_REFRESH_SECRET: ${JWT_REFRESH_SECRET}
      AWS_REGION: ${AWS_REGION}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_S3_BUCKET: ${AWS_S3_BUCKET}
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://backend:3001
      NEXTAUTH_URL: ${NEXTAUTH_URL}
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 7.2 Backend Dockerfile

```dockerfile
# backend/Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist

EXPOSE 3001
CMD ["node", "dist/main"]
```

## สรุป

Blog Platform ที่สร้างในบทนี้มีฟีเจอร์ครบครัน:

1. **Backend (NestJS)**:
   - ระบบ Authentication ด้วย JWT
   - CRUD สำหรับ Posts, Categories, Tags, Comments
   - ระบบ Likes และ Bookmarks
   - Full-text Search ด้วย PostgreSQL
   - Image Upload ไป AWS S3
   - RSS Feed Generation
   - SEO-friendly slugs
   - Markdown processing พร้อม sanitization
   - Role-based Access Control

2. **Frontend (Next.js)**:
   - Server-side Rendering สำหรับ SEO
   - React Query สำหรับ State Management
   - Rich Text Editor ด้วย TipTap
   - Infinite Scroll
   - Real-time Search Suggestions
   - Responsive Design ด้วย Tailwind CSS

3. **DevOps**:
   - Docker Compose สำหรับ Development
   - Database Migrations
   - E2E Testing ด้วย Supertest
