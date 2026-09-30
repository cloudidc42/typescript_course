# Part 5: Arrays และ Tuples ใน TypeScript

## บทนำ

Arrays และ Tuples เป็นโครงสร้างข้อมูลพื้นฐานที่ใช้กันมากที่สุดในการพัฒนาซอฟต์แวร์ TypeScript มีระบบ Type ที่ทรงพลังสำหรับทั้งสองชนิด ช่วยให้โค้ดปลอดภัยและป้องกันข้อผิดพลาดได้ตั้งแต่ขั้นตอนการเขียน

---

## 5.1 Array Type Syntax

### 5.1.1 รูปแบบการประกาศ Array

TypeScript มีสองรูปแบบหลักในการประกาศ Array:

```typescript
// ตัวอย่างที่ 1: รูปแบบ Type[] (แนะนำ)
let fruits: string[] = ["แอปเปิ้ล", "กล้วย", "ส้ม"];
let scores: number[] = [85, 92, 78, 95, 88];
let flags: boolean[] = [true, false, true, true];

console.log(fruits);  // ['แอปเปิ้ล', 'กล้วย', 'ส้ม']
console.log(scores);  // [85, 92, 78, 95, 88]
```

```typescript
// ตัวอย่างที่ 2: รูปแบบ Array<Type> (Generic Syntax)
let cities: Array<string> = ["กรุงเทพฯ", "เชียงใหม่", "ภูเก็ต"];
let prices: Array<number> = [299, 599, 999, 1299];
let statuses: Array<boolean> = [true, false, true];

// ทั้งสองรูปแบบเหมือนกันทุกประการ
// Type[] ใช้กันแพร่หลายกว่าในทีมส่วนใหญ่
```

```typescript
// ตัวอย่างที่ 3: Array ของ Object
interface Student {
    id: number;
    name: string;
    grade: string;
    gpa: number;
}

let students: Student[] = [
    { id: 1, name: "สมชาย จันทร์ดี", grade: "ม.4", gpa: 3.8 },
    { id: 2, name: "สมหญิง ใจงาม", grade: "ม.5", gpa: 3.5 },
    { id: 3, name: "สมศักดิ์ พงษ์ทอง", grade: "ม.6", gpa: 3.9 }
];

students.forEach(student => {
    console.log(`${student.name} (${student.grade}): GPA ${student.gpa}`);
});
```

```typescript
// ตัวอย่างที่ 4: Array ของ Union Types
let mixedData: (string | number)[] = ["ชื่อ", 1, "นามสกุล", 2, "อายุ", 25];
let optionalValues: (string | null | undefined)[] = ["ค่า1", null, undefined, "ค่า4"];

// กรอง null/undefined
const validValues = optionalValues.filter((v): v is string => v !== null && v !== undefined);
console.log(validValues); // ['ค่า1', 'ค่า4']
```

```typescript
// ตัวอย่างที่ 5: Empty Array พร้อม Type
let emptyStrings: string[] = [];
let emptyNumbers: Array<number> = [];

emptyStrings.push("สวัสดี");
emptyStrings.push("โลก");
emptyNumbers.push(1, 2, 3);

console.log(emptyStrings); // ['สวัสดี', 'โลก']
console.log(emptyNumbers); // [1, 2, 3]
```

---

## 5.2 Readonly Arrays

Readonly Arrays ป้องกันการแก้ไข Array หลังจากสร้างแล้ว เหมาะสำหรับค่าคงที่หรือข้อมูลที่ไม่ควรเปลี่ยนแปลง

```typescript
// ตัวอย่างที่ 6: readonly Array
const THAI_PROVINCES: readonly string[] = [
    "กรุงเทพมหานคร", "เชียงใหม่", "ภูเก็ต", 
    "ขอนแก่น", "นครราชสีมา"
];

// อ่านได้
console.log(THAI_PROVINCES[0]); // กรุงเทพมหานคร
console.log(THAI_PROVINCES.length); // 5

// ไม่สามารถแก้ไขได้
// THAI_PROVINCES.push("เชียงราย"); // Error!
// THAI_PROVINCES[0] = "ใหม่"; // Error!
// THAI_PROVINCES.pop(); // Error!
```

```typescript
// ตัวอย่างที่ 7: ReadonlyArray<T> (Generic รูปแบบ)
const WEEKDAYS: ReadonlyArray<string> = [
    "จันทร์", "อังคาร", "พุธ", "พฤหัสบดี", "ศุกร์"
];

// เข้าถึงได้ปกติ
WEEKDAYS.forEach(day => console.log(day));
const firstDay: string = WEEKDAYS[0];
console.log(`วันแรก: ${firstDay}`);

// แต่แก้ไขไม่ได้
// WEEKDAYS.push("เสาร์"); // Error!
```

```typescript
// ตัวอย่างที่ 8: Readonly กับ Object ใน Array
interface Config {
    readonly key: string;
    readonly value: string;
}

const APP_CONFIG: ReadonlyArray<Config> = [
    { key: "theme", value: "dark" },
    { key: "language", value: "th" },
    { key: "timezone", value: "Asia/Bangkok" }
];

// อ่านได้
const themeConfig = APP_CONFIG.find(c => c.key === "theme");
console.log(themeConfig?.value); // dark

// ไม่สามารถแก้ไขได้ทั้ง array และ object
// APP_CONFIG[0] = { key: "new", value: "val" }; // Error!
// APP_CONFIG[0].value = "light"; // Error!
```

```typescript
// ตัวอย่างที่ 9: แปลง Mutable เป็น Readonly
const mutableArray: number[] = [1, 2, 3, 4, 5];

function processReadonly(arr: readonly number[]): number {
    return arr.reduce((sum, n) => sum + n, 0);
}

// ส่ง mutable array เข้า function ที่รับ readonly ได้
const sum: number = processReadonly(mutableArray);
console.log(`ผลรวม: ${sum}`); // 15
```

---

## 5.3 Array Methods พร้อม Types

### 5.3.1 map - แปลงข้อมูลแต่ละ Element

```typescript
// ตัวอย่างที่ 10: map พื้นฐาน
const productPrices: number[] = [100, 250, 500, 750, 1000];
const pricesWithVat: number[] = productPrices.map(
    (price: number): number => price * 1.07
);

console.log("ราคาก่อน VAT:", productPrices);
console.log("ราคาหลัง VAT:", pricesWithVat.map(p => p.toFixed(2)));
```

```typescript
// ตัวอย่างที่ 11: map กับ Objects
interface Product {
    id: number;
    name: string;
    price: number;
}

interface ProductDisplay {
    id: number;
    name: string;
    formattedPrice: string;
    category: string;
}

const products: Product[] = [
    { id: 1, name: "แล็ปท็อป", price: 25000 },
    { id: 2, name: "เมาส์", price: 500 },
    { id: 3, name: "คีย์บอร์ด", price: 1200 }
];

const displayProducts: ProductDisplay[] = products.map(
    (product: Product): ProductDisplay => ({
        id: product.id,
        name: product.name,
        formattedPrice: `฿${product.price.toLocaleString()}`,
        category: product.price > 10000 ? "พรีเมียม" : "มาตรฐาน"
    })
);

displayProducts.forEach(p => {
    console.log(`[${p.category}] ${p.name}: ${p.formattedPrice}`);
});
```

### 5.3.2 filter - กรองข้อมูล

```typescript
// ตัวอย่างที่ 12: filter พื้นฐาน
const scores: number[] = [45, 72, 58, 89, 91, 63, 76, 84];
const passingScores: number[] = scores.filter(
    (score: number): boolean => score >= 60
);

console.log("คะแนนทั้งหมด:", scores);
console.log("คะแนนที่ผ่าน:", passingScores);
```

```typescript
// ตัวอย่างที่ 13: filter กับ Type Guards
interface Animal {
    name: string;
    type: "dog" | "cat" | "bird";
    age: number;
}

const animals: Animal[] = [
    { name: "บาวเซอร์", type: "dog", age: 3 },
    { name: "วิสเกอร์", type: "cat", age: 5 },
    { name: "ทวีต", type: "bird", age: 2 },
    { name: "แม็กซ์", type: "dog", age: 7 }
];

const dogs: Animal[] = animals.filter(
    (animal: Animal): boolean => animal.type === "dog"
);

console.log("สุนัข:", dogs.map(d => `${d.name} (${d.age} ปี)`));
```

```typescript
// ตัวอย่างที่ 14: filter เพื่อกรอง null/undefined (Type Guard)
const userNames: (string | null | undefined)[] = [
    "สมชาย", null, "สมหญิง", undefined, "สมศักดิ์"
];

// Type Guard function
function isString(value: string | null | undefined): value is string {
    return typeof value === "string";
}

const validNames: string[] = userNames.filter(isString);
console.log("ชื่อที่ถูกต้อง:", validNames);
```

### 5.3.3 reduce - สรุปข้อมูล

```typescript
// ตัวอย่างที่ 15: reduce พื้นฐาน
const monthlyExpenses: number[] = [5000, 3500, 8000, 2000, 4500];
const totalExpense: number = monthlyExpenses.reduce(
    (accumulator: number, current: number): number => accumulator + current,
    0
);

console.log(`ค่าใช้จ่ายรวม: ${totalExpense.toLocaleString()} บาท`);
```

```typescript
// ตัวอย่างที่ 16: reduce สร้าง Object จาก Array
interface SaleRecord {
    month: string;
    amount: number;
    category: string;
}

const sales: SaleRecord[] = [
    { month: "มกราคม", amount: 50000, category: "อิเล็กทรอนิกส์" },
    { month: "มกราคม", amount: 30000, category: "เสื้อผ้า" },
    { month: "กุมภาพันธ์", amount: 45000, category: "อิเล็กทรอนิกส์" },
    { month: "กุมภาพันธ์", amount: 25000, category: "เสื้อผ้า" }
];

const salesByMonth: Record<string, number> = sales.reduce(
    (acc: Record<string, number>, sale: SaleRecord): Record<string, number> => {
        acc[sale.month] = (acc[sale.month] || 0) + sale.amount;
        return acc;
    },
    {}
);

Object.entries(salesByMonth).forEach(([month, total]) => {
    console.log(`${month}: ${total.toLocaleString()} บาท`);
});
```

### 5.3.4 find และ findIndex

```typescript
// ตัวอย่างที่ 17: find
interface Employee {
    id: number;
    name: string;
    department: string;
    salary: number;
}

const employees: Employee[] = [
    { id: 1, name: "นิรันดร์ สุขสม", department: "IT", salary: 60000 },
    { id: 2, name: "พิมพ์ใจ ทองดี", department: "HR", salary: 45000 },
    { id: 3, name: "ชาตรี มีโชค", department: "Finance", salary: 55000 }
];

const itEmployee: Employee | undefined = employees.find(
    (emp: Employee): boolean => emp.department === "IT"
);

if (itEmployee) {
    console.log(`พบพนักงาน IT: ${itEmployee.name}`);
} else {
    console.log("ไม่พบพนักงาน IT");
}
```

```typescript
// ตัวอย่างที่ 18: findIndex
const productList: string[] = ["แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง"];
const mangoIndex: number = productList.findIndex(
    (item: string): boolean => item === "มะม่วง"
);

if (mangoIndex !== -1) {
    console.log(`พบมะม่วงที่ index: ${mangoIndex}`); // 3
    productList[mangoIndex] = "ทุเรียน"; // แทนที่ด้วยทุเรียน
}
console.log(productList);
```

### 5.3.5 sort และ reverse

```typescript
// ตัวอย่างที่ 19: sort พร้อม Type-safe Comparator
interface Student {
    name: string;
    gpa: number;
    year: number;
}

const studentList: Student[] = [
    { name: "อารยา", gpa: 3.5, year: 3 },
    { name: "บุญมี", gpa: 3.8, year: 2 },
    { name: "จีรพรรณ", gpa: 3.2, year: 4 },
    { name: "ธนัช", gpa: 3.9, year: 1 }
];

// เรียงตาม GPA จากมากไปน้อย
const sortedByGpa: Student[] = [...studentList].sort(
    (a: Student, b: Student): number => b.gpa - a.gpa
);

console.log("จัดเรียงตาม GPA:");
sortedByGpa.forEach((s, i) => {
    console.log(`${i + 1}. ${s.name}: ${s.gpa}`);
});
```

### 5.3.6 flatMap และ flat

```typescript
// ตัวอย่างที่ 20: flat - รวม Nested Arrays
const nestedScores: number[][] = [
    [85, 90, 78],
    [92, 88, 95],
    [71, 83, 79]
];

const allScores: number[] = nestedScores.flat();
console.log("คะแนนทั้งหมด:", allScores);
console.log(`ค่าเฉลี่ย: ${(allScores.reduce((a, b) => a + b, 0) / allScores.length).toFixed(2)}`);
```

```typescript
// ตัวอย่างที่ 21: flatMap
interface Category {
    name: string;
    products: string[];
}

const categories: Category[] = [
    { name: "ผลไม้", products: ["แอปเปิ้ล", "กล้วย", "ส้ม"] },
    { name: "ผัก", products: ["แครอท", "มะเขือเทศ", "ผักกาด"] },
    { name: "ธัญพืช", products: ["ข้าว", "ข้าวโพด", "ข้าวสาลี"] }
];

const allProducts: string[] = categories.flatMap(
    (category: Category): string[] => category.products
);

console.log("สินค้าทั้งหมด:", allProducts);
```

### 5.3.7 every และ some

```typescript
// ตัวอย่างที่ 22: every และ some
const examScores: number[] = [75, 82, 91, 68, 88];

const allPassed: boolean = examScores.every(
    (score: number): boolean => score >= 60
);

const anyExcellent: boolean = examScores.some(
    (score: number): boolean => score >= 90
);

console.log(`ทุกคนผ่าน: ${allPassed}`);     // true
console.log(`มีคนได้ A: ${anyExcellent}`);  // true

// ตรวจสอบข้อมูลในฟอร์ม
interface FormField {
    name: string;
    value: string;
    required: boolean;
}

const formFields: FormField[] = [
    { name: "ชื่อ", value: "สมชาย", required: true },
    { name: "อีเมล", value: "test@example.com", required: true },
    { name: "เบอร์โทร", value: "", required: false }
];

const isFormValid: boolean = formFields.every(
    (field: FormField): boolean => !field.required || field.value.trim() !== ""
);

console.log(`ฟอร์มถูกต้อง: ${isFormValid}`);
```

### 5.3.8 includes และ indexOf

```typescript
// ตัวอย่างที่ 23: includes
const allowedRoles: string[] = ["admin", "manager", "editor"];
const userRole: string = "editor";

if (allowedRoles.includes(userRole)) {
    console.log(`บทบาท ${userRole} มีสิทธิ์เข้าถึง`);
} else {
    console.log(`บทบาท ${userRole} ไม่มีสิทธิ์`);
}
```

### 5.3.9 forEach

```typescript
// ตัวอย่างที่ 24: forEach
interface OrderSummary {
    orderId: string;
    customer: string;
    total: number;
    isPaid: boolean;
}

const orders: OrderSummary[] = [
    { orderId: "ORD-001", customer: "นิตยา", total: 1500, isPaid: true },
    { orderId: "ORD-002", customer: "ประเสริฐ", total: 2800, isPaid: false },
    { orderId: "ORD-003", customer: "วรรณา", total: 950, isPaid: true }
];

let paidTotal: number = 0;
let unpaidTotal: number = 0;

orders.forEach((order: OrderSummary): void => {
    if (order.isPaid) {
        paidTotal += order.total;
        console.log(`✓ ${order.orderId}: ${order.customer} - ชำระแล้ว ${order.total} บาท`);
    } else {
        unpaidTotal += order.total;
        console.log(`✗ ${order.orderId}: ${order.customer} - ค้างชำระ ${order.total} บาท`);
    }
});

console.log(`รวมชำระแล้ว: ${paidTotal} บาท`);
console.log(`รวมค้างชำระ: ${unpaidTotal} บาท`);
```

---

## 5.4 Multi-dimensional Arrays

```typescript
// ตัวอย่างที่ 25: 2D Array
const matrix: number[][] = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
];

// เข้าถึง element
console.log(matrix[1][2]); // 6

// วนลูป 2D Array
for (let row: number = 0; row < matrix.length; row++) {
    for (let col: number = 0; col < matrix[row].length; col++) {
        process.stdout?.write(`${matrix[row][col]} `);
    }
    console.log();
}
```

```typescript
// ตัวอย่างที่ 26: 2D Array - ตาราง Scores
interface ClassRoom {
    className: string;
    subjects: string[];
    scores: number[][];
}

const classroom: ClassRoom = {
    className: "ม.6/1",
    subjects: ["คณิต", "วิทย์", "ภาษาไทย", "อังกฤษ"],
    scores: [
        [85, 90, 78, 92],  // นักเรียนคนที่ 1
        [72, 88, 81, 75],  // นักเรียนคนที่ 2
        [91, 85, 89, 95],  // นักเรียนคนที่ 3
        [68, 79, 72, 80]   // นักเรียนคนที่ 4
    ]
};

classroom.scores.forEach((studentScores: number[], studentIndex: number) => {
    const avg: number = studentScores.reduce((a, b) => a + b, 0) / studentScores.length;
    console.log(`นักเรียนคนที่ ${studentIndex + 1}: เฉลี่ย ${avg.toFixed(2)}`);
});
```

```typescript
// ตัวอย่างที่ 27: 3D Array
// ข้อมูลยอดขายตาม: [ปี][เดือน][สาขา]
const salesData: number[][][] = [
    [ // ปี 2023
        [100, 120, 90],  // มกราคม - [สาขา1, สาขา2, สาขา3]
        [110, 130, 95],  // กุมภาพันธ์
        [105, 125, 100]  // มีนาคม
    ],
    [ // ปี 2024
        [115, 140, 105],
        [120, 145, 110],
        [118, 138, 115]
    ]
];

const totalSales2024: number = salesData[1].flat().reduce((a, b) => a + b, 0);
console.log(`ยอดขายรวมไตรมาส 1 ปี 2024: ${totalSales2024}`);
```

---

## 5.5 Tuple Types

### 5.5.1 การประกาศ Tuple

Tuple เป็น Array ที่กำหนดจำนวนและชนิดข้อมูลของแต่ละตำแหน่งไว้แน่นอน

```typescript
// ตัวอย่างที่ 28: Tuple พื้นฐาน
let coordinate: [number, number] = [13.7563, 100.5018];
let person: [string, number] = ["สมชาย", 30];
let rgbColor: [number, number, number] = [255, 128, 0];

console.log(`พิกัด: ${coordinate[0]}, ${coordinate[1]}`);
console.log(`${person[0]} อายุ ${person[1]} ปี`);
console.log(`RGB: rgb(${rgbColor[0]}, ${rgbColor[1]}, ${rgbColor[2]})`);
```

```typescript
// ตัวอย่างที่ 29: Tuple กับหลายชนิดข้อมูล
type ProductEntry = [number, string, number, boolean];

const product: ProductEntry = [1001, "แล็ปท็อป Asus", 25000, true];

const [productId, productName, productPrice, inStock] = product;
console.log(`สินค้า: ${productName} (ID: ${productId})`);
console.log(`ราคา: ${productPrice.toLocaleString()} บาท`);
console.log(`มีสินค้า: ${inStock ? "ใช่" : "ไม่"}`);
```

### 5.5.2 Named Tuples (TypeScript 4.0+)

```typescript
// ตัวอย่างที่ 30: Named Tuples - ทำให้โค้ดอ่านง่ายขึ้น
type Coordinate = [latitude: number, longitude: number];
type PersonInfo = [name: string, age: number, email: string];
type DateRange = [startDate: Date, endDate: Date];

const bangkokLocation: Coordinate = [13.7563, 100.5018];
const userInfo: PersonInfo = ["สมหญิง พงษ์ดี", 28, "somying@example.com"];

// Named tuples ยังใช้ index ปกติได้
console.log(`ชื่อ: ${userInfo[0]}`);
console.log(`อีเมล: ${userInfo[2]}`);

// Destructuring กับ Named Tuples
const [latitude, longitude] = bangkokLocation;
const [name, age, email] = userInfo;
console.log(`${name} อายุ ${age} ปี`);
```

### 5.5.3 Optional Elements ใน Tuples

```typescript
// ตัวอย่างที่ 31: Optional Tuple Elements
type UserData = [string, number, string?]; // string ที่ 3 เป็น optional

const user1: UserData = ["นิรันดร์", 25];
const user2: UserData = ["ปรารถนา", 30, "prarathana@example.com"];

function displayUser(userData: UserData): void {
    const [name, age, email] = userData;
    console.log(`${name}, อายุ ${age}`);
    if (email) {
        console.log(`อีเมล: ${email}`);
    }
}

displayUser(user1);
displayUser(user2);
```

```typescript
// ตัวอย่างที่ 32: Tuple ใน Function Return
function divideAndRemainder(a: number, b: number): [quotient: number, remainder: number] {
    return [Math.floor(a / b), a % b];
}

const [quotient, remainder] = divideAndRemainder(17, 5);
console.log(`17 ÷ 5 = ${quotient} เศษ ${remainder}`);
```

```typescript
// ตัวอย่างที่ 33: useState pattern (React-like) ด้วย Tuples
type StateHook<T> = [T, (newValue: T) => void];

function useState<T>(initialValue: T): StateHook<T> {
    let state: T = initialValue;
    const setState = (newValue: T): void => {
        state = newValue;
        console.log(`State เปลี่ยนเป็น: ${JSON.stringify(newValue)}`);
    };
    return [state, setState];
}

const [count, setCount] = useState<number>(0);
console.log(`count เริ่มต้น: ${count}`); // 0
setCount(5);
setCount(10);

const [userName, setUserName] = useState<string>("ไม่ระบุ");
console.log(`userName เริ่มต้น: ${userName}`);
setUserName("สมชาย");
```

---

## 5.6 Rest Elements ใน Tuples

```typescript
// ตัวอย่างที่ 34: Rest Elements ที่ท้าย Tuple
type StringWithNumbers = [string, ...number[]];

const data1: StringWithNumbers = ["ค่าเฉลี่ย", 85, 90, 78];
const data2: StringWithNumbers = ["คะแนน", 95];
const data3: StringWithNumbers = ["ข้อมูล", 1, 2, 3, 4, 5];

function processLabeledData([label, ...values]: StringWithNumbers): string {
    const avg: number = values.reduce((a, b) => a + b, 0) / values.length;
    return `${label}: ${avg.toFixed(2)}`;
}

console.log(processLabeledData(data1)); // ค่าเฉลี่ย: 84.33
console.log(processLabeledData(data2)); // คะแนน: 95.00
```

```typescript
// ตัวอย่างที่ 35: Rest Elements ที่หน้า Tuple
type NumbersWithLabel = [...number[], string];

const scores1: NumbersWithLabel = [85, 90, 78, "คะแนน"];
const scores2: NumbersWithLabel = [100, "เต็ม"];
```

```typescript
// ตัวอย่างที่ 36: Tuple Spread
type First = [string, number];
type Second = [boolean, Date];
type Combined = [...First, ...Second];

const combined: Combined = ["ชื่อ", 25, true, new Date()];
const [name2, age2, active, date] = combined;
console.log(`${name2}, ${age2}, ${active}, ${date.toLocaleDateString()}`);
```

---

## 5.7 Tuple vs Array - การเปรียบเทียบ

```typescript
// ตัวอย่างที่ 37: เมื่อไหร่ใช้ Tuple vs Array

// ใช้ Array เมื่อ: ข้อมูลประเภทเดียวกัน ไม่ทราบจำนวน
const productNames: string[] = ["แล็ปท็อป", "เมาส์", "คีย์บอร์ด", "จอ"];
const dailyTemperatures: number[] = [28, 31, 29, 33, 27];

// ใช้ Tuple เมื่อ: ข้อมูลต่างชนิด ตำแหน่งมีความหมาย
type DatabaseRecord = [id: number, name: string, createdAt: Date, isActive: boolean];
type GeoPoint = [lat: number, lng: number, elevation?: number];
type HttpResponse = [statusCode: number, body: string, headers: Record<string, string>];

const record: DatabaseRecord = [1, "ผลิตภัณฑ์ A", new Date(), true];
const point: GeoPoint = [13.7563, 100.5018, 5];
const response: HttpResponse = [200, '{"status":"ok"}', { "Content-Type": "application/json" }];
```

```typescript
// ตัวอย่างที่ 38: ตัวอย่างจริง - CSV Parsing
type CsvRow = [id: string, name: string, amount: number, date: string];

function parseCsvLine(line: string): CsvRow {
    const parts: string[] = line.split(",");
    return [
        parts[0].trim(),
        parts[1].trim(),
        parseFloat(parts[2].trim()),
        parts[3].trim()
    ];
}

const csvData: string[] = [
    "001,สมชาย,1500.50,2024-01-15",
    "002,สมหญิง,2300.00,2024-01-16",
    "003,สมศักดิ์,875.25,2024-01-17"
];

const parsedData: CsvRow[] = csvData.map(parseCsvLine);
parsedData.forEach(([id, name, amount, date]) => {
    console.log(`${id}: ${name} - ${amount.toFixed(2)} บาท (${date})`);
});
```

---

## 5.8 Practical Use Cases

### 5.8.1 Pagination

```typescript
// ตัวอย่างที่ 39: Pagination ด้วย Tuple
type PaginatedResult<T> = [data: T[], total: number, page: number, pageSize: number];

function paginate<T>(items: T[], page: number, pageSize: number): PaginatedResult<T> {
    const start: number = (page - 1) * pageSize;
    const end: number = start + pageSize;
    const data: T[] = items.slice(start, end);
    return [data, items.length, page, pageSize];
}

const allProducts: string[] = [
    "สินค้า1", "สินค้า2", "สินค้า3", "สินค้า4", "สินค้า5",
    "สินค้า6", "สินค้า7", "สินค้า8", "สินค้า9", "สินค้า10"
];

const [pageData, total, currentPage, size] = paginate(allProducts, 2, 3);
console.log(`หน้า ${currentPage} จาก ${Math.ceil(total / size)} หน้า`);
console.log(`แสดง ${pageData.length} รายการ จาก ${total} รายการ`);
console.log("รายการ:", pageData);
```

### 5.8.2 Error Handling Pattern

```typescript
// ตัวอย่างที่ 40: Go-style Error Handling ด้วย Tuple
type Result<T> = [data: T | null, error: string | null];

async function fetchUser(userId: number): Promise<Result<{ id: number; name: string }>> {
    try {
        // จำลองการเรียก API
        if (userId <= 0) {
            return [null, "รหัสผู้ใช้ไม่ถูกต้อง"];
        }
        const user = { id: userId, name: `ผู้ใช้ ${userId}` };
        return [user, null];
    } catch (err) {
        return [null, "เกิดข้อผิดพลาดในการดึงข้อมูล"];
    }
}

async function main(): Promise<void> {
    const [user, error] = await fetchUser(1);
    
    if (error) {
        console.error(`ข้อผิดพลาด: ${error}`);
        return;
    }
    
    if (user) {
        console.log(`พบผู้ใช้: ${user.name}`);
    }
}

main();
```

### 5.8.3 Data Transformation Pipeline

```typescript
// ตัวอย่างที่ 41: Data Pipeline
interface RawSalesData {
    date: string;
    product: string;
    quantity: string;
    unitPrice: string;
}

interface ProcessedSalesData {
    date: Date;
    product: string;
    quantity: number;
    unitPrice: number;
    total: number;
    vatAmount: number;
    finalPrice: number;
}

const VAT_RATE: number = 0.07;

const rawData: RawSalesData[] = [
    { date: "2024-01-15", product: "เสื้อยืด", quantity: "5", unitPrice: "299" },
    { date: "2024-01-16", product: "กางเกงยีนส์", quantity: "2", unitPrice: "899" },
    { date: "2024-01-17", product: "รองเท้า", quantity: "3", unitPrice: "1299" }
];

const processedData: ProcessedSalesData[] = rawData
    .map((raw: RawSalesData): ProcessedSalesData => {
        const quantity: number = parseInt(raw.quantity);
        const unitPrice: number = parseFloat(raw.unitPrice);
        const total: number = quantity * unitPrice;
        const vatAmount: number = total * VAT_RATE;
        return {
            date: new Date(raw.date),
            product: raw.product,
            quantity,
            unitPrice,
            total,
            vatAmount,
            finalPrice: total + vatAmount
        };
    })
    .filter((item: ProcessedSalesData): boolean => item.total > 0)
    .sort((a: ProcessedSalesData, b: ProcessedSalesData): number => b.total - a.total);

const grandTotal: number = processedData.reduce(
    (sum: number, item: ProcessedSalesData): number => sum + item.finalPrice,
    0
);

console.log("=== รายงานยอดขาย ===");
processedData.forEach((item: ProcessedSalesData) => {
    console.log(`${item.product}: ${item.quantity} ชิ้น × ${item.unitPrice} = ${item.total} (รวม VAT: ${item.finalPrice.toFixed(2)})`);
});
console.log(`ยอดรวมทั้งหมด: ${grandTotal.toFixed(2)} บาท`);
```

---

## 5.9 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Array Methods

```typescript
// โจทย์: วิเคราะห์ข้อมูลนักเรียน

interface StudentRecord {
    name: string;
    class: string;
    scores: {
        math: number;
        science: number;
        thai: number;
        english: number;
    };
}

const classData: StudentRecord[] = [
    { name: "อรนุช", class: "ม.6/1", scores: { math: 85, science: 90, thai: 78, english: 92 } },
    { name: "ภาณุ", class: "ม.6/1", scores: { math: 72, science: 68, thai: 88, english: 75 } },
    { name: "กนิษฐา", class: "ม.6/2", scores: { math: 91, science: 95, thai: 89, english: 87 } },
    { name: "วิชาญ", class: "ม.6/2", scores: { math: 65, science: 70, thai: 72, english: 68 } },
    { name: "รัตนา", class: "ม.6/1", scores: { math: 88, science: 85, thai: 91, english: 94 } }
];

// 1. คำนวณ GPA ของแต่ละคน
const withGpa = classData.map(student => {
    const { math, science, thai, english } = student.scores;
    const gpa: number = (math + science + thai + english) / 4;
    return { ...student, gpa };
});

// 2. กรองนักเรียนที่ GPA >= 80
const topStudents = withGpa.filter(s => s.gpa >= 80);
console.log("\nนักเรียนเก่ง (GPA >= 80):");
topStudents.forEach(s => console.log(`  ${s.name} (${s.class}): GPA ${s.gpa.toFixed(2)}`));

// 3. จัดเรียงตาม GPA
const ranked = [...withGpa].sort((a, b) => b.gpa - a.gpa);
console.log("\nจัดอันดับทั้งหมด:");
ranked.forEach((s, i) => console.log(`  ${i + 1}. ${s.name}: ${s.gpa.toFixed(2)}`));

// 4. หาวิชาที่คะแนนรวมสูงสุด
type SubjectKey = "math" | "science" | "thai" | "english";
const subjects: SubjectKey[] = ["math", "science", "thai", "english"];
const subjectTotals: Record<SubjectKey, number> = subjects.reduce(
    (acc, subject) => {
        acc[subject] = classData.reduce((sum, s) => sum + s.scores[subject], 0);
        return acc;
    },
    {} as Record<SubjectKey, number>
);

const bestSubject = Object.entries(subjectTotals).reduce(
    (best, [sub, total]) => total > best[1] ? [sub, total] : best,
    ["", 0]
);

console.log(`\nวิชาที่คะแนนรวมสูงสุด: ${bestSubject[0]} (${bestSubject[1]} คะแนน)`);
```

### แบบฝึกหัดที่ 2: Tuples ในสถานการณ์จริง

```typescript
// โจทย์: ระบบติดตามพัสดุ

type TrackingEvent = [
    timestamp: Date,
    location: string,
    status: "รับพัสดุ" | "ออกจากคลัง" | "ระหว่างขนส่ง" | "ถึงปลายทาง" | "ส่งสำเร็จ",
    note?: string
];

type ParcelInfo = [
    trackingId: string,
    sender: string,
    recipient: string,
    weight: number,
    events: TrackingEvent[]
];

const parcel: ParcelInfo = [
    "TH-20240115-001",
    "บริษัท ABC จำกัด",
    "คุณสมชาย กรุงเทพฯ",
    2.5,
    [
        [new Date("2024-01-15 08:00"), "คลังสินค้ากรุงเทพฯ", "รับพัสดุ", "พัสดุสภาพดี"],
        [new Date("2024-01-15 14:00"), "คลังสินค้ากรุงเทพฯ", "ออกจากคลัง"],
        [new Date("2024-01-16 10:00"), "คลังจ่ายเชียงใหม่", "ระหว่างขนส่ง"],
        [new Date("2024-01-16 16:00"), "ที่ทำการไปรษณีย์ฯ", "ถึงปลายทาง"],
        [new Date("2024-01-17 11:30"), "บ้านผู้รับ", "ส่งสำเร็จ", "ผู้รับเซ็นรับแล้ว"]
    ]
];

const [trackingId, sender, recipient, weight, events] = parcel;
console.log(`\n=== ติดตามพัสดุ ${trackingId} ===`);
console.log(`ผู้ส่ง: ${sender}`);
console.log(`ผู้รับ: ${recipient}`);
console.log(`น้ำหนัก: ${weight} กก.`);
console.log("\nประวัติการขนส่ง:");
events.forEach(([timestamp, location, status, note]) => {
    const timeStr: string = timestamp.toLocaleString('th-TH');
    console.log(`  ${timeStr} - ${location}: ${status}${note ? ` (${note})` : ''}`);
});

const lastEvent = events[events.length - 1];
const [, , lastStatus] = lastEvent;
console.log(`\nสถานะล่าสุด: ${lastStatus}`);
```

---

## สรุปบทที่ 5

| หัวข้อ | รายละเอียด |
|--------|-----------|
| `Type[]` | รูปแบบหลักในการประกาศ Array |
| `Array<Type>` | รูปแบบ Generic สำหรับ Array |
| `readonly Type[]` | Array ที่ไม่สามารถแก้ไขได้ |
| `ReadonlyArray<T>` | Generic readonly Array |
| Array Methods | map, filter, reduce, find, sort, forEach, every, some, flat, flatMap |
| Multi-dimensional | `Type[][]` สำหรับ 2D Array |
| Tuple | `[Type1, Type2, ...]` - กำหนดชนิดตามตำแหน่ง |
| Named Tuple | `[name: Type, name2: Type2]` - ตั้งชื่อแต่ละตำแหน่ง |
| Optional Tuple | `[Type1, Type2?]` - element ที่ไม่จำเป็น |
| Rest in Tuple | `[Type1, ...Type2[]]` - rest elements |

ในบทถัดไปเราจะเรียนเรื่อง **Enums** ซึ่งเป็นชนิดข้อมูลพิเศษสำหรับชุดค่าคงที่ที่เกี่ยวข้องกัน
