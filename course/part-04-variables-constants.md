# Part 4: ตัวแปรและค่าคงที่ใน TypeScript (Variables & Constants)

## บทนำ

ตัวแปร (Variables) และค่าคงที่ (Constants) เป็นพื้นฐานที่สำคัญที่สุดในการเขียนโปรแกรม ใน TypeScript เราสามารถกำหนดชนิดข้อมูล (Type) ให้กับตัวแปรได้อย่างชัดเจน ซึ่งช่วยให้โค้ดมีความปลอดภัยและอ่านง่ายมากขึ้น

---

## 4.1 var, let และ const

### 4.1.1 var - ตัวแปรแบบเก่า

`var` เป็นวิธีการประกาศตัวแปรแบบดั้งเดิมใน JavaScript และ TypeScript โดยมีขอบเขต (Scope) เป็นระดับฟังก์ชัน (Function Scope)

```typescript
// ตัวอย่างที่ 1: การประกาศตัวแปรด้วย var
var studentName: string = "สมชาย";
var studentAge: number = 20;
var isGraduated: boolean = false;

console.log(studentName); // สมชาย
console.log(studentAge);  // 20
console.log(isGraduated); // false
```

```typescript
// ตัวอย่างที่ 2: ปัญหาของ var - Hoisting
function demonstrateHoisting(): void {
    console.log(message); // undefined (ไม่เกิด Error!)
    var message: string = "สวัสดี";
    console.log(message); // สวัสดี
}
demonstrateHoisting();
```

```typescript
// ตัวอย่างที่ 3: ปัญหาของ var - Function Scope
function varScope(): void {
    if (true) {
        var insideIf: string = "ฉันอยู่ใน if block";
    }
    console.log(insideIf); // สามารถเข้าถึงได้! (ไม่ดี)
}
varScope();
```

```typescript
// ตัวอย่างที่ 4: ปัญหา var ใน Loop
for (var i: number = 0; i < 3; i++) {
    // ปัญหาคลาสสิค - ทุก callback อ้างอิง i เดียวกัน
    setTimeout(function() {
        console.log(i); // พิมพ์ 3, 3, 3 แทนที่จะเป็น 0, 1, 2
    }, 100);
}
```

### 4.1.2 let - ตัวแปรแบบใหม่ (แนะนำให้ใช้)

`let` มีขอบเขต Block Scope และไม่มีปัญหาเรื่อง Hoisting แบบ `var`

```typescript
// ตัวอย่างที่ 5: การประกาศตัวแปรด้วย let
let productName: string = "แล็ปท็อป";
let price: number = 25000;
let inStock: boolean = true;

console.log(productName); // แล็ปท็อป
console.log(price);       // 25000
console.log(inStock);     // true
```

```typescript
// ตัวอย่างที่ 6: let มี Block Scope
function letBlockScope(): void {
    let outerVar: string = "ตัวแปรด้านนอก";
    
    if (true) {
        let innerVar: string = "ตัวแปรด้านใน";
        console.log(outerVar); // ตัวแปรด้านนอก
        console.log(innerVar); // ตัวแปรด้านใน
    }
    
    console.log(outerVar); // ตัวแปรด้านนอก
    // console.log(innerVar); // Error! innerVar ไม่มีในขอบเขตนี้
}
letBlockScope();
```

```typescript
// ตัวอย่างที่ 7: let แก้ปัญหา Loop
for (let i: number = 0; i < 3; i++) {
    setTimeout(function() {
        console.log(i); // พิมพ์ 0, 1, 2 อย่างถูกต้อง
    }, 100);
}
```

```typescript
// ตัวอย่างที่ 8: let สามารถเปลี่ยนค่าได้
let score: number = 0;
score = 10;
score = 20;
score += 5;
console.log(score); // 25
```

```typescript
// ตัวอย่างที่ 9: let กับ Temporal Dead Zone
function temporalDeadZone(): void {
    // console.log(x); // Error! Cannot access 'x' before initialization
    let x: number = 5;
    console.log(x); // 5
}
temporalDeadZone();
```

### 4.1.3 const - ค่าคงที่

`const` ใช้สำหรับประกาศค่าที่ไม่ต้องการให้เปลี่ยนแปลงหลังจากกำหนดค่าครั้งแรก

```typescript
// ตัวอย่างที่ 10: การประกาศค่าคงที่ด้วย const
const TAX_RATE: number = 0.07;
const APP_NAME: string = "ระบบจัดการสินค้า";
const MAX_USERS: number = 1000;
const IS_PRODUCTION: boolean = false;

console.log(TAX_RATE);    // 0.07
console.log(APP_NAME);    // ระบบจัดการสินค้า
console.log(MAX_USERS);   // 1000
console.log(IS_PRODUCTION); // false
```

```typescript
// ตัวอย่างที่ 11: const ไม่สามารถเปลี่ยนค่าได้
const PI: number = 3.14159;
// PI = 3.14; // Error! Cannot assign to 'PI' because it is a constant.
```

```typescript
// ตัวอย่างที่ 12: const กับ Object - Properties ยังเปลี่ยนได้!
const user = {
    name: "สมหญิง",
    age: 25,
    email: "somying@example.com"
};

// เปลี่ยน property ได้
user.age = 26;
user.email = "newemail@example.com";
console.log(user); // { name: 'สมหญิง', age: 26, email: 'newemail@example.com' }

// แต่ไม่สามารถ reassign object ใหม่ได้
// user = { name: "คนอื่น" }; // Error!
```

```typescript
// ตัวอย่างที่ 13: const กับ Array - Elements ยังเพิ่ม/ลบได้!
const fruits: string[] = ["แอปเปิ้ล", "กล้วย", "ส้ม"];

fruits.push("มะม่วง");    // ได้
fruits[0] = "สับปะรด";   // ได้
console.log(fruits); // ['สับปะรด', 'กล้วย', 'ส้ม', 'มะม่วง']

// แต่ไม่สามารถ reassign array ใหม่ได้
// fruits = ["ลูกแพร์"]; // Error!
```

---

## 4.2 การประกาศตัวแปรพร้อมชนิดข้อมูล

### 4.2.1 การกำหนดชนิดข้อมูลอย่างชัดเจน (Explicit Type Annotation)

```typescript
// ตัวอย่างที่ 14: Explicit Type Annotations
let fullName: string = "สมชาย ใจดี";
let age: number = 30;
let height: number = 175.5;
let isActive: boolean = true;
let profilePicture: null = null;
let middleName: undefined = undefined;

// bigint สำหรับตัวเลขขนาดใหญ่มาก
let bigNumber: bigint = 9007199254740991n;

// symbol สำหรับ unique identifiers
let uniqueId: symbol = Symbol("id");
```

```typescript
// ตัวอย่างที่ 15: การประกาศโดยไม่กำหนดค่า (และต้องระบุ type)
let userName: string;
let userScore: number;
let isLoggedIn: boolean;

// ต้องกำหนดค่าก่อนใช้งาน
userName = "ปรียา";
userScore = 95;
isLoggedIn = true;

console.log(`${userName} มีคะแนน ${userScore} คะแนน`);
```

```typescript
// ตัวอย่างที่ 16: Union Types - ตัวแปรที่รับได้หลายชนิด
let phoneNumber: string | number = "02-123-4567";
phoneNumber = 0812345678; // เปลี่ยนเป็น number ได้

let status: "active" | "inactive" | "pending" = "active";
status = "inactive"; // ได้
// status = "deleted"; // Error! ไม่ใช่ค่าที่กำหนดไว้

let result: number | null = null;
result = 42; // ได้
```

```typescript
// ตัวอย่างที่ 17: Any Type (ควรหลีกเลี่ยง)
let anyValue: any = "ข้อความ";
anyValue = 123;      // ได้
anyValue = true;     // ได้
anyValue = [];       // ได้
anyValue = {};       // ได้
// any ปิดการตรวจสอบ type - ใช้เมื่อจำเป็นเท่านั้น
```

```typescript
// ตัวอย่างที่ 18: Unknown Type (ปลอดภัยกว่า any)
let unknownValue: unknown = "อาจเป็นอะไรก็ได้";
unknownValue = 42;
unknownValue = { name: "test" };

// ต้องตรวจสอบชนิดก่อนใช้งาน
if (typeof unknownValue === "string") {
    console.log(unknownValue.toUpperCase());
} else if (typeof unknownValue === "number") {
    console.log(unknownValue.toFixed(2));
}
```

---

## 4.3 Block Scope vs Function Scope

```typescript
// ตัวอย่างที่ 19: Function Scope กับ var
function functionScopeDemo(): void {
    var x: number = 10;
    
    {
        var x: number = 20; // var ใช้ scope เดียวกัน!
        console.log(x); // 20
    }
    
    console.log(x); // 20 (ถูกเขียนทับ!)
}
functionScopeDemo();
```

```typescript
// ตัวอย่างที่ 20: Block Scope กับ let
function blockScopeDemo(): void {
    let x: number = 10;
    
    {
        let x: number = 20; // let สร้างตัวแปรใหม่ใน block
        console.log(x); // 20
    }
    
    console.log(x); // 10 (ค่าเดิม)
}
blockScopeDemo();
```

```typescript
// ตัวอย่างที่ 21: Nested Scopes
function nestedScopeExample(): void {
    let level: string = "ระดับ 1";
    
    function innerFunction(): void {
        let level: string = "ระดับ 2";
        
        {
            let level: string = "ระดับ 3";
            console.log(level); // ระดับ 3
        }
        
        console.log(level); // ระดับ 2
    }
    
    innerFunction();
    console.log(level); // ระดับ 1
}
nestedScopeExample();
```

```typescript
// ตัวอย่างที่ 22: Scope ใน Loop
function loopScopeExample(): void {
    // var - มีแค่ตัวแปรเดียวสำหรับทั้ง loop
    for (var varI: number = 0; varI < 3; varI++) {}
    console.log(varI); // 3 (ยังเข้าถึงได้นอก loop!)
    
    // let - แต่ละรอบสร้างตัวแปรใหม่
    for (let letI: number = 0; letI < 3; letI++) {}
    // console.log(letI); // Error! ไม่สามารถเข้าถึงได้
}
loopScopeExample();
```

```typescript
// ตัวอย่างที่ 23: ตัวอย่างจริงของ Scope
function calculateGrades(scores: number[]): string {
    let totalScore: number = 0;
    let grade: string;
    
    for (let i: number = 0; i < scores.length; i++) {
        let score: number = scores[i]; // scope แค่ใน loop
        totalScore += score;
    }
    
    let average: number = totalScore / scores.length;
    
    if (average >= 80) {
        let distinction: string = "เกียรตินิยม";
        grade = `A - ${distinction}`;
    } else if (average >= 70) {
        grade = "B - ดี";
    } else if (average >= 60) {
        grade = "C - พอใช้";
    } else {
        grade = "F - ไม่ผ่าน";
    }
    
    return `คะแนนเฉลี่ย: ${average.toFixed(2)}, เกรด: ${grade}`;
}

console.log(calculateGrades([85, 90, 78, 92, 88]));
```

---

## 4.4 Destructuring กับ Types

### 4.4.1 Object Destructuring

```typescript
// ตัวอย่างที่ 24: Object Destructuring พื้นฐาน
const employee = {
    id: 1001,
    firstName: "วิชัย",
    lastName: "มีสุข",
    department: "ไอที",
    salary: 50000
};

const { firstName, lastName, department }: { 
    firstName: string; 
    lastName: string; 
    department: string 
} = employee;

console.log(`${firstName} ${lastName} - ${department}`);
```

```typescript
// ตัวอย่างที่ 25: Object Destructuring พร้อม Rename
const product = {
    productName: "โทรศัพท์มือถือ",
    productPrice: 15000,
    productStock: 50
};

const { 
    productName: name, 
    productPrice: price, 
    productStock: stock 
}: {
    productName: string;
    productPrice: number;
    productStock: number;
} = product;

console.log(`${name}: ราคา ${price} บาท, สต็อก ${stock} ชิ้น`);
```

```typescript
// ตัวอย่างที่ 26: Object Destructuring พร้อม Default Values
interface UserConfig {
    theme?: string;
    language?: string;
    fontSize?: number;
    notifications?: boolean;
}

const config: UserConfig = {
    theme: "dark",
    language: "th"
};

const {
    theme = "light",
    language = "en",
    fontSize = 14,
    notifications = true
} = config;

console.log(theme);         // dark (จาก config)
console.log(language);      // th (จาก config)
console.log(fontSize);      // 14 (default)
console.log(notifications); // true (default)
```

```typescript
// ตัวอย่างที่ 27: Nested Object Destructuring
interface Address {
    street: string;
    city: string;
    province: string;
    zipCode: string;
}

interface PersonWithAddress {
    name: string;
    age: number;
    address: Address;
}

const person: PersonWithAddress = {
    name: "นิดา แสงทอง",
    age: 28,
    address: {
        street: "123 ถนนสุขุมวิท",
        city: "กรุงเทพมหานคร",
        province: "กรุงเทพฯ",
        zipCode: "10110"
    }
};

const {
    name,
    address: { city, province }
} = person;

console.log(`${name} อาศัยอยู่ที่ ${city}, ${province}`);
```

```typescript
// ตัวอย่างที่ 28: Destructuring ใน Function Parameters
interface OrderItem {
    id: number;
    productName: string;
    quantity: number;
    unitPrice: number;
}

function calculateOrderTotal({ id, productName, quantity, unitPrice }: OrderItem): string {
    const total: number = quantity * unitPrice;
    return `คำสั่งซื้อ #${id}: ${productName} x${quantity} = ${total.toFixed(2)} บาท`;
}

const order: OrderItem = {
    id: 5001,
    productName: "กระเป๋าหนัง",
    quantity: 2,
    unitPrice: 1500
};

console.log(calculateOrderTotal(order));
```

### 4.4.2 Array Destructuring

```typescript
// ตัวอย่างที่ 29: Array Destructuring พื้นฐาน
const coordinates: [number, number] = [13.7563, 100.5018];
const [latitude, longitude] = coordinates;

console.log(`ละติจูด: ${latitude}`);
console.log(`ลองจิจูด: ${longitude}`);
```

```typescript
// ตัวอย่างที่ 30: Array Destructuring กับ Skip Elements
const topScores: number[] = [98, 95, 92, 88, 85];
const [first, second, , fourth] = topScores;

console.log(`อันดับ 1: ${first}`);   // 98
console.log(`อันดับ 2: ${second}`);  // 95
console.log(`อันดับ 4: ${fourth}`);  // 88
```

```typescript
// ตัวอย่างที่ 31: Array Destructuring พร้อม Rest
const monthlyRevenue: number[] = [45000, 52000, 48000, 61000, 55000, 70000];
const [january, february, ...restMonths] = monthlyRevenue;

console.log(`มกราคม: ${january}`);   // 45000
console.log(`กุมภาพันธ์: ${february}`); // 52000
console.log(`เดือนที่เหลือ: ${restMonths}`); // [48000, 61000, 55000, 70000]
```

```typescript
// ตัวอย่างที่ 32: Swap Variables ด้วย Destructuring
let firstPlace: string = "สมชาย";
let secondPlace: string = "สมหญิง";

console.log(`ก่อน swap: ${firstPlace}, ${secondPlace}`);
[firstPlace, secondPlace] = [secondPlace, firstPlace];
console.log(`หลัง swap: ${firstPlace}, ${secondPlace}`);
```

---

## 4.5 Spread Operator กับ Types

```typescript
// ตัวอย่างที่ 33: Spread กับ Arrays
const thaiCities: string[] = ["กรุงเทพฯ", "เชียงใหม่", "ภูเก็ต"];
const southCities: string[] = ["หาดใหญ่", "สงขลา", "กระบี่"];

const allCities: string[] = [...thaiCities, ...southCities];
console.log(allCities);

// เพิ่มเมืองใหม่
const updatedCities: string[] = [...allCities, "พัทยา", "ขอนแก่น"];
console.log(updatedCities);
```

```typescript
// ตัวอย่างที่ 34: Spread กับ Objects
interface BasicInfo {
    name: string;
    age: number;
}

interface ContactInfo {
    email: string;
    phone: string;
}

const basicInfo: BasicInfo = {
    name: "ประสงค์ ดีงาม",
    age: 35
};

const contactInfo: ContactInfo = {
    email: "prasong@example.com",
    phone: "089-123-4567"
};

// รวม objects
const fullProfile = { ...basicInfo, ...contactInfo };
console.log(fullProfile);
```

```typescript
// ตัวอย่างที่ 35: Spread เพื่อสร้าง Immutable Updates
interface Product {
    id: number;
    name: string;
    price: number;
    stock: number;
    category: string;
}

const originalProduct: Product = {
    id: 1,
    name: "เสื้อยืด",
    price: 299,
    stock: 100,
    category: "เสื้อผ้า"
};

// อัปเดตราคาโดยไม่แก้ไข original
const updatedProduct: Product = {
    ...originalProduct,
    price: 250,
    stock: 80
};

console.log("ต้นฉบับ:", originalProduct);
console.log("อัปเดต:", updatedProduct);
```

```typescript
// ตัวอย่างที่ 36: Spread ใน Function Calls
function sumNumbers(...numbers: number[]): number {
    return numbers.reduce((sum, num) => sum + num, 0);
}

const monthlyData: number[] = [1000, 2000, 1500, 3000, 2500];
const total: number = sumNumbers(...monthlyData);
console.log(`ยอดรวม: ${total}`); // 10000
```

---

## 4.6 Template Literals

```typescript
// ตัวอย่างที่ 37: Template Literals พื้นฐาน
const studentName: string = "มานี";
const subject: string = "คณิตศาสตร์";
const score: number = 92.5;

const reportCard: string = `นักเรียน: ${studentName}
วิชา: ${subject}
คะแนน: ${score} คะแนน
เกรด: ${score >= 80 ? 'A' : score >= 70 ? 'B' : 'C'}`;

console.log(reportCard);
```

```typescript
// ตัวอย่างที่ 38: Template Literals กับ Expressions
const items: { name: string; price: number; qty: number }[] = [
    { name: "ข้าวสวย", price: 30, qty: 2 },
    { name: "ต้มยำกุ้ง", price: 120, qty: 1 },
    { name: "น้ำเปล่า", price: 15, qty: 3 }
];

const receipt: string = `
====== ใบเสร็จรับเงิน ======
${items.map(item => `${item.name} x${item.qty}: ${item.price * item.qty} บาท`).join('\n')}
------------------------
รวมทั้งหมด: ${items.reduce((sum, item) => sum + item.price * item.qty, 0)} บาท
`;

console.log(receipt);
```

```typescript
// ตัวอย่างที่ 39: Tagged Template Literals
function formatCurrency(strings: TemplateStringsArray, ...values: number[]): string {
    return strings.reduce((result, str, i) => {
        const value: number = values[i - 1];
        if (value !== undefined) {
            const formatted: string = new Intl.NumberFormat('th-TH', {
                style: 'currency',
                currency: 'THB'
            }).format(value);
            return result + formatted + str;
        }
        return result + str;
    });
}

const salary: number = 50000;
const bonus: number = 15000;
const message: string = formatCurrency`เงินเดือน: ${salary}, โบนัส: ${bonus}`;
console.log(message);
```

---

## 4.7 Type Inference vs Explicit Annotations

```typescript
// ตัวอย่างที่ 40: Type Inference
let inferredString = "สวัสดีโลก";    // TypeScript รู้ว่าเป็น string
let inferredNumber = 42;             // TypeScript รู้ว่าเป็น number
let inferredBoolean = true;          // TypeScript รู้ว่าเป็น boolean
let inferredArray = [1, 2, 3];       // TypeScript รู้ว่าเป็น number[]

// ไม่สามารถเปลี่ยนชนิดได้แม้ไม่ได้ระบุ type
// inferredString = 100; // Error!
// inferredNumber = "ข้อความ"; // Error!
```

```typescript
// ตัวอย่างที่ 41: เมื่อไหร่ควรระบุ Type ชัดเจน
// 1. เมื่อประกาศโดยไม่กำหนดค่า
let userRole: string;
// userRole ยังไม่ถูกกำหนดค่า แต่ TypeScript รู้ว่าต้องเป็น string

// 2. เมื่อ inference อาจไม่ถูกต้อง
let mixedValues: (string | number)[] = [];
mixedValues.push("ข้อความ");
mixedValues.push(42);

// 3. เมื่อต้องการ type ที่กว้างกว่า
let flexibleValue: string | number | boolean = "เริ่มต้น";
flexibleValue = 100; // ได้
flexibleValue = false; // ได้
```

```typescript
// ตัวอย่างที่ 42: Type Inference กับ Object
const inferredObject = {
    name: "สินค้า A",
    price: 100,
    categories: ["อิเล็กทรอนิกส์", "คอมพิวเตอร์"]
};

// TypeScript infers: { name: string; price: number; categories: string[] }
// inferredObject.name = 123; // Error!
// inferredObject.price = "ราคา"; // Error!
```

```typescript
// ตัวอย่างที่ 43: const Assertion สำหรับ Literal Types
const direction = "north" as const;
// direction มี type "north" ไม่ใช่ string ทั่วไป

const config = {
    endpoint: "https://api.example.com",
    timeout: 5000,
    retries: 3
} as const;

// ทุก property เป็น readonly และมี literal type
// config.endpoint = "อื่น"; // Error!
// config.timeout = 3000; // Error!

type ConfigType = typeof config;
```

---

## 4.8 Naming Conventions (มาตรฐานการตั้งชื่อ)

```typescript
// ตัวอย่างที่ 44: Naming Conventions ที่ดี

// camelCase สำหรับตัวแปรและฟังก์ชัน
let firstName: string = "สมชาย";
let totalOrderAmount: number = 1500;
let isUserLoggedIn: boolean = false;
let getUserProfile = () => {};

// PascalCase สำหรับ Classes, Interfaces, Types, Enums
interface UserProfile {
    userId: number;
    userName: string;
}

type OrderStatus = "pending" | "processing" | "completed" | "cancelled";

class ShoppingCart {
    private items: string[] = [];
}

// SCREAMING_SNAKE_CASE สำหรับ Constants
const MAX_RETRY_ATTEMPTS: number = 3;
const API_BASE_URL: string = "https://api.example.com";
const DEFAULT_TIMEOUT_MS: number = 5000;
const TAX_PERCENTAGE: number = 7;

// _ prefix สำหรับ private (convention บางทีม)
let _internalCounter: number = 0;
```

```typescript
// ตัวอย่างที่ 45: การตั้งชื่อ Boolean Variables
// ควรขึ้นต้นด้วย is, has, can, should, will
let isActive: boolean = true;
let hasPermission: boolean = false;
let canEdit: boolean = true;
let shouldRefresh: boolean = false;
let willExpire: boolean = true;

// หลีกเลี่ยงชื่อที่ไม่ชัดเจน
// let flag: boolean = true;     // ไม่ดี
// let status: boolean = false;  // สับสนกับ string status
```

```typescript
// ตัวอย่างที่ 46: ตัวอย่างโปรเจกต์จริง - ระบบจัดการออเดอร์

interface Customer {
    customerId: string;
    customerName: string;
    customerEmail: string;
    isVipMember: boolean;
    totalPurchaseAmount: number;
}

interface OrderDetail {
    orderId: string;
    customerId: string;
    orderDate: Date;
    items: OrderItem[];
    totalAmount: number;
    discountAmount: number;
    finalAmount: number;
    orderStatus: "pending" | "confirmed" | "shipped" | "delivered" | "cancelled";
}

interface OrderItem {
    itemId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    subtotal: number;
}

const VIP_DISCOUNT_RATE: number = 0.10;
const STANDARD_DISCOUNT_RATE: number = 0.05;
const MIN_ORDER_FOR_DISCOUNT: number = 1000;

function calculateDiscount(customer: Customer, orderAmount: number): number {
    if (customer.isVipMember) {
        return orderAmount * VIP_DISCOUNT_RATE;
    } else if (orderAmount >= MIN_ORDER_FOR_DISCOUNT) {
        return orderAmount * STANDARD_DISCOUNT_RATE;
    }
    return 0;
}

const exampleCustomer: Customer = {
    customerId: "CUST-001",
    customerName: "นิยม ชนะภัย",
    customerEmail: "niyom@example.com",
    isVipMember: true,
    totalPurchaseAmount: 25000
};

const orderAmount: number = 5000;
const discount: number = calculateDiscount(exampleCustomer, orderAmount);
const finalAmount: number = orderAmount - discount;

console.log(`ลูกค้า: ${exampleCustomer.customerName}`);
console.log(`ยอดสั่งซื้อ: ${orderAmount} บาท`);
console.log(`ส่วนลด: ${discount} บาท`);
console.log(`ยอดสุทธิ: ${finalAmount} บาท`);
```

---

## 4.9 Best Practices สรุป

```typescript
// ตัวอย่างที่ 47: Best Practices รวม

// 1. ใช้ const เป็นค่าเริ่มต้น เปลี่ยนเป็น let เมื่อจำเป็น
const API_KEY: string = "abc123";
let retryCount: number = 0;

// 2. ระบุ type ชัดเจนเมื่อ inference ไม่ถูกต้องหรือไม่ชัดเจน
const emptyArray: string[] = [];  // ต้องระบุ type
let result: string | null = null; // ต้องระบุ type

// 3. ใช้ readonly เมื่อต้องการป้องกันการแก้ไข
const SETTINGS: Readonly<{ apiUrl: string; timeout: number }> = {
    apiUrl: "https://api.example.com",
    timeout: 3000
};
// SETTINGS.apiUrl = "อื่น"; // Error!

// 4. หลีกเลี่ยง any - ใช้ unknown แทน
function processData(data: unknown): string {
    if (typeof data === "string") return data.toUpperCase();
    if (typeof data === "number") return data.toString();
    return "ไม่รู้จักชนิดข้อมูล";
}

// 5. ใช้ Type Alias หรือ Interface สำหรับ complex types
type PaymentMethod = "credit_card" | "bank_transfer" | "cash" | "qr_code";
type Currency = "THB" | "USD" | "EUR";

interface Payment {
    amount: number;
    currency: Currency;
    method: PaymentMethod;
    timestamp: Date;
    reference: string;
}
```

```typescript
// ตัวอย่างที่ 48: Anti-patterns ที่ควรหลีกเลี่ยง

// ไม่ดี - ใช้ var
var badVariable: string = "ไม่ดี";

// ดีกว่า - ใช้ let หรือ const
const goodConstant: string = "ดี";
let goodVariable: string = "ดี";

// ไม่ดี - ตั้งชื่อไม่ชัดเจน
let d: number = 5;
let f: boolean = true;
let arr: string[] = [];

// ดีกว่า - ตั้งชื่อชัดเจน
let daysUntilExpiry: number = 5;
let isFeatureEnabled: boolean = true;
let categoryNames: string[] = [];

// ไม่ดี - ใช้ any
function processInput(data: any): any {
    return data;
}

// ดีกว่า - ระบุ type ชัดเจน
function processUserInput(userId: string): { success: boolean; message: string } {
    return {
        success: true,
        message: `ดำเนินการสำเร็จสำหรับผู้ใช้ ${userId}`
    };
}
```

---

## 4.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ระบบจัดการพนักงาน
สร้างตัวแปรสำหรับเก็บข้อมูลพนักงาน พร้อมกำหนดชนิดข้อมูลให้ครบถ้วน

```typescript
// โจทย์: กำหนดชนิดข้อมูลและค่าตัวแปรต่อไปนี้

// 1. รหัสพนักงาน (ตัวเลข)
const employeeId: number = 10001;

// 2. ชื่อ-นามสกุล (ข้อความ)
const employeeFullName: string = "สุรศักดิ์ พงษ์ไทย";

// 3. ตำแหน่ง (ข้อความ, หนึ่งในสามตำแหน่ง)
let position: "manager" | "developer" | "designer" = "developer";

// 4. เงินเดือน (ตัวเลขทศนิยม)
let monthlySalary: number = 55000.00;

// 5. สถานะการทำงาน (boolean)
let isCurrentlyWorking: boolean = true;

// 6. วันที่เริ่มงาน
const startDate: Date = new Date("2020-03-15");

// 7. สิทธิ์การเข้าถึง (array ของ string)
const permissions: string[] = ["read", "write", "export"];

// 8. แผนก (union type หลายตัวเลือก)
const department: "IT" | "HR" | "Finance" | "Marketing" = "IT";

// คำนวณและแสดงผล
const yearsWorked: number = new Date().getFullYear() - startDate.getFullYear();
const annualSalary: number = monthlySalary * 12;

console.log(`พนักงาน: ${employeeFullName} (${employeeId})`);
console.log(`ตำแหน่ง: ${position} - แผนก: ${department}`);
console.log(`เงินเดือน: ${monthlySalary.toLocaleString('th-TH')} บาท/เดือน`);
console.log(`เงินเดือนรายปี: ${annualSalary.toLocaleString('th-TH')} บาท`);
console.log(`ทำงานมาแล้ว: ${yearsWorked} ปี`);
```

### แบบฝึกหัดที่ 2: Destructuring และ Spread

```typescript
// โจทย์: ใช้ Destructuring และ Spread ในสถานการณ์จริง

interface MenuItem {
    id: number;
    name: string;
    price: number;
    category: "อาหาร" | "เครื่องดื่ม" | "ของหวาน";
    isAvailable: boolean;
}

const menuItems: MenuItem[] = [
    { id: 1, name: "ข้าวมันไก่", price: 60, category: "อาหาร", isAvailable: true },
    { id: 2, name: "ผัดไทย", price: 80, category: "อาหาร", isAvailable: true },
    { id: 3, name: "ชาเย็น", price: 35, category: "เครื่องดื่ม", isAvailable: false },
    { id: 4, name: "กาแฟเย็น", price: 40, category: "เครื่องดื่ม", isAvailable: true },
    { id: 5, name: "ไอศกรีม", price: 45, category: "ของหวาน", isAvailable: true }
];

// Destructuring: แสดงรายการเมนูที่มีจำหน่าย
const availableItems = menuItems.filter(({ isAvailable }) => isAvailable);
availableItems.forEach(({ name, price, category }) => {
    console.log(`[${category}] ${name} - ${price} บาท`);
});

// Spread: เพิ่มเมนูใหม่
const newItem: MenuItem = { 
    id: 6, 
    name: "มังคุด", 
    price: 30, 
    category: "ของหวาน", 
    isAvailable: true 
};
const updatedMenu: MenuItem[] = [...menuItems, newItem];
console.log(`เมนูทั้งหมด: ${updatedMenu.length} รายการ`);

// Spread: อัปเดตราคา item แรก
const [firstItem, ...otherItems] = updatedMenu;
const updatedFirstItem: MenuItem = { ...firstItem, price: 65 };
const finalMenu: MenuItem[] = [updatedFirstItem, ...otherItems];
console.log(`ราคา ${finalMenu[0].name} ใหม่: ${finalMenu[0].price} บาท`);
```

---

## สรุปบทที่ 4

| หัวข้อ | สิ่งที่ต้องจำ |
|--------|-------------|
| `var` | Function scope, Hoisting, หลีกเลี่ยง |
| `let` | Block scope, Mutable, ใช้เมื่อต้องเปลี่ยนค่า |
| `const` | Block scope, Immutable (reference), ใช้เป็นค่าเริ่มต้น |
| Type Annotation | `: type` หลังชื่อตัวแปร |
| Type Inference | TypeScript เดาชนิดให้อัตโนมัติ |
| Destructuring | แยกค่าจาก Object/Array อย่างสะดวก |
| Spread | คัดลอกและรวม Object/Array |
| Template Literals | ใช้ `` ` `` และ `${}` สำหรับ string interpolation |

ในบทถัดไปเราจะเรียนเรื่อง **Arrays และ Tuples** ซึ่งเป็นโครงสร้างข้อมูลที่ใช้บ่อยมากใน TypeScript
