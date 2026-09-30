# ส่วนที่ 11: Classes & Object-Oriented Programming ใน TypeScript

## บทนำ

TypeScript มีการรองรับ Object-Oriented Programming (OOP) อย่างสมบูรณ์ ซึ่งรวมถึง classes, inheritance, abstract classes, interfaces, access modifiers และอื่นๆ อีกมากมาย TypeScript นำเอาแนวคิดของ OOP มาจาก Java และ C# แต่รันบน JavaScript runtime

ในบทนี้เราจะเรียนรู้:
- การประกาศ Class
- Constructor
- Properties และ Methods
- Access Modifiers
- Readonly Properties
- Static Members
- Inheritance
- Method Overriding
- Abstract Classes
- Implementing Interfaces
- Getters และ Setters
- Class Expressions
- Parameter Properties Shorthand
- ตัวอย่าง OOP Design จริง

---

## 11.1 การประกาศ Class พื้นฐาน

```typescript
// Class พื้นฐาน
class Person {
  name: string;
  age: number;
  
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
  
  greet(): string {
    return `สวัสดี ฉันชื่อ ${this.name} อายุ ${this.age} ปี`;
  }
  
  birthday(): void {
    this.age++;
    console.log(`สุขสันต์วันเกิด ${this.name}! ตอนนี้อายุ ${this.age} ปีแล้ว`);
  }
}

const person = new Person("สมชาย ใจดี", 30);
console.log(person.greet());  // สวัสดี ฉันชื่อ สมชาย ใจดี อายุ 30 ปี
person.birthday();             // สุขสันต์วันเกิด สมชาย ใจดี! ตอนนี้อายุ 31 ปีแล้ว
```

```typescript
// Class สำหรับสินค้า
class Product {
  id: number;
  name: string;
  price: number;
  stock: number;
  
  constructor(id: number, name: string, price: number, stock: number) {
    this.id = id;
    this.name = name;
    this.price = price;
    this.stock = stock;
  }
  
  getInfo(): string {
    return `[${this.id}] ${this.name} - ${this.price} บาท (คงเหลือ: ${this.stock})`;
  }
  
  isAvailable(): boolean {
    return this.stock > 0;
  }
  
  reduceStock(quantity: number): boolean {
    if (this.stock < quantity) {
      console.log(`สต็อกไม่เพียงพอ! มีเพียง ${this.stock} ชิ้น`);
      return false;
    }
    this.stock -= quantity;
    return true;
  }
  
  addStock(quantity: number): void {
    this.stock += quantity;
    console.log(`เพิ่มสต็อก ${quantity} ชิ้น รวม: ${this.stock} ชิ้น`);
  }
}

const laptop = new Product(1, "MacBook Pro", 59900, 5);
console.log(laptop.getInfo());
console.log(laptop.isAvailable()); // true
laptop.reduceStock(2);
console.log(laptop.stock);         // 3
```

---

## 11.2 Access Modifiers

TypeScript มี access modifiers 3 ระดับ: `public`, `private`, `protected`

```typescript
// Access Modifiers
class BankAccount {
  public accountNumber: string;     // ทุกคนเข้าถึงได้
  protected ownerName: string;      // เข้าถึงได้จาก class นี้และ subclass
  private balance: number;          // เข้าถึงได้เฉพาะใน class นี้
  private #secretPin: number;       // Private field (ES2022 syntax)
  
  constructor(accountNumber: string, ownerName: string, initialBalance: number) {
    this.accountNumber = accountNumber;
    this.ownerName = ownerName;
    this.balance = initialBalance;
    this.#secretPin = 1234; // ค่า default PIN
  }
  
  public deposit(amount: number): void {
    if (amount <= 0) {
      throw new Error("จำนวนเงินฝากต้องมากกว่า 0");
    }
    this.balance += amount;
    console.log(`ฝากเงิน ${amount} บาท ยอดคงเหลือ: ${this.balance} บาท`);
  }
  
  public withdraw(amount: number, pin: number): boolean {
    if (!this.verifyPin(pin)) {
      console.log("PIN ไม่ถูกต้อง");
      return false;
    }
    if (amount > this.balance) {
      console.log("ยอดเงินไม่เพียงพอ");
      return false;
    }
    this.balance -= amount;
    console.log(`ถอนเงิน ${amount} บาท ยอดคงเหลือ: ${this.balance} บาท`);
    return true;
  }
  
  public getBalance(): number {
    return this.balance;
  }
  
  private verifyPin(pin: number): boolean {
    return this.#secretPin === pin;
  }
  
  protected getOwnerInfo(): string {
    return `เจ้าของบัญชี: ${this.ownerName}`;
  }
}

const account = new BankAccount("1234567890", "สมชาย ใจดี", 10000);
account.deposit(5000);               // ฝากเงิน 5000 บาท ยอดคงเหลือ: 15000 บาท
account.withdraw(2000, 1234);        // ถอนเงิน 2000 บาท ยอดคงเหลือ: 13000 บาท
console.log(account.getBalance());   // 13000

// account.balance;     // Error! private
// account.verifyPin(); // Error! private
// account.ownerName;   // Error! protected
```

```typescript
// ตัวอย่าง Access Modifiers ใน Logger
class Logger {
  private static instance: Logger;
  private logs: string[] = [];
  private maxLogs: number;
  
  private constructor(maxLogs: number = 1000) {
    this.maxLogs = maxLogs;
  }
  
  public static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }
  
  public log(message: string, level: "INFO" | "WARN" | "ERROR" = "INFO"): void {
    const entry = this.formatEntry(message, level);
    this.addToLogs(entry);
    console.log(entry);
  }
  
  private formatEntry(message: string, level: string): string {
    const timestamp = new Date().toISOString();
    return `[${timestamp}] [${level}] ${message}`;
  }
  
  private addToLogs(entry: string): void {
    if (this.logs.length >= this.maxLogs) {
      this.logs.shift(); // ลบ log เก่าออก
    }
    this.logs.push(entry);
  }
  
  public getLogs(): readonly string[] {
    return this.logs;
  }
  
  public clearLogs(): void {
    this.logs = [];
    console.log("ล้าง logs ทั้งหมดแล้ว");
  }
}

const logger = Logger.getInstance();
logger.log("เริ่มต้นแอพพลิเคชัน");
logger.log("ผู้ใช้เข้าสู่ระบบ", "INFO");
logger.log("ไม่สามารถเชื่อมต่อฐานข้อมูลได้", "ERROR");
```

---

## 11.3 Readonly Properties

```typescript
// Readonly Properties ใน Class
class Circle {
  readonly id: number;
  readonly createdAt: Date;
  radius: number;
  
  constructor(id: number, radius: number) {
    this.id = id;
    this.radius = radius;
    this.createdAt = new Date();
  }
  
  get area(): number {
    return Math.PI * this.radius ** 2;
  }
  
  get circumference(): number {
    return 2 * Math.PI * this.radius;
  }
  
  scale(factor: number): Circle {
    // สร้าง circle ใหม่แทนการแก้ไข
    return new Circle(this.id, this.radius * factor);
  }
}

const circle = new Circle(1, 5);
console.log(`รัศมี: ${circle.radius}`);
console.log(`พื้นที่: ${circle.area.toFixed(2)}`);

circle.radius = 10;   // OK
// circle.id = 2;     // Error! readonly
// circle.createdAt = new Date(); // Error! readonly

const bigCircle = circle.scale(2);
console.log(`รัศมีใหม่: ${bigCircle.radius}`); // 20
```

---

## 11.4 Static Members

```typescript
// Static Properties และ Methods
class MathUtils {
  static readonly PI: number = 3.14159265358979;
  static readonly E: number = 2.71828182845905;
  private static callCount: number = 0;
  
  static add(a: number, b: number): number {
    MathUtils.callCount++;
    return a + b;
  }
  
  static subtract(a: number, b: number): number {
    MathUtils.callCount++;
    return a - b;
  }
  
  static multiply(a: number, b: number): number {
    MathUtils.callCount++;
    return a * b;
  }
  
  static divide(a: number, b: number): number {
    if (b === 0) throw new Error("หารด้วยศูนย์ไม่ได้");
    MathUtils.callCount++;
    return a / b;
  }
  
  static power(base: number, exponent: number): number {
    MathUtils.callCount++;
    return Math.pow(base, exponent);
  }
  
  static getCallCount(): number {
    return MathUtils.callCount;
  }
  
  static resetCallCount(): void {
    MathUtils.callCount = 0;
  }
}

console.log(MathUtils.add(5, 3));      // 8
console.log(MathUtils.multiply(4, 7)); // 28
console.log(MathUtils.PI);             // 3.14159265358979
console.log(`เรียกใช้ ${MathUtils.getCallCount()} ครั้ง`); // 2 ครั้ง
```

```typescript
// Singleton Pattern ด้วย Static
class DatabaseConnection {
  private static instance: DatabaseConnection | null = null;
  private static connectionCount: number = 0;
  
  private isConnected: boolean = false;
  private readonly connectionId: number;
  
  private constructor(private readonly host: string, private readonly port: number) {
    DatabaseConnection.connectionCount++;
    this.connectionId = DatabaseConnection.connectionCount;
  }
  
  static getInstance(host: string = "localhost", port: number = 5432): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection(host, port);
    }
    return DatabaseConnection.instance;
  }
  
  static getConnectionCount(): number {
    return DatabaseConnection.connectionCount;
  }
  
  static resetInstance(): void {
    if (DatabaseConnection.instance?.isConnected) {
      DatabaseConnection.instance.disconnect();
    }
    DatabaseConnection.instance = null;
  }
  
  connect(): void {
    this.isConnected = true;
    console.log(`เชื่อมต่อกับ ${this.host}:${this.port} (ID: ${this.connectionId})`);
  }
  
  disconnect(): void {
    this.isConnected = false;
    console.log(`ตัดการเชื่อมต่อ (ID: ${this.connectionId})`);
  }
  
  query<T>(sql: string): Promise<T[]> {
    if (!this.isConnected) throw new Error("ไม่ได้เชื่อมต่อกับฐานข้อมูล");
    console.log(`Query: ${sql}`);
    return Promise.resolve([]);
  }
}

const db1 = DatabaseConnection.getInstance("localhost", 5432);
const db2 = DatabaseConnection.getInstance(); // คืน instance เดิม

console.log(db1 === db2); // true - Singleton
db1.connect();
db2.query("SELECT * FROM users"); // ใช้ connection เดิม
```

---

## 11.5 Inheritance (การสืบทอด)

```typescript
// Base Class
class Animal {
  protected name: string;
  protected sound: string;
  protected speed: number;
  
  constructor(name: string, sound: string, speed: number) {
    this.name = name;
    this.sound = sound;
    this.speed = speed;
  }
  
  makeSound(): string {
    return `${this.name} ร้อง: "${this.sound}"`;
  }
  
  move(distance: number): string {
    return `${this.name} เดินทาง ${distance} เมตร ด้วยความเร็ว ${this.speed} กม./ชม.`;
  }
  
  describe(): string {
    return `${this.name} เป็นสัตว์`;
  }
  
  toString(): string {
    return `Animal(${this.name})`;
  }
}

// Derived Class
class Dog extends Animal {
  private breed: string;
  private owner: string;
  
  constructor(name: string, breed: string, owner: string) {
    super(name, "โฮ่ง", 45); // เรียก constructor ของ parent
    this.breed = breed;
    this.owner = owner;
  }
  
  fetch(item: string): string {
    return `${this.name} วิ่งไปเอา ${item} มาให้ ${this.owner}`;
  }
  
  override describe(): string {
    return `${this.name} เป็นสุนัขพันธุ์ ${this.breed} เจ้าของคือ ${this.owner}`;
  }
  
  override toString(): string {
    return `Dog(${this.name}, ${this.breed})`;
  }
}

class Cat extends Animal {
  private isIndoor: boolean;
  
  constructor(name: string, isIndoor: boolean = true) {
    super(name, "เมี้ยว", 30);
    this.isIndoor = isIndoor;
  }
  
  purr(): string {
    return `${this.name} กรรๆ อย่างเป็นสุข`;
  }
  
  override describe(): string {
    return `${this.name} เป็นแมว (${this.isIndoor ? "เลี้ยงในบ้าน" : "เลี้ยงนอกบ้าน"})`;
  }
}

class Bird extends Animal {
  private canFly: boolean;
  private wingspan: number;
  
  constructor(name: string, canFly: boolean, wingspan: number) {
    super(name, "จ๊อกๆ", canFly ? 100 : 10);
    this.canFly = canFly;
    this.wingspan = wingspan;
  }
  
  fly(altitude: number): string {
    if (!this.canFly) {
      return `${this.name} บินไม่ได้`;
    }
    return `${this.name} บินขึ้นไปที่ความสูง ${altitude} เมตร`;
  }
  
  override describe(): string {
    return `${this.name} เป็นนก ปีกกว้าง ${this.wingspan} ซม. (${this.canFly ? "บินได้" : "บินไม่ได้"})`;
  }
}

// การใช้งาน
const dog = new Dog("บัดดี้", "Golden Retriever", "สมชาย");
const cat = new Cat("วิสกี้", true);
const eagle = new Bird("อินทรี", true, 200);
const penguin = new Bird("นกเพนกวิน", false, 50);

const animals: Animal[] = [dog, cat, eagle, penguin];

animals.forEach(animal => {
  console.log(animal.describe());
  console.log(animal.makeSound());
  console.log(animal.move(100));
  console.log("---");
});

// Dog-specific methods
console.log(dog.fetch("ลูกบอล"));
console.log(cat.purr());
console.log(eagle.fly(500));
console.log(penguin.fly(100)); // บินไม่ได้
```

---

## 11.6 Method Overriding

```typescript
// Method Overriding ด้วย override keyword
class Shape {
  protected color: string;
  
  constructor(color: string = "black") {
    this.color = color;
  }
  
  area(): number {
    return 0;
  }
  
  perimeter(): number {
    return 0;
  }
  
  describe(): string {
    return `รูปทรงสี ${this.color}`;
  }
  
  toString(): string {
    return `Shape(${this.color}, area=${this.area().toFixed(2)})`;
  }
}

class CircleShape extends Shape {
  constructor(
    private radius: number,
    color: string = "red"
  ) {
    super(color);
  }
  
  override area(): number {
    return Math.PI * this.radius ** 2;
  }
  
  override perimeter(): number {
    return 2 * Math.PI * this.radius;
  }
  
  override describe(): string {
    return `วงกลมสี ${this.color} รัศมี ${this.radius}`;
  }
  
  override toString(): string {
    return `Circle(r=${this.radius}, color=${this.color})`;
  }
}

class RectangleShape extends Shape {
  constructor(
    private width: number,
    private height: number,
    color: string = "blue"
  ) {
    super(color);
  }
  
  override area(): number {
    return this.width * this.height;
  }
  
  override perimeter(): number {
    return 2 * (this.width + this.height);
  }
  
  override describe(): string {
    return `สี่เหลี่ยมสี ${this.color} ขนาด ${this.width}x${this.height}`;
  }
  
  isSquare(): boolean {
    return this.width === this.height;
  }
}

class TriangleShape extends Shape {
  constructor(
    private sideA: number,
    private sideB: number,
    private sideC: number,
    color: string = "green"
  ) {
    super(color);
  }
  
  override perimeter(): number {
    return this.sideA + this.sideB + this.sideC;
  }
  
  override area(): number {
    // สูตร Heron's Formula
    const s = this.perimeter() / 2;
    return Math.sqrt(s * (s - this.sideA) * (s - this.sideB) * (s - this.sideC));
  }
  
  override describe(): string {
    return `สามเหลี่ยมสี ${this.color} ด้าน: ${this.sideA}, ${this.sideB}, ${this.sideC}`;
  }
}

// Polymorphism
const shapes: Shape[] = [
  new CircleShape(5, "แดง"),
  new RectangleShape(4, 6, "น้ำเงิน"),
  new TriangleShape(3, 4, 5, "เขียว")
];

let totalArea = 0;
shapes.forEach(shape => {
  console.log(shape.describe());
  console.log(`  พื้นที่: ${shape.area().toFixed(2)}`);
  console.log(`  เส้นรอบวง: ${shape.perimeter().toFixed(2)}`);
  totalArea += shape.area();
});

console.log(`รวมพื้นที่ทั้งหมด: ${totalArea.toFixed(2)}`);
```

---

## 11.7 Abstract Classes

```typescript
// Abstract Class ที่ไม่สามารถสร้าง instance ได้โดยตรง
abstract class Vehicle {
  protected brand: string;
  protected model: string;
  protected year: number;
  protected mileage: number = 0;
  
  constructor(brand: string, model: string, year: number) {
    this.brand = brand;
    this.model = model;
    this.year = year;
  }
  
  // Abstract methods - ต้อง implement ใน subclass
  abstract getFuelType(): string;
  abstract getMaxSpeed(): number;
  abstract startEngine(): string;
  
  // Concrete methods - ใช้ได้ทันที
  getInfo(): string {
    return `${this.year} ${this.brand} ${this.model}`;
  }
  
  drive(distance: number): void {
    if (distance < 0) throw new Error("ระยะทางต้องเป็นค่าบวก");
    this.mileage += distance;
    console.log(`${this.getInfo()} วิ่งไป ${distance} กม. (รวม: ${this.mileage} กม.)`);
  }
  
  getMileage(): number {
    return this.mileage;
  }
  
  getAge(): number {
    return new Date().getFullYear() - this.year;
  }
  
  toString(): string {
    return `${this.getInfo()} (${this.getFuelType()})`;
  }
}

// Concrete Classes
class GasolineCar extends Vehicle {
  private fuelLevel: number; // เปอร์เซ็นต์
  private tankSize: number;  // ลิตร
  
  constructor(
    brand: string,
    model: string,
    year: number,
    tankSize: number
  ) {
    super(brand, model, year);
    this.tankSize = tankSize;
    this.fuelLevel = 100;
  }
  
  override getFuelType(): string {
    return "น้ำมันเบนซิน";
  }
  
  override getMaxSpeed(): number {
    return 220; // กม./ชม.
  }
  
  override startEngine(): string {
    if (this.fuelLevel === 0) {
      return `${this.getInfo()} น้ำมันหมด ไม่สามารถสตาร์ทได้`;
    }
    return `${this.getInfo()} วรูม! 🚗 (น้ำมัน ${this.fuelLevel}%)`;
  }
  
  refuel(amount: number): void {
    const liters = (amount / 100) * this.tankSize;
    this.fuelLevel = Math.min(100, this.fuelLevel + amount);
    console.log(`เติมน้ำมัน ${liters.toFixed(1)} ลิตร (${this.fuelLevel}%)`);
  }
  
  getFuelLevel(): number {
    return this.fuelLevel;
  }
}

class ElectricCar extends Vehicle {
  private batteryPercent: number;
  private maxRange: number; // กม. ต่อ ชาร์จเต็ม
  
  constructor(
    brand: string,
    model: string,
    year: number,
    maxRange: number
  ) {
    super(brand, model, year);
    this.maxRange = maxRange;
    this.batteryPercent = 100;
  }
  
  override getFuelType(): string {
    return "ไฟฟ้า";
  }
  
  override getMaxSpeed(): number {
    return 250;
  }
  
  override startEngine(): string {
    if (this.batteryPercent === 0) {
      return `${this.getInfo()} แบตหมด ต้องชาร์จก่อน`;
    }
    return `${this.getInfo()} เปิดระบบ ⚡ (แบต ${this.batteryPercent}%, ระยะทาง ~${this.getRemainingRange()} กม.)`;
  }
  
  charge(percent: number): void {
    this.batteryPercent = Math.min(100, this.batteryPercent + percent);
    console.log(`ชาร์จแบตเตอรี่: ${this.batteryPercent}%`);
  }
  
  getRemainingRange(): number {
    return Math.round((this.batteryPercent / 100) * this.maxRange);
  }
}

class HybridCar extends Vehicle {
  private fuelLevel: number;
  private batteryPercent: number;
  
  constructor(brand: string, model: string, year: number) {
    super(brand, model, year);
    this.fuelLevel = 100;
    this.batteryPercent = 100;
  }
  
  override getFuelType(): string {
    return "Hybrid (น้ำมัน + ไฟฟ้า)";
  }
  
  override getMaxSpeed(): number {
    return 200;
  }
  
  override startEngine(): string {
    return `${this.getInfo()} สตาร์ทระบบ Hybrid (น้ำมัน: ${this.fuelLevel}%, แบต: ${this.batteryPercent}%)`;
  }
}

// การใช้งาน
const vehicles: Vehicle[] = [
  new GasolineCar("Toyota", "Camry", 2023, 60),
  new ElectricCar("Tesla", "Model 3", 2024, 500),
  new HybridCar("Toyota", "Prius", 2023)
];

vehicles.forEach(v => {
  console.log(v.startEngine());
  v.drive(100);
  console.log(`ความเร็วสูงสุด: ${v.getMaxSpeed()} กม./ชม.`);
  console.log("---");
});
```

---

## 11.8 Implementing Interfaces ใน Class

```typescript
// Interfaces
interface Serializable {
  serialize(): string;
  toJSON(): Record<string, unknown>;
}

interface Comparable<T> {
  compareTo(other: T): number; // -1, 0, 1
  equals(other: T): boolean;
  lessThan(other: T): boolean;
  greaterThan(other: T): boolean;
}

interface Cloneable<T> {
  clone(): T;
}

// Class ที่ implement หลาย interfaces
class Temperature 
  implements Serializable, Comparable<Temperature>, Cloneable<Temperature> {
  
  constructor(
    private value: number,
    private unit: "C" | "F" | "K" = "C"
  ) {}
  
  // Convert to Celsius
  toCelsius(): number {
    switch (this.unit) {
      case "C": return this.value;
      case "F": return (this.value - 32) * 5/9;
      case "K": return this.value - 273.15;
    }
  }
  
  toFahrenheit(): number {
    const celsius = this.toCelsius();
    return (celsius * 9/5) + 32;
  }
  
  toKelvin(): number {
    return this.toCelsius() + 273.15;
  }
  
  toString(): string {
    return `${this.value}°${this.unit}`;
  }
  
  // Serializable
  serialize(): string {
    return JSON.stringify(this.toJSON());
  }
  
  toJSON(): Record<string, unknown> {
    return {
      value: this.value,
      unit: this.unit,
      celsius: this.toCelsius(),
      fahrenheit: this.toFahrenheit(),
      kelvin: this.toKelvin()
    };
  }
  
  static deserialize(json: string): Temperature {
    const data = JSON.parse(json) as { value: number; unit: "C" | "F" | "K" };
    return new Temperature(data.value, data.unit);
  }
  
  // Comparable
  compareTo(other: Temperature): number {
    const diff = this.toCelsius() - other.toCelsius();
    if (diff < 0) return -1;
    if (diff > 0) return 1;
    return 0;
  }
  
  equals(other: Temperature): boolean {
    return Math.abs(this.toCelsius() - other.toCelsius()) < 0.001;
  }
  
  lessThan(other: Temperature): boolean {
    return this.toCelsius() < other.toCelsius();
  }
  
  greaterThan(other: Temperature): boolean {
    return this.toCelsius() > other.toCelsius();
  }
  
  // Cloneable
  clone(): Temperature {
    return new Temperature(this.value, this.unit);
  }
}

const boiling = new Temperature(100, "C");
const bodyTemp = new Temperature(98.6, "F");
const absZero = new Temperature(0, "K");

console.log(`${boiling} = ${boiling.toFahrenheit()}°F = ${boiling.toKelvin()}K`);
console.log(`${bodyTemp} = ${bodyTemp.toCelsius().toFixed(1)}°C`);
console.log(`${absZero} = ${absZero.toCelsius().toFixed(2)}°C`);

console.log(`ร้อนกว่า: ${boiling.greaterThan(bodyTemp)}`);  // true
console.log(boiling.serialize());
```

---

## 11.9 Getters และ Setters

```typescript
// Getters และ Setters
class Temperature2 {
  private _celsius: number;
  
  constructor(celsius: number) {
    this._celsius = celsius;
  }
  
  // Getter
  get celsius(): number {
    return this._celsius;
  }
  
  // Setter พร้อม validation
  set celsius(value: number) {
    if (value < -273.15) {
      throw new Error("อุณหภูมิไม่สามารถต่ำกว่า Absolute Zero (-273.15°C)");
    }
    this._celsius = value;
  }
  
  get fahrenheit(): number {
    return (this._celsius * 9/5) + 32;
  }
  
  set fahrenheit(value: number) {
    this.celsius = (value - 32) * 5/9;
  }
  
  get kelvin(): number {
    return this._celsius + 273.15;
  }
  
  set kelvin(value: number) {
    if (value < 0) throw new Error("Kelvin ต้องไม่ติดลบ");
    this.celsius = value - 273.15;
  }
}

const temp = new Temperature2(25);
console.log(temp.celsius);    // 25
console.log(temp.fahrenheit); // 77
console.log(temp.kelvin);     // 298.15

temp.fahrenheit = 212;
console.log(temp.celsius);    // 100

// temp.celsius = -300; // Error!
```

```typescript
// Getters/Setters ที่ซับซ้อนขึ้น
class Rectangle2 {
  private _width: number;
  private _height: number;
  
  constructor(width: number, height: number) {
    this._width = this.validateDimension(width, "ความกว้าง");
    this._height = this.validateDimension(height, "ความสูง");
  }
  
  private validateDimension(value: number, name: string): number {
    if (value <= 0) throw new Error(`${name} ต้องมากกว่า 0`);
    if (!Number.isFinite(value)) throw new Error(`${name} ต้องเป็นตัวเลข finite`);
    return value;
  }
  
  get width(): number { return this._width; }
  set width(value: number) {
    this._width = this.validateDimension(value, "ความกว้าง");
  }
  
  get height(): number { return this._height; }
  set height(value: number) {
    this._height = this.validateDimension(value, "ความสูง");
  }
  
  get area(): number {
    return this._width * this._height;
  }
  
  get perimeter(): number {
    return 2 * (this._width + this._height);
  }
  
  get diagonal(): number {
    return Math.sqrt(this._width ** 2 + this._height ** 2);
  }
  
  get aspectRatio(): number {
    return this._width / this._height;
  }
  
  get isSquare(): boolean {
    return this._width === this._height;
  }
  
  toString(): string {
    return `Rectangle(${this._width}x${this._height})`;
  }
}

const rect = new Rectangle2(4, 6);
console.log(`ขนาด: ${rect.toString()}`);
console.log(`พื้นที่: ${rect.area}`);
console.log(`เส้นรอบวง: ${rect.perimeter}`);
console.log(`เส้นทแยงมุม: ${rect.diagonal.toFixed(2)}`);
console.log(`สัดส่วน: ${rect.aspectRatio.toFixed(2)}`);
console.log(`เป็นสี่เหลี่ยมจัตุรัส: ${rect.isSquare}`);
```

---

## 11.10 Parameter Properties Shorthand

```typescript
// Parameter Properties ย่อโค้ด constructor
class User {
  // แบบเต็ม
  // private id: number;
  // public name: string;
  // protected email: string;
  // constructor(id: number, name: string, email: string) {
  //   this.id = id;
  //   this.name = name;
  //   this.email = email;
  // }
  
  // แบบย่อ - TypeScript จะสร้าง property และ assign ให้อัตโนมัติ
  constructor(
    private readonly id: number,
    public name: string,
    protected email: string,
    private password: string,
    public readonly createdAt: Date = new Date()
  ) {}
  
  getId(): number {
    return this.id;
  }
  
  getEmail(): string {
    return this.email;
  }
  
  changePassword(oldPassword: string, newPassword: string): boolean {
    if (this.password !== oldPassword) {
      console.log("รหัสผ่านเดิมไม่ถูกต้อง");
      return false;
    }
    if (newPassword.length < 8) {
      console.log("รหัสผ่านใหม่ต้องมีอย่างน้อย 8 ตัวอักษร");
      return false;
    }
    this.password = newPassword;
    console.log("เปลี่ยนรหัสผ่านสำเร็จ");
    return true;
  }
  
  toPublicProfile(): Omit<User, "password" | "changePassword"> {
    return {
      id: this.id,
      name: this.name,
      email: this.email,
      createdAt: this.createdAt
    } as unknown as Omit<User, "password" | "changePassword">;
  }
}

const user = new User(1, "สมชาย ใจดี", "somchai@example.com", "password123");
console.log(user.name);        // สมชาย ใจดี
console.log(user.getId());     // 1
user.changePassword("password123", "newSecurePass456");
```

---

## 11.11 Class Expressions

```typescript
// Class Expression
const Point3D = class {
  constructor(
    public x: number,
    public y: number,
    public z: number
  ) {}
  
  distanceTo(other: InstanceType<typeof Point3D>): number {
    return Math.sqrt(
      (this.x - other.x) ** 2 +
      (this.y - other.y) ** 2 +
      (this.z - other.z) ** 2
    );
  }
  
  toString(): string {
    return `(${this.x}, ${this.y}, ${this.z})`;
  }
};

const p1 = new Point3D(0, 0, 0);
const p2 = new Point3D(1, 2, 3);
console.log(`ระยะห่าง: ${p1.distanceTo(p2).toFixed(2)}`);

// Named Class Expression
const createCounter = (initial: number = 0) => {
  return class Counter {
    private count: number = initial;
    private readonly initialValue: number = initial;
    
    increment(step: number = 1): void {
      this.count += step;
    }
    
    decrement(step: number = 1): void {
      this.count = Math.max(0, this.count - step);
    }
    
    reset(): void {
      this.count = this.initialValue;
    }
    
    getValue(): number {
      return this.count;
    }
    
    toString(): string {
      return `Counter(${this.count})`;
    }
  };
};

const Counter = createCounter(10);
const counter = new Counter();
counter.increment(5);
counter.increment(3);
counter.decrement(2);
console.log(counter.getValue()); // 16
console.log(counter.toString()); // Counter(16)
```

---

## 11.12 ตัวอย่าง OOP จริง: ระบบธนาคาร

```typescript
// ============ ระบบธนาคาร ============

// Types และ Interfaces
type TransactionType = "deposit" | "withdrawal" | "transfer" | "fee" | "interest";

interface ITransaction {
  id: string;
  type: TransactionType;
  amount: number;
  balance: number;
  description: string;
  timestamp: Date;
}

interface IAccount {
  accountNumber: string;
  holderName: string;
  balance: number;
  deposit(amount: number, description?: string): ITransaction;
  withdraw(amount: number, description?: string): ITransaction;
  getTransactions(): ITransaction[];
  getStatement(): string;
}

// Base Account Class
abstract class BaseAccount implements IAccount {
  abstract readonly accountType: string;
  protected _balance: number;
  protected transactions: ITransaction[] = [];
  private static accountCounter = 1000;
  
  readonly accountNumber: string;
  
  constructor(
    public readonly holderName: string,
    initialBalance: number = 0
  ) {
    this.accountNumber = `ACC-${++BaseAccount.accountCounter}`;
    this._balance = initialBalance;
    
    if (initialBalance > 0) {
      this.transactions.push(this.createTransaction(
        "deposit",
        initialBalance,
        "เปิดบัญชีพร้อมยอดเงินเริ่มต้น"
      ));
    }
  }
  
  get balance(): number {
    return this._balance;
  }
  
  protected createTransaction(
    type: TransactionType,
    amount: number,
    description: string
  ): ITransaction {
    return {
      id: `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      type,
      amount,
      balance: this._balance,
      description,
      timestamp: new Date()
    };
  }
  
  deposit(amount: number, description: string = "ฝากเงิน"): ITransaction {
    if (amount <= 0) throw new Error("จำนวนเงินต้องมากกว่า 0");
    this._balance += amount;
    const txn = this.createTransaction("deposit", amount, description);
    this.transactions.push(txn);
    console.log(`✅ ฝากเงิน ${amount.toLocaleString()} บาท | ยอดคงเหลือ: ${this._balance.toLocaleString()} บาท`);
    return txn;
  }
  
  withdraw(amount: number, description: string = "ถอนเงิน"): ITransaction {
    if (amount <= 0) throw new Error("จำนวนเงินต้องมากกว่า 0");
    this.validateWithdrawal(amount);
    this._balance -= amount;
    const txn = this.createTransaction("withdrawal", amount, description);
    this.transactions.push(txn);
    console.log(`✅ ถอนเงิน ${amount.toLocaleString()} บาท | ยอดคงเหลือ: ${this._balance.toLocaleString()} บาท`);
    return txn;
  }
  
  protected abstract validateWithdrawal(amount: number): void;
  
  transfer(amount: number, targetAccount: BaseAccount, description?: string): void {
    const desc = description ?? `โอนเงินไปบัญชี ${targetAccount.accountNumber}`;
    this.withdraw(amount, desc);
    targetAccount.deposit(amount, `รับโอนจากบัญชี ${this.accountNumber}`);
  }
  
  getTransactions(): ITransaction[] {
    return [...this.transactions];
  }
  
  getStatement(): string {
    const lines: string[] = [
      `=== บัญชี ${this.accountType} ===`,
      `เลขบัญชี: ${this.accountNumber}`,
      `ชื่อเจ้าของ: ${this.holderName}`,
      `ยอดคงเหลือ: ${this._balance.toLocaleString()} บาท`,
      ``,
      `รายการธุรกรรม:`,
      `-`.repeat(60)
    ];
    
    this.transactions.forEach(txn => {
      const sign = ["deposit", "interest"].includes(txn.type) ? "+" : "-";
      const amount = `${sign}${txn.amount.toLocaleString()}`.padStart(15);
      const balance = txn.balance.toLocaleString().padStart(15);
      lines.push(`${txn.timestamp.toLocaleDateString("th-TH")} ${amount} | ${balance} | ${txn.description}`);
    });
    
    lines.push(`-`.repeat(60));
    return lines.join("\n");
  }
}

// Savings Account
class SavingsAccount extends BaseAccount {
  readonly accountType = "ออมทรัพย์";
  private readonly interestRate: number; // % ต่อปี
  private readonly minBalance: number;
  
  constructor(
    holderName: string,
    initialBalance: number = 0,
    interestRate: number = 1.5,
    minBalance: number = 500
  ) {
    super(holderName, initialBalance);
    this.interestRate = interestRate;
    this.minBalance = minBalance;
  }
  
  protected override validateWithdrawal(amount: number): void {
    if (this._balance - amount < this.minBalance) {
      throw new Error(`ยอดเงินคงเหลือต้องไม่ต่ำกว่า ${this.minBalance.toLocaleString()} บาท`);
    }
  }
  
  calculateInterest(months: number = 12): number {
    return this._balance * (this.interestRate / 100) * (months / 12);
  }
  
  addInterest(months: number = 12): ITransaction {
    const interest = this.calculateInterest(months);
    this._balance += interest;
    const txn = this.createTransaction(
      "interest",
      interest,
      `ดอกเบี้ย ${this.interestRate}% (${months} เดือน)`
    );
    this.transactions.push(txn);
    console.log(`💰 ดอกเบี้ย ${interest.toFixed(2)} บาท | ยอดคงเหลือ: ${this._balance.toLocaleString()} บาท`);
    return txn;
  }
}

// Current Account (บัญชีกระแสรายวัน)
class CurrentAccount extends BaseAccount {
  readonly accountType = "กระแสรายวัน";
  private readonly overdraftLimit: number;
  
  constructor(
    holderName: string,
    initialBalance: number = 0,
    overdraftLimit: number = 5000
  ) {
    super(holderName, initialBalance);
    this.overdraftLimit = overdraftLimit;
  }
  
  protected override validateWithdrawal(amount: number): void {
    if (this._balance - amount < -this.overdraftLimit) {
      throw new Error(
        `เกินวงเงิน Overdraft (${this.overdraftLimit.toLocaleString()} บาท)`
      );
    }
  }
  
  getAvailableBalance(): number {
    return this._balance + this.overdraftLimit;
  }
  
  getOverdraftUsed(): number {
    return Math.max(0, -this._balance);
  }
}

// Fixed Deposit Account
class FixedDepositAccount extends BaseAccount {
  readonly accountType = "ฝากประจำ";
  private readonly maturityDate: Date;
  private readonly interestRate: number;
  private isMatured: boolean = false;
  
  constructor(
    holderName: string,
    amount: number,
    termMonths: number,
    interestRate: number = 3.5
  ) {
    super(holderName, amount);
    this.interestRate = interestRate;
    
    const maturity = new Date();
    maturity.setMonth(maturity.getMonth() + termMonths);
    this.maturityDate = maturity;
  }
  
  protected override validateWithdrawal(amount: number): void {
    if (!this.isMatured) {
      throw new Error(`ยังไม่ถึงกำหนดครบอายุ (${this.maturityDate.toLocaleDateString("th-TH")})`);
    }
    if (amount > this._balance) {
      throw new Error("จำนวนเงินเกินยอดคงเหลือ");
    }
  }
  
  checkMaturity(): void {
    if (new Date() >= this.maturityDate) {
      this.isMatured = true;
      const totalInterest = this._balance * (this.interestRate / 100);
      this._balance += totalInterest;
      console.log(`🎉 ฝากประจำครบกำหนด! ดอกเบี้ย: ${totalInterest.toFixed(2)} บาท`);
    }
  }
  
  getDaysToMaturity(): number {
    const diff = this.maturityDate.getTime() - Date.now();
    return Math.max(0, Math.ceil(diff / (1000 * 60 * 60 * 24)));
  }
}

// การใช้งาน
const savings = new SavingsAccount("สมชาย ใจดี", 10000, 2.0, 500);
const current = new CurrentAccount("สมหญิง รักเรียน", 5000, 10000);
const fixed = new FixedDepositAccount("สมศรี มีเงิน", 50000, 12, 3.5);

// ทำรายการ
savings.deposit(5000, "รับเงินเดือน");
savings.withdraw(2000, "ค่าอาหาร");
savings.addInterest(12);

current.deposit(10000, "รับค่าจ้าง");
current.withdraw(15000, "จ่ายค่าเช่า");

console.log(`ยอดบัญชีออมทรัพย์: ${savings.balance.toLocaleString()} บาท`);
console.log(`ยอดบัญชีกระแสรายวัน: ${current.balance.toLocaleString()} บาท`);
console.log(`ยอดใช้ Overdraft: ${current.getOverdraftUsed().toLocaleString()} บาท`);
console.log(`ฝากประจำครบใน: ${fixed.getDaysToMaturity()} วัน`);

// โอนเงิน
savings.transfer(3000, current, "โอนค่าใช้จ่าย");

console.log("\n" + savings.getStatement());
```

---

## 11.13 ตัวอย่าง OOP จริง: Animal Class Hierarchy

```typescript
// ============ Animal Class Hierarchy ============

interface IAnimal {
  name: string;
  species: string;
  sound(): string;
  eat(food: string): string;
  sleep(): string;
}

interface IDomesticAnimal extends IAnimal {
  owner: string;
  train(command: string): string;
}

interface IWildAnimal extends IAnimal {
  habitat: string;
  hunt(prey: string): string;
}

abstract class AnimalBase implements IAnimal {
  abstract species: string;
  private static totalAnimals = 0;
  protected readonly id: number;
  
  constructor(
    public name: string,
    protected age: number,
    protected weight: number // กก.
  ) {
    AnimalBase.totalAnimals++;
    this.id = AnimalBase.totalAnimals;
  }
  
  abstract sound(): string;
  
  eat(food: string): string {
    return `${this.name} กำลังกิน ${food}`;
  }
  
  sleep(): string {
    return `${this.name} กำลังนอนหลับ...`;
  }
  
  getInfo(): string {
    return `${this.name} (${this.species}) อายุ ${this.age} ปี น้ำหนัก ${this.weight} กก.`;
  }
  
  static getTotalCount(): number {
    return AnimalBase.totalAnimals;
  }
  
  abstract toString(): string;
}

// Domestic Animals
abstract class DomesticAnimal extends AnimalBase implements IDomesticAnimal {
  constructor(
    name: string,
    age: number,
    weight: number,
    public owner: string
  ) {
    super(name, age, weight);
  }
  
  train(command: string): string {
    return `${this.name} กำลังเรียนรู้คำสั่ง: "${command}"`;
  }
  
  override toString(): string {
    return `${this.species}(${this.name}, owner=${this.owner})`;
  }
}

class DomesticDog extends DomesticAnimal {
  readonly species = "สุนัขบ้าน";
  private tricks: string[] = [];
  
  constructor(
    name: string,
    age: number,
    weight: number,
    owner: string,
    private breed: string
  ) {
    super(name, age, weight, owner);
  }
  
  override sound(): string {
    return `${this.name}: "โฮ่ง โฮ่ง!"`;
  }
  
  learnTrick(trick: string): void {
    this.tricks.push(trick);
    console.log(`${this.name} เรียนรู้ท่า "${trick}" ได้แล้ว!`);
  }
  
  performTricks(): string {
    if (this.tricks.length === 0) {
      return `${this.name} ยังไม่รู้ท่าใดเลย`;
    }
    return `${this.name} แสดงท่า: ${this.tricks.join(", ")}`;
  }
  
  fetch(item: string): string {
    return `${this.name} วิ่งไปเอา ${item} มาให้ ${this.owner}!`;
  }
  
  override getInfo(): string {
    return `${super.getInfo()} พันธุ์: ${this.breed} เจ้าของ: ${this.owner}`;
  }
}

class DomesticCat extends DomesticAnimal {
  readonly species = "แมวบ้าน";
  private mood: "happy" | "grumpy" | "sleepy" | "playful" = "happy";
  
  constructor(
    name: string,
    age: number,
    weight: number,
    owner: string,
    private isIndoor: boolean = true
  ) {
    super(name, age, weight, owner);
  }
  
  override sound(): string {
    const sounds: Record<typeof this.mood, string> = {
      happy: "เมี้ยว~ 😊",
      grumpy: "ฟ่อ! 😾",
      sleepy: "เมี้ย...ว... 😴",
      playful: "เมี้ยวๆ! 😸"
    };
    return `${this.name}: "${sounds[this.mood]}"`;
  }
  
  setMood(mood: typeof this.mood): void {
    this.mood = mood;
    console.log(`${this.name} รู้สึก ${mood}`);
  }
  
  purr(): string {
    return `${this.name} กรรๆๆ อย่างเป็นสุข 🐱`;
  }
  
  override train(command: string): string {
    // แมวยากสอน!
    const chance = Math.random();
    if (chance > 0.7) {
      return `${this.name} เรียนรู้คำสั่ง "${command}" แต่จะทำตามก็ต่อเมื่ออยากทำเอง`;
    }
    return `${this.name} เดินออกไปโดยไม่สนใจ...`;
  }
}

// Wild Animals
abstract class WildAnimal extends AnimalBase implements IWildAnimal {
  constructor(
    name: string,
    age: number,
    weight: number,
    public habitat: string
  ) {
    super(name, age, weight);
  }
  
  hunt(prey: string): string {
    return `${this.name} ล่า ${prey} ในแหล่งที่อยู่อาศัย: ${this.habitat}`;
  }
  
  override toString(): string {
    return `${this.species}(${this.name}, habitat=${this.habitat})`;
  }
}

class Lion extends WildAnimal {
  readonly species = "สิงโต";
  private pride: string; // ฝูง
  
  constructor(
    name: string,
    age: number,
    weight: number,
    pride: string
  ) {
    super(name, age, weight, "ทุ่งหญ้าซาวันนา");
    this.pride = pride;
  }
  
  override sound(): string {
    return `${this.name}: "เกรี้ยว!!!" 🦁`;
  }
  
  override hunt(prey: string): string {
    return `${this.name} นำฝูง "${this.pride}" ล่า ${prey} บนทุ่งซาวันนา`;
  }
  
  roar(): string {
    return `${this.name} คำรามดังสนั่นป่า!`;
  }
}

class Eagle extends WildAnimal {
  readonly species = "นกอินทรี";
  private wingspan: number;
  
  constructor(
    name: string,
    age: number,
    weight: number,
    wingspan: number
  ) {
    super(name, age, weight, "ภูเขาสูง");
    this.wingspan = wingspan;
  }
  
  override sound(): string {
    return `${this.name}: "เจี๊ยวๆ" 🦅`;
  }
  
  soar(altitude: number): string {
    return `${this.name} บินร่อนที่ความสูง ${altitude} เมตร ด้วยปีกกว้าง ${this.wingspan} ซม.`;
  }
  
  dive(speed: number): string {
    return `${this.name} โฉบลงด้วยความเร็ว ${speed} กม./ชม.!`;
  }
  
  override hunt(prey: string): string {
    return `${this.name} มองเห็น ${prey} จากที่สูง แล้วโฉบลงจับ`;
  }
}

// Zoo/Animal Shelter
class AnimalShelter {
  private animals: AnimalBase[] = [];
  private name: string;
  
  constructor(name: string) {
    this.name = name;
  }
  
  addAnimal(animal: AnimalBase): void {
    this.animals.push(animal);
    console.log(`เพิ่ม ${animal.name} (${animal.species}) เข้า${this.name}`);
  }
  
  removeAnimal(animalId: number): AnimalBase | null {
    const index = this.animals.findIndex(a => (a as unknown as { id: number }).id === animalId);
    if (index === -1) return null;
    const [animal] = this.animals.splice(index, 1);
    return animal;
  }
  
  feedAll(food: string): void {
    console.log(`\n🍖 เวลาให้อาหาร - ${food}`);
    this.animals.forEach(a => console.log(a.eat(food)));
  }
  
  morningCheck(): void {
    console.log(`\n🌅 ตรวจเช้า ${this.name}`);
    this.animals.forEach(a => {
      console.log(`  ${a.getInfo()}`);
      console.log(`  ${a.sound()}`);
    });
  }
  
  getStats(): void {
    console.log(`\n📊 สถิติ ${this.name}`);
    console.log(`  สัตว์ทั้งหมด: ${this.animals.length} ตัว`);
    
    const species = new Map<string, number>();
    this.animals.forEach(a => {
      species.set(a.species, (species.get(a.species) ?? 0) + 1);
    });
    
    species.forEach((count, sp) => {
      console.log(`  ${sp}: ${count} ตัว`);
    });
  }
}

// การใช้งาน
const shelter = new AnimalShelter("สวนสัตว์แห่งชาติ");

const rex = new DomesticDog("เร็กซ์", 3, 25, "สมชาย", "German Shepherd");
const whiskers = new DomesticCat("วิสเกอร์", 2, 4, "สมหญิง");
const simba = new Lion("ซิมบ้า", 5, 180, "ฝูงราชา");
const sky = new Eagle("สกาย", 4, 6, 220);

rex.learnTrick("นั่ง");
rex.learnTrick("ล้มตาย");
rex.learnTrick("ดึงเอามือ");

shelter.addAnimal(rex);
shelter.addAnimal(whiskers);
shelter.addAnimal(simba);
shelter.addAnimal(sky);

shelter.morningCheck();
shelter.feedAll("เนื้อสด");
shelter.getStats();

console.log("\n🎪 การแสดง:");
console.log(rex.performTricks());
console.log(rex.fetch("ลูกบอล"));
console.log(whiskers.purr());
console.log(simba.roar());
console.log(sky.soar(1000));
console.log(sky.dive(320));

console.log(`\nสัตว์ทั้งหมดที่สร้าง: ${AnimalBase.getTotalCount()} ตัว`);
```

---

## 11.14 Design Patterns ด้วย Classes

```typescript
// Observer Pattern
interface Observer<T> {
  update(data: T): void;
}

abstract class Observable<T> {
  private observers: Observer<T>[] = [];
  
  subscribe(observer: Observer<T>): () => void {
    this.observers.push(observer);
    return () => this.unsubscribe(observer);
  }
  
  unsubscribe(observer: Observer<T>): void {
    this.observers = this.observers.filter(o => o !== observer);
  }
  
  protected notify(data: T): void {
    this.observers.forEach(o => o.update(data));
  }
  
  getObserverCount(): number {
    return this.observers.length;
  }
}

class StockPrice extends Observable<{ symbol: string; price: number }> {
  private prices: Map<string, number> = new Map();
  
  setPrice(symbol: string, price: number): void {
    const oldPrice = this.prices.get(symbol);
    this.prices.set(symbol, price);
    
    if (oldPrice !== price) {
      this.notify({ symbol, price });
    }
  }
  
  getPrice(symbol: string): number | undefined {
    return this.prices.get(symbol);
  }
}

class StockAlert implements Observer<{ symbol: string; price: number }> {
  private threshold: number;
  private symbol: string;
  
  constructor(symbol: string, threshold: number) {
    this.symbol = symbol;
    this.threshold = threshold;
  }
  
  update(data: { symbol: string; price: number }): void {
    if (data.symbol === this.symbol && data.price >= this.threshold) {
      console.log(`🔔 แจ้งเตือน! ${data.symbol} ราคาถึง ${data.price} บาท (เป้าหมาย: ${this.threshold})`);
    }
  }
}

const stockMarket = new StockPrice();
const alert1 = new StockAlert("KBANK", 160);
const alert2 = new StockAlert("PTT", 35);

const unsubscribe = stockMarket.subscribe(alert1);
stockMarket.subscribe(alert2);

stockMarket.setPrice("KBANK", 155);
stockMarket.setPrice("KBANK", 162);  // จะแจ้งเตือน
stockMarket.setPrice("PTT", 34);
stockMarket.setPrice("PTT", 36);    // จะแจ้งเตือน

unsubscribe(); // ยกเลิก subscription ของ alert1
stockMarket.setPrice("KBANK", 170); // ไม่มีการแจ้งเตือนแล้ว
```

```typescript
// Strategy Pattern
interface SortStrategy<T> {
  sort(data: T[]): T[];
  getName(): string;
}

class BubbleSort<T extends Comparable> implements SortStrategy<T> {
  sort(data: T[]): T[] {
    const arr = [...data];
    for (let i = 0; i < arr.length - 1; i++) {
      for (let j = 0; j < arr.length - i - 1; j++) {
        if (arr[j].compareTo(arr[j + 1]) > 0) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    return arr;
  }
  
  getName(): string { return "Bubble Sort"; }
}

interface Comparable {
  compareTo(other: this): number;
}

class SortedList<T extends Comparable> {
  private data: T[];
  private strategy: SortStrategy<T>;
  
  constructor(data: T[], strategy: SortStrategy<T>) {
    this.data = data;
    this.strategy = strategy;
  }
  
  setStrategy(strategy: SortStrategy<T>): void {
    this.strategy = strategy;
    console.log(`เปลี่ยน Strategy เป็น ${strategy.getName()}`);
  }
  
  sort(): T[] {
    const start = performance.now();
    const sorted = this.strategy.sort(this.data);
    const duration = (performance.now() - start).toFixed(3);
    console.log(`${this.strategy.getName()}: ${duration}ms`);
    return sorted;
  }
}
```

---

## 11.15 สรุปบทที่ 11

ในบทนี้เราได้เรียนรู้:

1. **Class Declaration** - การประกาศ class และสร้าง instances
2. **Access Modifiers** - `public`, `private`, `protected` และ private fields (`#`)
3. **Readonly Properties** - properties ที่ไม่สามารถแก้ไขหลัง construction
4. **Static Members** - properties และ methods ที่เป็นของ class ไม่ใช่ instance
5. **Inheritance** - `extends` สำหรับสืบทอดจาก parent class
6. **Method Overriding** - `override` keyword เพื่อ override parent methods
7. **Abstract Classes** - `abstract` class ที่ไม่สามารถสร้าง instance ได้ตรงๆ
8. **Implementing Interfaces** - `implements` หลาย interfaces ได้
9. **Getters/Setters** - computed properties พร้อม validation
10. **Parameter Properties** - shorthand ใน constructor
11. **Class Expressions** - สร้าง class โดยไม่ต้องตั้งชื่อทันที
12. **Real-world Examples** - ระบบธนาคาร, Animal Hierarchy

### หลักการ OOP หลัก

```typescript
// 1. Encapsulation - ซ่อน implementation details
class BankAccount2 {
  private balance: number = 0;  // ซ่อน internal state
  
  deposit(amount: number): void {
    // logic อยู่ใน method, ไม่ expose ตรงๆ
    if (amount > 0) this.balance += amount;
  }
}

// 2. Inheritance - สืบทอด behavior
class SavingsAccount2 extends BankAccount2 {
  // รับ deposit() มาจาก BankAccount
  addInterest(): void {
    // เพิ่ม behavior ใหม่
  }
}

// 3. Polymorphism - ใช้ base type แต่ behavior ต่างกัน
function processAccount(account: BankAccount2): void {
  account.deposit(100); // ทำงานกับ BankAccount ใดก็ได้
}

// 4. Abstraction - ซ่อน complexity ด้วย interface/abstract class
abstract class PaymentMethod {
  abstract process(amount: number): Promise<boolean>;
  
  async processWithRetry(amount: number, retries = 3): Promise<boolean> {
    // Logic ที่ซับซ้อนถูกซ่อนไว้ใน base class
    for (let i = 0; i < retries; i++) {
      const success = await this.process(amount);
      if (success) return true;
    }
    return false;
  }
}
```

---

## แบบฝึกหัดบทที่ 11

**แบบฝึกหัดที่ 1:** สร้าง class hierarchy สำหรับ Employee System ที่มี Employee, Manager, Director พร้อม salary calculation แต่ละระดับ

**แบบฝึกหัดที่ 2:** สร้าง Abstract Class สำหรับ Game Characters ที่มี Hero, Enemy, NPC พร้อม combat system

**แบบฝึกหัดที่ 3:** Implement Design Pattern (Factory Method หรือ Decorator) ด้วย TypeScript Classes

**แบบฝึกหัดที่ 4:** สร้าง Shopping Cart System ด้วย OOP ที่มี Cart, CartItem, Discount, Order classes ครบ lifecycle

---

*จบบทที่ 11: Classes & OOP*
*บทต่อไป: Part 12 - Generics*
