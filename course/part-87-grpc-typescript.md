# Part 87: gRPC กับ TypeScript

## บทนำ

gRPC (Google Remote Procedure Call) คือ framework สำหรับ RPC (Remote Procedure Call) ที่พัฒนาโดย Google ใช้ Protocol Buffers เป็น interface definition language และรองรับการสื่อสารแบบ streaming

---

## 1. gRPC Overview

### 1.1 เมื่อไหร่ควรใช้ gRPC

```
HTTP/REST                    gRPC
─────────────────────────    ───────────────────────────────
JSON text format             Binary Protocol Buffers
Schema-optional              Strongly typed schema
One request/response         Streaming support (4 modes)
Browser native               Requires gRPC-web for browsers
Human readable               Requires tooling to inspect
Loose coupling               Tight contract via .proto
~latency OK                  Lower latency, smaller payload
```

**ใช้ gRPC เมื่อ:**
- Microservices ที่ต้องการ performance สูง
- Real-time streaming data
- ต้องการ strongly typed API contracts
- polyglot environments (Go, Java, Python, TypeScript ทำงานร่วมกัน)

### 1.2 ติดตั้ง Dependencies

```json
{
  "dependencies": {
    "@grpc/grpc-js": "^1.10.1",
    "@grpc/proto-loader": "^0.7.10",
    "google-protobuf": "^3.21.4"
  },
  "devDependencies": {
    "grpc-tools": "^1.12.4",
    "ts-protoc-gen": "^0.15.0",
    "@types/google-protobuf": "^3.15.12",
    "protoc": "^1.0.4"
  }
}
```

---

## 2. Protocol Buffers (.proto)

### 2.1 Syntax เบื้องต้น

```protobuf
// proto/user.proto
syntax = "proto3";

package user;

option java_package = "com.example.user";
option java_outer_classname = "UserProto";

// Message definition
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  UserStatus status = 5;
  repeated string roles = 6;
  Address address = 7;
  google.protobuf.Timestamp created_at = 8;
}

// Enum
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0; // default ต้องเป็น 0
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_BANNED = 3;
}

// Nested message
message Address {
  string street = 1;
  string city = 2;
  string province = 3;
  string postal_code = 4;
  string country = 5;
}

// Request/Response messages
message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  bool has_more = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
}

message DeleteUserRequest {
  string user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  
  // Server-side streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client-side streaming
  rpc BulkCreateUsers(stream CreateUserRequest) returns (ListUsersResponse);
  
  // Bidirectional streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

### 2.2 FinTech Proto

```protobuf
// proto/payment.proto
syntax = "proto3";

package payment;

import "google/protobuf/timestamp.proto";

message Money {
  string amount = 1;  // decimal string เช่น "1000.50"
  string currency = 2; // "THB", "USD"
}

message TransferRequest {
  string from_account_id = 1;
  string to_account_id = 2;
  Money amount = 3;
  string description = 4;
  string idempotency_key = 5; // ป้องกัน duplicate
}

message TransferResponse {
  string transaction_id = 1;
  string reference_id = 2;
  TransferStatus status = 3;
  Money fee = 4;
  Money from_balance = 5;
  google.protobuf.Timestamp completed_at = 6;
}

enum TransferStatus {
  TRANSFER_STATUS_UNSPECIFIED = 0;
  TRANSFER_STATUS_SUCCESS = 1;
  TRANSFER_STATUS_FAILED = 2;
  TRANSFER_STATUS_PENDING = 3;
}

message GetBalanceRequest {
  string account_id = 1;
}

message GetBalanceResponse {
  string account_id = 1;
  Money balance = 2;
  Money available_balance = 3;
}

message TransactionStreamRequest {
  string account_id = 1;
  google.protobuf.Timestamp from_date = 2;
  google.protobuf.Timestamp to_date = 3;
}

message Transaction {
  string id = 1;
  string reference_id = 2;
  string type = 3;
  Money amount = 4;
  string description = 5;
  google.protobuf.Timestamp created_at = 6;
}

service PaymentService {
  // Unary: โอนเงิน
  rpc Transfer(TransferRequest) returns (TransferResponse);
  
  // Unary: ดูยอดเงิน
  rpc GetBalance(GetBalanceRequest) returns (GetBalanceResponse);
  
  // Server streaming: stream ประวัติธุรกรรม
  rpc StreamTransactions(TransactionStreamRequest) returns (stream Transaction);
  
  // Client streaming: bulk import transactions
  rpc BulkImportTransactions(stream Transaction) returns (TransferResponse);
  
  // Bidirectional streaming: real-time payment notification
  rpc PaymentNotifications(stream GetBalanceRequest) returns (stream Transaction);
}
```

---

## 3. Generate TypeScript จาก .proto

### 3.1 สคริปต์ generate

```bash
#!/bin/bash
# scripts/generate-proto.sh

PROTO_DIR="./proto"
OUT_DIR="./src/generated"

mkdir -p $OUT_DIR

# Generate JavaScript + TypeScript definitions
protoc \
  --plugin=protoc-gen-ts=./node_modules/.bin/protoc-gen-ts \
  --js_out=import_style=commonjs,binary:$OUT_DIR \
  --ts_out=service=grpc-node,mode=grpc-js:$OUT_DIR \
  --grpc_out=grpc_js:$OUT_DIR \
  --plugin=protoc-gen-grpc=./node_modules/.bin/grpc_tools_node_protoc_plugin \
  -I $PROTO_DIR \
  $PROTO_DIR/*.proto

echo "Generated TypeScript files in $OUT_DIR"
```

### 3.2 package.json scripts

```json
{
  "scripts": {
    "proto:generate": "bash scripts/generate-proto.sh",
    "build": "tsc",
    "start:server": "ts-node src/server.ts",
    "start:client": "ts-node src/client.ts"
  }
}
```

### 3.3 ใช้ @grpc/proto-loader โดยตรง (ไม่ต้อง generate)

```typescript
// src/proto-loader.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

const PROTO_PATH = path.join(__dirname, '../proto/payment.proto');

export const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
  includeDirs: [path.join(__dirname, '../proto')],
});

export const protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
export const paymentProto = (protoDescriptor as any).payment;
```

---

## 4. Unary RPC

### 4.1 Server Implementation

```typescript
// src/server/payment.server.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

// Type definitions สำหรับ service implementations
interface TransferRequest {
  from_account_id: string;
  to_account_id: string;
  amount: { amount: string; currency: string };
  description: string;
  idempotency_key: string;
}

interface TransferResponse {
  transaction_id: string;
  reference_id: string;
  status: string;
  fee: { amount: string; currency: string };
  from_balance: { amount: string; currency: string };
}

type UnaryCall<TReq, TRes> = grpc.ServerUnaryCall<TReq, TRes>;
type SendUnaryData<T> = grpc.sendUnaryData<T>;

// Service implementation
const paymentServiceImpl = {
  // Unary RPC
  Transfer: async (
    call: UnaryCall<TransferRequest, TransferResponse>,
    callback: SendUnaryData<TransferResponse>
  ) => {
    try {
      const { from_account_id, to_account_id, amount, description } = call.request;
      
      console.log(`รับคำขอโอนเงิน: ${amount.amount} ${amount.currency}`);
      console.log(`จาก: ${from_account_id} ไป: ${to_account_id}`);

      // ตรวจสอบ idempotency key
      const idempotencyKey = call.request.idempotency_key;
      const existing = await checkIdempotency(idempotencyKey);
      if (existing) {
        return callback(null, existing);
      }

      // Validate
      if (!from_account_id || !to_account_id) {
        return callback({
          code: grpc.status.INVALID_ARGUMENT,
          message: 'กรุณาระบุบัญชีต้นทางและปลายทาง',
        });
      }

      if (parseFloat(amount.amount) <= 0) {
        return callback({
          code: grpc.status.INVALID_ARGUMENT,
          message: 'จำนวนเงินต้องมากกว่า 0',
        });
      }

      // Process transfer
      const result: TransferResponse = {
        transaction_id: `TXN-${Date.now()}`,
        reference_id: `REF-${Math.random().toString(36).slice(2, 8).toUpperCase()}`,
        status: 'TRANSFER_STATUS_SUCCESS',
        fee: { amount: '0.00', currency: amount.currency },
        from_balance: { amount: '35000.00', currency: amount.currency },
      };

      // บันทึก idempotency result
      await saveIdempotencyResult(idempotencyKey, result);

      callback(null, result);
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: `เกิดข้อผิดพลาด: ${(error as Error).message}`,
      });
    }
  },

  GetBalance: async (
    call: grpc.ServerUnaryCall<{ account_id: string }, any>,
    callback: grpc.sendUnaryData<any>
  ) => {
    const { account_id } = call.request;
    
    // ดึงข้อมูล balance จาก database
    const balance = await getAccountBalance(account_id);
    
    if (!balance) {
      return callback({
        code: grpc.status.NOT_FOUND,
        message: `ไม่พบบัญชี: ${account_id}`,
      });
    }

    callback(null, {
      account_id,
      balance: { amount: balance.total, currency: 'THB' },
      available_balance: { amount: balance.available, currency: 'THB' },
    });
  },
};

async function checkIdempotency(key: string): Promise<TransferResponse | null> {
  // TODO: ตรวจสอบใน Redis
  return null;
}

async function saveIdempotencyResult(key: string, result: TransferResponse): Promise<void> {
  // TODO: บันทึกใน Redis พร้อม TTL 24 ชั่วโมง
}

async function getAccountBalance(accountId: string): Promise<{ total: string; available: string } | null> {
  // TODO: ดึงจาก database
  return { total: '50000.00', available: '49500.00' };
}
```

### 4.2 เริ่ม gRPC Server

```typescript
// src/server/index.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

const PROTO_PATH = path.join(__dirname, '../../proto/payment.proto');

const packageDef = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const protoDescriptor = grpc.loadPackageDefinition(packageDef) as any;
const PaymentService = protoDescriptor.payment.PaymentService;

function createServer(): grpc.Server {
  const server = new grpc.Server();

  server.addService(PaymentService.service, paymentServiceImpl);

  return server;
}

const server = createServer();
const PORT = process.env.GRPC_PORT || '50051';

server.bindAsync(
  `0.0.0.0:${PORT}`,
  grpc.ServerCredentials.createInsecure(), // Development only
  (err, port) => {
    if (err) {
      console.error('ไม่สามารถเริ่ม server ได้:', err);
      process.exit(1);
    }
    console.log(`gRPC Server ทำงานที่ port ${port}`);
    server.start();
  }
);

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('กำลังปิด gRPC Server...');
  server.tryShutdown((err) => {
    if (err) {
      console.error('เกิดข้อผิดพลาดขณะปิด:', err);
    }
    process.exit(0);
  });
});
```

### 4.3 Client Implementation

```typescript
// src/client/payment.client.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';
import { promisify } from 'util';

const PROTO_PATH = path.join(__dirname, '../../proto/payment.proto');

const packageDef = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const protoDescriptor = grpc.loadPackageDefinition(packageDef) as any;

// สร้าง client
const client = new protoDescriptor.payment.PaymentService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Promisify unary calls
const transfer = promisify(client.Transfer.bind(client));
const getBalance = promisify(client.GetBalance.bind(client));

export async function makeTransfer(
  fromAccountId: string,
  toAccountId: string,
  amount: string,
  currency: string,
  description: string
): Promise<any> {
  return transfer({
    from_account_id: fromAccountId,
    to_account_id: toAccountId,
    amount: { amount, currency },
    description,
    idempotency_key: `${fromAccountId}-${Date.now()}`,
  });
}

export async function checkBalance(accountId: string): Promise<any> {
  return getBalance({ account_id: accountId });
}

// ทดสอบ
async function runTest() {
  console.log('=== ทดสอบ gRPC Payment Client ===\n');

  // ตรวจสอบยอดเงิน
  const balance = await checkBalance('account-001');
  console.log('ยอดเงิน:', balance);

  // โอนเงิน
  const result = await makeTransfer(
    'account-001',
    'account-002',
    '5000.00',
    'THB',
    'โอนค่าอาหาร'
  );
  console.log('ผลการโอน:', result);
}

runTest().catch(console.error);
```

---

## 5. Server-side Streaming

### 5.1 Stream Transactions จาก Server

```typescript
// ใน paymentServiceImpl
StreamTransactions: (
  call: grpc.ServerWritableStream<
    { account_id: string; from_date: any; to_date: any },
    any
  >
) => {
  const { account_id } = call.request;
  
  console.log(`เริ่ม stream ธุรกรรมของ: ${account_id}`);

  // จำลองการส่ง stream ข้อมูล
  const transactions = [
    { id: 'TXN001', type: 'TRANSFER_IN', amount: { amount: '1000', currency: 'THB' } },
    { id: 'TXN002', type: 'TRANSFER_OUT', amount: { amount: '500', currency: 'THB' } },
    { id: 'TXN003', type: 'DEPOSIT', amount: { amount: '5000', currency: 'THB' } },
  ];

  let index = 0;
  
  const sendNext = () => {
    if (index >= transactions.length) {
      // ปิด stream เมื่อส่งครบ
      call.end();
      return;
    }

    const txn = transactions[index];
    const canContinue = call.write({
      ...txn,
      reference_id: `REF-${txn.id}`,
      description: 'ธุรกรรมตัวอย่าง',
      created_at: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
    });

    index++;

    if (canContinue) {
      // ส่งทันทีถ้า buffer ยังรับได้
      setImmediate(sendNext);
    } else {
      // รอ drain event ก่อนส่งต่อ (backpressure)
      call.once('drain', sendNext);
    }
  };

  // Handle client disconnect
  call.on('cancelled', () => {
    console.log(`Client ยกเลิก stream สำหรับ: ${account_id}`);
  });

  sendNext();
},
```

### 5.2 Client รับ Server Streaming

```typescript
export function streamTransactions(
  accountId: string,
  fromDate: Date,
  toDate: Date,
  onTransaction: (txn: any) => void,
  onError: (err: Error) => void,
  onEnd: () => void
): grpc.ClientReadableStream<any> {
  const stream = client.StreamTransactions({
    account_id: accountId,
    from_date: { 
      seconds: Math.floor(fromDate.getTime() / 1000), 
      nanos: 0 
    },
    to_date: { 
      seconds: Math.floor(toDate.getTime() / 1000), 
      nanos: 0 
    },
  });

  stream.on('data', (transaction: any) => {
    console.log('รับธุรกรรม:', transaction.id);
    onTransaction(transaction);
  });

  stream.on('error', (err: Error) => {
    console.error('Stream error:', err.message);
    onError(err);
  });

  stream.on('end', () => {
    console.log('Stream สิ้นสุด');
    onEnd();
  });

  return stream;
}

// ใช้งาน
const stream = streamTransactions(
  'account-001',
  new Date('2024-01-01'),
  new Date('2024-01-31'),
  (txn) => {
    console.log(`ธุรกรรม: ${txn.id} - ${txn.amount.amount} ${txn.amount.currency}`);
  },
  (err) => console.error('Error:', err),
  () => console.log('รับข้อมูลครบแล้ว')
);

// ยกเลิก stream ได้
setTimeout(() => {
  stream.cancel();
  console.log('ยกเลิก stream แล้ว');
}, 5000);
```

---

## 6. Client-side Streaming

### 6.1 Bulk Import Transactions

```typescript
// Server
BulkImportTransactions: (
  call: grpc.ServerReadableStream<any, any>,
  callback: grpc.sendUnaryData<any>
) => {
  const transactions: any[] = [];
  let totalAmount = 0;

  call.on('data', (transaction: any) => {
    transactions.push(transaction);
    totalAmount += parseFloat(transaction.amount?.amount || '0');
    console.log(`รับธุรกรรม #${transactions.length}: ${transaction.id}`);
  });

  call.on('end', () => {
    console.log(`รับธุรกรรมทั้งหมด ${transactions.length} รายการ`);
    
    // บันทึกทั้งหมดลง database
    callback(null, {
      transaction_id: `BULK-${Date.now()}`,
      reference_id: `BULK-REF-${Math.random().toString(36).slice(2, 6)}`,
      status: 'TRANSFER_STATUS_SUCCESS',
      fee: { amount: '0', currency: 'THB' },
      from_balance: { amount: totalAmount.toFixed(2), currency: 'THB' },
    });
  });

  call.on('error', (err: Error) => {
    console.error('Client stream error:', err);
  });
},
```

### 6.2 Client ส่ง Stream

```typescript
export async function bulkImportTransactions(
  transactions: Array<{ id: string; amount: string; type: string; description: string }>
): Promise<any> {
  return new Promise((resolve, reject) => {
    const stream = client.BulkImportTransactions(
      (err: grpc.ServiceError | null, response: any) => {
        if (err) {
          reject(err);
        } else {
          resolve(response);
        }
      }
    );

    // ส่งทีละ record
    for (const txn of transactions) {
      stream.write({
        id: txn.id,
        reference_id: `REF-${txn.id}`,
        type: txn.type,
        amount: { amount: txn.amount, currency: 'THB' },
        description: txn.description,
        created_at: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
      });
    }

    // ปิด stream เมื่อส่งครบ
    stream.end();
  });
}

// ใช้งาน
const txnsToImport = [
  { id: 'TXN001', amount: '1000', type: 'DEPOSIT', description: 'ฝากเงิน' },
  { id: 'TXN002', amount: '500', type: 'WITHDRAWAL', description: 'ถอนเงิน' },
  { id: 'TXN003', amount: '2000', type: 'TRANSFER_IN', description: 'รับโอน' },
];

const result = await bulkImportTransactions(txnsToImport);
console.log('Import result:', result);
```

---

## 7. Bidirectional Streaming

### 7.1 Real-time Payment Notifications

```typescript
// Server
PaymentNotifications: (
  call: grpc.ServerDuplexStream<any, any>
) => {
  console.log('Client เชื่อมต่อสำหรับ payment notifications');

  // รับ request จาก client
  call.on('data', (request: { account_id: string }) => {
    const { account_id } = request;
    console.log(`Subscribing to notifications for: ${account_id}`);

    // ส่ง notification กลับ (simulate real-time events)
    const sendNotification = (data: any) => {
      if (!call.cancelled) {
        call.write(data);
      }
    };

    // Subscribe to account events (ในระบบจริงใช้ event bus/Redis pub/sub)
    const interval = setInterval(() => {
      sendNotification({
        id: `NOTIF-${Date.now()}`,
        reference_id: `REF-${Math.random().toString(36).slice(2)}`,
        type: 'TRANSFER_IN',
        amount: { amount: (Math.random() * 1000).toFixed(2), currency: 'THB' },
        description: `รับโอนเงิน (Account: ${account_id})`,
        created_at: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
      });
    }, 3000);

    // Cleanup เมื่อ client disconnect
    call.on('cancelled', () => {
      clearInterval(interval);
      console.log(`Client disconnect: ${account_id}`);
    });
  });

  call.on('end', () => {
    call.end();
  });

  call.on('error', (err: Error) => {
    console.error('Duplex stream error:', err);
  });
},
```

### 7.2 Client Bidirectional Stream

```typescript
export function subscribeToNotifications(
  accountIds: string[],
  onNotification: (notification: any) => void
): grpc.ClientDuplexStream<any, any> {
  const stream = client.PaymentNotifications();

  stream.on('data', (notification: any) => {
    onNotification(notification);
  });

  stream.on('error', (err: Error) => {
    console.error('Notification stream error:', err);
  });

  stream.on('end', () => {
    console.log('Notification stream สิ้นสุด');
  });

  // ส่ง subscription requests
  for (const accountId of accountIds) {
    stream.write({ account_id: accountId });
  }

  return stream;
}

// ใช้งาน
const notifStream = subscribeToNotifications(
  ['account-001', 'account-002'],
  (notification) => {
    console.log(`📬 การแจ้งเตือน: ${notification.type}`);
    console.log(`   จำนวน: ${notification.amount.amount} ${notification.amount.currency}`);
    console.log(`   รายละเอียด: ${notification.description}`);
  }
);

// ยกเลิกหลัง 30 วินาที
setTimeout(() => {
  notifStream.end();
}, 30000);
```

---

## 8. Metadata และ Interceptors

### 8.1 การส่ง Metadata

```typescript
// Client: ส่ง metadata
const metadata = new grpc.Metadata();
metadata.set('authorization', 'Bearer eyJhbGci...');
metadata.set('x-request-id', '12345');
metadata.set('x-trace-id', 'trace-abc');

const result = await new Promise((resolve, reject) => {
  client.GetBalance(
    { account_id: 'account-001' },
    metadata, // ส่ง metadata พร้อมกับ request
    (err: grpc.ServiceError | null, response: any) => {
      if (err) reject(err);
      else resolve(response);
    }
  );
});
```

### 8.2 Server อ่าน Metadata

```typescript
GetBalance: async (
  call: grpc.ServerUnaryCall<any, any>,
  callback: grpc.sendUnaryData<any>
) => {
  // อ่าน metadata
  const metadata = call.metadata;
  const authToken = metadata.get('authorization')[0] as string;
  const requestId = metadata.get('x-request-id')[0] as string;
  
  console.log(`Request ID: ${requestId}`);
  
  // Validate auth token
  if (!authToken || !authToken.startsWith('Bearer ')) {
    return callback({
      code: grpc.status.UNAUTHENTICATED,
      message: 'กรุณาเข้าสู่ระบบก่อน',
    });
  }

  const token = authToken.replace('Bearer ', '');
  // TODO: verify JWT token
  
  // ส่ง trailing metadata กลับ
  const responseMetadata = new grpc.Metadata();
  responseMetadata.set('x-processing-time', '25ms');
  call.sendMetadata(responseMetadata);

  callback(null, { account_id: call.request.account_id, balance: { amount: '50000', currency: 'THB' } });
},
```

### 8.3 gRPC Interceptor (Client-side)

```typescript
// Client interceptor สำหรับ logging
function loggingInterceptor(
  options: grpc.InterceptorOptions,
  nextCall: (options: grpc.InterceptorOptions) => grpc.InterceptingCall
): grpc.InterceptingCall {
  const startTime = Date.now();
  
  return new grpc.InterceptingCall(nextCall(options), {
    start(metadata, listener, next) {
      console.log(`→ gRPC Call: ${options.method_definition?.path}`);
      
      // เพิ่ม metadata อัตโนมัติ
      metadata.set('x-request-id', `req-${Date.now()}`);
      metadata.set('x-client-version', '1.0.0');
      
      next(metadata, {
        onReceiveMessage(message, nextMessage) {
          console.log(`← gRPC Response received`);
          nextMessage(message);
        },
        onReceiveStatus(status, nextStatus) {
          const duration = Date.now() - startTime;
          console.log(`← gRPC Status: ${status.code} (${duration}ms)`);
          nextStatus(status);
        },
      });
    },
    sendMessage(message, next) {
      console.log(`→ Sending message`);
      next(message);
    },
  });
}

// สร้าง client พร้อม interceptor
const clientWithInterceptor = new protoDescriptor.payment.PaymentService(
  'localhost:50051',
  grpc.credentials.createInsecure(),
  {
    interceptors: [loggingInterceptor],
  }
);
```

---

## 9. Authentication กับ JWT

### 9.1 JWT Authentication Interceptor

```typescript
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'secret-key';

// Server-side auth interceptor
function createAuthInterceptor(skipMethods: string[] = []) {
  return {
    methodName: '*',
    interceptor: (
      call: grpc.ServerUnaryCall<any, any>,
      next: (err?: grpc.ServiceError) => void
    ) => {
      const methodPath = (call as any).call.handler.path as string;
      
      // ข้าม method ที่ไม่ต้อง auth
      if (skipMethods.some(m => methodPath.endsWith(m))) {
        return next();
      }

      const metadata = call.metadata;
      const authHeader = metadata.get('authorization')[0] as string;

      if (!authHeader?.startsWith('Bearer ')) {
        return next({
          code: grpc.status.UNAUTHENTICATED,
          message: 'กรุณาระบุ Authorization token',
        });
      }

      const token = authHeader.replace('Bearer ', '');

      try {
        const decoded = jwt.verify(token, JWT_SECRET) as { userId: string; roles: string[] };
        
        // แนบข้อมูล user ไปกับ request
        (call as any).user = decoded;
        
        next();
      } catch (err) {
        next({
          code: grpc.status.UNAUTHENTICATED,
          message: 'Token ไม่ถูกต้องหรือหมดอายุ',
        });
      }
    },
  };
}
```

### 9.2 Client ส่ง JWT

```typescript
export class AuthenticatedPaymentClient {
  private client: any;
  private token: string;

  constructor(serverAddress: string, token: string) {
    this.token = token;
    this.client = new protoDescriptor.payment.PaymentService(
      serverAddress,
      grpc.credentials.createInsecure()
    );
  }

  private getMetadata(): grpc.Metadata {
    const metadata = new grpc.Metadata();
    metadata.set('authorization', `Bearer ${this.token}`);
    return metadata;
  }

  async getBalance(accountId: string): Promise<any> {
    return new Promise((resolve, reject) => {
      this.client.GetBalance(
        { account_id: accountId },
        this.getMetadata(),
        (err: grpc.ServiceError | null, response: any) => {
          if (err) reject(err);
          else resolve(response);
        }
      );
    });
  }

  async transfer(
    fromAccountId: string,
    toAccountId: string,
    amount: string
  ): Promise<any> {
    return new Promise((resolve, reject) => {
      this.client.Transfer(
        {
          from_account_id: fromAccountId,
          to_account_id: toAccountId,
          amount: { amount, currency: 'THB' },
          description: 'โอนเงิน',
          idempotency_key: `${fromAccountId}-${Date.now()}`,
        },
        this.getMetadata(),
        (err: grpc.ServiceError | null, response: any) => {
          if (err) reject(err);
          else resolve(response);
        }
      );
    });
  }
}
```

---

## 10. Error Handling กับ gRPC Status Codes

### 10.1 gRPC Status Codes

```typescript
// gRPC status codes ที่ใช้บ่อย
const gRPCStatusCodes = {
  OK: 0,                // สำเร็จ
  CANCELLED: 1,         // ถูกยกเลิก
  UNKNOWN: 2,           // ไม่ทราบสาเหตุ
  INVALID_ARGUMENT: 3,  // ข้อมูลไม่ถูกต้อง
  DEADLINE_EXCEEDED: 4, // หมดเวลา
  NOT_FOUND: 5,         // ไม่พบข้อมูล
  ALREADY_EXISTS: 6,    // มีอยู่แล้ว
  PERMISSION_DENIED: 7, // ไม่มีสิทธิ์
  UNAUTHENTICATED: 16,  // ไม่ได้ login
  RESOURCE_EXHAUSTED: 8,// ทรัพยากรเต็ม (rate limit)
  FAILED_PRECONDITION: 9,// เงื่อนไขไม่ตรง
  INTERNAL: 13,         // Internal server error
  UNAVAILABLE: 14,      // Service ไม่พร้อม
};

// Custom error handler
export class GRPCError extends Error {
  constructor(
    public readonly code: grpc.status,
    message: string,
    public readonly details?: string
  ) {
    super(message);
    this.name = 'GRPCError';
  }

  toGRPCError(): grpc.ServiceError {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      details: this.details || '',
      metadata: new grpc.Metadata(),
    };
  }
}

// Domain errors → gRPC errors
export function mapDomainErrorToGRPC(error: Error): grpc.ServiceError {
  if (error.name === 'InsufficientFundsError') {
    return new GRPCError(
      grpc.status.FAILED_PRECONDITION,
      error.message
    ).toGRPCError();
  }
  if (error.name === 'AccountFrozenError') {
    return new GRPCError(
      grpc.status.PERMISSION_DENIED,
      error.message
    ).toGRPCError();
  }
  if (error.name === 'LimitExceededError') {
    return new GRPCError(
      grpc.status.RESOURCE_EXHAUSTED,
      error.message
    ).toGRPCError();
  }
  // Default
  return {
    name: 'Error',
    message: 'เกิดข้อผิดพลาดภายในระบบ',
    code: grpc.status.INTERNAL,
    details: '',
    metadata: new grpc.Metadata(),
  };
}
```

### 10.2 Client Error Handling

```typescript
async function callWithRetry<T>(
  operation: () => Promise<T>,
  maxRetries = 3,
  delayMs = 1000
): Promise<T> {
  let lastError: Error | undefined;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error: unknown) {
      const grpcError = error as grpc.ServiceError;
      lastError = grpcError;
      
      // Retry เฉพาะ errors ที่ retry ได้
      const retryableStatuses = [
        grpc.status.UNAVAILABLE,
        grpc.status.DEADLINE_EXCEEDED,
        grpc.status.RESOURCE_EXHAUSTED,
      ];
      
      if (!retryableStatuses.includes(grpcError.code)) {
        throw error; // ไม่ retry สำหรับ client errors
      }
      
      if (attempt < maxRetries) {
        const waitMs = delayMs * Math.pow(2, attempt - 1); // exponential backoff
        console.log(`Retry attempt ${attempt}/${maxRetries} ใน ${waitMs}ms...`);
        await new Promise(resolve => setTimeout(resolve, waitMs));
      }
    }
  }
  
  throw lastError;
}

// ใช้งาน
const balance = await callWithRetry(() => paymentClient.getBalance('account-001'));
```

---

## 11. gRPC-web สำหรับ Browsers

### 11.1 กำหนดค่า gRPC-web Proxy

```typescript
// grpc-web ต้องการ proxy (Envoy หรือ grpc-web middleware)
// docker-compose.yml

/*
version: '3'
services:
  grpc-server:
    build: .
    ports:
      - "50051:50051"
  
  envoy-proxy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "8080:8080"  # Browser เชื่อมผ่าน port นี้
*/
```

### 11.2 Browser Client ด้วย grpc-web

```typescript
// ติดตั้ง: npm install grpc-web
// Browser client ใช้ grpc-web package แทน @grpc/grpc-js

import { PaymentServiceClient } from './generated/payment_grpc_web_pb';
import { TransferRequest, GetBalanceRequest } from './generated/payment_pb';

const client = new PaymentServiceClient('http://localhost:8080');

// Unary call
export function getBrowserBalance(accountId: string): Promise<any> {
  return new Promise((resolve, reject) => {
    const request = new GetBalanceRequest();
    request.setAccountId(accountId);

    const metadata = { 'authorization': 'Bearer token123' };

    client.getBalance(request, metadata, (err, response) => {
      if (err) {
        reject(err);
      } else {
        resolve({
          accountId: response.getAccountId(),
          balance: response.getBalance()?.getAmount(),
          currency: response.getBalance()?.getCurrency(),
        });
      }
    });
  });
}

// Server streaming ใน browser
export function streamBrowserTransactions(accountId: string): void {
  const request = new TransactionStreamRequest();
  request.setAccountId(accountId);

  const stream = client.streamTransactions(request, {});

  stream.on('data', (transaction) => {
    console.log('Transaction:', transaction.toObject());
  });

  stream.on('error', (err) => {
    console.error('Error:', err);
  });

  stream.on('end', () => {
    console.log('Stream ended');
  });
}
```

---

## 12. NestJS กับ gRPC Microservices

### 12.1 NestJS gRPC Setup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.GRPC,
      options: {
        package: 'payment',
        protoPath: join(__dirname, '../proto/payment.proto'),
        url: '0.0.0.0:50051',
      },
    }
  );

  await app.listen();
  console.log('gRPC Microservice ทำงานแล้ว');
}

bootstrap();
```

### 12.2 NestJS gRPC Controller

```typescript
// payment.controller.ts
import { Controller } from '@nestjs/common';
import { GrpcMethod, GrpcStreamMethod } from '@nestjs/microservices';
import { Observable, Subject } from 'rxjs';

@Controller()
export class PaymentController {
  // Unary
  @GrpcMethod('PaymentService', 'Transfer')
  async transfer(data: TransferRequest): Promise<TransferResponse> {
    return {
      transaction_id: `TXN-${Date.now()}`,
      reference_id: 'REF-001',
      status: 'TRANSFER_STATUS_SUCCESS',
      fee: { amount: '0', currency: 'THB' },
      from_balance: { amount: '45000', currency: 'THB' },
    };
  }

  // Server streaming
  @GrpcMethod('PaymentService', 'StreamTransactions')
  streamTransactions(data: TransactionStreamRequest): Observable<any> {
    const subject = new Subject<any>();
    
    // Emit transactions
    setTimeout(() => {
      subject.next({ id: 'TXN001', type: 'DEPOSIT', amount: { amount: '1000', currency: 'THB' } });
      subject.next({ id: 'TXN002', type: 'WITHDRAWAL', amount: { amount: '500', currency: 'THB' } });
      subject.complete();
    }, 100);
    
    return subject.asObservable();
  }

  // Bidirectional streaming
  @GrpcStreamMethod('PaymentService', 'PaymentNotifications')
  paymentNotifications(messages: Observable<any>): Observable<any> {
    const subject = new Subject<any>();
    
    messages.subscribe({
      next: (request) => {
        console.log('Subscribe request for:', request.account_id);
        
        // ส่ง notification กลับ
        const interval = setInterval(() => {
          subject.next({
            id: `NOTIF-${Date.now()}`,
            type: 'TRANSFER_IN',
            amount: { amount: '100', currency: 'THB' },
          });
        }, 2000);
        
        // cleanup
        return () => clearInterval(interval);
      },
      error: (err) => subject.error(err),
      complete: () => subject.complete(),
    });
    
    return subject.asObservable();
  }
}
```

---

## สรุปบทที่ 87

ในบทนี้เราได้เรียนรู้:

1. **gRPC Overview** - ทำความเข้าใจว่าเมื่อไหร่ควรใช้ gRPC แทน REST
2. **Protocol Buffers** - เขียน .proto file กำหนด messages และ services
3. **Code Generation** - generate TypeScript จาก .proto
4. **Unary RPC** - Request/Response แบบธรรมดา
5. **Server-side Streaming** - Server ส่งข้อมูลหลาย messages
6. **Client-side Streaming** - Client ส่งข้อมูลหลาย messages
7. **Bidirectional Streaming** - ทั้งสองฝ่ายส่งพร้อมกัน
8. **Metadata** - ส่งข้อมูลเพิ่มเติมนอก message
9. **JWT Authentication** - ตรวจสอบตัวตนผ่าน metadata
10. **Error Handling** - ใช้ gRPC status codes อย่างถูกต้อง
11. **gRPC-web** - ใช้ gRPC ใน browser
12. **NestJS Integration** - ใช้ gRPC กับ NestJS microservices
