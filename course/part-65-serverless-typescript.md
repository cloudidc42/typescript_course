# ส่วนที่ 65: Serverless with TypeScript

## บทนำ

Serverless architecture ช่วยให้นักพัฒนาสามารถสร้างและ deploy แอปพลิเคชันโดยไม่ต้องจัดการ infrastructure ด้วยตัวเอง ในบทนี้เราจะเรียนรู้การสร้าง serverless applications ด้วย TypeScript โดยใช้ Serverless Framework

---

## 1. Serverless Framework กับ TypeScript

### 1.1 ติดตั้งและตั้งค่า

```bash
# ติดตั้ง Serverless Framework
npm install -g serverless

# สร้าง project ใหม่
serverless create --template aws-nodejs-typescript --path my-serverless-app
cd my-serverless-app

# ติดตั้ง dependencies
npm install
npm install --save-dev @types/aws-lambda serverless-esbuild serverless-offline
```

### 1.2 serverless.yml

```yaml
# serverless.yml
service: my-serverless-app

frameworkVersion: '3'

plugins:
  - serverless-esbuild
  - serverless-offline

provider:
  name: aws
  runtime: nodejs18.x
  region: ${opt:region, 'ap-southeast-1'}
  stage: ${opt:stage, 'dev'}
  
  # Environment variables
  environment:
    STAGE: ${self:provider.stage}
    REGION: ${self:provider.region}
    TABLE_NAME: ${self:custom.tableName}
    BUCKET_NAME: ${self:custom.bucketName}
  
  # IAM permissions
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
            - dynamodb:Query
            - dynamodb:Scan
          Resource:
            - !GetAtt UsersTable.Arn
            - !Sub "${UsersTable.Arn}/index/*"
        - Effect: Allow
          Action:
            - s3:GetObject
            - s3:PutObject
            - s3:DeleteObject
          Resource:
            - !Sub "arn:aws:s3:::${self:custom.bucketName}/*"

custom:
  tableName: users-${self:provider.stage}
  bucketName: my-app-uploads-${self:provider.stage}
  
  esbuild:
    bundle: true
    minify: false
    sourcemap: true
    exclude:
      - '@aws-sdk/*'
    target: 'node18'
    platform: 'node'
    concurrency: 10
  
  serverless-offline:
    httpPort: 3000
    lambdaPort: 3002

functions:
  # Users API
  getUsers:
    handler: src/handlers/users/get-users.handler
    events:
      - http:
          path: /users
          method: GET
          cors: true
  
  getUserById:
    handler: src/handlers/users/get-user.handler
    events:
      - http:
          path: /users/{userId}
          method: GET
          cors: true
  
  createUser:
    handler: src/handlers/users/create-user.handler
    events:
      - http:
          path: /users
          method: POST
          cors: true
  
  updateUser:
    handler: src/handlers/users/update-user.handler
    events:
      - http:
          path: /users/{userId}
          method: PUT
          cors: true
  
  deleteUser:
    handler: src/handlers/users/delete-user.handler
    events:
      - http:
          path: /users/{userId}
          method: DELETE
          cors: true
  
  # S3 Triggers
  processUpload:
    handler: src/handlers/s3/process-upload.handler
    events:
      - s3:
          bucket: ${self:custom.bucketName}
          event: s3:ObjectCreated:*
          rules:
            - prefix: uploads/
  
  # SQS Triggers
  processEmailQueue:
    handler: src/handlers/sqs/process-emails.handler
    events:
      - sqs:
          arn: !GetAtt EmailQueue.Arn
          batchSize: 10
          functionResponseType: ReportBatchItemFailures
  
  # Scheduled function
  cleanupOldFiles:
    handler: src/handlers/scheduled/cleanup.handler
    events:
      - schedule:
          rate: rate(1 day)
          description: 'Cleanup old files daily'

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:custom.tableName}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: PK
            AttributeType: S
          - AttributeName: SK
            AttributeType: S
          - AttributeName: GSI1PK
            AttributeType: S
        KeySchema:
          - AttributeName: PK
            KeyType: HASH
          - AttributeName: SK
            KeyType: RANGE
        GlobalSecondaryIndexes:
          - IndexName: GSI1
            KeySchema:
              - AttributeName: GSI1PK
                KeyType: HASH
            Projection:
              ProjectionType: ALL
    
    EmailQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: email-queue-${self:provider.stage}
        VisibilityTimeout: 60
        RedrivePolicy:
          deadLetterTargetArn: !GetAtt EmailDLQ.Arn
          maxReceiveCount: 3
    
    EmailDLQ:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: email-dlq-${self:provider.stage}
        MessageRetentionPeriod: 1209600
```

---

## 2. Handler Types และ Event Types

### 2.1 API Gateway Handler

```typescript
// src/handlers/users/get-users.ts
import {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
  Context
} from 'aws-lambda';
import { DynamoDBDocumentClient, ScanCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

interface UserItem {
  userId: string;
  name: string;
  email: string;
  role: string;
  createdAt: string;
}

// Helper สำหรับสร้าง response
const createResponse = (
  statusCode: number,
  body: unknown
): APIGatewayProxyResult => ({
  statusCode,
  headers: {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Credentials': 'true'
  },
  body: JSON.stringify(body)
});

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  // ลด cold start latency
  context.callbackWaitsForEmptyEventLoop = false;

  try {
    const tableName = process.env.TABLE_NAME;
    if (!tableName) {
      return createResponse(500, { error: 'TABLE_NAME ไม่ถูกตั้งค่า' });
    }

    // Query parameters
    const limit = parseInt(event.queryStringParameters?.limit ?? '20');
    const lastKey = event.queryStringParameters?.lastKey
      ? JSON.parse(decodeURIComponent(event.queryStringParameters.lastKey))
      : undefined;

    const response = await docClient.send(
      new ScanCommand({
        TableName: tableName,
        FilterExpression: 'begins_with(PK, :prefix)',
        ExpressionAttributeValues: {
          ':prefix': 'USER#'
        },
        Limit: limit,
        ExclusiveStartKey: lastKey
      })
    );

    const users = (response.Items ?? []).map((item) => ({
      userId: item.userId,
      name: item.name,
      email: item.email,
      role: item.role,
      createdAt: item.createdAt
    })) as UserItem[];

    return createResponse(200, {
      users,
      count: users.length,
      lastKey: response.LastEvaluatedKey
        ? encodeURIComponent(JSON.stringify(response.LastEvaluatedKey))
        : null
    });
  } catch (error) {
    console.error('Error getting users:', error);
    return createResponse(500, { error: 'Internal Server Error' });
  }
};
```

### 2.2 S3 Event Handler

```typescript
// src/handlers/s3/process-upload.ts
import { S3Event, S3EventRecord, Context } from 'aws-lambda';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { Readable } from 'stream';

const s3Client = new S3Client({});
const dynamoClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

interface FileMetadata {
  key: string;
  bucket: string;
  size: number;
  contentType: string;
  processedAt: string;
}

async function processRecord(record: S3EventRecord): Promise<void> {
  const bucket = record.s3.bucket.name;
  const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
  const size = record.s3.object.size;

  console.log(`Processing: ${bucket}/${key} (${size} bytes)`);

  // ดึง object metadata
  const headResponse = await s3Client.send(
    new GetObjectCommand({ Bucket: bucket, Key: key })
  );

  const metadata: FileMetadata = {
    key,
    bucket,
    size,
    contentType: headResponse.ContentType ?? 'application/octet-stream',
    processedAt: new Date().toISOString()
  };

  // บันทึก metadata ลง DynamoDB
  await dynamoClient.send(
    new PutCommand({
      TableName: process.env.TABLE_NAME!,
      Item: {
        PK: `FILE#${key}`,
        SK: `FILE#${key}`,
        ...metadata
      }
    })
  );

  console.log(`Processed file metadata for: ${key}`);
}

export const handler = async (
  event: S3Event,
  context: Context
): Promise<void> => {
  console.log(`Processing ${event.Records.length} S3 events`);

  const promises = event.Records.map(processRecord);
  const results = await Promise.allSettled(promises);

  const failed = results.filter(r => r.status === 'rejected');
  if (failed.length > 0) {
    console.error(`${failed.length} records failed:`, failed);
    throw new Error(`${failed.length} S3 records failed to process`);
  }
};
```

### 2.3 SQS Event Handler กับ Partial Batch Failures

```typescript
// src/handlers/sqs/process-emails.ts
import {
  SQSEvent,
  SQSRecord,
  SQSBatchResponse,
  SQSBatchItemFailure,
  Context
} from 'aws-lambda';
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses';

const sesClient = new SESClient({});

interface EmailMessage {
  to: string;
  subject: string;
  body: string;
  templateId?: string;
  templateData?: Record<string, string>;
}

async function processEmailRecord(record: SQSRecord): Promise<void> {
  const message: EmailMessage = JSON.parse(record.body);

  console.log(`Sending email to: ${message.to}`);

  await sesClient.send(
    new SendEmailCommand({
      Source: process.env.FROM_EMAIL ?? 'noreply@example.com',
      Destination: {
        ToAddresses: [message.to]
      },
      Message: {
        Subject: { Data: message.subject },
        Body: {
          Html: { Data: message.body },
          Text: { Data: message.body.replace(/<[^>]*>/g, '') }
        }
      }
    })
  );

  console.log(`Email sent to: ${message.to}`);
}

// ReportBatchItemFailures ช่วยให้ retry เฉพาะ records ที่ fail
export const handler = async (
  event: SQSEvent,
  context: Context
): Promise<SQSBatchResponse> => {
  const batchItemFailures: SQSBatchItemFailure[] = [];

  const promises = event.Records.map(async (record) => {
    try {
      await processEmailRecord(record);
    } catch (error) {
      console.error(`Failed to process record ${record.messageId}:`, error);
      // เพิ่ม record ที่ fail เข้า failures list
      batchItemFailures.push({
        itemIdentifier: record.messageId
      });
    }
  });

  await Promise.all(promises);

  console.log(
    `Processed ${event.Records.length} records, ${batchItemFailures.length} failures`
  );

  return { batchItemFailures };
};
```

### 2.4 Scheduled Event Handler

```typescript
// src/handlers/scheduled/cleanup.ts
import { ScheduledEvent, Context } from 'aws-lambda';
import { S3Client, ListObjectsV2Command, DeleteObjectsCommand } from '@aws-sdk/client-s3';

const s3Client = new S3Client({});

async function cleanupOldFiles(
  bucket: string,
  prefix: string,
  olderThanDays: number
): Promise<number> {
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - olderThanDays);

  const objectsToDelete: { Key: string }[] = [];
  let continuationToken: string | undefined;

  // ดึงรายการ files ทั้งหมด
  do {
    const response = await s3Client.send(
      new ListObjectsV2Command({
        Bucket: bucket,
        Prefix: prefix,
        ContinuationToken: continuationToken
      })
    );

    for (const obj of response.Contents ?? []) {
      if (obj.Key && obj.LastModified && obj.LastModified < cutoffDate) {
        objectsToDelete.push({ Key: obj.Key });
      }
    }

    continuationToken = response.NextContinuationToken;
  } while (continuationToken);

  if (objectsToDelete.length === 0) {
    return 0;
  }

  // ลบ objects ทีละ 1000 (AWS limit)
  let deletedCount = 0;
  for (let i = 0; i < objectsToDelete.length; i += 1000) {
    const batch = objectsToDelete.slice(i, i + 1000);
    
    await s3Client.send(
      new DeleteObjectsCommand({
        Bucket: bucket,
        Delete: { Objects: batch }
      })
    );

    deletedCount += batch.length;
  }

  return deletedCount;
}

export const handler = async (
  event: ScheduledEvent,
  context: Context
): Promise<void> => {
  console.log('Starting cleanup job:', event.time);

  const bucket = process.env.BUCKET_NAME!;
  const cleanupDays = parseInt(process.env.CLEANUP_DAYS ?? '30');

  const deletedCount = await cleanupOldFiles(bucket, 'temp/', cleanupDays);
  
  console.log(`Cleaned up ${deletedCount} files older than ${cleanupDays} days`);
};
```

---

## 3. Middleware กับ Middy

### 3.1 ติดตั้ง Middy

```bash
npm install @middy/core
npm install @middy/http-json-body-parser
npm install @middy/http-error-handler
npm install @middy/http-cors
npm install @middy/validator
npm install @middy/warmup
```

### 3.2 สร้าง Middleware ด้วย Middy

```typescript
// src/middleware/index.ts
import middy from '@middy/core';
import httpJsonBodyParser from '@middy/http-json-body-parser';
import httpErrorHandler from '@middy/http-error-handler';
import cors from '@middy/http-cors';
import { injectLambdaContext } from '@aws-lambda-powertools/logger/middleware';
import { Logger } from '@aws-lambda-powertools/logger';

export const logger = new Logger({ serviceName: 'my-serverless-app' });

// Middleware สำหรับ API handlers ทั้งหมด
export function withMiddleware(handler: middy.MiddlewareFn) {
  return middy(handler)
    .use(httpJsonBodyParser())
    .use(cors({
      origin: process.env.ALLOWED_ORIGIN ?? '*',
      headers: 'Content-Type,Authorization',
      methods: 'GET,POST,PUT,DELETE,OPTIONS'
    }))
    .use(injectLambdaContext(logger, { clearState: true }))
    .use(httpErrorHandler());
}

// Custom middleware สำหรับ validation
export function withValidation<T>(schema: object) {
  return (handler: middy.MiddlewareFn) =>
    middy(handler)
      .use(httpJsonBodyParser())
      .use(createValidatorMiddleware<T>(schema))
      .use(httpErrorHandler());
}

// Middleware สำหรับ authentication
export function withAuth() {
  return {
    before: async (request: any) => {
      const token = request.event.headers?.Authorization?.replace('Bearer ', '');
      
      if (!token) {
        throw createError(401, 'Authorization header ต้องการ');
      }

      try {
        const decoded = verifyToken(token);
        request.event.user = decoded;
      } catch {
        throw createError(401, 'Token ไม่ถูกต้อง');
      }
    }
  };
}

function createError(statusCode: number, message: string) {
  const error = new Error(message) as any;
  error.statusCode = statusCode;
  return error;
}

function verifyToken(token: string): any {
  // JWT verification logic
  return { userId: '1', role: 'user' }; // mock
}

function createValidatorMiddleware<T>(schema: object) {
  return {
    before: async (request: any) => {
      // Validation logic
      const { error } = validateSchema(schema, request.event.body);
      if (error) {
        throw createError(400, error.message);
      }
    }
  };
}

function validateSchema(schema: object, data: any): { error?: Error } {
  // Schema validation using ajv or joi
  return {};
}
```

### 3.3 Handler ที่ใช้ Middy

```typescript
// src/handlers/users/create-user-with-middy.ts
import middy from '@middy/core';
import {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
  Context
} from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { v4 as uuidv4 } from 'uuid';
import { withMiddleware } from '../../middleware';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

interface CreateUserBody {
  name: string;
  email: string;
  role?: 'admin' | 'user';
}

const baseHandler = async (
  event: APIGatewayProxyEvent & { body: CreateUserBody },
  context: Context
): Promise<APIGatewayProxyResult> => {
  const { name, email, role = 'user' } = event.body;

  if (!name || !email) {
    return {
      statusCode: 400,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: 'name และ email ต้องการ' })
    };
  }

  const userId = uuidv4();
  const now = new Date().toISOString();

  const user = {
    PK: `USER#${userId}`,
    SK: `USER#${userId}`,
    userId,
    name,
    email,
    role,
    createdAt: now,
    updatedAt: now,
    GSI1PK: `EMAIL#${email}`,
    GSI1SK: `USER#${userId}`
  };

  await docClient.send(
    new PutCommand({
      TableName: process.env.TABLE_NAME!,
      Item: user,
      ConditionExpression: 'attribute_not_exists(PK)'
    })
  );

  return {
    statusCode: 201,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId, name, email, role })
  };
};

export const handler = withMiddleware(baseHandler as any);
```

---

## 4. Cold Start Optimization

### 4.1 การลด Cold Start Time

```typescript
// src/utils/lazy-client.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { S3Client } from '@aws-sdk/client-s3';

// Singleton pattern สำหรับ AWS clients
// สร้างครั้งเดียวและใช้ซ้ำใน Lambda invocations ในตัวเดียวกัน
let dynamoDbClient: DynamoDBDocumentClient | null = null;
let s3Client: S3Client | null = null;

export function getDynamoClient(): DynamoDBDocumentClient {
  if (!dynamoDbClient) {
    const rawClient = new DynamoDBClient({
      maxAttempts: 3
    });
    dynamoDbClient = DynamoDBDocumentClient.from(rawClient, {
      marshallOptions: { removeUndefinedValues: true }
    });
  }
  return dynamoDbClient;
}

export function getS3Client(): S3Client {
  if (!s3Client) {
    s3Client = new S3Client({
      maxAttempts: 3
    });
  }
  return s3Client;
}
```

```typescript
// src/utils/warmup.ts

// ตรวจสอบ warmup event จาก serverless-plugin-warmup
export function isWarmupEvent(event: any): boolean {
  return event.source === 'serverless-plugin-warmup';
}

// Decorator สำหรับ warmup
export function withWarmup(handler: Function) {
  return async (event: any, context: any) => {
    if (isWarmupEvent(event)) {
      console.log('Warmup event received');
      return { statusCode: 200, body: 'warmed' };
    }
    return handler(event, context);
  };
}

// Pre-initialize heavy operations ที่ module level
// (ทำงานเมื่อ Lambda container เริ่มต้น)
console.log('Lambda container initialized');
const startTime = Date.now();

export function getContainerAge(): number {
  return Date.now() - startTime;
}
```

### 4.2 Provisioned Concurrency

```yaml
# serverless.yml - กำหนด provisioned concurrency
functions:
  criticalApi:
    handler: src/handlers/critical.handler
    events:
      - http:
          path: /critical
          method: GET
    
    # Provisioned concurrency ลด cold starts
    provisionedConcurrency: 3
    
    # หรือใช้ Auto Scaling
    autoScaling:
      minCapacity: 2
      maxCapacity: 10
      targetUtilization: 0.7
```

---

## 5. Environment Variables

### 5.1 การจัดการ Environment Variables

```typescript
// src/config/environment.ts

// Type-safe environment variables
interface EnvironmentConfig {
  tableName: string;
  bucketName: string;
  region: string;
  stage: string;
  fromEmail: string;
  jwtSecret: string;
  allowedOrigin: string;
}

function getRequiredEnv(key: string): string {
  const value = process.env[key];
  if (!value) {
    throw new Error(`Environment variable ${key} ต้องการ`);
  }
  return value;
}

function getOptionalEnv(key: string, defaultValue: string): string {
  return process.env[key] ?? defaultValue;
}

export const config: EnvironmentConfig = {
  tableName: getRequiredEnv('TABLE_NAME'),
  bucketName: getRequiredEnv('BUCKET_NAME'),
  region: getOptionalEnv('AWS_REGION', 'ap-southeast-1'),
  stage: getOptionalEnv('STAGE', 'dev'),
  fromEmail: getOptionalEnv('FROM_EMAIL', 'noreply@example.com'),
  jwtSecret: getRequiredEnv('JWT_SECRET'),
  allowedOrigin: getOptionalEnv('ALLOWED_ORIGIN', '*')
};

// Validate config ณ startup
Object.entries(config).forEach(([key, value]) => {
  if (value === undefined || value === null) {
    throw new Error(`Config validation failed: ${key} is undefined`);
  }
});
```

### 5.2 Secrets Manager Integration

```typescript
// src/config/secrets.ts
import {
  SecretsManagerClient,
  GetSecretValueCommand
} from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({});
const secretsCache: Map<string, { value: any; expiry: number }> = new Map();

export async function getSecret<T>(
  secretName: string,
  cacheTtlMs: number = 5 * 60 * 1000
): Promise<T> {
  const cached = secretsCache.get(secretName);
  if (cached && Date.now() < cached.expiry) {
    return cached.value;
  }

  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await client.send(command);

  const value = response.SecretString
    ? JSON.parse(response.SecretString)
    : null;

  secretsCache.set(secretName, {
    value,
    expiry: Date.now() + cacheTtlMs
  });

  return value;
}

// การใช้งาน
interface DatabaseConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

let dbConfig: DatabaseConfig | null = null;

export async function getDatabaseConfig(): Promise<DatabaseConfig> {
  if (!dbConfig) {
    dbConfig = await getSecret<DatabaseConfig>('myapp/database');
  }
  return dbConfig;
}
```

---

## 6. DynamoDB Integration

### 6.1 Single-Table Design

```typescript
// src/repositories/single-table.repository.ts
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { getDynamoClient } from '../utils/lazy-client';

export type EntityType = 'USER' | 'POST' | 'COMMENT' | 'ORDER';

export interface BaseEntity {
  PK: string;
  SK: string;
  type: EntityType;
  createdAt: string;
  updatedAt: string;
  [key: string]: any;
}

export class SingleTableRepository {
  private readonly client: DynamoDBDocumentClient;
  private readonly tableName: string;

  constructor() {
    this.client = getDynamoClient();
    this.tableName = process.env.TABLE_NAME!;
  }

  // สร้าง key patterns
  static userKey(userId: string) {
    return { PK: `USER#${userId}`, SK: `USER#${userId}` };
  }

  static postKey(userId: string, postId: string) {
    return { PK: `USER#${userId}`, SK: `POST#${postId}` };
  }

  static commentKey(postId: string, commentId: string) {
    return { PK: `POST#${postId}`, SK: `COMMENT#${commentId}` };
  }

  // ดึง user พร้อม posts ทั้งหมด
  async getUserWithPosts(userId: string) {
    const { Items } = await this.client.send({
      ...{ TableName: this.tableName },
      KeyConditionExpression: 'PK = :pk',
      ExpressionAttributeValues: { ':pk': `USER#${userId}` }
    } as any);

    if (!Items?.length) return null;

    const user = Items.find(item => item.SK === `USER#${userId}`);
    const posts = Items.filter(item => item.SK.startsWith('POST#'));

    return { user, posts };
  }
}
```

---

## 7. Complete Serverless Application

### 7.1 Todo API Application

```typescript
// src/handlers/todos/create-todo.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { v4 as uuidv4 } from 'uuid';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

interface CreateTodoBody {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';
  dueDate?: string;
}

interface TodoItem {
  PK: string;
  SK: string;
  todoId: string;
  userId: string;
  title: string;
  description: string;
  status: 'todo' | 'in-progress' | 'done';
  priority: 'low' | 'medium' | 'high';
  dueDate?: string;
  createdAt: string;
  updatedAt: string;
  // GSI สำหรับค้นหาตาม status
  GSI1PK: string;
  GSI1SK: string;
}

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  context.callbackWaitsForEmptyEventLoop = false;

  try {
    // ดึง userId จาก authorizer
    const userId = event.requestContext.authorizer?.principalId;
    if (!userId) {
      return {
        statusCode: 401,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Unauthorized' })
      };
    }

    if (!event.body) {
      return {
        statusCode: 400,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Request body ต้องการ' })
      };
    }

    const body: CreateTodoBody = JSON.parse(event.body);

    if (!body.title?.trim()) {
      return {
        statusCode: 400,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'title ต้องการ' })
      };
    }

    const todoId = uuidv4();
    const now = new Date().toISOString();

    const todo: TodoItem = {
      PK: `USER#${userId}`,
      SK: `TODO#${todoId}`,
      todoId,
      userId,
      title: body.title.trim(),
      description: body.description?.trim() ?? '',
      status: 'todo',
      priority: body.priority ?? 'medium',
      dueDate: body.dueDate,
      createdAt: now,
      updatedAt: now,
      GSI1PK: `STATUS#todo`,
      GSI1SK: `USER#${userId}#TODO#${todoId}`
    };

    await docClient.send(
      new PutCommand({
        TableName: process.env.TABLE_NAME!,
        Item: todo
      })
    );

    return {
      statusCode: 201,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({
        todoId: todo.todoId,
        title: todo.title,
        description: todo.description,
        status: todo.status,
        priority: todo.priority,
        dueDate: todo.dueDate,
        createdAt: todo.createdAt
      })
    };
  } catch (error) {
    console.error('Error creating todo:', error);
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: 'Internal Server Error' })
    };
  }
};
```

```typescript
// src/handlers/todos/list-todos.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  QueryCommand,
  QueryCommandInput
} from '@aws-sdk/lib-dynamodb';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  context.callbackWaitsForEmptyEventLoop = false;

  try {
    const userId = event.requestContext.authorizer?.principalId;
    if (!userId) {
      return {
        statusCode: 401,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Unauthorized' })
      };
    }

    const status = event.queryStringParameters?.status;
    const limit = parseInt(event.queryStringParameters?.limit ?? '20');
    const lastKey = event.queryStringParameters?.lastKey
      ? JSON.parse(decodeURIComponent(event.queryStringParameters.lastKey))
      : undefined;

    let queryParams: QueryCommandInput;

    if (status) {
      // ค้นหาตาม status ผ่าน GSI
      queryParams = {
        TableName: process.env.TABLE_NAME!,
        IndexName: 'GSI1',
        KeyConditionExpression: 'GSI1PK = :status AND begins_with(GSI1SK, :userId)',
        ExpressionAttributeValues: {
          ':status': `STATUS#${status}`,
          ':userId': `USER#${userId}`
        },
        Limit: limit,
        ExclusiveStartKey: lastKey
      };
    } else {
      // ดึงทุก todos ของ user
      queryParams = {
        TableName: process.env.TABLE_NAME!,
        KeyConditionExpression: 'PK = :pk AND begins_with(SK, :prefix)',
        ExpressionAttributeValues: {
          ':pk': `USER#${userId}`,
          ':prefix': 'TODO#'
        },
        Limit: limit,
        ExclusiveStartKey: lastKey,
        ScanIndexForward: false // เรียงจากใหม่ไปเก่า
      };
    }

    const response = await docClient.send(new QueryCommand(queryParams));

    const todos = (response.Items ?? []).map(item => ({
      todoId: item.todoId,
      title: item.title,
      description: item.description,
      status: item.status,
      priority: item.priority,
      dueDate: item.dueDate,
      createdAt: item.createdAt,
      updatedAt: item.updatedAt
    }));

    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({
        todos,
        count: todos.length,
        lastKey: response.LastEvaluatedKey
          ? encodeURIComponent(JSON.stringify(response.LastEvaluatedKey))
          : null
      })
    };
  } catch (error) {
    console.error('Error listing todos:', error);
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: 'Internal Server Error' })
    };
  }
};
```

---

## 8. Lambda Authorizer

```typescript
// src/handlers/auth/authorizer.ts
import {
  APIGatewayTokenAuthorizerEvent,
  APIGatewayAuthorizerResult,
  PolicyDocument,
  Statement,
  Context
} from 'aws-lambda';
import * as jwt from 'jsonwebtoken';

interface TokenPayload {
  userId: string;
  email: string;
  role: 'admin' | 'user';
}

function generatePolicy(
  principalId: string,
  effect: 'Allow' | 'Deny',
  resource: string,
  context?: Record<string, string>
): APIGatewayAuthorizerResult {
  const statement: Statement = {
    Action: 'execute-api:Invoke',
    Effect: effect,
    Resource: resource
  };

  const policyDocument: PolicyDocument = {
    Version: '2012-10-17',
    Statement: [statement]
  };

  return {
    principalId,
    policyDocument,
    context: context ?? {}
  };
}

export const handler = async (
  event: APIGatewayTokenAuthorizerEvent,
  context: Context
): Promise<APIGatewayAuthorizerResult> => {
  const token = event.authorizationToken?.replace('Bearer ', '');

  if (!token) {
    throw new Error('Unauthorized');
  }

  try {
    const secret = process.env.JWT_SECRET!;
    const decoded = jwt.verify(token, secret) as TokenPayload;

    return generatePolicy(
      decoded.userId,
      'Allow',
      event.methodArn,
      {
        userId: decoded.userId,
        email: decoded.email,
        role: decoded.role
      }
    );
  } catch (error) {
    throw new Error('Unauthorized');
  }
};
```

---

## 9. Testing Serverless Functions

### 9.1 Unit Tests สำหรับ Lambda

```typescript
// src/handlers/todos/create-todo.test.ts
import { APIGatewayProxyEvent, Context } from 'aws-lambda';
import { mockClient } from 'aws-sdk-client-mock';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { handler } from './create-todo';

const ddbMock = mockClient(DynamoDBDocumentClient);

describe('Create Todo Handler', () => {
  beforeEach(() => {
    ddbMock.reset();
    process.env.TABLE_NAME = 'todos-test';
  });

  const mockEvent = (body: object, userId: string = 'user-123'): APIGatewayProxyEvent => ({
    body: JSON.stringify(body),
    headers: {},
    multiValueHeaders: {},
    httpMethod: 'POST',
    isBase64Encoded: false,
    path: '/todos',
    pathParameters: null,
    queryStringParameters: null,
    multiValueQueryStringParameters: null,
    stageVariables: null,
    requestContext: {
      authorizer: { principalId: userId }
    } as any,
    resource: '/todos'
  });

  const mockContext: Context = {
    callbackWaitsForEmptyEventLoop: false,
    functionName: 'create-todo',
    functionVersion: '1',
    invokedFunctionArn: 'arn:aws:lambda:ap-southeast-1:123456789:function:create-todo',
    memoryLimitInMB: '256',
    awsRequestId: 'test-request-id',
    logGroupName: '/aws/lambda/create-todo',
    logStreamName: 'test-stream',
    getRemainingTimeInMillis: () => 30000,
    done: jest.fn(),
    fail: jest.fn(),
    succeed: jest.fn()
  };

  test('ควรสร้าง todo สำเร็จ', async () => {
    ddbMock.on(PutCommand).resolves({});

    const event = mockEvent({ title: 'ทดสอบ Todo', priority: 'high' });
    const result = await handler(event, mockContext);

    expect(result.statusCode).toBe(201);
    
    const body = JSON.parse(result.body);
    expect(body.title).toBe('ทดสอบ Todo');
    expect(body.priority).toBe('high');
    expect(body.status).toBe('todo');
    expect(body.todoId).toBeDefined();
  });

  test('ควรคืน 400 เมื่อไม่มี title', async () => {
    const event = mockEvent({ description: 'ไม่มี title' });
    const result = await handler(event, mockContext);

    expect(result.statusCode).toBe(400);
    expect(JSON.parse(result.body).error).toBe('title ต้องการ');
  });

  test('ควรคืน 401 เมื่อไม่มี userId', async () => {
    const event = mockEvent({ title: 'Todo' }, '');
    const result = await handler(event, mockContext);

    expect(result.statusCode).toBe(401);
  });

  test('ควรคืน 500 เมื่อ DynamoDB error', async () => {
    ddbMock.on(PutCommand).rejects(new Error('DynamoDB Error'));

    const event = mockEvent({ title: 'Todo ที่จะ error' });
    const result = await handler(event, mockContext);

    expect(result.statusCode).toBe(500);
  });
});
```

---

## 10. Monitoring และ Observability

### 10.1 AWS Lambda Powertools

```typescript
// src/utils/observability.ts
import { Logger } from '@aws-lambda-powertools/logger';
import { Metrics, MetricUnits } from '@aws-lambda-powertools/metrics';
import { Tracer } from '@aws-lambda-powertools/tracer';

// กำหนด service name
const SERVICE_NAME = 'my-serverless-app';

export const logger = new Logger({ serviceName: SERVICE_NAME });

export const metrics = new Metrics({
  namespace: 'MyServerlessApp',
  serviceName: SERVICE_NAME
});

export const tracer = new Tracer({ serviceName: SERVICE_NAME });

// Custom metrics helper
export function recordMetric(
  name: string,
  value: number,
  unit: MetricUnits = MetricUnits.Count
) {
  metrics.addMetric(name, unit, value);
}

// Structured logging
export function logEvent(
  message: string,
  extra?: Record<string, unknown>
) {
  logger.info(message, extra);
}

export function logError(
  message: string,
  error: Error,
  extra?: Record<string, unknown>
) {
  logger.error(message, { error: error.message, stack: error.stack, ...extra });
}
```

---

## 11. Deployment และ CI/CD

### 11.1 GitHub Actions สำหรับ Serverless

```yaml
# .github/workflows/serverless-deploy.yml
name: Deploy Serverless

on:
  push:
    branches:
      - main
      - develop

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Build TypeScript
        run: npm run build

  deploy-dev:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Deploy to dev
        run: npx serverless deploy --stage dev
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID_DEV }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY_DEV }}

  deploy-prod:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Deploy to production
        run: npx serverless deploy --stage prod
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
```

---

## 12. Error Handling และ Retry Logic

```typescript
// src/utils/retry.ts
interface RetryOptions {
  maxAttempts: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
  retryableErrors?: string[];
}

const defaultOptions: RetryOptions = {
  maxAttempts: 3,
  initialDelayMs: 100,
  maxDelayMs: 5000,
  backoffMultiplier: 2
};

export async function withRetry<T>(
  operation: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const config = { ...defaultOptions, ...options };
  let lastError: Error;
  let delay = config.initialDelayMs;

  for (let attempt = 1; attempt <= config.maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;

      // ตรวจสอบว่าควร retry หรือไม่
      if (config.retryableErrors) {
        const isRetryable = config.retryableErrors.some(
          errType => lastError.message.includes(errType)
        );
        if (!isRetryable) throw lastError;
      }

      if (attempt === config.maxAttempts) {
        throw lastError;
      }

      console.warn(
        `Attempt ${attempt} failed: ${lastError.message}. Retrying in ${delay}ms...`
      );

      await new Promise(resolve => setTimeout(resolve, delay));
      delay = Math.min(delay * config.backoffMultiplier, config.maxDelayMs);
    }
  }

  throw lastError!;
}

// ตัวอย่างการใช้งาน
async function fetchWithRetry(url: string): Promise<Response> {
  return withRetry(
    () => fetch(url),
    {
      maxAttempts: 3,
      initialDelayMs: 500,
      retryableErrors: ['NetworkError', 'ECONNRESET']
    }
  );
}
```

---

## สรุปบทที่ 65

ในบทนี้เราได้เรียนรู้:

1. **Serverless Framework** - การตั้งค่าและ configuration
2. **Handler Types** - API Gateway, S3, SQS, Scheduled handlers
3. **Middy Middleware** - middleware สำหรับ Lambda functions
4. **Cold Start Optimization** - การลด latency
5. **Environment Variables** - การจัดการ configuration
6. **DynamoDB Integration** - single-table design
7. **Complete Serverless App** - Todo API
8. **Lambda Authorizer** - JWT authentication
9. **Testing** - unit tests ด้วย aws-sdk-client-mock
10. **Monitoring** - AWS Lambda Powertools
11. **CI/CD** - GitHub Actions deployment
12. **Error Handling** - retry logic

---

## แบบฝึกหัด

1. สร้าง serverless blog API ด้วย TypeScript
2. Implement image processing pipeline ด้วย S3 triggers
3. สร้าง notification system ด้วย SQS และ SNS
4. เพิ่ม authentication ด้วย Cognito
5. ตั้งค่า monitoring dashboard ด้วย CloudWatch

---

## Resources เพิ่มเติม

- [Serverless Framework Documentation](https://www.serverless.com/framework/docs)
- [AWS Lambda Powertools for TypeScript](https://docs.powertools.aws.dev/lambda/typescript/)
- [AWS CDK TypeScript Reference](https://docs.aws.amazon.com/cdk/api/v2/)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)

---

*จบส่วนที่ 65 - Serverless with TypeScript*

*ยินดีด้วยที่เรียนจบ Advanced TypeScript Course! คุณได้เรียนรู้ทั้ง Testing Patterns, TDD, E2E Testing, AWS Integration และ Serverless Development ครบถ้วนแล้ว*
