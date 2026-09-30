# ตอนที่ 40: GraphQL กับ TypeScript

## บทนำ

GraphQL คือ query language สำหรับ API ที่พัฒนาโดย Facebook ซึ่งช่วยให้ client สามารถระบุได้อย่างชัดเจนว่าต้องการข้อมูลอะไร TypeScript และ GraphQL ทำงานร่วมกันได้อย่างยอดเยี่ยม เนื่องจากทั้งคู่มีระบบ type ที่แข็งแกร่ง

---

## 40.1 พื้นฐาน GraphQL กับ TypeScript

### การติดตั้ง

```bash
npm install graphql apollo-server-express express
npm install -D typescript @types/node @types/express ts-node nodemon
```

### โครงสร้าง GraphQL Schema เบื้องต้น

```typescript
// src/schema/types.ts
import { gql } from 'apollo-server-express';

export const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    age: Int
    posts: [Post!]!
    createdAt: String!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    published: Boolean!
    author: User!
    tags: [String!]!
    createdAt: String!
  }

  type Query {
    user(id: ID!): User
    users: [User!]!
    post(id: ID!): Post
    posts(published: Boolean): [Post!]!
  }

  type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User
    deleteUser(id: ID!): Boolean!
  }

  input CreateUserInput {
    name: String!
    email: String!
    age: Int
  }

  input UpdateUserInput {
    name: String
    email: String
    age: Int
  }
`;
```

### TypeScript Interfaces สำหรับ GraphQL

```typescript
// src/types/index.ts

// User types
export interface User {
  id: string;
  name: string;
  email: string;
  age?: number;
  posts: Post[];
  createdAt: Date;
}

export interface CreateUserInput {
  name: string;
  email: string;
  age?: number;
}

export interface UpdateUserInput {
  name?: string;
  email?: string;
  age?: number;
}

// Post types
export interface Post {
  id: string;
  title: string;
  content: string;
  published: boolean;
  authorId: string;
  author: User;
  tags: string[];
  createdAt: Date;
}

export interface CreatePostInput {
  title: string;
  content: string;
  authorId: string;
  tags?: string[];
}

// Resolver types
export interface ResolverContext {
  userId?: string;
  db: Database;
  loaders: DataLoaders;
}

export interface Database {
  users: Map<string, User>;
  posts: Map<string, Post>;
}

export interface DataLoaders {
  userLoader: DataLoader<string, User>;
  postsByUserLoader: DataLoader<string, Post[]>;
}
```

---

## 40.2 Apollo Server กับ TypeScript

### การตั้งค่า Apollo Server

```typescript
// src/server.ts
import express from 'express';
import { ApolloServer } from 'apollo-server-express';
import { typeDefs } from './schema/types';
import { resolvers } from './resolvers';
import { createContext } from './context';

async function startServer(): Promise<void> {
  const app = express();

  const server = new ApolloServer({
    typeDefs,
    resolvers,
    context: createContext,
    formatError: (error) => {
      console.error('GraphQL Error:', error);
      return {
        message: error.message,
        code: error.extensions?.code,
        path: error.path,
      };
    },
    plugins: [
      {
        requestDidStart() {
          return {
            didResolveOperation({ request, document }) {
              console.log(`Operation: ${request.operationName}`);
            },
          };
        },
      },
    ],
  });

  await server.start();
  server.applyMiddleware({ app, path: '/graphql' });

  const PORT = process.env.PORT || 4000;
  app.listen(PORT, () => {
    console.log(`🚀 Server ready at http://localhost:${PORT}${server.graphqlPath}`);
  });
}

startServer().catch(console.error);
```

### Context การตั้งค่า

```typescript
// src/context.ts
import { Request, Response } from 'express';
import { verify } from 'jsonwebtoken';
import { ResolverContext, Database } from './types';
import { createDataLoaders } from './dataloaders';

// In-memory database simulation
const db: Database = {
  users: new Map([
    ['1', {
      id: '1',
      name: 'สมชาย ใจดี',
      email: 'somchai@example.com',
      age: 30,
      posts: [],
      createdAt: new Date('2024-01-01'),
    }],
    ['2', {
      id: '2',
      name: 'สมหญิง รักเรียน',
      email: 'somying@example.com',
      age: 25,
      posts: [],
      createdAt: new Date('2024-01-15'),
    }],
  ]),
  posts: new Map([
    ['1', {
      id: '1',
      title: 'เรียน TypeScript ให้ได้ผล',
      content: 'TypeScript เป็นภาษาที่ยอดเยี่ยม...',
      published: true,
      authorId: '1',
      author: null as any,
      tags: ['typescript', 'programming'],
      createdAt: new Date('2024-02-01'),
    }],
  ]),
};

export async function createContext({
  req,
}: {
  req: Request;
  res: Response;
}): Promise<ResolverContext> {
  // ดึง token จาก header
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  let userId: string | undefined;
  
  if (token) {
    try {
      const decoded = verify(token, process.env.JWT_SECRET || 'secret') as {
        userId: string;
      };
      userId = decoded.userId;
    } catch (error) {
      // Token ไม่ถูกต้อง - ไม่ต้อง throw error ที่นี่
    }
  }

  return {
    userId,
    db,
    loaders: createDataLoaders(db),
  };
}
```

---

## 40.3 Schema-First Approach

### การออกแบบ Schema แบบ Schema-First

```typescript
// src/schema/user.schema.ts
import { gql } from 'apollo-server-express';

export const userTypeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    age: Int
    role: UserRole!
    posts: [Post!]!
    profile: UserProfile
    createdAt: String!
    updatedAt: String!
  }

  type UserProfile {
    bio: String
    avatar: String
    website: String
    social: SocialLinks
  }

  type SocialLinks {
    twitter: String
    github: String
    linkedin: String
  }

  enum UserRole {
    ADMIN
    EDITOR
    VIEWER
  }

  type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type UserEdge {
    node: User!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  extend type Query {
    user(id: ID!): User
    users(
      first: Int
      after: String
      last: Int
      before: String
      filter: UserFilter
      orderBy: UserOrderBy
    ): UserConnection!
    me: User
  }

  extend type Mutation {
    createUser(input: CreateUserInput!): CreateUserPayload!
    updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
    deleteUser(id: ID!): DeleteUserPayload!
    changePassword(oldPassword: String!, newPassword: String!): Boolean!
  }

  input CreateUserInput {
    name: String!
    email: String!
    password: String!
    role: UserRole = VIEWER
  }

  input UpdateUserInput {
    name: String
    email: String
    age: Int
    profile: UpdateProfileInput
  }

  input UpdateProfileInput {
    bio: String
    avatar: String
    website: String
  }

  input UserFilter {
    name: String
    email: String
    role: UserRole
    ageMin: Int
    ageMax: Int
  }

  input UserOrderBy {
    field: UserOrderField!
    direction: OrderDirection!
  }

  enum UserOrderField {
    NAME
    EMAIL
    CREATED_AT
  }

  enum OrderDirection {
    ASC
    DESC
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
    success: Boolean!
    errors: [UserError!]!
  }

  type UserError {
    field: String
    message: String!
    code: ErrorCode!
  }

  enum ErrorCode {
    NOT_FOUND
    VALIDATION_ERROR
    PERMISSION_DENIED
    DUPLICATE_EMAIL
  }
`;
```

### Resolvers สำหรับ Schema-First

```typescript
// src/resolvers/user.resolver.ts
import { IResolvers } from '@graphql-tools/utils';
import { ResolverContext, User, UserRole } from '../types';
import { AuthenticationError, UserInputError } from 'apollo-server-express';
import { hash } from 'bcryptjs';
import { v4 as uuidv4 } from 'uuid';

export const userResolvers: IResolvers<any, ResolverContext> = {
  Query: {
    user: async (_, { id }, { db }) => {
      const user = db.users.get(id);
      if (!user) return null;
      return user;
    },

    users: async (_, { first = 10, after, filter, orderBy }, { db }) => {
      let users = Array.from(db.users.values());

      // Apply filters
      if (filter) {
        if (filter.name) {
          users = users.filter(u => 
            u.name.toLowerCase().includes(filter.name.toLowerCase())
          );
        }
        if (filter.email) {
          users = users.filter(u => 
            u.email.toLowerCase().includes(filter.email.toLowerCase())
          );
        }
        if (filter.role) {
          users = users.filter(u => u.role === filter.role);
        }
      }

      // Apply ordering
      if (orderBy) {
        users.sort((a, b) => {
          const aVal = a[orderBy.field.toLowerCase() as keyof User];
          const bVal = b[orderBy.field.toLowerCase() as keyof User];
          const comparison = String(aVal).localeCompare(String(bVal));
          return orderBy.direction === 'ASC' ? comparison : -comparison;
        });
      }

      // Pagination
      const totalCount = users.length;
      let startIndex = 0;

      if (after) {
        const afterIndex = users.findIndex(u => u.id === after);
        if (afterIndex !== -1) startIndex = afterIndex + 1;
      }

      const paginatedUsers = users.slice(startIndex, startIndex + first);

      return {
        edges: paginatedUsers.map(user => ({
          node: user,
          cursor: user.id,
        })),
        pageInfo: {
          hasNextPage: startIndex + first < totalCount,
          hasPreviousPage: startIndex > 0,
          startCursor: paginatedUsers[0]?.id,
          endCursor: paginatedUsers[paginatedUsers.length - 1]?.id,
        },
        totalCount,
      };
    },

    me: async (_, __, { userId, db }) => {
      if (!userId) throw new AuthenticationError('กรุณาเข้าสู่ระบบ');
      return db.users.get(userId) || null;
    },
  },

  Mutation: {
    createUser: async (_, { input }, { db }) => {
      const errors: any[] = [];

      // Validate email uniqueness
      const existingUser = Array.from(db.users.values()).find(
        u => u.email === input.email
      );

      if (existingUser) {
        errors.push({
          field: 'email',
          message: 'อีเมลนี้ถูกใช้งานแล้ว',
          code: 'DUPLICATE_EMAIL',
        });
        return { user: null, errors };
      }

      // Validate password
      if (input.password.length < 8) {
        errors.push({
          field: 'password',
          message: 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
          code: 'VALIDATION_ERROR',
        });
        return { user: null, errors };
      }

      const hashedPassword = await hash(input.password, 12);
      const newUser: User = {
        id: uuidv4(),
        name: input.name,
        email: input.email,
        role: input.role || UserRole.VIEWER,
        posts: [],
        createdAt: new Date(),
        updatedAt: new Date(),
      };

      db.users.set(newUser.id, newUser);
      return { user: newUser, errors: [] };
    },

    updateUser: async (_, { id, input }, { userId, db }) => {
      if (!userId) throw new AuthenticationError('กรุณาเข้าสู่ระบบ');

      const user = db.users.get(id);
      if (!user) {
        return {
          user: null,
          errors: [{
            field: 'id',
            message: 'ไม่พบผู้ใช้',
            code: 'NOT_FOUND',
          }],
        };
      }

      // ตรวจสอบสิทธิ์
      if (userId !== id) {
        const currentUser = db.users.get(userId);
        if (currentUser?.role !== UserRole.ADMIN) {
          return {
            user: null,
            errors: [{
              message: 'ไม่มีสิทธิ์แก้ไขข้อมูลผู้ใช้อื่น',
              code: 'PERMISSION_DENIED',
            }],
          };
        }
      }

      const updatedUser: User = {
        ...user,
        ...input,
        updatedAt: new Date(),
      };

      db.users.set(id, updatedUser);
      return { user: updatedUser, errors: [] };
    },

    deleteUser: async (_, { id }, { userId, db }) => {
      if (!userId) throw new AuthenticationError('กรุณาเข้าสู่ระบบ');

      const currentUser = db.users.get(userId);
      if (currentUser?.role !== UserRole.ADMIN) {
        return {
          success: false,
          errors: [{
            message: 'เฉพาะผู้ดูแลระบบเท่านั้นที่สามารถลบผู้ใช้ได้',
            code: 'PERMISSION_DENIED',
          }],
        };
      }

      const deleted = db.users.delete(id);
      return { success: deleted, errors: [] };
    },
  },

  User: {
    posts: async (parent, _, { db }) => {
      return Array.from(db.posts.values()).filter(
        post => post.authorId === parent.id
      );
    },
  },
};
```

---

## 40.4 Code-First Approach กับ TypeGraphQL

### การติดตั้ง TypeGraphQL

```bash
npm install type-graphql reflect-metadata class-validator
npm install -D @types/node
```

### Entity Definitions

```typescript
// src/entities/User.ts
import {
  ObjectType,
  Field,
  ID,
  Int,
  registerEnumType,
} from 'type-graphql';
import { Post } from './Post';

export enum UserRole {
  ADMIN = 'ADMIN',
  EDITOR = 'EDITOR',
  VIEWER = 'VIEWER',
}

registerEnumType(UserRole, {
  name: 'UserRole',
  description: 'บทบาทของผู้ใช้ในระบบ',
});

@ObjectType({ description: 'ข้อมูลผู้ใช้' })
export class User {
  @Field(() => ID)
  id!: string;

  @Field(() => String, { description: 'ชื่อผู้ใช้' })
  name!: string;

  @Field(() => String, { description: 'อีเมล' })
  email!: string;

  @Field(() => Int, { nullable: true, description: 'อายุ' })
  age?: number;

  @Field(() => UserRole)
  role!: UserRole;

  @Field(() => [Post], { description: 'โพสต์ของผู้ใช้' })
  posts!: Post[];

  @Field(() => String)
  createdAt!: Date;

  // ไม่ expose password ใน GraphQL
  password!: string;
}
```

```typescript
// src/entities/Post.ts
import {
  ObjectType,
  Field,
  ID,
} from 'type-graphql';
import { User } from './User';

@ObjectType()
export class Post {
  @Field(() => ID)
  id!: string;

  @Field()
  title!: string;

  @Field()
  content!: string;

  @Field()
  published!: boolean;

  @Field(() => User)
  author!: User;

  @Field(() => [String])
  tags!: string[];

  @Field()
  createdAt!: Date;
}
```

### Input Types กับ Validation

```typescript
// src/inputs/CreateUserInput.ts
import { InputType, Field } from 'type-graphql';
import { IsEmail, MinLength, MaxLength, IsOptional, Min, Max } from 'class-validator';
import { UserRole } from '../entities/User';

@InputType({ description: 'ข้อมูลสำหรับสร้างผู้ใช้ใหม่' })
export class CreateUserInput {
  @Field()
  @MinLength(2, { message: 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร' })
  @MaxLength(100, { message: 'ชื่อต้องไม่เกิน 100 ตัวอักษร' })
  name!: string;

  @Field()
  @IsEmail({}, { message: 'รูปแบบอีเมลไม่ถูกต้อง' })
  email!: string;

  @Field()
  @MinLength(8, { message: 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร' })
  password!: string;

  @Field(() => UserRole, { nullable: true, defaultValue: UserRole.VIEWER })
  role?: UserRole;

  @Field({ nullable: true })
  @IsOptional()
  @Min(0, { message: 'อายุต้องไม่ต่ำกว่า 0' })
  @Max(150, { message: 'อายุต้องไม่เกิน 150' })
  age?: number;
}
```

### Resolver แบบ Code-First

```typescript
// src/resolvers/UserResolver.ts
import {
  Resolver,
  Query,
  Mutation,
  Arg,
  Ctx,
  Authorized,
  FieldResolver,
  Root,
  ID,
} from 'type-graphql';
import { User, UserRole } from '../entities/User';
import { Post } from '../entities/Post';
import { CreateUserInput } from '../inputs/CreateUserInput';
import { ResolverContext } from '../types';
import { v4 as uuidv4 } from 'uuid';
import { hash } from 'bcryptjs';

@Resolver(() => User)
export class UserResolver {
  @Query(() => User, { nullable: true, description: 'ดึงข้อมูลผู้ใช้ตาม ID' })
  async user(
    @Arg('id', () => ID) id: string,
    @Ctx() { db }: ResolverContext
  ): Promise<User | null> {
    return db.users.get(id) || null;
  }

  @Query(() => [User], { description: 'ดึงรายการผู้ใช้ทั้งหมด' })
  async users(@Ctx() { db }: ResolverContext): Promise<User[]> {
    return Array.from(db.users.values());
  }

  @Authorized()
  @Query(() => User, { nullable: true, description: 'ดึงข้อมูลผู้ใช้ปัจจุบัน' })
  async me(@Ctx() { userId, db }: ResolverContext): Promise<User | null> {
    if (!userId) return null;
    return db.users.get(userId) || null;
  }

  @Mutation(() => User, { description: 'สร้างผู้ใช้ใหม่' })
  async createUser(
    @Arg('input') input: CreateUserInput,
    @Ctx() { db }: ResolverContext
  ): Promise<User> {
    // ตรวจสอบอีเมลซ้ำ
    const existing = Array.from(db.users.values()).find(
      u => u.email === input.email
    );
    if (existing) {
      throw new Error('อีเมลนี้ถูกใช้งานแล้ว');
    }

    const hashedPassword = await hash(input.password, 12);
    const user: User = {
      id: uuidv4(),
      name: input.name,
      email: input.email,
      password: hashedPassword,
      age: input.age,
      role: input.role || UserRole.VIEWER,
      posts: [],
      createdAt: new Date(),
    };

    db.users.set(user.id, user);
    return user;
  }

  @Authorized(UserRole.ADMIN)
  @Mutation(() => Boolean, { description: 'ลบผู้ใช้ (เฉพาะผู้ดูแลระบบ)' })
  async deleteUser(
    @Arg('id', () => ID) id: string,
    @Ctx() { db }: ResolverContext
  ): Promise<boolean> {
    return db.users.delete(id);
  }

  @FieldResolver(() => [Post])
  async posts(
    @Root() user: User,
    @Ctx() { loaders }: ResolverContext
  ): Promise<Post[]> {
    return loaders.postsByUserLoader.load(user.id);
  }
}
```

---

## 40.5 Type Definitions ขั้นสูง

### Union Types และ Interface

```typescript
// src/schema/advanced.ts
import { gql } from 'apollo-server-express';

export const advancedTypeDefs = gql`
  interface Node {
    id: ID!
    createdAt: String!
  }

  interface Searchable {
    searchText: String!
  }

  type Article implements Node & Searchable {
    id: ID!
    title: String!
    content: String!
    author: User!
    searchText: String!
    createdAt: String!
  }

  type Video implements Node & Searchable {
    id: ID!
    title: String!
    url: String!
    duration: Int!
    searchText: String!
    createdAt: String!
  }

  type Comment implements Node {
    id: ID!
    text: String!
    author: User!
    createdAt: String!
  }

  union SearchResult = Article | Video | User

  extend type Query {
    search(query: String!): [SearchResult!]!
    node(id: ID!): Node
  }
`;
```

### Resolvers สำหรับ Union และ Interface

```typescript
// src/resolvers/search.resolver.ts
import { IResolvers } from '@graphql-tools/utils';
import { ResolverContext } from '../types';

export const searchResolvers: IResolvers<any, ResolverContext> = {
  SearchResult: {
    __resolveType(obj) {
      if ('content' in obj && 'author' in obj) return 'Article';
      if ('url' in obj && 'duration' in obj) return 'Video';
      if ('email' in obj) return 'User';
      return null;
    },
  },

  Node: {
    __resolveType(obj) {
      if ('email' in obj) return 'User';
      if ('url' in obj) return 'Video';
      if ('content' in obj) return 'Article';
      return null;
    },
  },

  Query: {
    search: async (_, { query }, { db }) => {
      const results: any[] = [];
      const lowerQuery = query.toLowerCase();

      // ค้นหาใน users
      for (const user of db.users.values()) {
        if (
          user.name.toLowerCase().includes(lowerQuery) ||
          user.email.toLowerCase().includes(lowerQuery)
        ) {
          results.push(user);
        }
      }

      return results;
    },
  },
};
```

---

## 40.6 Resolvers พร้อม Type Safety

### Typed Resolver Context

```typescript
// src/types/context.ts
import DataLoader from 'dataloader';

export interface AppContext {
  userId?: string;
  userRole?: string;
  requestId: string;
  db: AppDatabase;
  loaders: AppDataLoaders;
  logger: Logger;
}

export interface AppDatabase {
  users: UserRepository;
  posts: PostRepository;
}

export interface UserRepository {
  findById(id: string): Promise<UserModel | null>;
  findAll(filter?: UserFilter): Promise<UserModel[]>;
  create(data: CreateUserData): Promise<UserModel>;
  update(id: string, data: UpdateUserData): Promise<UserModel | null>;
  delete(id: string): Promise<boolean>;
}

export interface PostRepository {
  findById(id: string): Promise<PostModel | null>;
  findByAuthorId(authorId: string): Promise<PostModel[]>;
  findAll(filter?: PostFilter): Promise<PostModel[]>;
  create(data: CreatePostData): Promise<PostModel>;
}

export interface AppDataLoaders {
  user: DataLoader<string, UserModel | null>;
  postsByUser: DataLoader<string, PostModel[]>;
}

export interface Logger {
  info(message: string, meta?: Record<string, any>): void;
  error(message: string, error?: Error): void;
  warn(message: string, meta?: Record<string, any>): void;
}
```

### Repository Pattern

```typescript
// src/repositories/UserRepository.ts
import { v4 as uuidv4 } from 'uuid';
import { UserModel, CreateUserData, UpdateUserData, UserFilter } from '../types';

export class UserRepositoryImpl implements UserRepository {
  private users = new Map<string, UserModel>();

  async findById(id: string): Promise<UserModel | null> {
    return this.users.get(id) || null;
  }

  async findAll(filter?: UserFilter): Promise<UserModel[]> {
    let result = Array.from(this.users.values());

    if (filter?.name) {
      result = result.filter(u =>
        u.name.toLowerCase().includes(filter.name!.toLowerCase())
      );
    }

    if (filter?.role) {
      result = result.filter(u => u.role === filter.role);
    }

    return result;
  }

  async create(data: CreateUserData): Promise<UserModel> {
    const user: UserModel = {
      id: uuidv4(),
      ...data,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.users.set(user.id, user);
    return user;
  }

  async update(id: string, data: UpdateUserData): Promise<UserModel | null> {
    const user = this.users.get(id);
    if (!user) return null;

    const updated: UserModel = {
      ...user,
      ...data,
      updatedAt: new Date(),
    };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }
}
```

---

## 40.7 Mutations พร้อม Validation

### Complex Mutations

```typescript
// src/resolvers/post.resolver.ts
import { IResolvers } from '@graphql-tools/utils';
import { AuthenticationError, UserInputError, ForbiddenError } from 'apollo-server-express';
import { ResolverContext } from '../types';
import { v4 as uuidv4 } from 'uuid';

export const postResolvers: IResolvers<any, ResolverContext> = {
  Mutation: {
    createPost: async (_, { input }, { userId, db }) => {
      if (!userId) {
        throw new AuthenticationError('กรุณาเข้าสู่ระบบก่อนสร้างโพสต์');
      }

      // Validation
      if (!input.title || input.title.trim().length < 3) {
        throw new UserInputError('หัวข้อต้องมีอย่างน้อย 3 ตัวอักษร', {
          field: 'title',
        });
      }

      if (!input.content || input.content.trim().length < 10) {
        throw new UserInputError('เนื้อหาต้องมีอย่างน้อย 10 ตัวอักษร', {
          field: 'content',
        });
      }

      const post = {
        id: uuidv4(),
        title: input.title.trim(),
        content: input.content.trim(),
        published: input.published ?? false,
        authorId: userId,
        tags: input.tags || [],
        createdAt: new Date(),
        updatedAt: new Date(),
      };

      db.posts.set(post.id, post);
      return post;
    },

    publishPost: async (_, { id }, { userId, db }) => {
      if (!userId) {
        throw new AuthenticationError('กรุณาเข้าสู่ระบบ');
      }

      const post = db.posts.get(id);
      if (!post) {
        throw new UserInputError('ไม่พบโพสต์');
      }

      // ตรวจสอบว่าเป็นเจ้าของ
      if (post.authorId !== userId) {
        const user = db.users.get(userId);
        if (user?.role !== 'ADMIN') {
          throw new ForbiddenError('คุณไม่มีสิทธิ์เผยแพร่โพสต์นี้');
        }
      }

      const updatedPost = {
        ...post,
        published: true,
        publishedAt: new Date(),
        updatedAt: new Date(),
      };

      db.posts.set(id, updatedPost);
      return updatedPost;
    },

    addTagsToPost: async (_, { id, tags }, { userId, db }) => {
      if (!userId) {
        throw new AuthenticationError('กรุณาเข้าสู่ระบบ');
      }

      const post = db.posts.get(id);
      if (!post) {
        throw new UserInputError('ไม่พบโพสต์');
      }

      if (post.authorId !== userId) {
        throw new ForbiddenError('คุณไม่มีสิทธิ์แก้ไขโพสต์นี้');
      }

      // เพิ่ม tags โดยไม่ซ้ำ
      const existingTags = new Set(post.tags);
      tags.forEach((tag: string) => existingTags.add(tag.toLowerCase()));

      const updatedPost = {
        ...post,
        tags: Array.from(existingTags),
        updatedAt: new Date(),
      };

      db.posts.set(id, updatedPost);
      return updatedPost;
    },
  },
};
```

---

## 40.8 Subscriptions

### การตั้งค่า Subscriptions

```typescript
// src/server.ts (with subscriptions)
import { createServer } from 'http';
import { execute, subscribe } from 'graphql';
import { SubscriptionServer } from 'subscriptions-transport-ws';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { PubSub } from 'graphql-subscriptions';
import { ApolloServer } from 'apollo-server-express';
import express from 'express';

export const pubsub = new PubSub();

// Events
export const EVENTS = {
  POST_CREATED: 'POST_CREATED',
  POST_UPDATED: 'POST_UPDATED',
  USER_JOINED: 'USER_JOINED',
  MESSAGE_SENT: 'MESSAGE_SENT',
  NOTIFICATION: 'NOTIFICATION',
} as const;

export type EventName = typeof EVENTS[keyof typeof EVENTS];
```

### Subscription Schema

```typescript
// src/schema/subscription.schema.ts
import { gql } from 'apollo-server-express';

export const subscriptionTypeDefs = gql`
  type Subscription {
    postCreated: Post!
    postUpdated(id: ID!): Post!
    userJoined: User!
    messageSent(roomId: ID!): Message!
    notificationReceived(userId: ID!): Notification!
  }

  type Message {
    id: ID!
    content: String!
    sender: User!
    roomId: String!
    createdAt: String!
  }

  type Notification {
    id: ID!
    type: NotificationType!
    message: String!
    data: String
    userId: String!
    read: Boolean!
    createdAt: String!
  }

  enum NotificationType {
    POST_LIKE
    NEW_FOLLOWER
    COMMENT
    SYSTEM
  }
`;
```

### Subscription Resolvers

```typescript
// src/resolvers/subscription.resolver.ts
import { IResolvers } from '@graphql-tools/utils';
import { withFilter } from 'graphql-subscriptions';
import { pubsub, EVENTS } from '../server';
import { AuthenticationError } from 'apollo-server-express';
import { ResolverContext } from '../types';

export const subscriptionResolvers: IResolvers<any, ResolverContext> = {
  Subscription: {
    postCreated: {
      subscribe: (_, __, { userId }) => {
        if (!userId) {
          throw new AuthenticationError('กรุณาเข้าสู่ระบบเพื่อรับการแจ้งเตือน');
        }
        return pubsub.asyncIterator([EVENTS.POST_CREATED]);
      },
      resolve: (payload) => payload.postCreated,
    },

    postUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.POST_UPDATED]),
        (payload, variables) => {
          return payload.postUpdated.id === variables.id;
        }
      ),
      resolve: (payload) => payload.postUpdated,
    },

    messageSent: {
      subscribe: withFilter(
        (_, __, { userId }) => {
          if (!userId) throw new AuthenticationError('กรุณาเข้าสู่ระบบ');
          return pubsub.asyncIterator([EVENTS.MESSAGE_SENT]);
        },
        (payload, variables) => {
          return payload.messageSent.roomId === variables.roomId;
        }
      ),
      resolve: (payload) => payload.messageSent,
    },

    notificationReceived: {
      subscribe: withFilter(
        (_, __, { userId }) => {
          if (!userId) throw new AuthenticationError('กรุณาเข้าสู่ระบบ');
          return pubsub.asyncIterator([EVENTS.NOTIFICATION]);
        },
        (payload, variables, { userId }) => {
          return (
            payload.notificationReceived.userId === variables.userId &&
            payload.notificationReceived.userId === userId
          );
        }
      ),
      resolve: (payload) => payload.notificationReceived,
    },
  },
};
```

### การ Publish Events

```typescript
// src/services/notification.service.ts
import { pubsub, EVENTS } from '../server';
import { v4 as uuidv4 } from 'uuid';

export interface NotificationData {
  type: string;
  message: string;
  data?: string;
  userId: string;
}

export async function sendNotification(data: NotificationData): Promise<void> {
  const notification = {
    id: uuidv4(),
    ...data,
    read: false,
    createdAt: new Date(),
  };

  await pubsub.publish(EVENTS.NOTIFICATION, {
    notificationReceived: notification,
  });
}

export async function broadcastPostCreated(post: any): Promise<void> {
  await pubsub.publish(EVENTS.POST_CREATED, {
    postCreated: post,
  });
}
```

---

## 40.9 Context Typing ขั้นสูง

```typescript
// src/context/index.ts
import { Request } from 'express';
import { verify, JwtPayload } from 'jsonwebtoken';
import DataLoader from 'dataloader';
import { AppContext, AppDatabase } from '../types';

interface TokenPayload extends JwtPayload {
  userId: string;
  role: string;
}

export function createContext({ req }: { req: Request }): AppContext {
  const context: AppContext = {
    requestId: generateRequestId(),
    db: getDatabase(),
    loaders: createLoaders(getDatabase()),
    logger: createLogger(),
  };

  const token = extractToken(req);

  if (token) {
    try {
      const decoded = verify(
        token,
        process.env.JWT_SECRET!
      ) as TokenPayload;

      context.userId = decoded.userId;
      context.userRole = decoded.role;
    } catch {
      // Token หมดอายุหรือไม่ถูกต้อง
    }
  }

  return context;
}

function extractToken(req: Request): string | null {
  const authHeader = req.headers.authorization;
  if (!authHeader) return null;

  const parts = authHeader.split(' ');
  if (parts.length !== 2 || parts[0] !== 'Bearer') return null;

  return parts[1];
}

function generateRequestId(): string {
  return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

function createLogger() {
  return {
    info: (message: string, meta?: Record<string, any>) => {
      console.log(`[INFO] ${message}`, meta);
    },
    error: (message: string, error?: Error) => {
      console.error(`[ERROR] ${message}`, error);
    },
    warn: (message: string, meta?: Record<string, any>) => {
      console.warn(`[WARN] ${message}`, meta);
    },
  };
}
```

---

## 40.10 DataLoader Pattern

### การตั้งค่า DataLoader

```typescript
// src/dataloaders/index.ts
import DataLoader from 'dataloader';
import { UserModel, PostModel, AppDatabase, AppDataLoaders } from '../types';

export function createDataLoaders(db: AppDatabase): AppDataLoaders {
  return {
    user: createUserLoader(db),
    postsByUser: createPostsByUserLoader(db),
  };
}

// Batch load users
function createUserLoader(db: AppDatabase): DataLoader<string, UserModel | null> {
  return new DataLoader<string, UserModel | null>(
    async (ids: readonly string[]) => {
      console.log(`[DataLoader] Loading ${ids.length} users: ${ids.join(', ')}`);
      
      const users = await db.users.findAll({
        ids: ids as string[],
      });

      // จัดเรียงผลลัพธ์ให้ตรงกับลำดับของ ids
      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) || null);
    },
    {
      // Cache ผลลัพธ์ใน request เดียวกัน
      cache: true,
      // Batch requests ภายใน 1 tick
      batchScheduleFn: (callback) => setTimeout(callback, 0),
      // จำนวนสูงสุดต่อ batch
      maxBatchSize: 100,
    }
  );
}

// Batch load posts by user
function createPostsByUserLoader(
  db: AppDatabase
): DataLoader<string, PostModel[]> {
  return new DataLoader<string, PostModel[]>(
    async (userIds: readonly string[]) => {
      console.log(`[DataLoader] Loading posts for ${userIds.length} users`);

      const allPosts = await db.posts.findAll({
        authorIds: userIds as string[],
      });

      // Group posts by authorId
      const postsByUser = new Map<string, PostModel[]>();
      userIds.forEach(id => postsByUser.set(id, []));

      allPosts.forEach(post => {
        const userPosts = postsByUser.get(post.authorId) || [];
        userPosts.push(post);
        postsByUser.set(post.authorId, userPosts);
      });

      return userIds.map(id => postsByUser.get(id) || []);
    },
    {
      cache: true,
    }
  );
}
```

### การใช้งาน DataLoader ใน Resolvers

```typescript
// src/resolvers/post.resolver.ts
export const postFieldResolvers: IResolvers = {
  Post: {
    // ใช้ DataLoader แทน N+1 queries
    author: async (parent, _, { loaders }) => {
      // DataLoader จะ batch requests โดยอัตโนมัติ
      return loaders.user.load(parent.authorId);
    },
  },

  User: {
    posts: async (parent, _, { loaders }) => {
      return loaders.postsByUser.load(parent.id);
    },
  },
};
```

---

## 40.11 Authentication ใน GraphQL

### Auth Middleware

```typescript
// src/middleware/auth.ts
import { GraphQLResolveInfo } from 'graphql';
import { AuthenticationError, ForbiddenError } from 'apollo-server-express';
import { AppContext } from '../types';

type ResolverFn = (
  parent: any,
  args: any,
  context: AppContext,
  info: GraphQLResolveInfo
) => any;

// Higher-order function สำหรับ authentication
export function requireAuth(resolver: ResolverFn): ResolverFn {
  return (parent, args, context, info) => {
    if (!context.userId) {
      throw new AuthenticationError(
        'กรุณาเข้าสู่ระบบก่อนใช้งาน feature นี้'
      );
    }
    return resolver(parent, args, context, info);
  };
}

// Higher-order function สำหรับ role-based access
export function requireRole(...roles: string[]): (resolver: ResolverFn) => ResolverFn {
  return (resolver) => (parent, args, context, info) => {
    if (!context.userId) {
      throw new AuthenticationError('กรุณาเข้าสู่ระบบ');
    }

    if (!roles.includes(context.userRole || '')) {
      throw new ForbiddenError(
        `ต้องการสิทธิ์ระดับ ${roles.join(' หรือ ')} เท่านั้น`
      );
    }

    return resolver(parent, args, context, info);
  };
}

// การใช้งาน
export const adminResolver = requireRole('ADMIN');
export const editorResolver = requireRole('ADMIN', 'EDITOR');
```

### JWT Authentication Service

```typescript
// src/services/auth.service.ts
import { sign, verify, JwtPayload } from 'jsonwebtoken';
import { compare, hash } from 'bcryptjs';
import { UserModel } from '../types';

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

export interface TokenPayload extends JwtPayload {
  userId: string;
  email: string;
  role: string;
}

const ACCESS_TOKEN_SECRET = process.env.JWT_SECRET || 'access-secret';
const REFRESH_TOKEN_SECRET = process.env.JWT_REFRESH_SECRET || 'refresh-secret';
const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '7d';

export class AuthService {
  generateTokens(user: UserModel): AuthTokens {
    const payload: Omit<TokenPayload, keyof JwtPayload> = {
      userId: user.id,
      email: user.email,
      role: user.role,
    };

    const accessToken = sign(payload, ACCESS_TOKEN_SECRET, {
      expiresIn: ACCESS_TOKEN_EXPIRY,
    });

    const refreshToken = sign(
      { userId: user.id },
      REFRESH_TOKEN_SECRET,
      { expiresIn: REFRESH_TOKEN_EXPIRY }
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: 15 * 60, // 15 minutes in seconds
    };
  }

  verifyAccessToken(token: string): TokenPayload {
    return verify(token, ACCESS_TOKEN_SECRET) as TokenPayload;
  }

  verifyRefreshToken(token: string): { userId: string } {
    return verify(token, REFRESH_TOKEN_SECRET) as { userId: string };
  }

  async hashPassword(password: string): Promise<string> {
    return hash(password, 12);
  }

  async comparePasswords(plain: string, hashed: string): Promise<boolean> {
    return compare(plain, hashed);
  }
}

export const authService = new AuthService();
```

### Auth Resolvers

```typescript
// src/resolvers/auth.resolver.ts
import { IResolvers } from '@graphql-tools/utils';
import { UserInputError, AuthenticationError } from 'apollo-server-express';
import { authService } from '../services/auth.service';
import { AppContext } from '../types';

export const authResolvers: IResolvers<any, AppContext> = {
  Mutation: {
    login: async (_, { email, password }, { db }) => {
      const user = Array.from(db.users.values()).find(u => u.email === email);

      if (!user) {
        throw new UserInputError('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
      }

      const validPassword = await authService.comparePasswords(
        password,
        user.password
      );

      if (!validPassword) {
        throw new UserInputError('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
      }

      const tokens = authService.generateTokens(user);

      return {
        user,
        ...tokens,
      };
    },

    register: async (_, { input }, { db }) => {
      const existing = Array.from(db.users.values()).find(
        u => u.email === input.email
      );

      if (existing) {
        throw new UserInputError('อีเมลนี้ถูกใช้งานแล้ว');
      }

      const hashedPassword = await authService.hashPassword(input.password);

      const user = {
        id: Math.random().toString(36).substr(2, 9),
        name: input.name,
        email: input.email,
        password: hashedPassword,
        role: 'VIEWER',
        posts: [],
        createdAt: new Date(),
      };

      db.users.set(user.id, user);

      const tokens = authService.generateTokens(user);

      return {
        user,
        ...tokens,
      };
    },

    refreshToken: async (_, { token }, { db }) => {
      try {
        const { userId } = authService.verifyRefreshToken(token);
        const user = db.users.get(userId);

        if (!user) {
          throw new AuthenticationError('Invalid token');
        }

        return authService.generateTokens(user);
      } catch {
        throw new AuthenticationError('Token หมดอายุหรือไม่ถูกต้อง');
      }
    },
  },
};
```

---

## 40.12 การแก้ปัญหา N+1

### ปัญหา N+1

```typescript
// ❌ ปัญหา N+1 - ทุก post จะ query user แยก
const badResolvers = {
  Post: {
    author: async (parent, _, { db }) => {
      // นี่คือปัญหา N+1!
      // ถ้ามี 100 posts จะมี 100 queries
      console.log(`Query user: ${parent.authorId}`);
      return await db.findUserById(parent.authorId);
    },
  },
};

// ✅ วิธีแก้ด้วย DataLoader
const goodResolvers = {
  Post: {
    author: async (parent, _, { loaders }) => {
      // DataLoader จะรวม requests เป็น batch เดียว
      return loaders.user.load(parent.authorId);
    },
  },
};
```

### Advanced DataLoader กับ Cache

```typescript
// src/dataloaders/advanced.ts
import DataLoader from 'dataloader';

interface CacheEntry<T> {
  value: T;
  timestamp: number;
}

// DataLoader พร้อม TTL Cache
export function createCachedLoader<K, V>(
  batchFn: (keys: readonly K[]) => Promise<(V | Error)[]>,
  ttlMs: number = 60000
): DataLoader<K, V> {
  const cache = new Map<K, CacheEntry<Promise<V>>>();

  return new DataLoader(batchFn, {
    cacheMap: {
      get: (key: K) => {
        const entry = cache.get(key);
        if (!entry) return undefined;

        // ตรวจสอบว่า cache หมดอายุหรือยัง
        if (Date.now() - entry.timestamp > ttlMs) {
          cache.delete(key);
          return undefined;
        }

        return entry.value;
      },
      set: (key: K, value: Promise<V>) => {
        cache.set(key, { value, timestamp: Date.now() });
      },
      delete: (key: K) => {
        cache.delete(key);
      },
      clear: () => {
        cache.clear();
      },
    },
  });
}

// Global cache ที่แชร์ระหว่าง requests
const globalUserCache = new Map<string, UserModel>();

export function createUserLoaderWithCache(db: AppDatabase) {
  return new DataLoader<string, UserModel | null>(
    async (ids: readonly string[]) => {
      // ดูว่ามีใน cache แล้วหรือยัง
      const missing: string[] = [];
      const results = new Map<string, UserModel | null>();

      for (const id of ids) {
        const cached = globalUserCache.get(id);
        if (cached) {
          results.set(id, cached);
        } else {
          missing.push(id);
        }
      }

      if (missing.length > 0) {
        const users = await db.users.findByIds(missing);
        users.forEach(user => {
          globalUserCache.set(user.id, user);
          results.set(user.id, user);
        });
      }

      return ids.map(id => results.get(id) || null);
    }
  );
}
```

---

## 40.13 ตัวอย่าง GraphQL API สมบูรณ์

### Blog API Schema

```typescript
// src/schema/blog.schema.ts
export const blogSchema = gql`
  type Query {
    # Users
    me: User
    user(id: ID!): User
    users(filter: UserFilter, pagination: PaginationInput): UserConnection!

    # Posts
    post(id: ID!): Post
    posts(filter: PostFilter, pagination: PaginationInput): PostConnection!
    featuredPosts: [Post!]!

    # Tags
    tags: [Tag!]!
    popularTags(limit: Int = 10): [Tag!]!
  }

  type Mutation {
    # Auth
    login(email: String!, password: String!): AuthPayload!
    register(input: RegisterInput!): AuthPayload!
    logout: Boolean!
    refreshToken(token: String!): AuthPayload!

    # Posts
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!
    unpublishPost(id: ID!): Post!
    likePost(id: ID!): Post!
    unlikePost(id: ID!): Post!

    # Comments
    addComment(postId: ID!, content: String!): Comment!
    deleteComment(id: ID!): Boolean!
    replyToComment(commentId: ID!, content: String!): Comment!
  }

  type Subscription {
    postPublished: Post!
    commentAdded(postId: ID!): Comment!
    newFollower(userId: ID!): User!
  }

  type AuthPayload {
    user: User!
    accessToken: String!
    refreshToken: String!
    expiresIn: Int!
  }

  type Post {
    id: ID!
    title: String!
    slug: String!
    excerpt: String!
    content: String!
    coverImage: String
    published: Boolean!
    featured: Boolean!
    author: User!
    tags: [Tag!]!
    comments: [Comment!]!
    commentCount: Int!
    likeCount: Int!
    likedByMe: Boolean!
    viewCount: Int!
    readingTime: Int!
    createdAt: String!
    updatedAt: String!
    publishedAt: String
  }

  type Comment {
    id: ID!
    content: String!
    author: User!
    post: Post!
    replies: [Comment!]!
    parentComment: Comment
    createdAt: String!
  }

  type Tag {
    id: ID!
    name: String!
    slug: String!
    postCount: Int!
  }

  input CreatePostInput {
    title: String!
    content: String!
    excerpt: String
    coverImage: String
    tags: [String!]
    published: Boolean = false
  }

  input UpdatePostInput {
    title: String
    content: String
    excerpt: String
    coverImage: String
    tags: [String!]
  }

  input PostFilter {
    authorId: ID
    tags: [String!]
    published: Boolean
    featured: Boolean
    search: String
    dateFrom: String
    dateTo: String
  }

  input PaginationInput {
    page: Int = 1
    perPage: Int = 10
  }

  type PostConnection {
    nodes: [Post!]!
    totalCount: Int!
    pageInfo: PageInfo!
  }

  type UserConnection {
    nodes: [User!]!
    totalCount: Int!
    pageInfo: PageInfo!
  }

  type PageInfo {
    currentPage: Int!
    totalPages: Int!
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
  }

  input RegisterInput {
    name: String!
    email: String!
    password: String!
    username: String!
  }

  input UserFilter {
    search: String
    role: UserRole
  }
`;
```

### Complete Blog Resolvers

```typescript
// src/resolvers/blog.resolver.ts
import { IResolvers } from '@graphql-tools/utils';
import { AppContext } from '../types';
import { pubsub, EVENTS } from '../server';
import { authService } from '../services/auth.service';
import { v4 as uuidv4 } from 'uuid';

export const blogResolvers: IResolvers<any, AppContext> = {
  Query: {
    me: (_, __, { userId, db }) => {
      if (!userId) return null;
      return db.users.get(userId) || null;
    },

    posts: async (_, { filter, pagination }, { db }) => {
      let posts = Array.from(db.posts.values());

      // Apply filters
      if (filter) {
        if (filter.published !== undefined) {
          posts = posts.filter(p => p.published === filter.published);
        }
        if (filter.authorId) {
          posts = posts.filter(p => p.authorId === filter.authorId);
        }
        if (filter.search) {
          const search = filter.search.toLowerCase();
          posts = posts.filter(p =>
            p.title.toLowerCase().includes(search) ||
            p.content.toLowerCase().includes(search)
          );
        }
        if (filter.tags?.length) {
          posts = posts.filter(p =>
            filter.tags.some((tag: string) => p.tags.includes(tag))
          );
        }
      }

      const totalCount = posts.length;
      const { page = 1, perPage = 10 } = pagination || {};
      const skip = (page - 1) * perPage;
      const nodes = posts.slice(skip, skip + perPage);
      const totalPages = Math.ceil(totalCount / perPage);

      return {
        nodes,
        totalCount,
        pageInfo: {
          currentPage: page,
          totalPages,
          hasNextPage: page < totalPages,
          hasPreviousPage: page > 1,
        },
      };
    },

    featuredPosts: async (_, __, { db }) => {
      return Array.from(db.posts.values())
        .filter(p => p.published && p.featured)
        .sort((a, b) => b.likeCount - a.likeCount)
        .slice(0, 5);
    },
  },

  Mutation: {
    createPost: async (_, { input }, { userId, db }) => {
      if (!userId) throw new Error('กรุณาเข้าสู่ระบบ');

      const slug = input.title
        .toLowerCase()
        .replace(/[^a-z0-9฀-๿\s]/g, '')
        .replace(/\s+/g, '-');

      const post = {
        id: uuidv4(),
        title: input.title,
        slug: `${slug}-${Date.now()}`,
        excerpt: input.excerpt || input.content.substring(0, 200),
        content: input.content,
        coverImage: input.coverImage,
        published: input.published || false,
        featured: false,
        authorId: userId,
        tags: input.tags || [],
        likeCount: 0,
        likedBy: new Set<string>(),
        viewCount: 0,
        readingTime: Math.ceil(input.content.split(' ').length / 200),
        createdAt: new Date(),
        updatedAt: new Date(),
        publishedAt: input.published ? new Date() : null,
      };

      db.posts.set(post.id, post);

      if (post.published) {
        await pubsub.publish(EVENTS.POST_CREATED, { postPublished: post });
      }

      return post;
    },

    likePost: async (_, { id }, { userId, db }) => {
      if (!userId) throw new Error('กรุณาเข้าสู่ระบบ');

      const post = db.posts.get(id);
      if (!post) throw new Error('ไม่พบโพสต์');

      const likedBy = new Set(post.likedBy || []);
      if (likedBy.has(userId)) {
        throw new Error('คุณกด Like แล้ว');
      }

      likedBy.add(userId);
      const updated = {
        ...post,
        likedBy,
        likeCount: likedBy.size,
        updatedAt: new Date(),
      };

      db.posts.set(id, updated);
      return updated;
    },
  },

  Post: {
    author: async (parent, _, { loaders }) => {
      return loaders.user.load(parent.authorId);
    },

    likedByMe: (parent, _, { userId }) => {
      if (!userId) return false;
      return parent.likedBy?.has(userId) || false;
    },

    comments: async (parent, _, { db }) => {
      return Array.from(db.comments?.values() || []).filter(
        (c: any) => c.postId === parent.id && !c.parentCommentId
      );
    },

    commentCount: async (parent, _, { db }) => {
      return Array.from(db.comments?.values() || []).filter(
        (c: any) => c.postId === parent.id
      ).length;
    },
  },

  Subscription: {
    postPublished: {
      subscribe: () => pubsub.asyncIterator([EVENTS.POST_CREATED]),
    },

    commentAdded: {
      subscribe: (_, { postId }) =>
        pubsub.asyncIterator([`COMMENT_ADDED_${postId}`]),
    },
  },
};
```

---

## 40.14 Error Handling

```typescript
// src/errors/index.ts
import { ApolloError } from 'apollo-server-express';

export class NotFoundError extends ApolloError {
  constructor(resource: string) {
    super(`ไม่พบ ${resource}`, 'NOT_FOUND');
    Object.defineProperty(this, 'name', { value: 'NotFoundError' });
  }
}

export class ValidationError extends ApolloError {
  constructor(message: string, field?: string) {
    super(message, 'VALIDATION_ERROR', { field });
    Object.defineProperty(this, 'name', { value: 'ValidationError' });
  }
}

export class UnauthorizedError extends ApolloError {
  constructor(message = 'กรุณาเข้าสู่ระบบ') {
    super(message, 'UNAUTHORIZED');
    Object.defineProperty(this, 'name', { value: 'UnauthorizedError' });
  }
}

export class ForbiddenError extends ApolloError {
  constructor(message = 'ไม่มีสิทธิ์เข้าถึง') {
    super(message, 'FORBIDDEN');
    Object.defineProperty(this, 'name', { value: 'ForbiddenError' });
  }
}

export class RateLimitError extends ApolloError {
  constructor(retryAfter: number) {
    super('คำขอมากเกินไป กรุณาลองใหม่ภายหลัง', 'RATE_LIMIT_EXCEEDED', {
      retryAfter,
    });
  }
}
```

### Error Formatting

```typescript
// src/server.ts
const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (error) => {
    // Log ข้อผิดพลาดภายใน
    if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
      console.error('[INTERNAL ERROR]', error);
      // ซ่อน details จาก client ใน production
      if (process.env.NODE_ENV === 'production') {
        return new ApolloError(
          'เกิดข้อผิดพลาดภายในระบบ',
          'INTERNAL_SERVER_ERROR'
        );
      }
    }
    
    return error;
  },
});
```

---

## สรุป

GraphQL กับ TypeScript ทำงานได้อย่างยอดเยี่ยม เนื่องจาก:

1. **Type Safety** - Schema ของ GraphQL สร้าง type definitions อัตโนมัติ
2. **Code-First vs Schema-First** - เลือกแนวทางที่เหมาะกับทีม
3. **DataLoader** - แก้ปัญหา N+1 ได้อย่างมีประสิทธิภาพ
4. **Authentication** - JWT + middleware pattern
5. **Subscriptions** - Real-time updates ด้วย WebSockets
6. **Error Handling** - Custom error types สำหรับ user-friendly messages

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Microservices กับ TypeScript
