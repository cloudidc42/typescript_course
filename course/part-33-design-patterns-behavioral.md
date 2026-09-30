# Part 33: Behavioral Design Patterns ใน TypeScript

## บทนำ

Behavioral Design Patterns มุ่งเน้นการสื่อสารและการกำหนดความรับผิดชอบระหว่าง object รูปแบบเหล่านี้ช่วยให้การออกแบบระบบที่มีความยืดหยุ่นในเรื่องของ behavior ทำได้ง่ายขึ้น

## รูปแบบที่จะเรียนรู้ในบทนี้

1. **Observer Pattern** - การแจ้งเตือน object หลายตัวเมื่อ state เปลี่ยน
2. **Strategy Pattern** - การเลือก algorithm ตอน runtime
3. **Command Pattern** - encapsulate คำขอเป็น object
4. **Chain of Responsibility** - ส่งต่อคำขอตามลำดับ
5. **Iterator Pattern** - เข้าถึง element ของ collection แบบ sequential
6. **Mediator Pattern** - ลดการ coupling ระหว่าง object
7. **Memento Pattern** - บันทึกและกู้คืน state ของ object
8. **State Pattern** - เปลี่ยน behavior ตาม state
9. **Template Method Pattern** - กำหนด skeleton ของ algorithm
10. **Visitor Pattern** - เพิ่ม operation ให้ object โดยไม่แก้ class

---

## 1. Observer Pattern (รูปแบบผู้สังเกตการณ์)

### แนวคิด

Observer Pattern กำหนดความสัมพันธ์แบบ one-to-many ระหว่าง object เมื่อ object หนึ่ง (subject) เปลี่ยน state, observer ทั้งหมดที่ subscribe ไว้จะได้รับการแจ้งเตือนโดยอัตโนมัติ

### เมื่อไหรควรใช้ Observer Pattern

- เมื่อ object หลายตัวต้องรับรู้การเปลี่ยนแปลงของ object อื่น
- เมื่อต้องการ loosely coupled system
- เมื่อ subject ไม่รู้ว่ามี observer กี่ตัว

### โครงสร้าง Observer Pattern

```typescript
// Observer Interface
interface Observer<T> {
  update(data: T): void;
}

// Subject Interface
interface Subject<T> {
  subscribe(observer: Observer<T>): void;
  unsubscribe(observer: Observer<T>): void;
  notify(data: T): void;
}

// Generic Subject Implementation
class EventEmitter<T> implements Subject<T> {
  private observers: Set<Observer<T>> = new Set();

  subscribe(observer: Observer<T>): void {
    this.observers.add(observer);
  }

  unsubscribe(observer: Observer<T>): void {
    this.observers.delete(observer);
  }

  notify(data: T): void {
    this.observers.forEach((observer) => observer.update(data));
  }
}
```

### ตัวอย่างจริง: ระบบ Event-Driven Stock Price

```typescript
// ระบบแจ้งราคาหุ้นแบบ real-time

interface StockPrice {
  symbol: string;
  price: number;
  change: number;
  changePercent: number;
  timestamp: Date;
}

// Subject
class StockMarket {
  private prices: Map<string, StockPrice> = new Map();
  private subscribers: Map<string, Set<(price: StockPrice) => void>> = new Map();
  private globalSubscribers: Set<(price: StockPrice) => void> = new Set();

  // Subscribe ราคาหุ้นเฉพาะตัว
  subscribeStock(symbol: string, callback: (price: StockPrice) => void): () => void {
    if (!this.subscribers.has(symbol)) {
      this.subscribers.set(symbol, new Set());
    }
    this.subscribers.get(symbol)!.add(callback);
    
    // Return unsubscribe function
    return () => {
      this.subscribers.get(symbol)?.delete(callback);
    };
  }

  // Subscribe ราคาหุ้นทุกตัว
  subscribeAll(callback: (price: StockPrice) => void): () => void {
    this.globalSubscribers.add(callback);
    return () => this.globalSubscribers.delete(callback);
  }

  updatePrice(symbol: string, newPrice: number): void {
    const oldPrice = this.prices.get(symbol)?.price || newPrice;
    const change = newPrice - oldPrice;
    const changePercent = oldPrice > 0 ? (change / oldPrice) * 100 : 0;

    const priceData: StockPrice = {
      symbol,
      price: newPrice,
      change,
      changePercent,
      timestamp: new Date(),
    };

    this.prices.set(symbol, priceData);

    // แจ้ง subscribers เฉพาะหุ้นนั้น
    this.subscribers.get(symbol)?.forEach((cb) => cb(priceData));
    
    // แจ้ง global subscribers
    this.globalSubscribers.forEach((cb) => cb(priceData));
  }

  getCurrentPrice(symbol: string): StockPrice | undefined {
    return this.prices.get(symbol);
  }
}

// Observers
class PriceAlert {
  private alerts: Array<{
    symbol: string;
    targetPrice: number;
    type: "above" | "below";
    triggered: boolean;
  }> = [];

  constructor(private market: StockMarket) {}

  addAlert(symbol: string, targetPrice: number, type: "above" | "below"): void {
    this.alerts.push({ symbol, targetPrice, type, triggered: false });
    
    this.market.subscribeStock(symbol, (price) => {
      this.checkAlerts(price);
    });
  }

  private checkAlerts(price: StockPrice): void {
    for (const alert of this.alerts) {
      if (alert.symbol !== price.symbol || alert.triggered) continue;
      
      const triggered =
        (alert.type === "above" && price.price >= alert.targetPrice) ||
        (alert.type === "below" && price.price <= alert.targetPrice);
      
      if (triggered) {
        alert.triggered = true;
        console.log(
          `🔔 Alert! ${price.symbol} ${alert.type === "above" ? "↑" : "↓"} ` +
          `${alert.targetPrice} บาท (ราคาปัจจุบัน: ${price.price} บาท)`
        );
      }
    }
  }
}

class PortfolioTracker {
  private portfolio: Map<string, { shares: number; avgCost: number }> = new Map();
  private currentValues: Map<string, number> = new Map();
  private unsubscribers: (() => void)[] = [];

  constructor(private market: StockMarket) {}

  addStock(symbol: string, shares: number, avgCost: number): void {
    this.portfolio.set(symbol, { shares, avgCost });
    
    const unsubscribe = this.market.subscribeStock(symbol, (price) => {
      this.updateValue(symbol, price.price);
    });
    
    this.unsubscribers.push(unsubscribe);
  }

  private updateValue(symbol: string, currentPrice: number): void {
    this.currentValues.set(symbol, currentPrice);
    this.displaySummary();
  }

  displaySummary(): void {
    let totalInvestment = 0;
    let totalCurrentValue = 0;
    
    console.log("\n📊 Portfolio Summary:");
    
    for (const [symbol, { shares, avgCost }] of this.portfolio) {
      const currentPrice = this.currentValues.get(symbol) || avgCost;
      const investment = shares * avgCost;
      const currentValue = shares * currentPrice;
      const pnl = currentValue - investment;
      const pnlPercent = (pnl / investment) * 100;
      
      totalInvestment += investment;
      totalCurrentValue += currentValue;
      
      console.log(
        `  ${symbol}: ${shares} หุ้น @ ${currentPrice.toFixed(2)} | ` +
        `P&L: ${pnl >= 0 ? "+" : ""}${pnl.toFixed(2)} (${pnlPercent.toFixed(2)}%)`
      );
    }
    
    const totalPnL = totalCurrentValue - totalInvestment;
    const totalPnLPercent = (totalPnL / totalInvestment) * 100;
    console.log(
      `\n  รวม: ${totalCurrentValue.toFixed(2)} บาท | ` +
      `P&L: ${totalPnL >= 0 ? "+" : ""}${totalPnL.toFixed(2)} (${totalPnLPercent.toFixed(2)}%)`
    );
  }

  dispose(): void {
    this.unsubscribers.forEach((unsub) => unsub());
  }
}

class MarketDataLogger {
  private log: StockPrice[] = [];

  constructor(market: StockMarket) {
    market.subscribeAll((price) => {
      this.log.push(price);
    });
  }

  getHistory(symbol: string): StockPrice[] {
    return this.log.filter((p) => p.symbol === symbol);
  }
}

// การใช้งาน
const market = new StockMarket();

// ตั้งค่า portfolio
const portfolio = new PortfolioTracker(market);
portfolio.addStock("KBANK", 100, 160);
portfolio.addStock("PTT", 200, 35);

// ตั้ง alerts
const alerts = new PriceAlert(market);
alerts.addAlert("KBANK", 170, "above");
alerts.addAlert("PTT", 30, "below");

// Logger
const logger = new MarketDataLogger(market);

// จำลองการเปลี่ยนราคา
console.log("=== Market Open ===");
market.updatePrice("KBANK", 162);
market.updatePrice("PTT", 36);
market.updatePrice("KBANK", 168);
market.updatePrice("PTT", 32);
market.updatePrice("KBANK", 172); // จะ trigger alert
market.updatePrice("PTT", 28);   // จะ trigger alert

console.log(`\nประวัติการซื้อขาย KBANK: ${logger.getHistory("KBANK").length} รายการ`);
```

---

## 2. Strategy Pattern (รูปแบบกลยุทธ์)

### แนวคิด

Strategy Pattern กำหนด family ของ algorithm, encapsulate แต่ละตัว และทำให้สามารถ interchange กันได้ ช่วยให้ algorithm สามารถเปลี่ยนแปลงได้อิสระจาก client ที่ใช้งาน

### เมื่อไหรควรใช้ Strategy Pattern

- เมื่อต้องการ algorithm หลายตัวที่สามารถสลับกันได้
- เมื่อต้องการซ่อน implementation details ของ algorithm
- เมื่อ class มี conditional statements สำหรับ behavior ต่างๆ

### ตัวอย่างจริง: ระบบ Sorting และ Filtering

```typescript
// Strategy สำหรับ Data Processing

interface SortStrategy<T> {
  sort(data: T[], compareFn: (a: T, b: T) => number): T[];
}

interface FilterStrategy<T> {
  filter(data: T[], predicate: (item: T) => boolean): T[];
}

// Sorting Strategies
class BubbleSortStrategy<T> implements SortStrategy<T> {
  sort(data: T[], compareFn: (a: T, b: T) => number): T[] {
    const arr = [...data];
    const n = arr.length;
    
    for (let i = 0; i < n - 1; i++) {
      for (let j = 0; j < n - i - 1; j++) {
        if (compareFn(arr[j], arr[j + 1]) > 0) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    
    console.log("[Strategy] ใช้ Bubble Sort");
    return arr;
  }
}

class QuickSortStrategy<T> implements SortStrategy<T> {
  sort(data: T[], compareFn: (a: T, b: T) => number): T[] {
    if (data.length <= 1) return data;
    
    const pivot = data[Math.floor(data.length / 2)];
    const left = data.filter((item) => compareFn(item, pivot) < 0);
    const middle = data.filter((item) => compareFn(item, pivot) === 0);
    const right = data.filter((item) => compareFn(item, pivot) > 0);
    
    console.log("[Strategy] ใช้ Quick Sort");
    return [...this.sort(left, compareFn), ...middle, ...this.sort(right, compareFn)];
  }
}

class MergeSortStrategy<T> implements SortStrategy<T> {
  sort(data: T[], compareFn: (a: T, b: T) => number): T[] {
    if (data.length <= 1) return data;
    
    const mid = Math.floor(data.length / 2);
    const left = this.sort(data.slice(0, mid), compareFn);
    const right = this.sort(data.slice(mid), compareFn);
    
    console.log("[Strategy] ใช้ Merge Sort");
    return this.merge(left, right, compareFn);
  }

  private merge(left: T[], right: T[], compareFn: (a: T, b: T) => number): T[] {
    const result: T[] = [];
    let i = 0, j = 0;
    
    while (i < left.length && j < right.length) {
      if (compareFn(left[i], right[j]) <= 0) {
        result.push(left[i++]);
      } else {
        result.push(right[j++]);
      }
    }
    
    return [...result, ...left.slice(i), ...right.slice(j)];
  }
}

// DataProcessor Context
class DataProcessor<T> {
  private sortStrategy: SortStrategy<T>;

  constructor(sortStrategy: SortStrategy<T>) {
    this.sortStrategy = sortStrategy;
  }

  setSortStrategy(strategy: SortStrategy<T>): void {
    this.sortStrategy = strategy;
  }

  processData(
    data: T[],
    sortField: keyof T,
    filterFn?: (item: T) => boolean
  ): T[] {
    let processed = filterFn ? data.filter(filterFn) : [...data];
    
    return this.sortStrategy.sort(processed, (a, b) => {
      if (a[sortField] < b[sortField]) return -1;
      if (a[sortField] > b[sortField]) return 1;
      return 0;
    });
  }
}

// ตัวอย่าง: Product Catalog
interface Product {
  id: string;
  name: string;
  price: number;
  rating: number;
  category: string;
}

const products: Product[] = [
  { id: "1", name: "MacBook Pro", price: 89000, rating: 4.8, category: "laptop" },
  { id: "2", name: "iPhone 15", price: 45000, rating: 4.7, category: "phone" },
  { id: "3", name: "iPad Air", price: 32000, rating: 4.6, category: "tablet" },
  { id: "4", name: "AirPods Pro", price: 9500, rating: 4.5, category: "audio" },
  { id: "5", name: "Apple Watch", price: 18000, rating: 4.4, category: "watch" },
];

// ใช้ Strategy ต่างๆ
const processor = new DataProcessor<Product>(new QuickSortStrategy<Product>());

console.log("=== เรียงตามราคา (Quick Sort) ===");
const byPrice = processor.processData(products, "price");
byPrice.forEach((p) => console.log(`  ${p.name}: ${p.price.toLocaleString()} บาท`));

processor.setSortStrategy(new MergeSortStrategy<Product>());
console.log("\n=== เรียงตาม rating (Merge Sort) ===");
const byRating = processor.processData(products, "rating");
byRating.forEach((p) => console.log(`  ${p.name}: ⭐ ${p.rating}`));
```

### ตัวอย่างที่ 2: Payment Strategy

```typescript
// Payment Strategy Pattern

interface PaymentStrategy {
  name: string;
  validate(amount: number): boolean;
  pay(amount: number): Promise<PaymentResult>;
}

interface PaymentResult {
  success: boolean;
  transactionId: string;
  fee: number;
  message: string;
}

class CreditCardStrategy implements PaymentStrategy {
  name = "บัตรเครดิต";

  constructor(
    private cardNumber: string,
    private expiryDate: string,
    private cvv: string
  ) {}

  validate(amount: number): boolean {
    if (!this.cardNumber || this.cardNumber.length !== 16) {
      console.log("หมายเลขบัตรไม่ถูกต้อง");
      return false;
    }
    if (amount > 100000) {
      console.log("วงเงินเกินที่กำหนด");
      return false;
    }
    return true;
  }

  async pay(amount: number): Promise<PaymentResult> {
    await new Promise((resolve) => setTimeout(resolve, 300));
    const fee = amount * 0.015; // 1.5% fee
    
    return {
      success: true,
      transactionId: `CC-${Date.now()}`,
      fee,
      message: `ชำระผ่านบัตรเครดิต ****${this.cardNumber.slice(-4)}`,
    };
  }
}

class QRPaymentStrategy implements PaymentStrategy {
  name = "QR Code";

  validate(amount: number): boolean {
    return amount > 0 && amount <= 500000;
  }

  async pay(amount: number): Promise<PaymentResult> {
    console.log(`📱 กรุณาสแกน QR Code เพื่อชำระ ${amount} บาท`);
    await new Promise((resolve) => setTimeout(resolve, 500));
    
    return {
      success: true,
      transactionId: `QR-${Date.now()}`,
      fee: 0,
      message: "ชำระผ่าน QR Code สำเร็จ",
    };
  }
}

class CryptoPaymentStrategy implements PaymentStrategy {
  name = "Cryptocurrency";

  constructor(
    private walletAddress: string,
    private currency: "BTC" | "ETH" | "USDT"
  ) {}

  validate(amount: number): boolean {
    if (!this.walletAddress || this.walletAddress.length < 26) {
      return false;
    }
    return amount > 0;
  }

  async pay(amount: number): Promise<PaymentResult> {
    const exchangeRates: Record<string, number> = {
      BTC: 1500000,
      ETH: 120000,
      USDT: 35,
    };
    
    const cryptoAmount = amount / exchangeRates[this.currency];
    console.log(`💰 โอน ${cryptoAmount.toFixed(8)} ${this.currency} ไปยัง ${this.walletAddress.slice(0, 10)}...`);
    
    await new Promise((resolve) => setTimeout(resolve, 1000));
    const fee = amount * 0.001; // 0.1% fee
    
    return {
      success: true,
      transactionId: `CRYPTO-${Date.now()}`,
      fee,
      message: `ชำระด้วย ${this.currency} สำเร็จ`,
    };
  }
}

class ShoppingCart {
  private items: Array<{ name: string; price: number; quantity: number }> = [];
  private paymentStrategy: PaymentStrategy | null = null;

  addItem(name: string, price: number, quantity: number = 1): void {
    this.items.push({ name, price, quantity });
  }

  setPaymentStrategy(strategy: PaymentStrategy): void {
    this.paymentStrategy = strategy;
    console.log(`เลือกวิธีชำระเงิน: ${strategy.name}`);
  }

  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  async checkout(): Promise<void> {
    if (!this.paymentStrategy) {
      throw new Error("กรุณาเลือกวิธีชำระเงิน");
    }

    const total = this.getTotal();
    console.log(`\nยอดรวม: ${total.toLocaleString()} บาท`);

    if (!this.paymentStrategy.validate(total)) {
      throw new Error("การชำระเงินไม่ผ่านการตรวจสอบ");
    }

    const result = await this.paymentStrategy.pay(total);

    if (result.success) {
      console.log(`✅ ${result.message}`);
      console.log(`Transaction ID: ${result.transactionId}`);
      if (result.fee > 0) {
        console.log(`ค่าธรรมเนียม: ${result.fee.toFixed(2)} บาท`);
      }
    }
  }
}

// การใช้งาน
async function demoStrategy() {
  const cart = new ShoppingCart();
  cart.addItem("MacBook Pro", 89000);
  cart.addItem("Magic Mouse", 3500);

  // ชำระด้วยบัตรเครดิต
  cart.setPaymentStrategy(
    new CreditCardStrategy("1234567890123456", "12/27", "123")
  );
  await cart.checkout();

  console.log("\n--- เปลี่ยนวิธีชำระเงิน ---");

  // เปลี่ยนเป็น QR Code
  cart.setPaymentStrategy(new QRPaymentStrategy());
  await cart.checkout();
}

demoStrategy();
```

---

## 3. Command Pattern (รูปแบบคำสั่ง)

### แนวคิด

Command Pattern encapsulate คำขอ (request) เป็น object ทำให้สามารถ parameterize clients ด้วย different requests, queue หรือ log requests, และรองรับ undo operations

### เมื่อไหรควรใช้ Command Pattern

- เมื่อต้องการ parameterize object ด้วย actions
- เมื่อต้องการ queue, schedule, หรือ execute operations
- เมื่อต้องการรองรับ undo/redo operations
- เมื่อต้องการ logging ของ operations

### ตัวอย่างจริง: Text Editor พร้อม Undo/Redo

```typescript
// Command Pattern สำหรับ Text Editor

interface Command {
  execute(): void;
  undo(): void;
  redo(): void;
  getDescription(): string;
}

class TextEditor {
  private content: string = "";
  private selectionStart: number = 0;
  private selectionEnd: number = 0;

  getContent(): string {
    return this.content;
  }

  setContent(content: string): void {
    this.content = content;
  }

  getSelection(): { start: number; end: number } {
    return { start: this.selectionStart, end: this.selectionEnd };
  }

  setSelection(start: number, end: number): void {
    this.selectionStart = start;
    this.selectionEnd = end;
  }

  insertText(position: number, text: string): void {
    this.content =
      this.content.slice(0, position) + text + this.content.slice(position);
  }

  deleteText(start: number, end: number): string {
    const deleted = this.content.slice(start, end);
    this.content = this.content.slice(0, start) + this.content.slice(end);
    return deleted;
  }
}

// Concrete Commands
class InsertCommand implements Command {
  private position: number;
  private text: string;

  constructor(private editor: TextEditor, position: number, text: string) {
    this.position = position;
    this.text = text;
  }

  execute(): void {
    this.editor.insertText(this.position, this.text);
  }

  undo(): void {
    this.editor.deleteText(this.position, this.position + this.text.length);
  }

  redo(): void {
    this.execute();
  }

  getDescription(): string {
    return `แทรกข้อความ "${this.text}" ที่ตำแหน่ง ${this.position}`;
  }
}

class DeleteCommand implements Command {
  private start: number;
  private end: number;
  private deletedText: string = "";

  constructor(private editor: TextEditor, start: number, end: number) {
    this.start = start;
    this.end = end;
  }

  execute(): void {
    this.deletedText = this.editor.deleteText(this.start, this.end);
  }

  undo(): void {
    this.editor.insertText(this.start, this.deletedText);
  }

  redo(): void {
    this.execute();
  }

  getDescription(): string {
    return `ลบข้อความตำแหน่ง ${this.start}-${this.end}`;
  }
}

class ReplaceCommand implements Command {
  private originalText: string = "";
  private start: number;
  private end: number;

  constructor(
    private editor: TextEditor,
    start: number,
    end: number,
    private newText: string
  ) {
    this.start = start;
    this.end = end;
  }

  execute(): void {
    this.originalText = this.editor.deleteText(this.start, this.end);
    this.editor.insertText(this.start, this.newText);
  }

  undo(): void {
    this.editor.deleteText(this.start, this.start + this.newText.length);
    this.editor.insertText(this.start, this.originalText);
  }

  redo(): void {
    this.execute();
  }

  getDescription(): string {
    return `แทนที่ด้วย "${this.newText}"`;
  }
}

// Command Manager - จัดการ undo/redo stack
class CommandManager {
  private undoStack: Command[] = [];
  private redoStack: Command[] = [];

  executeCommand(command: Command): void {
    command.execute();
    this.undoStack.push(command);
    this.redoStack = []; // ล้าง redo stack เมื่อมีคำสั่งใหม่
    
    console.log(`✅ Execute: ${command.getDescription()}`);
  }

  undo(): boolean {
    const command = this.undoStack.pop();
    if (!command) {
      console.log("ไม่มีคำสั่งที่จะ undo");
      return false;
    }
    
    command.undo();
    this.redoStack.push(command);
    console.log(`↩️ Undo: ${command.getDescription()}`);
    return true;
  }

  redo(): boolean {
    const command = this.redoStack.pop();
    if (!command) {
      console.log("ไม่มีคำสั่งที่จะ redo");
      return false;
    }
    
    command.redo();
    this.undoStack.push(command);
    console.log(`↪️ Redo: ${command.getDescription()}`);
    return true;
  }

  canUndo(): boolean {
    return this.undoStack.length > 0;
  }

  canRedo(): boolean {
    return this.redoStack.length > 0;
  }

  getHistory(): string[] {
    return this.undoStack.map((cmd, i) => `${i + 1}. ${cmd.getDescription()}`);
  }
}

// การใช้งาน
const editor = new TextEditor();
const manager = new CommandManager();

manager.executeCommand(new InsertCommand(editor, 0, "สวัสดีครับ"));
console.log(`เนื้อหา: "${editor.getContent()}"`);

manager.executeCommand(new InsertCommand(editor, 10, " TypeScript!"));
console.log(`เนื้อหา: "${editor.getContent()}"`);

manager.executeCommand(new ReplaceCommand(editor, 10, 11, ", "));
console.log(`เนื้อหา: "${editor.getContent()}"`);

console.log("\n--- Undo ---");
manager.undo();
console.log(`เนื้อหา: "${editor.getContent()}"`);

manager.undo();
console.log(`เนื้อหา: "${editor.getContent()}"`);

console.log("\n--- Redo ---");
manager.redo();
console.log(`เนื้อหา: "${editor.getContent()}"`);

console.log("\n--- History ---");
manager.getHistory().forEach((h) => console.log(h));
```

---

## 4. Chain of Responsibility Pattern

### แนวคิด

Chain of Responsibility Pattern ส่งต่อ request ผ่าน chain ของ handlers แต่ละ handler ตัดสินใจว่าจะประมวลผล request หรือส่งต่อไปให้ handler ถัดไป

### เมื่อไหรควรใช้

- เมื่อมี handler มากกว่าหนึ่งตัวสำหรับ request
- เมื่อไม่ต้องการ specify handler explicitly
- เมื่อต้องการ issue request ให้ handler หลายตัวโดยไม่รู้ว่าตัวไหนจะจัดการ

### ตัวอย่างจริง: ระบบ Approval Workflow

```typescript
// ระบบอนุมัติค่าใช้จ่าย

interface ExpenseRequest {
  id: string;
  employeeId: string;
  amount: number;
  description: string;
  category: string;
}

interface ApprovalResult {
  approved: boolean;
  approver: string;
  comment: string;
}

// Abstract Handler
abstract class ApprovalHandler {
  protected nextHandler: ApprovalHandler | null = null;

  setNext(handler: ApprovalHandler): ApprovalHandler {
    this.nextHandler = handler;
    return handler;
  }

  abstract getApprovalLimit(): number;
  abstract getTitle(): string;

  handle(request: ExpenseRequest): ApprovalResult {
    if (request.amount <= this.getApprovalLimit()) {
      return this.approve(request);
    }
    
    if (this.nextHandler) {
      console.log(`  ${this.getTitle()}: ส่งต่อให้ระดับที่สูงกว่า (${request.amount} > ${this.getApprovalLimit()})`);
      return this.nextHandler.handle(request);
    }
    
    return {
      approved: false,
      approver: "ระบบ",
      comment: "ค่าใช้จ่ายเกินวงเงินที่อนุมัติได้",
    };
  }

  protected approve(request: ExpenseRequest): ApprovalResult {
    console.log(`✅ ${this.getTitle()} อนุมัติ ${request.amount.toLocaleString()} บาท`);
    return {
      approved: true,
      approver: this.getTitle(),
      comment: `อนุมัติโดย ${this.getTitle()}`,
    };
  }
}

// Concrete Handlers
class TeamLeadHandler extends ApprovalHandler {
  getApprovalLimit(): number { return 5000; }
  getTitle(): string { return "หัวหน้าทีม"; }
}

class ManagerHandler extends ApprovalHandler {
  getApprovalLimit(): number { return 20000; }
  getTitle(): string { return "ผู้จัดการ"; }
}

class DirectorHandler extends ApprovalHandler {
  getApprovalLimit(): number { return 100000; }
  getTitle(): string { return "ผู้อำนวยการ"; }
}

class CFOHandler extends ApprovalHandler {
  getApprovalLimit(): number { return 1000000; }
  getTitle(): string { return "CFO"; }
}

// สร้าง chain
function createApprovalChain(): ApprovalHandler {
  const teamLead = new TeamLeadHandler();
  const manager = new ManagerHandler();
  const director = new DirectorHandler();
  const cfo = new CFOHandler();

  teamLead.setNext(manager).setNext(director).setNext(cfo);

  return teamLead;
}

// การใช้งาน
const approvalChain = createApprovalChain();

const requests: ExpenseRequest[] = [
  { id: "E001", employeeId: "EMP001", amount: 3000, description: "ค่าเดินทาง", category: "travel" },
  { id: "E002", employeeId: "EMP002", amount: 15000, description: "ค่าอุปกรณ์", category: "equipment" },
  { id: "E003", employeeId: "EMP003", amount: 75000, description: "ค่าฝึกอบรม", category: "training" },
  { id: "E004", employeeId: "EMP004", amount: 500000, description: "ซื้อซอฟต์แวร์", category: "software" },
  { id: "E005", employeeId: "EMP005", amount: 2000000, description: "โครงการ IT", category: "project" },
];

for (const request of requests) {
  console.log(`\nคำขอ #${request.id}: ${request.description} (${request.amount.toLocaleString()} บาท)`);
  const result = approvalChain.handle(request);
  if (!result.approved) {
    console.log(`❌ ไม่อนุมัติ: ${result.comment}`);
  }
}
```

---

## 5. Iterator Pattern (รูปแบบตัวทำซ้ำ)

### แนวคิด

Iterator Pattern ให้วิธีการเข้าถึง element ของ collection แบบ sequential โดยไม่ expose โครงสร้างภายใน

### ตัวอย่างจริง: Tree Iterator และ Pagination Iterator

```typescript
// Tree Structure Iterator

interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[];
}

class BreadthFirstIterator<T> implements Iterator<T> {
  private queue: TreeNode<T>[];
  private done: boolean = false;

  constructor(root: TreeNode<T>) {
    this.queue = [root];
  }

  next(): IteratorResult<T> {
    if (this.queue.length === 0) {
      this.done = true;
      return { value: undefined as any, done: true };
    }

    const node = this.queue.shift()!;
    this.queue.push(...node.children);

    return { value: node.value, done: false };
  }

  [Symbol.iterator](): Iterator<T> {
    return this;
  }
}

class DepthFirstIterator<T> implements Iterator<T> {
  private stack: TreeNode<T>[];

  constructor(root: TreeNode<T>) {
    this.stack = [root];
  }

  next(): IteratorResult<T> {
    if (this.stack.length === 0) {
      return { value: undefined as any, done: true };
    }

    const node = this.stack.pop()!;
    // เพิ่ม children แบบ reverse เพื่อให้ left child มาก่อน
    this.stack.push(...[...node.children].reverse());

    return { value: node.value, done: false };
  }

  [Symbol.iterator](): Iterator<T> {
    return this;
  }
}

// Category Tree ตัวอย่าง
const categoryTree: TreeNode<string> = {
  value: "สินค้าทั้งหมด",
  children: [
    {
      value: "อิเล็กทรอนิกส์",
      children: [
        { value: "โทรศัพท์", children: [] },
        { value: "คอมพิวเตอร์", children: [
          { value: "แล็ปท็อป", children: [] },
          { value: "เดสก์ท็อป", children: [] },
        ]},
      ],
    },
    {
      value: "เสื้อผ้า",
      children: [
        { value: "เสื้อผู้ชาย", children: [] },
        { value: "เสื้อผู้หญิง", children: [] },
      ],
    },
  ],
};

console.log("=== Breadth First (ตามระดับ) ===");
const bfsIter = new BreadthFirstIterator(categoryTree);
const bfsResults: string[] = [];
let bfsResult = bfsIter.next();
while (!bfsResult.done) {
  bfsResults.push(bfsResult.value);
  bfsResult = bfsIter.next();
}
console.log(bfsResults.join(" → "));

console.log("\n=== Depth First (ตามความลึก) ===");
const dfsIter = new DepthFirstIterator(categoryTree);
const dfsResults: string[] = [];
let dfsResult = dfsIter.next();
while (!dfsResult.done) {
  dfsResults.push(dfsResult.value);
  dfsResult = dfsIter.next();
}
console.log(dfsResults.join(" → "));
```

### Pagination Iterator

```typescript
// Iterable สำหรับ Pagination

interface PaginatedResponse<T> {
  data: T[];
  page: number;
  totalPages: number;
  hasNext: boolean;
}

class PaginationIterator<T> implements AsyncIterator<T[]> {
  private currentPage: number = 1;
  private finished: boolean = false;

  constructor(
    private fetchFn: (page: number) => Promise<PaginatedResponse<T>>,
    private startPage: number = 1
  ) {
    this.currentPage = startPage;
  }

  async next(): Promise<IteratorResult<T[]>> {
    if (this.finished) {
      return { value: [], done: true };
    }

    const response = await this.fetchFn(this.currentPage);
    this.currentPage++;

    if (!response.hasNext || this.currentPage > response.totalPages + 1) {
      this.finished = true;
    }

    return { value: response.data, done: false };
  }

  [Symbol.asyncIterator](): AsyncIterator<T[]> {
    return this;
  }
}

// จำลอง API
interface User {
  id: number;
  name: string;
  email: string;
}

const allUsers: User[] = Array.from({ length: 25 }, (_, i) => ({
  id: i + 1,
  name: `ผู้ใช้ ${i + 1}`,
  email: `user${i + 1}@example.com`,
}));

async function fetchUsers(page: number, perPage: number = 10): Promise<PaginatedResponse<User>> {
  const start = (page - 1) * perPage;
  const data = allUsers.slice(start, start + perPage);
  const totalPages = Math.ceil(allUsers.length / perPage);
  
  console.log(`[API] ดึงข้อมูลหน้า ${page}/${totalPages}`);
  
  return {
    data,
    page,
    totalPages,
    hasNext: page < totalPages,
  };
}

// การใช้งาน Pagination Iterator
async function processAllUsers() {
  const iterator = new PaginationIterator<User>((page) => fetchUsers(page));
  let totalProcessed = 0;

  for await (const users of iterator) {
    totalProcessed += users.length;
    console.log(`  ประมวลผล ${users.length} users (รวม ${totalProcessed})`);
  }

  console.log(`\nประมวลผลทั้งหมด: ${totalProcessed} users`);
}

processAllUsers();
```

---

## 6. Mediator Pattern (รูปแบบผู้ไกล่เกลี่ย)

### แนวคิด

Mediator Pattern ลดการ coupling ระหว่าง objects โดยให้ communicate ผ่าน mediator object แทนที่จะ reference กันโดยตรง

### ตัวอย่างจริง: Chat Room

```typescript
// Chat Room Mediator

interface ChatMediator {
  register(user: ChatUser): void;
  sendMessage(sender: ChatUser, message: string, room?: string): void;
  createRoom(roomName: string): void;
  joinRoom(user: ChatUser, roomName: string): void;
}

class ChatUser {
  private rooms: Set<string> = new Set();

  constructor(
    private mediator: ChatMediator,
    public readonly name: string,
    public readonly id: string
  ) {
    mediator.register(this);
  }

  send(message: string, room?: string): void {
    console.log(`[${this.name}] ส่งข้อความ${room ? ` ห้อง ${room}` : ""}: ${message}`);
    this.mediator.sendMessage(this, message, room);
  }

  receive(message: string, from: string, room?: string): void {
    console.log(
      `  📩 [${this.name}] ได้รับจาก ${from}${room ? ` (${room})` : ""}: ${message}`
    );
  }

  joinRoom(roomName: string): void {
    this.mediator.joinRoom(this, roomName);
    this.rooms.add(roomName);
    console.log(`[${this.name}] เข้าห้อง ${roomName}`);
  }

  getRooms(): string[] {
    return [...this.rooms];
  }
}

class ChatRoomMediator implements ChatMediator {
  private users: Map<string, ChatUser> = new Map();
  private rooms: Map<string, Set<string>> = new Map(); // roomName -> Set of user IDs

  register(user: ChatUser): void {
    this.users.set(user.id, user);
  }

  createRoom(roomName: string): void {
    if (!this.rooms.has(roomName)) {
      this.rooms.set(roomName, new Set());
      console.log(`🏠 สร้างห้อง: ${roomName}`);
    }
  }

  joinRoom(user: ChatUser, roomName: string): void {
    if (!this.rooms.has(roomName)) {
      this.createRoom(roomName);
    }
    this.rooms.get(roomName)!.add(user.id);
  }

  sendMessage(sender: ChatUser, message: string, room?: string): void {
    if (room) {
      // ส่งในห้อง
      const roomMembers = this.rooms.get(room);
      if (!roomMembers) {
        console.log(`  ❌ ไม่พบห้อง ${room}`);
        return;
      }

      roomMembers.forEach((userId) => {
        const user = this.users.get(userId);
        if (user && user.id !== sender.id) {
          user.receive(message, sender.name, room);
        }
      });
    } else {
      // Broadcast ทุกคน
      this.users.forEach((user) => {
        if (user.id !== sender.id) {
          user.receive(message, sender.name);
        }
      });
    }
  }
}

// การใช้งาน
const chatMediator = new ChatRoomMediator();

const alice = new ChatUser(chatMediator, "Alice", "alice");
const bob = new ChatUser(chatMediator, "Bob", "bob");
const charlie = new ChatUser(chatMediator, "Charlie", "charlie");
const diana = new ChatUser(chatMediator, "Diana", "diana");

chatMediator.createRoom("typescript");
chatMediator.createRoom("general");

alice.joinRoom("typescript");
bob.joinRoom("typescript");
charlie.joinRoom("typescript");
diana.joinRoom("general");

console.log("\n=== ข้อความในห้อง typescript ===");
alice.send("สวัสดีทุกคน!", "typescript");

console.log("\n=== Alice ส่งข้อความ broadcast ===");
alice.send("ประกาศทั่วไป!");
```

---

## 7. Memento Pattern (รูปแบบความทรงจำ)

### แนวคิด

Memento Pattern บันทึก (capture) และ externalize state ภายในของ object เพื่อให้สามารถ restore กลับมาได้ในภายหลัง

### ตัวอย่างจริง: Game State

```typescript
// Game Save/Load System

interface GameState {
  level: number;
  score: number;
  playerPosition: { x: number; y: number };
  health: number;
  inventory: string[];
  checkpoint: string;
}

// Memento
class GameMemento {
  private state: GameState;
  private savedAt: Date;

  constructor(state: GameState) {
    this.state = JSON.parse(JSON.stringify(state)); // Deep copy
    this.savedAt = new Date();
  }

  getState(): GameState {
    return JSON.parse(JSON.stringify(this.state));
  }

  getSavedAt(): Date {
    return this.savedAt;
  }

  getSummary(): string {
    return (
      `Level ${this.state.level} | Score: ${this.state.score} | ` +
      `HP: ${this.state.health} | บันทึกเมื่อ: ${this.savedAt.toLocaleTimeString("th-TH")}`
    );
  }
}

// Originator
class Game {
  private state: GameState;

  constructor() {
    this.state = {
      level: 1,
      score: 0,
      playerPosition: { x: 0, y: 0 },
      health: 100,
      inventory: ["ดาบไม้"],
      checkpoint: "เริ่มต้นเกม",
    };
  }

  // Actions
  collectItem(item: string): void {
    this.state.inventory.push(item);
    this.state.score += 50;
    console.log(`  เก็บ: ${item} (+50 คะแนน)`);
  }

  movePlayer(x: number, y: number): void {
    this.state.playerPosition = { x, y };
    console.log(`  ย้ายไปยัง (${x}, ${y})`);
  }

  takeDamage(amount: number): void {
    this.state.health = Math.max(0, this.state.health - amount);
    console.log(`  รับดาเมจ ${amount} HP เหลือ: ${this.state.health}`);
  }

  gainScore(points: number): void {
    this.state.score += points;
    console.log(`  ได้รับ ${points} คะแนน รวม: ${this.state.score}`);
  }

  levelUp(): void {
    this.state.level++;
    this.state.health = 100;
    this.state.checkpoint = `เริ่ม Level ${this.state.level}`;
    console.log(`  🎉 Level Up! ตอนนี้อยู่ Level ${this.state.level}`);
  }

  // Memento operations
  save(): GameMemento {
    const memento = new GameMemento(this.state);
    console.log(`  💾 บันทึกเกม: ${memento.getSummary()}`);
    return memento;
  }

  load(memento: GameMemento): void {
    this.state = memento.getState();
    console.log(`  📂 โหลดเกม: ${memento.getSummary()}`);
  }

  getStatus(): string {
    return (
      `Level ${this.state.level} | ` +
      `Score: ${this.state.score} | ` +
      `HP: ${this.state.health} | ` +
      `Position: (${this.state.playerPosition.x}, ${this.state.playerPosition.y}) | ` +
      `ไอเทม: [${this.state.inventory.join(", ")}]`
    );
  }
}

// Caretaker - จัดการ save slots
class SaveManager {
  private saves: Map<string, GameMemento> = new Map();
  private autoSave: GameMemento | null = null;

  save(slotName: string, memento: GameMemento): void {
    this.saves.set(slotName, memento);
    console.log(`  Slot "${slotName}" บันทึกแล้ว`);
  }

  load(slotName: string): GameMemento | null {
    return this.saves.get(slotName) || null;
  }

  setAutoSave(memento: GameMemento): void {
    this.autoSave = memento;
  }

  getAutoSave(): GameMemento | null {
    return this.autoSave;
  }

  listSaves(): void {
    console.log("\n=== Save Slots ===");
    this.saves.forEach((memento, name) => {
      console.log(`  [${name}] ${memento.getSummary()}`);
    });
    if (this.autoSave) {
      console.log(`  [Auto Save] ${this.autoSave.getSummary()}`);
    }
  }
}

// การใช้งาน
const game = new Game();
const saveManager = new SaveManager();

console.log("=== เริ่มเกม ===");
console.log(`สถานะ: ${game.getStatus()}`);

console.log("\n=== เล่นเกม ===");
game.movePlayer(10, 5);
game.collectItem("โล่เหล็ก");
game.gainScore(100);

saveManager.save("Slot 1", game.save());

game.movePlayer(20, 10);
game.collectItem("ยาฮีล");
game.takeDamage(30);
game.levelUp();

saveManager.save("Slot 2", game.save());

console.log(`\nสถานะปัจจุบัน: ${game.getStatus()}`);

console.log("\n=== โหลด Slot 1 ===");
const slot1 = saveManager.load("Slot 1");
if (slot1) {
  game.load(slot1);
  console.log(`สถานะหลังโหลด: ${game.getStatus()}`);
}

saveManager.listSaves();
```

---

## 8. State Pattern (รูปแบบสถานะ)

### แนวคิด

State Pattern ให้ object เปลี่ยน behavior เมื่อ internal state เปลี่ยน object จะดูเหมือนเปลี่ยน class ของตัวเอง

### ตัวอย่างจริง: Order State Machine

```typescript
// Order State Machine

interface OrderState {
  getStatus(): string;
  confirm(order: Order): void;
  pay(order: Order): void;
  ship(order: Order): void;
  deliver(order: Order): void;
  cancel(order: Order): void;
}

class Order {
  private state: OrderState;
  private history: string[] = [];
  
  constructor(
    public readonly id: string,
    public readonly items: string[],
    public readonly totalAmount: number
  ) {
    this.state = new PendingState();
    this.logHistory("สร้างคำสั่งซื้อ");
  }

  setState(state: OrderState): void {
    this.logHistory(`เปลี่ยนสถานะเป็น: ${state.getStatus()}`);
    this.state = state;
  }

  private logHistory(action: string): void {
    const entry = `[${new Date().toLocaleTimeString("th-TH")}] ${action}`;
    this.history.push(entry);
    console.log(`  ${entry}`);
  }

  confirm(): void { this.state.confirm(this); }
  pay(): void { this.state.pay(this); }
  ship(): void { this.state.ship(this); }
  deliver(): void { this.state.deliver(this); }
  cancel(): void { this.state.cancel(this); }

  getStatus(): string { return this.state.getStatus(); }
  getHistory(): string[] { return [...this.history]; }
}

// States
class PendingState implements OrderState {
  getStatus(): string { return "รอดำเนินการ"; }

  confirm(order: Order): void {
    console.log(`  ✅ ยืนยันคำสั่งซื้อ #${order.id}`);
    order.setState(new ConfirmedState());
  }

  pay(order: Order): void {
    console.log("  ❌ ต้องยืนยันก่อนชำระเงิน");
  }

  ship(order: Order): void {
    console.log("  ❌ ไม่สามารถจัดส่งได้ ยังไม่ได้ชำระเงิน");
  }

  deliver(order: Order): void {
    console.log("  ❌ ยังไม่ได้จัดส่ง");
  }

  cancel(order: Order): void {
    console.log(`  ❌ ยกเลิกคำสั่งซื้อ #${order.id}`);
    order.setState(new CancelledState());
  }
}

class ConfirmedState implements OrderState {
  getStatus(): string { return "ยืนยันแล้ว"; }

  confirm(order: Order): void {
    console.log("  ⚠️ ยืนยันแล้ว");
  }

  pay(order: Order): void {
    console.log(`  💳 ชำระเงินสำหรับ #${order.id} จำนวน ${order.totalAmount} บาท`);
    order.setState(new PaidState());
  }

  ship(order: Order): void {
    console.log("  ❌ ต้องชำระเงินก่อนจัดส่ง");
  }

  deliver(order: Order): void {
    console.log("  ❌ ยังไม่ได้จัดส่ง");
  }

  cancel(order: Order): void {
    console.log(`  ❌ ยกเลิกคำสั่งซื้อ #${order.id}`);
    order.setState(new CancelledState());
  }
}

class PaidState implements OrderState {
  getStatus(): string { return "ชำระเงินแล้ว"; }

  confirm(order: Order): void { console.log("  ⚠️ ยืนยันแล้ว"); }

  pay(order: Order): void { console.log("  ⚠️ ชำระเงินแล้ว"); }

  ship(order: Order): void {
    console.log(`  🚚 จัดส่งคำสั่งซื้อ #${order.id}`);
    order.setState(new ShippedState());
  }

  deliver(order: Order): void {
    console.log("  ❌ ยังไม่ได้จัดส่ง");
  }

  cancel(order: Order): void {
    console.log(`  🔄 ยกเลิกและคืนเงินสำหรับ #${order.id}`);
    order.setState(new CancelledState());
  }
}

class ShippedState implements OrderState {
  getStatus(): string { return "กำลังจัดส่ง"; }

  confirm(order: Order): void { console.log("  ⚠️ ยืนยันแล้ว"); }
  pay(order: Order): void { console.log("  ⚠️ ชำระเงินแล้ว"); }
  ship(order: Order): void { console.log("  ⚠️ จัดส่งแล้ว"); }

  deliver(order: Order): void {
    console.log(`  📦 ส่งมอบสำเร็จ #${order.id}`);
    order.setState(new DeliveredState());
  }

  cancel(order: Order): void {
    console.log("  ❌ ไม่สามารถยกเลิกได้ กำลังจัดส่ง");
  }
}

class DeliveredState implements OrderState {
  getStatus(): string { return "ส่งมอบแล้ว"; }

  confirm(order: Order): void { console.log("  ⚠️ สั่งซื้อเสร็จสิ้น"); }
  pay(order: Order): void { console.log("  ⚠️ สั่งซื้อเสร็จสิ้น"); }
  ship(order: Order): void { console.log("  ⚠️ ส่งมอบแล้ว"); }
  deliver(order: Order): void { console.log("  ⚠️ ส่งมอบแล้ว"); }

  cancel(order: Order): void {
    console.log("  ❌ ไม่สามารถยกเลิกได้ ส่งมอบแล้ว");
  }
}

class CancelledState implements OrderState {
  getStatus(): string { return "ยกเลิกแล้ว"; }

  confirm(order: Order): void { console.log("  ❌ คำสั่งซื้อถูกยกเลิก"); }
  pay(order: Order): void { console.log("  ❌ คำสั่งซื้อถูกยกเลิก"); }
  ship(order: Order): void { console.log("  ❌ คำสั่งซื้อถูกยกเลิก"); }
  deliver(order: Order): void { console.log("  ❌ คำสั่งซื้อถูกยกเลิก"); }
  cancel(order: Order): void { console.log("  ⚠️ ยกเลิกแล้ว"); }
}

// การใช้งาน
const order = new Order("ORD-001", ["MacBook Pro", "Magic Mouse"], 92500);

console.log(`\nสถานะเริ่มต้น: ${order.getStatus()}`);

order.confirm();
order.pay();
order.ship();
order.deliver();

console.log(`\nสถานะสุดท้าย: ${order.getStatus()}`);

console.log("\n=== ประวัติคำสั่งซื้อ ===");
order.getHistory().forEach((h) => console.log(h));
```

---

## 9. Template Method Pattern

### แนวคิด

Template Method Pattern กำหนด skeleton ของ algorithm ใน base class และ defer บางขั้นตอนไปให้ subclasses

### ตัวอย่างจริง: Report Generator

```typescript
// Abstract Report Generator

abstract class ReportGenerator {
  // Template Method
  async generate(data: any[]): Promise<string> {
    console.log(`📊 เริ่มสร้าง ${this.getReportType()} Report`);
    
    const validated = await this.validateData(data);
    const processed = await this.processData(validated);
    const formatted = await this.formatData(processed);
    const header = this.generateHeader();
    const footer = this.generateFooter();
    const report = this.compileReport(header, formatted, footer);
    
    await this.saveReport(report);
    
    console.log(`✅ สร้าง ${this.getReportType()} Report สำเร็จ`);
    return report;
  }

  // Abstract methods - subclass ต้องนำไปใช้
  protected abstract getReportType(): string;
  protected abstract processData(data: any[]): Promise<any>;
  protected abstract formatData(data: any): Promise<string>;

  // Hook methods - subclass อาจ override หรือไม่ก็ได้
  protected async validateData(data: any[]): Promise<any[]> {
    console.log(`  ตรวจสอบข้อมูล ${data.length} รายการ`);
    return data.filter((item) => item !== null && item !== undefined);
  }

  protected generateHeader(): string {
    return `=== รายงาน ${this.getReportType()} ===\nวันที่: ${new Date().toLocaleDateString("th-TH")}\n\n`;
  }

  protected generateFooter(): string {
    return "\n\n--- สิ้นสุดรายงาน ---";
  }

  protected compileReport(header: string, body: string, footer: string): string {
    return header + body + footer;
  }

  protected async saveReport(report: string): Promise<void> {
    console.log(`  บันทึกรายงาน (${report.length} ตัวอักษร)`);
  }
}

// Sales Report
class SalesReportGenerator extends ReportGenerator {
  protected getReportType(): string {
    return "ยอดขาย";
  }

  protected async processData(data: any[]): Promise<any> {
    const totalRevenue = data.reduce((sum, item) => sum + item.revenue, 0);
    const byCategory = data.reduce((acc, item) => {
      acc[item.category] = (acc[item.category] || 0) + item.revenue;
      return acc;
    }, {});

    return { items: data, totalRevenue, byCategory };
  }

  protected async formatData(data: any): Promise<string> {
    let result = "ยอดขายรายละเอียด:\n";
    
    for (const item of data.items) {
      result += `  - ${item.product}: ${item.revenue.toLocaleString()} บาท\n`;
    }
    
    result += "\nยอดขายตามหมวดหมู่:\n";
    for (const [category, revenue] of Object.entries(data.byCategory)) {
      result += `  ${category}: ${(revenue as number).toLocaleString()} บาท\n`;
    }
    
    result += `\nยอดรวม: ${data.totalRevenue.toLocaleString()} บาท`;
    
    return result;
  }
}

// Inventory Report
class InventoryReportGenerator extends ReportGenerator {
  protected getReportType(): string {
    return "คลังสินค้า";
  }

  protected async validateData(data: any[]): Promise<any[]> {
    const validated = await super.validateData(data);
    return validated.filter((item) => item.quantity >= 0);
  }

  protected async processData(data: any[]): Promise<any> {
    const lowStock = data.filter((item) => item.quantity < item.minStock);
    const outOfStock = data.filter((item) => item.quantity === 0);
    const totalValue = data.reduce(
      (sum, item) => sum + item.quantity * item.unitCost,
      0
    );

    return { items: data, lowStock, outOfStock, totalValue };
  }

  protected async formatData(data: any): Promise<string> {
    let result = "";
    
    if (data.outOfStock.length > 0) {
      result += "⚠️ สินค้าหมด:\n";
      data.outOfStock.forEach((item: any) => {
        result += `  ❌ ${item.name} (รหัส: ${item.sku})\n`;
      });
    }
    
    if (data.lowStock.length > 0) {
      result += "\n⚠️ สินค้าใกล้หมด:\n";
      data.lowStock.forEach((item: any) => {
        result += `  ⚡ ${item.name}: เหลือ ${item.quantity}/${item.minStock}\n`;
      });
    }
    
    result += `\nมูลค่าคลังสินค้ารวม: ${data.totalValue.toLocaleString()} บาท`;
    
    return result;
  }
}

// การใช้งาน
async function demoTemplate() {
  const salesGenerator = new SalesReportGenerator();
  const salesData = [
    { product: "MacBook Pro", revenue: 267000, category: "คอมพิวเตอร์" },
    { product: "iPhone 15", revenue: 180000, category: "โทรศัพท์" },
    { product: "iPad Air", revenue: 96000, category: "แท็บเล็ต" },
  ];
  
  const salesReport = await salesGenerator.generate(salesData);
  console.log(salesReport);

  const inventoryGenerator = new InventoryReportGenerator();
  const inventoryData = [
    { name: "MacBook Pro", sku: "MBP-001", quantity: 5, minStock: 10, unitCost: 89000 },
    { name: "iPhone 15", sku: "IP15-001", quantity: 0, minStock: 20, unitCost: 45000 },
    { name: "AirPods Pro", sku: "APP-001", quantity: 50, minStock: 15, unitCost: 9500 },
  ];
  
  const inventoryReport = await inventoryGenerator.generate(inventoryData);
  console.log(inventoryReport);
}

demoTemplate();
```

---

## 10. Visitor Pattern (รูปแบบผู้เยี่ยมชม)

### แนวคิด

Visitor Pattern ช่วยให้สามารถเพิ่ม operation ใหม่ๆ ให้กับ class hierarchy โดยไม่ต้องแก้ไข class เหล่านั้น

### ตัวอย่างจริง: AST (Abstract Syntax Tree) Visitor

```typescript
// Document Element Visitor

interface DocumentVisitor {
  visitHeading(heading: HeadingElement): string;
  visitParagraph(paragraph: ParagraphElement): string;
  visitImage(image: ImageElement): string;
  visitList(list: ListElement): string;
  visitCodeBlock(code: CodeBlockElement): string;
}

interface DocumentElement {
  accept(visitor: DocumentVisitor): string;
}

// Element classes
class HeadingElement implements DocumentElement {
  constructor(
    public text: string,
    public level: 1 | 2 | 3 | 4 | 5 | 6
  ) {}

  accept(visitor: DocumentVisitor): string {
    return visitor.visitHeading(this);
  }
}

class ParagraphElement implements DocumentElement {
  constructor(public text: string) {}

  accept(visitor: DocumentVisitor): string {
    return visitor.visitParagraph(this);
  }
}

class ImageElement implements DocumentElement {
  constructor(
    public src: string,
    public alt: string,
    public width?: number,
    public height?: number
  ) {}

  accept(visitor: DocumentVisitor): string {
    return visitor.visitImage(this);
  }
}

class ListElement implements DocumentElement {
  constructor(
    public items: string[],
    public ordered: boolean = false
  ) {}

  accept(visitor: DocumentVisitor): string {
    return visitor.visitList(this);
  }
}

class CodeBlockElement implements DocumentElement {
  constructor(
    public code: string,
    public language: string
  ) {}

  accept(visitor: DocumentVisitor): string {
    return visitor.visitCodeBlock(this);
  }
}

// Visitors
class HTMLExporter implements DocumentVisitor {
  visitHeading(heading: HeadingElement): string {
    return `<h${heading.level}>${heading.text}</h${heading.level}>`;
  }

  visitParagraph(paragraph: ParagraphElement): string {
    return `<p>${paragraph.text}</p>`;
  }

  visitImage(image: ImageElement): string {
    const dims = image.width ? ` width="${image.width}" height="${image.height}"` : "";
    return `<img src="${image.src}" alt="${image.alt}"${dims} />`;
  }

  visitList(list: ListElement): string {
    const tag = list.ordered ? "ol" : "ul";
    const items = list.items.map((item) => `  <li>${item}</li>`).join("\n");
    return `<${tag}>\n${items}\n</${tag}>`;
  }

  visitCodeBlock(code: CodeBlockElement): string {
    return `<pre><code class="language-${code.language}">${code.code}</code></pre>`;
  }
}

class MarkdownExporter implements DocumentVisitor {
  visitHeading(heading: HeadingElement): string {
    return `${"#".repeat(heading.level)} ${heading.text}`;
  }

  visitParagraph(paragraph: ParagraphElement): string {
    return paragraph.text;
  }

  visitImage(image: ImageElement): string {
    return `![${image.alt}](${image.src})`;
  }

  visitList(list: ListElement): string {
    return list.items
      .map((item, i) => (list.ordered ? `${i + 1}. ${item}` : `- ${item}`))
      .join("\n");
  }

  visitCodeBlock(code: CodeBlockElement): string {
    return `\`\`\`${code.language}\n${code.code}\n\`\`\``;
  }
}

class WordCountVisitor implements DocumentVisitor {
  private totalWords: number = 0;

  visitHeading(heading: HeadingElement): string {
    this.totalWords += heading.text.split(/\s+/).length;
    return "";
  }

  visitParagraph(paragraph: ParagraphElement): string {
    this.totalWords += paragraph.text.split(/\s+/).length;
    return "";
  }

  visitImage(image: ImageElement): string {
    this.totalWords += image.alt.split(/\s+/).length;
    return "";
  }

  visitList(list: ListElement): string {
    list.items.forEach((item) => {
      this.totalWords += item.split(/\s+/).length;
    });
    return "";
  }

  visitCodeBlock(code: CodeBlockElement): string {
    return ""; // ไม่นับ code
  }

  getTotalWords(): number {
    return this.totalWords;
  }
}

// Document class
class Document {
  private elements: DocumentElement[] = [];

  add(element: DocumentElement): this {
    this.elements.push(element);
    return this;
  }

  export(visitor: DocumentVisitor): string {
    return this.elements.map((el) => el.accept(visitor)).join("\n\n");
  }
}

// การใช้งาน
const doc = new Document();

doc
  .add(new HeadingElement("TypeScript Design Patterns", 1))
  .add(new ParagraphElement("Design Patterns คือแนวทางการแก้ปัญหาที่พบบ่อยในการพัฒนาซอฟต์แวร์"))
  .add(new HeadingElement("Structural Patterns", 2))
  .add(new ListElement(["Adapter", "Bridge", "Composite", "Decorator"], true))
  .add(new CodeBlockElement(`interface Observer {
  update(data: any): void;
}`, "typescript"))
  .add(new ImageElement("/images/patterns.png", "Design Patterns Diagram", 800, 600));

console.log("=== HTML Export ===");
const htmlExporter = new HTMLExporter();
console.log(doc.export(htmlExporter));

console.log("\n=== Markdown Export ===");
const markdownExporter = new MarkdownExporter();
console.log(doc.export(markdownExporter));

console.log("\n=== Word Count ===");
const wordCounter = new WordCountVisitor();
doc.export(wordCounter);
console.log(`จำนวนคำทั้งหมด: ${wordCounter.getTotalWords()} คำ`);
```

---

## สรุป Behavioral Design Patterns

| Pattern | วัตถุประสงค์ | Use Case หลัก |
|---------|------------|--------------|
| **Observer** | แจ้งเตือน objects หลายตัว | Event systems, Pub/Sub, Real-time updates |
| **Strategy** | เลือก algorithm ตอน runtime | Payment methods, Sorting, Compression |
| **Command** | Encapsulate request | Undo/Redo, Transaction, Queue |
| **Chain of Responsibility** | ส่งต่อ request | Middleware, Approval workflow |
| **Iterator** | เข้าถึง collection | Data traversal, Pagination |
| **Mediator** | ลด coupling | Chat systems, Air traffic control |
| **Memento** | บันทึก/กู้คืน state | Game saves, Editor history |
| **State** | เปลี่ยน behavior ตาม state | Order status, Traffic lights |
| **Template Method** | กำหนด algorithm skeleton | Report generation, Data import |
| **Visitor** | เพิ่ม operation ใหม่ | Code analysis, Document export |

## แบบฝึกหัด

1. สร้างระบบ Event Bus ที่รองรับ wildcard subscriptions ด้วย Observer Pattern
2. ใช้ Strategy Pattern สำหรับระบบ compression ที่รองรับ gzip, brotli, zstd
3. สร้าง Command pattern พร้อม macro commands (หลาย commands ใน command เดียว)
4. ออกแบบ Chain of Responsibility สำหรับ HTTP request validation
5. สร้าง Custom Iterator สำหรับ Infinite Scroll
6. ใช้ Mediator สำหรับ form validation ที่มี dependent fields
7. สร้าง Memento สำหรับ document collaboration (multiple users)
8. ออกแบบ State Machine สำหรับ payment flow
9. ใช้ Template Method สำหรับ data import pipeline
10. สร้าง Visitor Pattern สำหรับ query builder
