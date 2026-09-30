# ส่วนที่ 64: AWS with TypeScript

## บทนำ

Amazon Web Services (AWS) เป็นบริการ cloud computing ที่ครองตลาด TypeScript มี integration ที่ดีมากกับ AWS SDK v3 ในบทนี้เราจะเรียนรู้การใช้ AWS services ต่างๆ ผ่าน TypeScript

---

## 1. AWS SDK v3 กับ TypeScript

### 1.1 ติดตั้ง AWS SDK v3

```bash
# ติดตั้ง core packages
npm install @aws-sdk/client-s3
npm install @aws-sdk/client-dynamodb
npm install @aws-sdk/lib-dynamodb
npm install @aws-sdk/client-lambda
npm install @aws-sdk/client-sqs
npm install @aws-sdk/client-sns
npm install @aws-sdk/client-secrets-manager
npm install @aws-sdk/client-api-gateway

# TypeScript types
npm install --save-dev @types/node
```

### 1.2 การตั้งค่า AWS Credentials

```typescript
// aws-config.ts
import { 
  fromEnv, 
  fromIni, 
  fromContainerMetadata,
  fromInstanceMetadata 
} from '@aws-sdk/credential-providers';
import { STSClient, GetCallerIdentityCommand } from '@aws-sdk/client-sts';

// กำหนด configuration แบบต่างๆ
export const awsConfig = {
  region: process.env.AWS_REGION ?? 'ap-southeast-1',
  credentials: process.env.AWS_ACCESS_KEY_ID
    ? fromEnv()
    : process.env.AWS_PROFILE
    ? fromIni({ profile: process.env.AWS_PROFILE })
    : fromInstanceMetadata() // สำหรับ EC2/ECS
};

// ตรวจสอบ credentials
async function verifyCredentials(): Promise<void> {
  const stsClient = new STSClient(awsConfig);
  const command = new GetCallerIdentityCommand({});
  
  try {
    const response = await stsClient.send(command);
    console.log('AWS Account:', response.Account);
    console.log('AWS User ARN:', response.Arn);
  } catch (error) {
    console.error('ไม่สามารถยืนยัน AWS credentials:', error);
    throw error;
  }
}
```

---

## 2. S3 Operations กับ TypeScript

### 2.1 S3 Client และ Basic Operations

```typescript
// services/s3.service.ts
import {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  ListObjectsV2Command,
  HeadObjectCommand,
  CopyObjectCommand,
  CreateBucketCommand,
  DeleteBucketCommand,
  GetObjectCommandOutput,
  ListObjectsV2CommandOutput,
  _Object
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { Upload } from '@aws-sdk/lib-storage';
import { Readable } from 'stream';
import * as fs from 'fs';
import * as path from 'path';

export interface S3ObjectMetadata {
  key: string;
  size: number;
  lastModified: Date;
  contentType: string;
  eTag: string;
}

export interface UploadResult {
  bucket: string;
  key: string;
  url: string;
  eTag: string;
}

export class S3Service {
  private readonly client: S3Client;

  constructor(private readonly bucketName: string, region: string = 'ap-southeast-1') {
    this.client = new S3Client({
      region,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
      }
    });
  }

  // Upload ไฟล์
  async upload(
    key: string,
    body: Buffer | Readable | string,
    contentType: string = 'application/octet-stream',
    metadata?: Record<string, string>
  ): Promise<UploadResult> {
    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: body,
      ContentType: contentType,
      Metadata: metadata,
      ServerSideEncryption: 'AES256'
    });

    const response = await this.client.send(command);

    return {
      bucket: this.bucketName,
      key,
      url: `https://${this.bucketName}.s3.amazonaws.com/${key}`,
      eTag: response.ETag ?? ''
    };
  }

  // Upload ไฟล์ขนาดใหญ่ด้วย multipart upload
  async uploadLargeFile(
    key: string,
    filePath: string,
    contentType: string = 'application/octet-stream'
  ): Promise<UploadResult> {
    const fileStream = fs.createReadStream(filePath);
    const fileSize = fs.statSync(filePath).size;

    const upload = new Upload({
      client: this.client,
      params: {
        Bucket: this.bucketName,
        Key: key,
        Body: fileStream,
        ContentType: contentType
      },
      // Multipart upload threshold (5MB)
      partSize: 5 * 1024 * 1024,
      queueSize: 4 // parallel parts
    });

    // Track progress
    upload.on('httpUploadProgress', (progress) => {
      if (progress.loaded && progress.total) {
        const percentage = Math.round((progress.loaded / progress.total) * 100);
        console.log(`Upload progress: ${percentage}%`);
      }
    });

    const result = await upload.done();

    return {
      bucket: this.bucketName,
      key,
      url: `https://${this.bucketName}.s3.amazonaws.com/${key}`,
      eTag: result.ETag ?? ''
    };
  }

  // ดาวน์โหลด object เป็น Buffer
  async download(key: string): Promise<Buffer> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key
    });

    const response: GetObjectCommandOutput = await this.client.send(command);

    if (!response.Body) {
      throw new Error(`ไม่พบ object: ${key}`);
    }

    // แปลง ReadableStream เป็น Buffer
    const chunks: Buffer[] = [];
    const body = response.Body as Readable;

    return new Promise((resolve, reject) => {
      body.on('data', (chunk: Buffer) => chunks.push(chunk));
      body.on('end', () => resolve(Buffer.concat(chunks)));
      body.on('error', reject);
    });
  }

  // ลบ object
  async delete(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucketName,
      Key: key
    });

    await this.client.send(command);
  }

  // แสดงรายการ objects
  async list(prefix?: string, maxKeys: number = 1000): Promise<S3ObjectMetadata[]> {
    const objects: S3ObjectMetadata[] = [];
    let continuationToken: string | undefined;

    do {
      const command = new ListObjectsV2Command({
        Bucket: this.bucketName,
        Prefix: prefix,
        MaxKeys: maxKeys,
        ContinuationToken: continuationToken
      });

      const response: ListObjectsV2CommandOutput = await this.client.send(command);

      if (response.Contents) {
        for (const obj of response.Contents) {
          if (obj.Key && obj.Size !== undefined && obj.LastModified && obj.ETag) {
            objects.push({
              key: obj.Key,
              size: obj.Size,
              lastModified: obj.LastModified,
              contentType: '',
              eTag: obj.ETag
            });
          }
        }
      }

      continuationToken = response.NextContinuationToken;
    } while (continuationToken);

    return objects;
  }

  // สร้าง presigned URL สำหรับ download
  async getDownloadUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key
    });

    return getSignedUrl(this.client, command, { expiresIn });
  }

  // สร้าง presigned URL สำหรับ upload
  async getUploadUrl(
    key: string,
    contentType: string,
    expiresIn: number = 3600
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      ContentType: contentType
    });

    return getSignedUrl(this.client, command, { expiresIn });
  }

  // Copy object
  async copy(sourceKey: string, destinationKey: string): Promise<void> {
    const command = new CopyObjectCommand({
      Bucket: this.bucketName,
      CopySource: `${this.bucketName}/${sourceKey}`,
      Key: destinationKey
    });

    await this.client.send(command);
  }

  // ตรวจสอบว่า object มีอยู่
  async exists(key: string): Promise<boolean> {
    try {
      const command = new HeadObjectCommand({
        Bucket: this.bucketName,
        Key: key
      });
      await this.client.send(command);
      return true;
    } catch {
      return false;
    }
  }
}
```

---

## 3. DynamoDB กับ TypeScript

### 3.1 DynamoDB Document Client

```typescript
// services/dynamodb.service.ts
import {
  DynamoDBClient,
  CreateTableCommand,
  DeleteTableCommand,
  DescribeTableCommand,
  AttributeDefinition,
  KeySchemaElement,
  BillingMode
} from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  PutCommand,
  GetCommand,
  DeleteCommand,
  UpdateCommand,
  QueryCommand,
  ScanCommand,
  BatchWriteCommand,
  BatchGetCommand,
  TransactWriteCommand,
  QueryCommandInput,
  ScanCommandInput
} from '@aws-sdk/lib-dynamodb';

// Types
export interface DynamoDBItem {
  [key: string]: any;
}

export interface PaginatedResult<T> {
  items: T[];
  lastEvaluatedKey?: Record<string, any>;
  count: number;
}

export class DynamoDBService {
  private readonly docClient: DynamoDBDocumentClient;
  private readonly rawClient: DynamoDBClient;

  constructor(region: string = 'ap-southeast-1') {
    this.rawClient = new DynamoDBClient({ region });
    this.docClient = DynamoDBDocumentClient.from(this.rawClient, {
      marshallOptions: {
        removeUndefinedValues: true,
        convertClassInstanceToMap: true
      }
    });
  }

  // สร้าง item
  async put(tableName: string, item: DynamoDBItem): Promise<void> {
    const command = new PutCommand({
      TableName: tableName,
      Item: item
    });

    await this.docClient.send(command);
  }

  // ดึง item ตาม key
  async get<T extends DynamoDBItem>(
    tableName: string,
    key: DynamoDBItem
  ): Promise<T | null> {
    const command = new GetCommand({
      TableName: tableName,
      Key: key
    });

    const response = await this.docClient.send(command);
    return (response.Item as T) ?? null;
  }

  // ลบ item
  async delete(tableName: string, key: DynamoDBItem): Promise<void> {
    const command = new DeleteCommand({
      TableName: tableName,
      Key: key
    });

    await this.docClient.send(command);
  }

  // อัปเดต item
  async update(
    tableName: string,
    key: DynamoDBItem,
    updates: DynamoDBItem
  ): Promise<DynamoDBItem | null> {
    const updateExpressions: string[] = [];
    const expressionAttributeNames: Record<string, string> = {};
    const expressionAttributeValues: Record<string, any> = {};

    for (const [field, value] of Object.entries(updates)) {
      const attrName = `#${field}`;
      const attrValue = `:${field}`;
      
      updateExpressions.push(`${attrName} = ${attrValue}`);
      expressionAttributeNames[attrName] = field;
      expressionAttributeValues[attrValue] = value;
    }

    const command = new UpdateCommand({
      TableName: tableName,
      Key: key,
      UpdateExpression: `SET ${updateExpressions.join(', ')}`,
      ExpressionAttributeNames: expressionAttributeNames,
      ExpressionAttributeValues: expressionAttributeValues,
      ReturnValues: 'ALL_NEW'
    });

    const response = await this.docClient.send(command);
    return response.Attributes ?? null;
  }

  // Query ตาม partition key
  async query<T extends DynamoDBItem>(
    tableName: string,
    keyConditionExpression: string,
    expressionAttributeValues: Record<string, any>,
    options?: {
      filterExpression?: string;
      expressionAttributeNames?: Record<string, string>;
      limit?: number;
      lastEvaluatedKey?: Record<string, any>;
      indexName?: string;
      scanIndexForward?: boolean;
    }
  ): Promise<PaginatedResult<T>> {
    const params: QueryCommandInput = {
      TableName: tableName,
      KeyConditionExpression: keyConditionExpression,
      ExpressionAttributeValues: expressionAttributeValues,
      FilterExpression: options?.filterExpression,
      ExpressionAttributeNames: options?.expressionAttributeNames,
      Limit: options?.limit,
      ExclusiveStartKey: options?.lastEvaluatedKey,
      IndexName: options?.indexName,
      ScanIndexForward: options?.scanIndexForward
    };

    const response = await this.docClient.send(new QueryCommand(params));

    return {
      items: (response.Items as T[]) ?? [],
      lastEvaluatedKey: response.LastEvaluatedKey,
      count: response.Count ?? 0
    };
  }

  // Scan table
  async scan<T extends DynamoDBItem>(
    tableName: string,
    options?: {
      filterExpression?: string;
      expressionAttributeNames?: Record<string, string>;
      expressionAttributeValues?: Record<string, any>;
      limit?: number;
      lastEvaluatedKey?: Record<string, any>;
    }
  ): Promise<PaginatedResult<T>> {
    const params: ScanCommandInput = {
      TableName: tableName,
      FilterExpression: options?.filterExpression,
      ExpressionAttributeNames: options?.expressionAttributeNames,
      ExpressionAttributeValues: options?.expressionAttributeValues,
      Limit: options?.limit,
      ExclusiveStartKey: options?.lastEvaluatedKey
    };

    const response = await this.docClient.send(new ScanCommand(params));

    return {
      items: (response.Items as T[]) ?? [],
      lastEvaluatedKey: response.LastEvaluatedKey,
      count: response.Count ?? 0
    };
  }

  // Batch write
  async batchWrite(
    tableName: string,
    items: DynamoDBItem[],
    keysToDelete?: DynamoDBItem[]
  ): Promise<void> {
    const putRequests = items.map(item => ({
      PutRequest: { Item: item }
    }));

    const deleteRequests = (keysToDelete ?? []).map(key => ({
      DeleteRequest: { Key: key }
    }));

    const allRequests = [...putRequests, ...deleteRequests];

    // DynamoDB มี limit 25 items ต่อ batch
    const chunks = [];
    for (let i = 0; i < allRequests.length; i += 25) {
      chunks.push(allRequests.slice(i, i + 25));
    }

    for (const chunk of chunks) {
      const command = new BatchWriteCommand({
        RequestItems: {
          [tableName]: chunk
        }
      });

      await this.docClient.send(command);
    }
  }

  // Transaction write
  async transactWrite(
    operations: Array<{
      type: 'Put' | 'Update' | 'Delete' | 'ConditionCheck';
      tableName: string;
      item?: DynamoDBItem;
      key?: DynamoDBItem;
      updateExpression?: string;
      conditionExpression?: string;
      expressionAttributeValues?: Record<string, any>;
      expressionAttributeNames?: Record<string, string>;
    }>
  ): Promise<void> {
    const transactItems = operations.map(op => {
      switch (op.type) {
        case 'Put':
          return {
            Put: {
              TableName: op.tableName,
              Item: op.item!,
              ConditionExpression: op.conditionExpression,
              ExpressionAttributeValues: op.expressionAttributeValues,
              ExpressionAttributeNames: op.expressionAttributeNames
            }
          };
        case 'Delete':
          return {
            Delete: {
              TableName: op.tableName,
              Key: op.key!,
              ConditionExpression: op.conditionExpression
            }
          };
        case 'Update':
          return {
            Update: {
              TableName: op.tableName,
              Key: op.key!,
              UpdateExpression: op.updateExpression!,
              ExpressionAttributeValues: op.expressionAttributeValues,
              ExpressionAttributeNames: op.expressionAttributeNames
            }
          };
        default:
          throw new Error(`Operation type ไม่รองรับ: ${op.type}`);
      }
    });

    const command = new TransactWriteCommand({ TransactItems: transactItems });
    await this.docClient.send(command);
  }
}
```

### 3.2 Repository Pattern กับ DynamoDB

```typescript
// repositories/user.repository.ts
import { DynamoDBService } from '../services/dynamodb.service';
import { v4 as uuidv4 } from 'uuid';

export interface DynamoUser {
  PK: string; // USER#userId
  SK: string; // USER#userId
  userId: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: string;
  updatedAt: string;
  GSI1PK?: string; // EMAIL#email
  GSI1SK?: string;
}

export class UserRepository {
  private readonly TABLE_NAME = 'users';

  constructor(private readonly db: DynamoDBService) {}

  async create(data: {
    name: string;
    email: string;
    role: 'admin' | 'user';
  }): Promise<DynamoUser> {
    const userId = uuidv4();
    const now = new Date().toISOString();

    const user: DynamoUser = {
      PK: `USER#${userId}`,
      SK: `USER#${userId}`,
      userId,
      name: data.name,
      email: data.email,
      role: data.role,
      createdAt: now,
      updatedAt: now,
      GSI1PK: `EMAIL#${data.email}`,
      GSI1SK: `USER#${userId}`
    };

    await this.db.put(this.TABLE_NAME, user);
    return user;
  }

  async findById(userId: string): Promise<DynamoUser | null> {
    return this.db.get<DynamoUser>(this.TABLE_NAME, {
      PK: `USER#${userId}`,
      SK: `USER#${userId}`
    });
  }

  async findByEmail(email: string): Promise<DynamoUser | null> {
    const result = await this.db.query<DynamoUser>(
      this.TABLE_NAME,
      'GSI1PK = :pk',
      { ':pk': `EMAIL#${email}` },
      { indexName: 'GSI1' }
    );

    return result.items[0] ?? null;
  }

  async update(
    userId: string,
    updates: Partial<Pick<DynamoUser, 'name' | 'role'>>
  ): Promise<DynamoUser | null> {
    const result = await this.db.update(
      this.TABLE_NAME,
      { PK: `USER#${userId}`, SK: `USER#${userId}` },
      { ...updates, updatedAt: new Date().toISOString() }
    );

    return result as DynamoUser | null;
  }

  async delete(userId: string): Promise<void> {
    await this.db.delete(this.TABLE_NAME, {
      PK: `USER#${userId}`,
      SK: `USER#${userId}`
    });
  }
}
```

---

## 4. Lambda Functions ใน TypeScript

### 4.1 Lambda Handler Types

```typescript
// lambda/types.ts
import {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
  APIGatewayProxyEventV2,
  APIGatewayProxyResultV2,
  Context,
  SQSEvent,
  SNSEvent,
  S3Event,
  ScheduledEvent
} from 'aws-lambda';

// ประเภท handlers ต่างๆ
export type ApiGatewayHandler = (
  event: APIGatewayProxyEvent,
  context: Context
) => Promise<APIGatewayProxyResult>;

export type ApiGatewayV2Handler = (
  event: APIGatewayProxyEventV2,
  context: Context
) => Promise<APIGatewayProxyResultV2>;

export type SqsHandler = (
  event: SQSEvent,
  context: Context
) => Promise<void>;

export type S3Handler = (
  event: S3Event,
  context: Context
) => Promise<void>;

export type ScheduledHandler = (
  event: ScheduledEvent,
  context: Context
) => Promise<void>;

// Helper function สร้าง response
export function createResponse(
  statusCode: number,
  body: object | string,
  headers: Record<string, string> = {}
): APIGatewayProxyResult {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Headers': 'Content-Type,Authorization',
      ...headers
    },
    body: typeof body === 'string' ? body : JSON.stringify(body)
  };
}

export const ok = (body: object) => createResponse(200, body);
export const created = (body: object) => createResponse(201, body);
export const badRequest = (message: string) => createResponse(400, { error: message });
export const unauthorized = () => createResponse(401, { error: 'Unauthorized' });
export const forbidden = () => createResponse(403, { error: 'Forbidden' });
export const notFound = (resource: string) => createResponse(404, { error: `${resource} not found` });
export const internalError = () => createResponse(500, { error: 'Internal Server Error' });
```

### 4.2 Lambda Function ตัวอย่าง

```typescript
// lambda/users/get-user.ts
import { APIGatewayProxyEvent, Context } from 'aws-lambda';
import { DynamoDBService } from '../../services/dynamodb.service';
import { UserRepository } from '../../repositories/user.repository';
import { ok, notFound, badRequest, internalError } from '../types';

const db = new DynamoDBService();
const userRepo = new UserRepository(db);

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
) => {
  console.log('Event:', JSON.stringify(event, null, 2));
  console.log('Context:', context.functionName);

  try {
    const userId = event.pathParameters?.userId;

    if (!userId) {
      return badRequest('userId ต้องการ');
    }

    const user = await userRepo.findById(userId);

    if (!user) {
      return notFound(`User ${userId}`);
    }

    return ok({
      userId: user.userId,
      name: user.name,
      email: user.email,
      role: user.role
    });
  } catch (error) {
    console.error('Error:', error);
    return internalError();
  }
};
```

```typescript
// lambda/users/create-user.ts
import { APIGatewayProxyEvent, Context } from 'aws-lambda';
import { DynamoDBService } from '../../services/dynamodb.service';
import { UserRepository } from '../../repositories/user.repository';
import { ok, created, badRequest, internalError } from '../types';

const db = new DynamoDBService();
const userRepo = new UserRepository(db);

interface CreateUserBody {
  name: string;
  email: string;
  role?: 'admin' | 'user';
}

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
) => {
  try {
    if (!event.body) {
      return badRequest('Request body ต้องการ');
    }

    let body: CreateUserBody;
    try {
      body = JSON.parse(event.body);
    } catch {
      return badRequest('JSON ไม่ถูกต้อง');
    }

    const { name, email, role = 'user' } = body;

    if (!name || !email) {
      return badRequest('name และ email ต้องการ');
    }

    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      return badRequest('รูปแบบ email ไม่ถูกต้อง');
    }

    const existing = await userRepo.findByEmail(email);
    if (existing) {
      return { statusCode: 409, body: JSON.stringify({ error: 'Email ถูกใช้แล้ว' }) };
    }

    const user = await userRepo.create({ name, email, role });

    return created({
      userId: user.userId,
      name: user.name,
      email: user.email,
      role: user.role
    });
  } catch (error) {
    console.error('Error:', error);
    return internalError();
  }
};
```

---

## 5. SQS และ SNS

### 5.1 SQS Service

```typescript
// services/sqs.service.ts
import {
  SQSClient,
  SendMessageCommand,
  ReceiveMessageCommand,
  DeleteMessageCommand,
  SendMessageBatchCommand,
  GetQueueAttributesCommand,
  PurgeQueueCommand,
  Message
} from '@aws-sdk/client-sqs';
import { v4 as uuidv4 } from 'uuid';

export interface SqsMessage<T = any> {
  messageId: string;
  receiptHandle: string;
  body: T;
  attributes: Record<string, string>;
  sentTimestamp: Date;
}

export class SqsService {
  private readonly client: SQSClient;

  constructor(region: string = 'ap-southeast-1') {
    this.client = new SQSClient({ region });
  }

  // ส่ง message
  async send<T>(
    queueUrl: string,
    body: T,
    options?: {
      delaySeconds?: number;
      messageGroupId?: string;
      deduplicationId?: string;
      attributes?: Record<string, { DataType: string; StringValue: string }>;
    }
  ): Promise<string> {
    const command = new SendMessageCommand({
      QueueUrl: queueUrl,
      MessageBody: JSON.stringify(body),
      DelaySeconds: options?.delaySeconds,
      MessageGroupId: options?.messageGroupId,
      MessageDeduplicationId: options?.deduplicationId ?? uuidv4(),
      MessageAttributes: options?.attributes
    });

    const response = await this.client.send(command);
    return response.MessageId ?? '';
  }

  // ส่ง batch messages
  async sendBatch<T>(
    queueUrl: string,
    messages: Array<{
      id: string;
      body: T;
      delaySeconds?: number;
    }>
  ): Promise<{ successful: string[]; failed: string[] }> {
    const command = new SendMessageBatchCommand({
      QueueUrl: queueUrl,
      Entries: messages.map(msg => ({
        Id: msg.id,
        MessageBody: JSON.stringify(msg.body),
        DelaySeconds: msg.delaySeconds
      }))
    });

    const response = await this.client.send(command);

    return {
      successful: response.Successful?.map(s => s.MessageId ?? '') ?? [],
      failed: response.Failed?.map(f => f.Id ?? '') ?? []
    };
  }

  // รับ messages
  async receive<T>(
    queueUrl: string,
    options?: {
      maxMessages?: number;
      waitTimeSeconds?: number;
      visibilityTimeout?: number;
    }
  ): Promise<SqsMessage<T>[]> {
    const command = new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      MaxNumberOfMessages: options?.maxMessages ?? 10,
      WaitTimeSeconds: options?.waitTimeSeconds ?? 20,
      VisibilityTimeout: options?.visibilityTimeout ?? 30,
      AttributeNames: ['All'],
      MessageAttributeNames: ['All']
    });

    const response = await this.client.send(command);

    return (response.Messages ?? []).map((msg: Message) => ({
      messageId: msg.MessageId ?? '',
      receiptHandle: msg.ReceiptHandle ?? '',
      body: JSON.parse(msg.Body ?? '{}') as T,
      attributes: msg.Attributes ?? {},
      sentTimestamp: new Date(
        parseInt(msg.Attributes?.SentTimestamp ?? '0')
      )
    }));
  }

  // ลบ message หลังจากประมวลผลเสร็จ
  async delete(queueUrl: string, receiptHandle: string): Promise<void> {
    const command = new DeleteMessageCommand({
      QueueUrl: queueUrl,
      ReceiptHandle: receiptHandle
    });

    await this.client.send(command);
  }

  // ดูจำนวน messages ใน queue
  async getQueueDepth(queueUrl: string): Promise<number> {
    const command = new GetQueueAttributesCommand({
      QueueUrl: queueUrl,
      AttributeNames: ['ApproximateNumberOfMessages']
    });

    const response = await this.client.send(command);
    return parseInt(
      response.Attributes?.ApproximateNumberOfMessages ?? '0'
    );
  }
}
```

### 5.2 SNS Service

```typescript
// services/sns.service.ts
import {
  SNSClient,
  PublishCommand,
  PublishBatchCommand,
  SubscribeCommand,
  UnsubscribeCommand,
  ListSubscriptionsByTopicCommand
} from '@aws-sdk/client-sns';

export interface SnsMessage<T = any> {
  subject?: string;
  body: T;
  attributes?: Record<string, { DataType: string; StringValue: string }>;
}

export class SnsService {
  private readonly client: SNSClient;

  constructor(region: string = 'ap-southeast-1') {
    this.client = new SNSClient({ region });
  }

  // Publish message
  async publish<T>(
    topicArn: string,
    message: SnsMessage<T>
  ): Promise<string> {
    const command = new PublishCommand({
      TopicArn: topicArn,
      Subject: message.subject,
      Message: JSON.stringify(message.body),
      MessageAttributes: message.attributes
    });

    const response = await this.client.send(command);
    return response.MessageId ?? '';
  }

  // Subscribe endpoint
  async subscribe(
    topicArn: string,
    protocol: 'https' | 'http' | 'sqs' | 'lambda' | 'email',
    endpoint: string,
    filterPolicy?: Record<string, any>
  ): Promise<string> {
    const command = new SubscribeCommand({
      TopicArn: topicArn,
      Protocol: protocol,
      Endpoint: endpoint,
      Attributes: filterPolicy
        ? { FilterPolicy: JSON.stringify(filterPolicy) }
        : undefined
    });

    const response = await this.client.send(command);
    return response.SubscriptionArn ?? '';
  }
}
```

---

## 6. Secrets Manager

```typescript
// services/secrets.service.ts
import {
  SecretsManagerClient,
  GetSecretValueCommand,
  CreateSecretCommand,
  UpdateSecretCommand,
  DeleteSecretCommand,
  RotateSecretCommand
} from '@aws-sdk/client-secrets-manager';

export class SecretsManagerService {
  private readonly client: SecretsManagerClient;
  private readonly cache: Map<string, { value: any; expiry: number }> = new Map();
  private readonly cacheTtl: number;

  constructor(
    region: string = 'ap-southeast-1',
    cacheTtlSeconds: number = 300
  ) {
    this.client = new SecretsManagerClient({ region });
    this.cacheTtl = cacheTtlSeconds * 1000;
  }

  // ดึง secret value
  async getSecret<T = string>(secretName: string): Promise<T> {
    // ตรวจสอบ cache
    const cached = this.cache.get(secretName);
    if (cached && Date.now() < cached.expiry) {
      return cached.value as T;
    }

    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await this.client.send(command);

    let value: T;
    if (response.SecretString) {
      try {
        value = JSON.parse(response.SecretString) as T;
      } catch {
        value = response.SecretString as unknown as T;
      }
    } else if (response.SecretBinary) {
      value = Buffer.from(response.SecretBinary).toString('utf-8') as unknown as T;
    } else {
      throw new Error(`ไม่พบ secret: ${secretName}`);
    }

    // บันทึก cache
    this.cache.set(secretName, {
      value,
      expiry: Date.now() + this.cacheTtl
    });

    return value;
  }

  // สร้าง secret ใหม่
  async createSecret(
    name: string,
    value: string | object,
    description?: string
  ): Promise<string> {
    const command = new CreateSecretCommand({
      Name: name,
      SecretString: typeof value === 'string' ? value : JSON.stringify(value),
      Description: description
    });

    const response = await this.client.send(command);
    return response.ARN ?? '';
  }

  // อัปเดต secret
  async updateSecret(name: string, value: string | object): Promise<void> {
    const command = new UpdateSecretCommand({
      SecretId: name,
      SecretString: typeof value === 'string' ? value : JSON.stringify(value)
    });

    await this.client.send(command);
    this.cache.delete(name);
  }
}

// การใช้งาน
interface DatabaseSecrets {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

async function getDbConnection() {
  const secretsManager = new SecretsManagerService();
  const secrets = await secretsManager.getSecret<DatabaseSecrets>('myapp/database');
  
  return {
    host: secrets.host,
    port: secrets.port,
    user: secrets.username,
    password: secrets.password,
    database: secrets.database
  };
}
```

---

## 7. AWS CDK กับ TypeScript

### 7.1 ติดตั้งและตั้งค่า CDK

```bash
npm install -g aws-cdk
npm install aws-cdk-lib constructs
cdk init app --language typescript
```

### 7.2 CDK Stack พื้นฐาน

```typescript
// lib/my-app-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';
import * as path from 'path';

export class MyAppStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 Bucket
    const uploadBucket = new s3.Bucket(this, 'UploadBucket', {
      bucketName: `myapp-uploads-${this.account}`,
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
      cors: [
        {
          allowedMethods: [s3.HttpMethods.GET, s3.HttpMethods.PUT],
          allowedOrigins: ['*'],
          allowedHeaders: ['*'],
          maxAge: 3600
        }
      ]
    });

    // DynamoDB Table
    const usersTable = new dynamodb.Table(this, 'UsersTable', {
      tableName: 'users',
      partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      encryption: dynamodb.TableEncryption.AWS_MANAGED,
      pointInTimeRecovery: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY
    });

    // GSI สำหรับค้นหาตาม email
    usersTable.addGlobalSecondaryIndex({
      indexName: 'GSI1',
      partitionKey: { name: 'GSI1PK', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'GSI1SK', type: dynamodb.AttributeType.STRING }
    });

    // SQS Queue
    const deadLetterQueue = new sqs.Queue(this, 'DeadLetterQueue', {
      queueName: 'myapp-dlq',
      retentionPeriod: cdk.Duration.days(14)
    });

    const processingQueue = new sqs.Queue(this, 'ProcessingQueue', {
      queueName: 'myapp-processing',
      visibilityTimeout: cdk.Duration.seconds(30),
      deadLetterQueue: {
        queue: deadLetterQueue,
        maxReceiveCount: 3
      }
    });

    // Lambda Function
    const getUserFunction = new lambda.Function(this, 'GetUserFunction', {
      functionName: 'myapp-get-user',
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'get-user.handler',
      code: lambda.Code.fromAsset(path.join(__dirname, '../dist/lambda/users')),
      timeout: cdk.Duration.seconds(30),
      memorySize: 256,
      environment: {
        TABLE_NAME: usersTable.tableName,
        REGION: this.region
      },
      tracing: lambda.Tracing.ACTIVE
    });

    // ให้สิทธิ์ Lambda อ่าน DynamoDB
    usersTable.grantReadData(getUserFunction);

    const createUserFunction = new lambda.Function(this, 'CreateUserFunction', {
      functionName: 'myapp-create-user',
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'create-user.handler',
      code: lambda.Code.fromAsset(path.join(__dirname, '../dist/lambda/users')),
      timeout: cdk.Duration.seconds(30),
      memorySize: 256,
      environment: {
        TABLE_NAME: usersTable.tableName,
        QUEUE_URL: processingQueue.queueUrl
      }
    });

    usersTable.grantWriteData(createUserFunction);
    processingQueue.grantSendMessages(createUserFunction);

    // API Gateway
    const api = new apigateway.RestApi(this, 'MyApi', {
      restApiName: 'MyApp API',
      description: 'API สำหรับ MyApp',
      deployOptions: {
        stageName: 'v1',
        tracingEnabled: true,
        loggingLevel: apigateway.MethodLoggingLevel.INFO
      },
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS
      }
    });

    const users = api.root.addResource('users');
    users.addMethod('GET', new apigateway.LambdaIntegration(getUserFunction));
    users.addMethod('POST', new apigateway.LambdaIntegration(createUserFunction));

    const user = users.addResource('{userId}');
    user.addMethod('GET', new apigateway.LambdaIntegration(getUserFunction));

    // Outputs
    new cdk.CfnOutput(this, 'ApiUrl', {
      value: api.url,
      description: 'API Gateway URL'
    });

    new cdk.CfnOutput(this, 'BucketName', {
      value: uploadBucket.bucketName,
      description: 'S3 Bucket Name'
    });

    new cdk.CfnOutput(this, 'TableName', {
      value: usersTable.tableName,
      description: 'DynamoDB Table Name'
    });
  }
}
```

### 7.3 CDK App Entry Point

```typescript
// bin/my-app.ts
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { MyAppStack } from '../lib/my-app-stack';

const app = new cdk.App();

// Deploy ไปหลาย environments
const envDev = {
  account: process.env.CDK_DEFAULT_ACCOUNT,
  region: 'ap-southeast-1'
};

const envProd = {
  account: process.env.PROD_ACCOUNT,
  region: 'ap-southeast-1'
};

new MyAppStack(app, 'MyAppDev', {
  env: envDev,
  tags: {
    Environment: 'dev',
    Project: 'MyApp'
  }
});

new MyAppStack(app, 'MyAppProd', {
  env: envProd,
  tags: {
    Environment: 'prod',
    Project: 'MyApp'
  }
});
```

---

## 8. Lambda Layers กับ TypeScript

```typescript
// lib/lambda-layers-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Construct } from 'constructs';

export class LambdaLayersStack extends cdk.Stack {
  public readonly commonLayer: lambda.LayerVersion;
  public readonly depsLayer: lambda.LayerVersion;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Layer สำหรับ shared utilities
    this.commonLayer = new lambda.LayerVersion(this, 'CommonLayer', {
      code: lambda.Code.fromAsset('layers/common'),
      compatibleRuntimes: [lambda.Runtime.NODEJS_18_X],
      description: 'Common utilities layer'
    });

    // Layer สำหรับ dependencies ขนาดใหญ่
    this.depsLayer = new lambda.LayerVersion(this, 'DepsLayer', {
      code: lambda.Code.fromAsset('layers/deps'),
      compatibleRuntimes: [lambda.Runtime.NODEJS_18_X],
      description: 'Node.js dependencies layer'
    });
  }
}
```

---

## 9. IAM Types และ Permissions

```typescript
// lib/iam-policies.ts
import * as iam from 'aws-cdk-lib/aws-iam';

// สร้าง custom policy สำหรับ Lambda
export function createLambdaPolicy(
  resources: {
    dynamoTableArns?: string[];
    s3BucketArns?: string[];
    sqsQueueArns?: string[];
    snsTopicArns?: string[];
    secretArns?: string[];
  }
): iam.PolicyDocument {
  const statements: iam.PolicyStatement[] = [];

  if (resources.dynamoTableArns?.length) {
    statements.push(
      new iam.PolicyStatement({
        actions: [
          'dynamodb:GetItem',
          'dynamodb:PutItem',
          'dynamodb:UpdateItem',
          'dynamodb:DeleteItem',
          'dynamodb:Query',
          'dynamodb:Scan',
          'dynamodb:BatchGetItem',
          'dynamodb:BatchWriteItem',
          'dynamodb:TransactWriteItems'
        ],
        resources: [
          ...resources.dynamoTableArns,
          ...resources.dynamoTableArns.map(arn => `${arn}/index/*`)
        ]
      })
    );
  }

  if (resources.s3BucketArns?.length) {
    statements.push(
      new iam.PolicyStatement({
        actions: [
          's3:GetObject',
          's3:PutObject',
          's3:DeleteObject',
          's3:ListBucket'
        ],
        resources: [
          ...resources.s3BucketArns,
          ...resources.s3BucketArns.map(arn => `${arn}/*`)
        ]
      })
    );
  }

  if (resources.sqsQueueArns?.length) {
    statements.push(
      new iam.PolicyStatement({
        actions: [
          'sqs:SendMessage',
          'sqs:ReceiveMessage',
          'sqs:DeleteMessage',
          'sqs:GetQueueAttributes'
        ],
        resources: resources.sqsQueueArns
      })
    );
  }

  if (resources.secretArns?.length) {
    statements.push(
      new iam.PolicyStatement({
        actions: ['secretsmanager:GetSecretValue'],
        resources: resources.secretArns
      })
    );
  }

  return new iam.PolicyDocument({ statements });
}
```

---

## 10. Deployment TypeScript Apps ไปยัง AWS

### 10.1 tsconfig สำหรับ Lambda

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### 10.2 Build Script

```typescript
// scripts/build.ts
import * as esbuild from 'esbuild';
import * as path from 'path';
import * as fs from 'fs';
import * as glob from 'glob';

async function build() {
  // ค้นหา Lambda handler files
  const handlerFiles = glob.sync('src/lambda/**/*.ts', {
    ignore: ['**/*.test.ts', '**/*.spec.ts']
  });

  console.log(`พบ Lambda handlers: ${handlerFiles.length} ไฟล์`);

  const buildPromises = handlerFiles.map(async (file) => {
    const relativePath = path.relative('src/lambda', file);
    const outputFile = path.join('dist/lambda', relativePath.replace('.ts', '.js'));

    await esbuild.build({
      entryPoints: [file],
      bundle: true,
      platform: 'node',
      target: 'node18',
      outfile: outputFile,
      external: ['aws-sdk', '@aws-sdk/*'],
      minify: process.env.NODE_ENV === 'production',
      sourcemap: true,
      treeShaking: true
    });

    console.log(`Built: ${file} -> ${outputFile}`);
  });

  await Promise.all(buildPromises);
  console.log('Build เสร็จสิ้น!');
}

build().catch(console.error);
```

---

## สรุปบทที่ 64

ในบทนี้เราได้เรียนรู้:

1. **AWS SDK v3** - การใช้งานและตั้งค่า
2. **S3 Operations** - upload, download, presigned URLs
3. **DynamoDB** - CRUD operations, repository pattern
4. **Lambda Functions** - handler types, implementation
5. **SQS** - queue operations, batch processing
6. **SNS** - publish/subscribe pattern
7. **Secrets Manager** - จัดการ secrets อย่างปลอดภัย
8. **AWS CDK** - infrastructure as code ด้วย TypeScript
9. **Lambda Layers** - shared code และ dependencies
10. **IAM Types** - จัดการ permissions

---

## แบบฝึกหัด

1. สร้าง S3 service ที่รองรับ multipart upload
2. ออกแบบ DynamoDB schema สำหรับ multi-tenant app
3. สร้าง CDK stack สำหรับ full-stack application
4. Implement SQS processor สำหรับ email notifications
5. ใช้ CDK pipelines สำหรับ CI/CD

---

*ต่อไป: ส่วนที่ 65 - Serverless with TypeScript*
