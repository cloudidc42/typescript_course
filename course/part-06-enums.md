# Part 6: Enums ใน TypeScript

## บทนำ

Enum (Enumeration) คือชนิดข้อมูลพิเศษที่ช่วยให้เราสามารถกำหนดชุดค่าคงที่ที่มีชื่อและเกี่ยวข้องกันได้อย่างเป็นระเบียบ TypeScript รองรับ Enum หลายรูปแบบ ได้แก่ Numeric, String, Const, และ Heterogeneous Enums ซึ่งแต่ละแบบมีจุดเด่นและการใช้งานที่แตกต่างกัน

---

## 6.1 Numeric Enums

### 6.1.1 การประกาศ Numeric Enum พื้นฐาน

```typescript
// ตัวอย่างที่ 1: Numeric Enum พื้นฐาน
enum Direction {
    North,  // 0
    East,   // 1
    South,  // 2
    West    // 3
}

let myDirection: Direction = Direction.North;
console.log(myDirection);         // 0
console.log(Direction.North);    // 0
console.log(Direction.East);     // 1
console.log(Direction.South);    // 2
console.log(Direction.West);     // 3
```

```typescript
// ตัวอย่างที่ 2: Numeric Enum กับค่าเริ่มต้น
enum StatusCode {
    OK = 200,
    Created = 201,
    NoContent = 204,
    BadRequest = 400,
    Unauthorized = 401,
    Forbidden = 403,
    NotFound = 404,
    InternalServerError = 500
}

function handleResponse(code: StatusCode): string {
    switch (code) {
        case StatusCode.OK:
            return "สำเร็จ";
        case StatusCode.Created:
            return "สร้างข้อมูลแล้ว";
        case StatusCode.NotFound:
            return "ไม่พบข้อมูล";
        case StatusCode.InternalServerError:
            return "เกิดข้อผิดพลาดในเซิร์ฟเวอร์";
        default:
            return `รหัส HTTP: ${code}`;
    }
}

console.log(handleResponse(StatusCode.OK));         // สำเร็จ
console.log(handleResponse(StatusCode.NotFound));   // ไม่พบข้อมูล
console.log(handleResponse(StatusCode.Created));    // สร้างข้อมูลแล้ว
```

```typescript
// ตัวอย่างที่ 3: Numeric Enum กับ Auto-increment
enum Priority {
    Critical = 1,   // 1
    High,           // 2 (auto-increment)
    Medium,         // 3
    Low,            // 4
    None            // 5
}

function getTasksByPriority(priority: Priority): string[] {
    return [`งานที่มีความสำคัญระดับ ${Priority[priority]}`];
}

console.log(Priority.Critical);  // 1
console.log(Priority.High);      // 2
console.log(Priority.Medium);    // 3
```

### 6.1.2 Numeric Enum ในสถานการณ์จริง

```typescript
// ตัวอย่างที่ 4: Enum สำหรับระดับการแจ้งเตือน
enum AlertLevel {
    Info = 1,
    Warning = 2,
    Error = 3,
    Critical = 4
}

interface Alert {
    id: number;
    level: AlertLevel;
    message: string;
    timestamp: Date;
    isRead: boolean;
}

function createAlert(level: AlertLevel, message: string): Alert {
    return {
        id: Date.now(),
        level,
        message,
        timestamp: new Date(),
        isRead: false
    };
}

function formatAlertLevel(level: AlertLevel): string {
    const levelNames: Record<AlertLevel, string> = {
        [AlertLevel.Info]: "ℹ️ ข้อมูล",
        [AlertLevel.Warning]: "⚠️ คำเตือน",
        [AlertLevel.Error]: "❌ ข้อผิดพลาด",
        [AlertLevel.Critical]: "🚨 วิกฤต"
    };
    return levelNames[level] || "ไม่ทราบ";
}

const alerts: Alert[] = [
    createAlert(AlertLevel.Info, "ระบบเริ่มทำงานแล้ว"),
    createAlert(AlertLevel.Warning, "พื้นที่จัดเก็บเหลือน้อย"),
    createAlert(AlertLevel.Critical, "ฐานข้อมูลไม่ตอบสนอง")
];

// กรองเฉพาะ Critical
const criticalAlerts = alerts.filter(a => a.level >= AlertLevel.Error);
criticalAlerts.forEach(alert => {
    console.log(`${formatAlertLevel(alert.level)}: ${alert.message}`);
});
```

---

## 6.2 String Enums

### 6.2.1 การประกาศ String Enum

```typescript
// ตัวอย่างที่ 5: String Enum พื้นฐาน
enum Color {
    Red = "RED",
    Green = "GREEN",
    Blue = "BLUE",
    Yellow = "YELLOW",
    Orange = "ORANGE"
}

let primaryColor: Color = Color.Red;
console.log(primaryColor);        // "RED"
console.log(Color.Green);         // "GREEN"
console.log(Color.Blue);          // "BLUE"
```

```typescript
// ตัวอย่างที่ 6: String Enum สำหรับ API Endpoints
enum ApiEndpoint {
    Users = "/api/users",
    Products = "/api/products",
    Orders = "/api/orders",
    Auth = "/api/auth",
    Reports = "/api/reports"
}

enum HttpMethod {
    GET = "GET",
    POST = "POST",
    PUT = "PUT",
    PATCH = "PATCH",
    DELETE = "DELETE"
}

interface ApiRequest {
    endpoint: ApiEndpoint;
    method: HttpMethod;
    body?: Record<string, unknown>;
}

function makeRequest(request: ApiRequest): string {
    const { endpoint, method, body } = request;
    let message = `${method} ${endpoint}`;
    if (body) {
        message += `\nBody: ${JSON.stringify(body)}`;
    }
    return message;
}

const getUsersRequest: ApiRequest = {
    endpoint: ApiEndpoint.Users,
    method: HttpMethod.GET
};

const createUserRequest: ApiRequest = {
    endpoint: ApiEndpoint.Users,
    method: HttpMethod.POST,
    body: { name: "สมชาย", email: "somchai@example.com" }
};

console.log(makeRequest(getUsersRequest));
console.log(makeRequest(createUserRequest));
```

```typescript
// ตัวอย่างที่ 7: String Enum สำหรับ Order Status
enum OrderStatus {
    Pending = "PENDING",
    Confirmed = "CONFIRMED",
    Processing = "PROCESSING",
    Shipped = "SHIPPED",
    Delivered = "DELIVERED",
    Cancelled = "CANCELLED",
    Refunded = "REFUNDED"
}

interface Order {
    orderId: string;
    customerName: string;
    totalAmount: number;
    status: OrderStatus;
    createdAt: Date;
    updatedAt: Date;
}

function getStatusDisplay(status: OrderStatus): { label: string; color: string } {
    const statusMap: Record<OrderStatus, { label: string; color: string }> = {
        [OrderStatus.Pending]: { label: "รอดำเนินการ", color: "#FFA500" },
        [OrderStatus.Confirmed]: { label: "ยืนยันแล้ว", color: "#2196F3" },
        [OrderStatus.Processing]: { label: "กำลังดำเนินการ", color: "#9C27B0" },
        [OrderStatus.Shipped]: { label: "จัดส่งแล้ว", color: "#00BCD4" },
        [OrderStatus.Delivered]: { label: "ส่งถึงแล้ว", color: "#4CAF50" },
        [OrderStatus.Cancelled]: { label: "ยกเลิกแล้ว", color: "#F44336" },
        [OrderStatus.Refunded]: { label: "คืนเงินแล้ว", color: "#607D8B" }
    };
    return statusMap[status];
}

const sampleOrders: Order[] = [
    {
        orderId: "ORD-001",
        customerName: "นิรันดร์ สุขใจ",
        totalAmount: 1500,
        status: OrderStatus.Shipped,
        createdAt: new Date("2024-01-15"),
        updatedAt: new Date("2024-01-17")
    },
    {
        orderId: "ORD-002",
        customerName: "พิมพ์ใจ มีโชค",
        totalAmount: 2800,
        status: OrderStatus.Pending,
        createdAt: new Date("2024-01-18"),
        updatedAt: new Date("2024-01-18")
    }
];

sampleOrders.forEach(order => {
    const display = getStatusDisplay(order.status);
    console.log(`${order.orderId}: ${order.customerName} - ${display.label}`);
});
```

```typescript
// ตัวอย่างที่ 8: String Enum สำหรับสิทธิ์การเข้าถึง
enum Permission {
    Read = "read",
    Write = "write",
    Delete = "delete",
    Admin = "admin"
}

enum UserRole {
    Guest = "GUEST",
    User = "USER",
    Editor = "EDITOR",
    Admin = "ADMIN",
    SuperAdmin = "SUPER_ADMIN"
}

const rolePermissions: Record<UserRole, Permission[]> = {
    [UserRole.Guest]: [Permission.Read],
    [UserRole.User]: [Permission.Read, Permission.Write],
    [UserRole.Editor]: [Permission.Read, Permission.Write, Permission.Delete],
    [UserRole.Admin]: [Permission.Read, Permission.Write, Permission.Delete, Permission.Admin],
    [UserRole.SuperAdmin]: [Permission.Read, Permission.Write, Permission.Delete, Permission.Admin]
};

function hasPermission(role: UserRole, permission: Permission): boolean {
    return rolePermissions[role].includes(permission);
}

console.log(hasPermission(UserRole.Guest, Permission.Write));  // false
console.log(hasPermission(UserRole.Editor, Permission.Delete)); // true
console.log(hasPermission(UserRole.User, Permission.Admin));    // false
```

---

## 6.3 Const Enums

Const Enums ถูก compile เป็นค่าตรงๆ ไม่สร้าง object จริง ทำให้ประสิทธิภาพดีขึ้น

```typescript
// ตัวอย่างที่ 9: Const Enum
const enum Weekday {
    Monday = 1,
    Tuesday = 2,
    Wednesday = 3,
    Thursday = 4,
    Friday = 5,
    Saturday = 6,
    Sunday = 7
}

function isWeekend(day: Weekday): boolean {
    return day === Weekday.Saturday || day === Weekday.Sunday;
}

function getWorkHours(day: Weekday): number {
    if (isWeekend(day)) return 0;
    if (day === Weekday.Friday) return 7; // วันศุกร์ทำงานน้อยกว่า
    return 8;
}

// const enum จะถูก inline เป็นค่าตรงๆ เมื่อ compile
// isWeekend(Weekday.Saturday) -> isWeekend(6) ใน JavaScript
const today: Weekday = Weekday.Wednesday;
console.log(`วันนี้ทำงาน ${getWorkHours(today)} ชั่วโมง`);
console.log(`วันนี้เป็นวันหยุด: ${isWeekend(today)}`);
```

```typescript
// ตัวอย่างที่ 10: Const Enum สำหรับ Configuration
const enum Config {
    MaxRetries = 3,
    TimeoutMs = 5000,
    MaxConnections = 10,
    DefaultPageSize = 20,
    MaxPageSize = 100
}

function fetchWithRetry(url: string): string {
    let attempts: number = 0;
    while (attempts < Config.MaxRetries) {
        attempts++;
        console.log(`ลองครั้งที่ ${attempts}/${Config.MaxRetries}`);
        // จำลองการเรียก API
        if (attempts === Config.MaxRetries) {
            return "สำเร็จ";
        }
    }
    return "ล้มเหลว";
}

console.log(fetchWithRetry("https://api.example.com/data"));
```

```typescript
// ตัวอย่างที่ 11: ความแตกต่าง Const Enum vs Regular Enum
// Regular Enum - สร้าง object จริง
enum RegularEnum {
    A = 1,
    B = 2,
    C = 3
}

// ใช้งานได้ทั้ง:
console.log(RegularEnum.A);    // 1
console.log(RegularEnum[1]);   // "A" (Reverse mapping)
console.log(Object.keys(RegularEnum)); // ดูได้

// Const Enum - ไม่สร้าง object
const enum ConstEnum {
    X = 10,
    Y = 20,
    Z = 30
}

// ใช้ได้แค่ค่าตรงๆ (ไม่มี reverse mapping)
console.log(ConstEnum.X);  // 10
// console.log(ConstEnum[10]); // Error! ไม่สามารถ index const enum
```

---

## 6.4 Heterogeneous Enums

Heterogeneous Enums ผสมระหว่าง number และ string (ใช้น้อย แต่มีกรณีเฉพาะ)

```typescript
// ตัวอย่างที่ 12: Heterogeneous Enum
enum BooleanLike {
    No = 0,
    Yes = "YES"
}

console.log(BooleanLike.No);  // 0
console.log(BooleanLike.Yes); // "YES"

// ใช้สำหรับ Legacy API ที่ส่งค่าผสม
enum LegacyStatus {
    Inactive = 0,
    Active = "ACTIVE",
    Suspended = 2,
    Banned = "BANNED"
}
```

```typescript
// ตัวอย่างที่ 13: Heterogeneous Enum กับ Real-world
// (ควรหลีกเลี่ยงในกรณีทั่วไป แต่มีบางกรณีที่จำเป็น)
enum AppMode {
    Debug = 0,
    Test = "TEST",
    Production = "PRODUCTION"
}

function getEnvironmentConfig(mode: AppMode): object {
    switch (mode) {
        case AppMode.Debug:
            return { logLevel: "verbose", showErrors: true };
        case AppMode.Test:
            return { logLevel: "info", showErrors: true };
        case AppMode.Production:
            return { logLevel: "error", showErrors: false };
        default:
            return {};
    }
}
```

---

## 6.5 Enum Members เป็น Types

```typescript
// ตัวอย่างที่ 14: Enum Member ใช้เป็น Type
enum ShapeType {
    Circle = "CIRCLE",
    Rectangle = "RECTANGLE",
    Triangle = "TRIANGLE"
}

interface Circle {
    type: ShapeType.Circle;
    radius: number;
}

interface Rectangle {
    type: ShapeType.Rectangle;
    width: number;
    height: number;
}

interface Triangle {
    type: ShapeType.Triangle;
    base: number;
    height: number;
}

type Shape = Circle | Rectangle | Triangle;

function calculateArea(shape: Shape): number {
    switch (shape.type) {
        case ShapeType.Circle:
            return Math.PI * shape.radius ** 2;
        case ShapeType.Rectangle:
            return shape.width * shape.height;
        case ShapeType.Triangle:
            return 0.5 * shape.base * shape.height;
    }
}

const myCircle: Circle = { type: ShapeType.Circle, radius: 5 };
const myRect: Rectangle = { type: ShapeType.Rectangle, width: 4, height: 6 };
const myTriangle: Triangle = { type: ShapeType.Triangle, base: 3, height: 8 };

console.log(`วงกลม พื้นที่: ${calculateArea(myCircle).toFixed(2)}`);
console.log(`สี่เหลี่ยม พื้นที่: ${calculateArea(myRect).toFixed(2)}`);
console.log(`สามเหลี่ยม พื้นที่: ${calculateArea(myTriangle).toFixed(2)}`);
```

```typescript
// ตัวอย่างที่ 15: Discriminated Union กับ Enum
enum NotificationType {
    Email = "EMAIL",
    SMS = "SMS",
    PushNotification = "PUSH"
}

interface EmailNotification {
    type: NotificationType.Email;
    to: string;
    subject: string;
    body: string;
}

interface SMSNotification {
    type: NotificationType.SMS;
    phoneNumber: string;
    message: string;
}

interface PushNotification {
    type: NotificationType.PushNotification;
    deviceToken: string;
    title: string;
    body: string;
}

type Notification = EmailNotification | SMSNotification | PushNotification;

function sendNotification(notification: Notification): void {
    switch (notification.type) {
        case NotificationType.Email:
            console.log(`ส่งอีเมล ถึง: ${notification.to}`);
            console.log(`หัวข้อ: ${notification.subject}`);
            break;
        case NotificationType.SMS:
            console.log(`ส่ง SMS ถึง: ${notification.phoneNumber}`);
            console.log(`ข้อความ: ${notification.message}`);
            break;
        case NotificationType.PushNotification:
            console.log(`ส่ง Push Notification: ${notification.title}`);
            break;
    }
}

sendNotification({
    type: NotificationType.Email,
    to: "user@example.com",
    subject: "ยืนยันคำสั่งซื้อ",
    body: "คำสั่งซื้อของคุณได้รับการยืนยันแล้ว"
});

sendNotification({
    type: NotificationType.SMS,
    phoneNumber: "0812345678",
    message: "รหัส OTP ของคุณคือ: 123456"
});
```

---

## 6.6 Reverse Mapping

Numeric Enums รองรับ Reverse Mapping (แปลงค่าตัวเลขกลับเป็นชื่อ)

```typescript
// ตัวอย่างที่ 16: Reverse Mapping พื้นฐาน
enum Season {
    Spring = 1,
    Summer = 2,
    Autumn = 3,
    Winter = 4
}

// Forward mapping (ชื่อ -> ค่า)
console.log(Season.Spring);  // 1
console.log(Season.Summer);  // 2

// Reverse mapping (ค่า -> ชื่อ)
console.log(Season[1]);      // "Spring"
console.log(Season[2]);      // "Summer"
console.log(Season[3]);      // "Autumn"
console.log(Season[4]);      // "Winter"
```

```typescript
// ตัวอย่างที่ 17: Reverse Mapping ประยุกต์ใช้
enum ErrorCode {
    Success = 0,
    NetworkError = 1001,
    AuthError = 1002,
    ValidationError = 1003,
    NotFoundError = 1004,
    ServerError = 5000
}

function getErrorMessage(code: number): string {
    const errorName: string = ErrorCode[code];
    if (!errorName) {
        return `รหัสข้อผิดพลาดไม่รู้จัก: ${code}`;
    }
    
    const messages: Partial<Record<keyof typeof ErrorCode, string>> = {
        Success: "ดำเนินการสำเร็จ",
        NetworkError: "เกิดข้อผิดพลาดทางเครือข่าย",
        AuthError: "ไม่ผ่านการยืนยันตัวตน",
        ValidationError: "ข้อมูลไม่ถูกต้อง",
        NotFoundError: "ไม่พบข้อมูล",
        ServerError: "เกิดข้อผิดพลาดในเซิร์ฟเวอร์"
    };
    
    return messages[errorName as keyof typeof messages] || `ข้อผิดพลาด: ${errorName}`;
}

console.log(getErrorMessage(0));    // ดำเนินการสำเร็จ
console.log(getErrorMessage(1002)); // ไม่ผ่านการยืนยันตัวตน
console.log(getErrorMessage(9999)); // รหัสข้อผิดพลาดไม่รู้จัก: 9999
```

```typescript
// ตัวอย่างที่ 18: String Enum ไม่มี Reverse Mapping
enum StringEnum {
    First = "FIRST",
    Second = "SECOND"
}

// String Enums ไม่มี reverse mapping
console.log(StringEnum.First);  // "FIRST"
// console.log(StringEnum["FIRST"]); // undefined (ไม่ work)

// ถ้าต้องการ reverse กับ string enum ต้องทำเอง
function getStringEnumKey(value: string): string | undefined {
    return Object.keys(StringEnum).find(
        key => StringEnum[key as keyof typeof StringEnum] === value
    );
}

console.log(getStringEnumKey("FIRST"));   // "First"
console.log(getStringEnumKey("SECOND"));  // "Second"
```

---

## 6.7 Computed Enum Values

```typescript
// ตัวอย่างที่ 19: Computed Values ใน Enum
function getBaseValue(): number {
    return 100;
}

enum ComputedEnum {
    A = getBaseValue(),        // computed
    B = A + 1,                 // computed based on A
    C = Math.pow(2, 4),        // computed using Math
}

console.log(ComputedEnum.A); // 100
console.log(ComputedEnum.B); // 101
console.log(ComputedEnum.C); // 16
```

```typescript
// ตัวอย่างที่ 20: Bit Flags ด้วย Enum
// ใช้สำหรับ permission flags หรือ feature flags
enum FilePermission {
    None    = 0,         // 0000
    Read    = 1 << 0,    // 0001 = 1
    Write   = 1 << 1,    // 0010 = 2
    Execute = 1 << 2,    // 0100 = 4
    Delete  = 1 << 3,    // 1000 = 8
    
    // Combined permissions
    ReadWrite = Read | Write,          // 3
    ReadExecute = Read | Execute,      // 5
    All = Read | Write | Execute | Delete // 15
}

function hasFilePermission(userPerms: number, requiredPerm: FilePermission): boolean {
    return (userPerms & requiredPerm) === requiredPerm;
}

function addPermission(current: number, perm: FilePermission): number {
    return current | perm;
}

function removePermission(current: number, perm: FilePermission): number {
    return current & ~perm;
}

// ทดสอบ
let userPermissions: number = FilePermission.Read | FilePermission.Write;

console.log(`มีสิทธิ์อ่าน: ${hasFilePermission(userPermissions, FilePermission.Read)}`);     // true
console.log(`มีสิทธิ์เขียน: ${hasFilePermission(userPermissions, FilePermission.Write)}`);   // true
console.log(`มีสิทธิ์รัน: ${hasFilePermission(userPermissions, FilePermission.Execute)}`); // false

// เพิ่มสิทธิ์ Execute
userPermissions = addPermission(userPermissions, FilePermission.Execute);
console.log(`หลังเพิ่ม Execute - มีสิทธิ์รัน: ${hasFilePermission(userPermissions, FilePermission.Execute)}`); // true

// ลบสิทธิ์ Write
userPermissions = removePermission(userPermissions, FilePermission.Write);
console.log(`หลังลบ Write - มีสิทธิ์เขียน: ${hasFilePermission(userPermissions, FilePermission.Write)}`); // false
```

---

## 6.8 Ambient Enums

```typescript
// ตัวอย่างที่ 21: Ambient Enum (ใช้ใน .d.ts files)
// declare enum สำหรับ enum ที่นิยามภายนอก TypeScript
// ใช้เมื่อต้องการ type ให้กับ enum จาก library ภายนอก

declare enum ExternalStatus {
    Active,
    Inactive,
    Pending
}

// ใช้ใน code (ค่าจริงมาจาก runtime environment)
// function checkStatus(status: ExternalStatus): void { ... }
```

---

## 6.9 Enum Best Practices

```typescript
// ตัวอย่างที่ 22: เมื่อไหร่ควรใช้ Enum vs Union Types

// String Union Types (บ่อยครั้งดีกว่า Enum!)
type PaymentStatus = "pending" | "processing" | "completed" | "failed" | "refunded";
type Tier = "free" | "basic" | "premium" | "enterprise";

// Enum (ดีสำหรับกรณีเหล่านี้)
enum Month {
    January = 1,
    February,
    March,
    April,
    May,
    June,
    July,
    August,
    September,
    October,
    November,
    December
}

// เมื่อต้องการ Reverse Mapping
function getMonthName(month: Month): string {
    return Month[month];
}
console.log(getMonthName(Month.March)); // "March"
```

```typescript
// ตัวอย่างที่ 23: Enum ในระบบจริง - Payment Gateway
enum PaymentProvider {
    PromptPay = "PROMPTPAY",
    CreditCard = "CREDIT_CARD",
    BankTransfer = "BANK_TRANSFER",
    TrueMoney = "TRUEMONEY",
    GBPrimePay = "GBPRIMEPAY",
    Omise = "OMISE"
}

enum PaymentResult {
    Success = "SUCCESS",
    Failed = "FAILED",
    Pending = "PENDING",
    Cancelled = "CANCELLED"
}

interface PaymentRequest {
    amount: number;
    currency: "THB" | "USD";
    provider: PaymentProvider;
    orderId: string;
    description: string;
}

interface PaymentResponse {
    transactionId: string;
    result: PaymentResult;
    provider: PaymentProvider;
    amount: number;
    timestamp: Date;
    message: string;
}

function processPayment(request: PaymentRequest): PaymentResponse {
    // จำลองการประมวลผลการชำระเงิน
    const isSuccess: boolean = Math.random() > 0.2;
    
    return {
        transactionId: `TXN-${Date.now()}`,
        result: isSuccess ? PaymentResult.Success : PaymentResult.Failed,
        provider: request.provider,
        amount: request.amount,
        timestamp: new Date(),
        message: isSuccess ? "ชำระเงินสำเร็จ" : "ชำระเงินไม่สำเร็จ กรุณาลองใหม่"
    };
}

const payment: PaymentRequest = {
    amount: 1500,
    currency: "THB",
    provider: PaymentProvider.PromptPay,
    orderId: "ORD-12345",
    description: "ชำระค่าสินค้า"
};

const response = processPayment(payment);
console.log(`การชำระเงิน: ${response.result}`);
console.log(`ผ่าน: ${response.provider}`);
console.log(`จำนวน: ${response.amount} บาท`);
```

```typescript
// ตัวอย่างที่ 24: Enum กับ Class
enum VehicleType {
    Car = "CAR",
    Motorcycle = "MOTORCYCLE",
    Truck = "TRUCK",
    Bus = "BUS"
}

enum FuelType {
    Gasoline = "GASOLINE",
    Diesel = "DIESEL",
    Electric = "ELECTRIC",
    Hybrid = "HYBRID"
}

class Vehicle {
    private readonly type: VehicleType;
    private readonly fuelType: FuelType;
    private readonly plateNumber: string;
    private fuelLevel: number; // 0-100
    
    constructor(
        type: VehicleType,
        fuelType: FuelType,
        plateNumber: string,
        initialFuel: number = 100
    ) {
        this.type = type;
        this.fuelType = fuelType;
        this.plateNumber = plateNumber;
        this.fuelLevel = initialFuel;
    }
    
    refuel(amount: number): void {
        if (this.fuelType === FuelType.Electric) {
            console.log(`${this.plateNumber}: ชาร์จไฟ ${amount}%`);
        } else {
            console.log(`${this.plateNumber}: เติมน้ำมัน ${amount} ลิตร`);
        }
        this.fuelLevel = Math.min(100, this.fuelLevel + amount);
    }
    
    getInfo(): string {
        return `${this.plateNumber} (${this.type}) - ${this.fuelType}: ${this.fuelLevel}%`;
    }
}

const mycar = new Vehicle(VehicleType.Car, FuelType.Gasoline, "กข-1234", 50);
const myEV = new Vehicle(VehicleType.Car, FuelType.Electric, "ขค-5678", 80);

console.log(mycar.getInfo());
mycar.refuel(30);
console.log(mycar.getInfo());

console.log(myEV.getInfo());
myEV.refuel(20);
```

---

## 6.10 เมื่อไหร่ไม่ควรใช้ Enum

```typescript
// ตัวอย่างที่ 25: กรณีที่ Union Types ดีกว่า Enum

// แบบ Enum (โค้ดมากกว่า)
enum Direction1 {
    North = "NORTH",
    South = "SOUTH",
    East = "EAST",
    West = "WEST"
}

// แบบ Union Type (กระชับกว่า)
type Direction2 = "NORTH" | "SOUTH" | "EAST" | "WEST";

// ทั้งคู่ป้องกัน type error ได้เหมือนกัน
function move1(dir: Direction1): void {
    console.log(`เคลื่อนที่ไป ${dir}`);
}

function move2(dir: Direction2): void {
    console.log(`เคลื่อนที่ไป ${dir}`);
}

move1(Direction1.North);
move2("NORTH");

// Union Types มีข้อดี:
// 1. โค้ดน้อยกว่า
// 2. ไม่สร้าง JavaScript object
// 3. ทำงานกับ JSON ได้โดยตรง
// 4. intellisense ดีพอกัน
```

```typescript
// ตัวอย่างที่ 26: Object as Const (ทางเลือกอื่น)
// บางทีนี้ดีกว่า enum เพราะได้ทั้ง type และ value

const COLORS = {
    Red: "RED",
    Green: "GREEN",
    Blue: "BLUE",
    Yellow: "YELLOW"
} as const;

type ColorValue = typeof COLORS[keyof typeof COLORS];
// ColorValue = "RED" | "GREEN" | "BLUE" | "YELLOW"

function paintColor(color: ColorValue): void {
    console.log(`ทาสี ${color}`);
}

paintColor(COLORS.Red);     // "RED"
paintColor("GREEN");         // ได้เช่นกัน
// paintColor("PURPLE");    // Error!
```

---

## 6.11 Enum กับ keyof และ typeof

```typescript
// ตัวอย่างที่ 27: keyof typeof Enum
enum Language {
    Thai = "th",
    English = "en",
    Japanese = "ja",
    Chinese = "zh",
    Korean = "ko"
}

type LanguageKey = keyof typeof Language;
// type LanguageKey = "Thai" | "English" | "Japanese" | "Chinese" | "Korean"

function getLanguageName(key: LanguageKey): string {
    const names: Record<LanguageKey, string> = {
        Thai: "ภาษาไทย",
        English: "ภาษาอังกฤษ",
        Japanese: "ภาษาญี่ปุ่น",
        Chinese: "ภาษาจีน",
        Korean: "ภาษาเกาหลี"
    };
    return names[key];
}

console.log(getLanguageName("Thai"));     // ภาษาไทย
console.log(getLanguageName("English"));  // ภาษาอังกฤษ
// console.log(getLanguageName("French")); // Error!
```

---

## 6.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ระบบจัดการพนักงาน

```typescript
// โจทย์: สร้าง Enum สำหรับระบบจัดการพนักงาน

enum Department {
    Engineering = "ENGINEERING",
    Marketing = "MARKETING",
    Finance = "FINANCE",
    HumanResources = "HR",
    Operations = "OPERATIONS"
}

enum EmploymentType {
    FullTime = "FULL_TIME",
    PartTime = "PART_TIME",
    Contract = "CONTRACT",
    Intern = "INTERN"
}

enum PerformanceRating {
    Exceptional = 5,
    Excellent = 4,
    Good = 3,
    NeedsImprovement = 2,
    Unsatisfactory = 1
}

interface Employee {
    id: number;
    name: string;
    department: Department;
    employmentType: EmploymentType;
    salary: number;
    performanceRating: PerformanceRating;
    yearsOfService: number;
}

function calculateBonus(employee: Employee): number {
    const baseBonus: number = employee.salary * 0.1;
    const performanceMultiplier: number = employee.performanceRating / 5;
    const loyaltyBonus: number = employee.yearsOfService * 1000;
    
    let typeMultiplier: number = 1;
    switch (employee.employmentType) {
        case EmploymentType.FullTime:
            typeMultiplier = 1.0;
            break;
        case EmploymentType.PartTime:
            typeMultiplier = 0.5;
            break;
        case EmploymentType.Contract:
            typeMultiplier = 0.7;
            break;
        case EmploymentType.Intern:
            typeMultiplier = 0;
            break;
    }
    
    return (baseBonus * performanceMultiplier + loyaltyBonus) * typeMultiplier;
}

const employees: Employee[] = [
    { id: 1, name: "ธนาพร ชัยวัฒน์", department: Department.Engineering, 
      employmentType: EmploymentType.FullTime, salary: 80000, 
      performanceRating: PerformanceRating.Exceptional, yearsOfService: 5 },
    { id: 2, name: "วรรณภา สุขสม", department: Department.Marketing, 
      employmentType: EmploymentType.FullTime, salary: 55000, 
      performanceRating: PerformanceRating.Good, yearsOfService: 3 },
    { id: 3, name: "ณัฐพล มีสุข", department: Department.Finance, 
      employmentType: EmploymentType.Contract, salary: 65000, 
      performanceRating: PerformanceRating.Excellent, yearsOfService: 2 }
];

console.log("=== คำนวณโบนัสพนักงาน ===");
employees.forEach(emp => {
    const bonus = calculateBonus(emp);
    console.log(`${emp.name} (${emp.department}): โบนัส ${bonus.toLocaleString()} บาท`);
});
```

### แบบฝึกหัดที่ 2: State Machine ด้วย Enum

```typescript
// โจทย์: State Machine สำหรับ Document Workflow

enum DocumentState {
    Draft = "DRAFT",
    UnderReview = "UNDER_REVIEW",
    Approved = "APPROVED",
    Rejected = "REJECTED",
    Published = "PUBLISHED",
    Archived = "ARCHIVED"
}

type Transition = {
    from: DocumentState;
    to: DocumentState;
    action: string;
};

const allowedTransitions: Transition[] = [
    { from: DocumentState.Draft, to: DocumentState.UnderReview, action: "ส่งเพื่อตรวจสอบ" },
    { from: DocumentState.UnderReview, to: DocumentState.Approved, action: "อนุมัติ" },
    { from: DocumentState.UnderReview, to: DocumentState.Rejected, action: "ปฏิเสธ" },
    { from: DocumentState.Rejected, to: DocumentState.Draft, action: "แก้ไข" },
    { from: DocumentState.Approved, to: DocumentState.Published, action: "เผยแพร่" },
    { from: DocumentState.Published, to: DocumentState.Archived, action: "เก็บเข้าคลัง" }
];

function canTransition(from: DocumentState, to: DocumentState): boolean {
    return allowedTransitions.some(t => t.from === from && t.to === to);
}

function getAvailableActions(state: DocumentState): string[] {
    return allowedTransitions
        .filter(t => t.from === state)
        .map(t => t.action);
}

let currentState: DocumentState = DocumentState.Draft;

console.log(`\nสถานะเริ่มต้น: ${currentState}`);
console.log(`การดำเนินการที่ทำได้: ${getAvailableActions(currentState).join(", ")}`);

// ทดสอบ transition
const transitions = [
    DocumentState.UnderReview,
    DocumentState.Approved,
    DocumentState.Published,
    DocumentState.Archived
];

transitions.forEach(nextState => {
    if (canTransition(currentState, nextState)) {
        console.log(`\nเปลี่ยนจาก ${currentState} -> ${nextState}: สำเร็จ`);
        currentState = nextState;
        const actions = getAvailableActions(currentState);
        if (actions.length > 0) {
            console.log(`การดำเนินการที่ทำได้: ${actions.join(", ")}`);
        } else {
            console.log("ไม่มีการดำเนินการเพิ่มเติม");
        }
    }
});
```

---

## สรุปบทที่ 6

| ชนิด Enum | รายละเอียด | เมื่อไหร่ใช้ |
|-----------|-----------|------------|
| Numeric Enum | ค่าเป็นตัวเลข, มี auto-increment | เมื่อต้องการ ordering หรือ bit flags |
| String Enum | ค่าเป็น string, อ่านง่าย | ส่วนใหญ่แนะนำ, debugging ง่าย |
| Const Enum | ถูก inline, ประสิทธิภาพดี | เมื่อ performance สำคัญ |
| Heterogeneous | ผสม number/string | หลีกเลี่ยงถ้าเป็นไปได้ |
| Reverse Mapping | number -> name | เฉพาะ Numeric Enums |

**เมื่อไหร่ไม่ควรใช้ Enum:**
- เมื่อ Union Types เพียงพอ
- เมื่อต้องการ zero runtime overhead
- เมื่อต้องการ serialize/deserialize ง่ายๆ

ในบทถัดไปเราจะเรียนเรื่อง **Functions** ซึ่งเป็นหัวใจสำคัญของ TypeScript
