# ส่วนที่ 62: TDD with TypeScript (Test-Driven Development)

## บทนำ

Test-Driven Development (TDD) เป็นวิธีการพัฒนาซอฟต์แวร์ที่เขียน test ก่อนแล้วค่อยเขียน code ให้ผ่าน test นั้น วงจรหลักของ TDD คือ Red-Green-Refactor ซึ่งช่วยให้โค้ดมีคุณภาพสูงและออกแบบได้ดีขึ้น

---

## 1. วงจร Red-Green-Refactor

```
🔴 Red   → เขียน test ที่ fail
🟢 Green → เขียน code ให้ test ผ่าน
🔵 Blue  → Refactor code ให้ดีขึ้น
```

### 1.1 ตัวอย่าง TDD วงจรพื้นฐาน

**ขั้นตอนที่ 1: Red - เขียน test ที่ fail**

```typescript
// calculator.test.ts - เขียนก่อน
import { Calculator } from './calculator';

describe('Calculator', () => {
  let calc: Calculator;

  beforeEach(() => {
    calc = new Calculator();
  });

  // Test นี้จะ fail เพราะยังไม่มี Calculator class
  test('บวกเลขสองตัว', () => {
    expect(calc.add(2, 3)).toBe(5);
  });
});
```

**ขั้นตอนที่ 2: Green - เขียน code ให้ผ่าน**

```typescript
// calculator.ts - เขียนขั้นต่ำสุดให้ test ผ่าน
export class Calculator {
  add(a: number, b: number): number {
    return a + b;
  }
}
```

**ขั้นตอนที่ 3: เพิ่ม tests แล้ว Refactor**

```typescript
// calculator.test.ts - เพิ่ม tests มากขึ้น
describe('Calculator', () => {
  let calc: Calculator;

  beforeEach(() => {
    calc = new Calculator();
  });

  describe('add', () => {
    test('บวกเลขบวกสองตัว', () => {
      expect(calc.add(2, 3)).toBe(5);
    });

    test('บวกเลขลบ', () => {
      expect(calc.add(-2, 3)).toBe(1);
    });

    test('บวกศูนย์', () => {
      expect(calc.add(5, 0)).toBe(5);
    });

    test('บวกเลขทศนิยม', () => {
      expect(calc.add(1.5, 2.5)).toBeCloseTo(4.0);
    });
  });

  describe('subtract', () => {
    test('ลบเลขสองตัว', () => {
      expect(calc.subtract(5, 3)).toBe(2);
    });

    test('ผลลัพธ์เป็นลบ', () => {
      expect(calc.subtract(3, 5)).toBe(-2);
    });
  });

  describe('multiply', () => {
    test('คูณเลขสองตัว', () => {
      expect(calc.multiply(4, 3)).toBe(12);
    });

    test('คูณด้วยศูนย์', () => {
      expect(calc.multiply(5, 0)).toBe(0);
    });
  });

  describe('divide', () => {
    test('หารเลขสองตัว', () => {
      expect(calc.divide(10, 2)).toBe(5);
    });

    test('หารด้วยศูนย์ควรโยน error', () => {
      expect(() => calc.divide(5, 0)).toThrow('ไม่สามารถหารด้วยศูนย์ได้');
    });
  });
});
```

```typescript
// calculator.ts - Refactor ให้สมบูรณ์
export class Calculator {
  add(a: number, b: number): number {
    return a + b;
  }

  subtract(a: number, b: number): number {
    return a - b;
  }

  multiply(a: number, b: number): number {
    return a * b;
  }

  divide(a: number, b: number): number {
    if (b === 0) {
      throw new Error('ไม่สามารถหารด้วยศูนย์ได้');
    }
    return a / b;
  }

  // เพิ่ม method ใหม่หลังจาก test ผ่านทั้งหมด
  power(base: number, exponent: number): number {
    return Math.pow(base, exponent);
  }

  sqrt(n: number): number {
    if (n < 0) {
      throw new Error('ไม่สามารถหารากที่สองของเลขลบได้');
    }
    return Math.sqrt(n);
  }
}
```

---

## 2. TDD สำหรับ Classes

### 2.1 การสร้าง Shopping Cart ด้วย TDD

**ขั้นตอนที่ 1: เขียน tests ทั้งหมดก่อน**

```typescript
// shopping-cart.test.ts
import { ShoppingCart } from './shopping-cart';
import { Product } from './product';

describe('ShoppingCart', () => {
  let cart: ShoppingCart;
  let product1: Product;
  let product2: Product;

  beforeEach(() => {
    cart = new ShoppingCart();
    product1 = new Product('p1', 'หนังสือ TypeScript', 350, 10);
    product2 = new Product('p2', 'เมาส์ไร้สาย', 850, 5);
  });

  describe('การเพิ่มสินค้า', () => {
    test('ตะกร้าใหม่ควรว่าง', () => {
      expect(cart.isEmpty()).toBe(true);
      expect(cart.getItemCount()).toBe(0);
    });

    test('เพิ่มสินค้าได้', () => {
      cart.addItem(product1, 2);
      expect(cart.isEmpty()).toBe(false);
      expect(cart.getItemCount()).toBe(1);
    });

    test('เพิ่มสินค้าชนิดเดียวกันสองครั้งรวมจำนวน', () => {
      cart.addItem(product1, 2);
      cart.addItem(product1, 3);
      
      const items = cart.getItems();
      expect(items).toHaveLength(1);
      expect(items[0].quantity).toBe(5);
    });

    test('ไม่สามารถเพิ่มจำนวนเกิน stock', () => {
      expect(() => cart.addItem(product1, 15)).toThrow('จำนวนสินค้าเกิน stock');
    });
  });

  describe('การลบสินค้า', () => {
    test('ลบสินค้าออกจากตะกร้า', () => {
      cart.addItem(product1, 2);
      cart.removeItem(product1.id);
      
      expect(cart.isEmpty()).toBe(true);
    });

    test('ลดจำนวนสินค้า', () => {
      cart.addItem(product1, 5);
      cart.decreaseQuantity(product1.id, 2);
      
      const items = cart.getItems();
      expect(items[0].quantity).toBe(3);
    });

    test('ลดจำนวนเป็น 0 ควรลบสินค้าออก', () => {
      cart.addItem(product1, 2);
      cart.decreaseQuantity(product1.id, 2);
      
      expect(cart.isEmpty()).toBe(true);
    });
  });

  describe('การคำนวณราคา', () => {
    test('คำนวณราคารวมได้ถูกต้อง', () => {
      cart.addItem(product1, 2); // 350 * 2 = 700
      cart.addItem(product2, 1); // 850 * 1 = 850
      
      expect(cart.getTotal()).toBe(1550);
    });

    test('ตะกร้าว่างมีราคา 0', () => {
      expect(cart.getTotal()).toBe(0);
    });

    test('คำนวณส่วนลดได้', () => {
      cart.addItem(product1, 2); // 700
      cart.applyDiscount(10); // 10%
      
      expect(cart.getTotalWithDiscount()).toBe(630);
    });
  });

  describe('การล้างตะกร้า', () => {
    test('ล้างตะกร้าได้', () => {
      cart.addItem(product1, 2);
      cart.addItem(product2, 1);
      cart.clear();
      
      expect(cart.isEmpty()).toBe(true);
      expect(cart.getTotal()).toBe(0);
    });
  });
});
```

**ขั้นตอนที่ 2: เขียน implementation**

```typescript
// product.ts
export class Product {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly price: number,
    public readonly stock: number
  ) {
    if (price < 0) throw new Error('ราคาต้องไม่ติดลบ');
    if (stock < 0) throw new Error('จำนวน stock ต้องไม่ติดลบ');
  }
}

// shopping-cart.ts
interface CartItem {
  product: Product;
  quantity: number;
}

export class ShoppingCart {
  private items: Map<string, CartItem> = new Map();
  private discountPercentage: number = 0;

  addItem(product: Product, quantity: number): void {
    const existingItem = this.items.get(product.id);
    const currentQuantity = existingItem?.quantity ?? 0;
    const newQuantity = currentQuantity + quantity;

    if (newQuantity > product.stock) {
      throw new Error('จำนวนสินค้าเกิน stock');
    }

    this.items.set(product.id, { product, quantity: newQuantity });
  }

  removeItem(productId: string): void {
    this.items.delete(productId);
  }

  decreaseQuantity(productId: string, amount: number): void {
    const item = this.items.get(productId);
    if (!item) return;

    const newQuantity = item.quantity - amount;

    if (newQuantity <= 0) {
      this.items.delete(productId);
    } else {
      this.items.set(productId, { ...item, quantity: newQuantity });
    }
  }

  isEmpty(): boolean {
    return this.items.size === 0;
  }

  getItemCount(): number {
    return this.items.size;
  }

  getItems(): CartItem[] {
    return Array.from(this.items.values());
  }

  getTotal(): number {
    return Array.from(this.items.values()).reduce(
      (total, item) => total + item.product.price * item.quantity,
      0
    );
  }

  applyDiscount(percentage: number): void {
    if (percentage < 0 || percentage > 100) {
      throw new Error('ส่วนลดต้องอยู่ระหว่าง 0-100%');
    }
    this.discountPercentage = percentage;
  }

  getTotalWithDiscount(): number {
    const total = this.getTotal();
    return total * (1 - this.discountPercentage / 100);
  }

  clear(): void {
    this.items.clear();
    this.discountPercentage = 0;
  }
}
```

---

## 3. TDD สำหรับ Functions

### 3.1 String Processing ด้วย TDD

```typescript
// string-utils.test.ts
import { 
  capitalize, 
  truncate, 
  slugify,
  countWords,
  extractEmails
} from './string-utils';

describe('String Utils', () => {
  describe('capitalize', () => {
    test('ทำให้ตัวอักษรแรกเป็นตัวพิมพ์ใหญ่', () => {
      expect(capitalize('hello')).toBe('Hello');
    });

    test('string ว่างคืนค่าว่าง', () => {
      expect(capitalize('')).toBe('');
    });

    test('ตัวอักษรแรกที่เป็นตัวพิมพ์ใหญ่อยู่แล้วไม่เปลี่ยน', () => {
      expect(capitalize('Hello')).toBe('Hello');
    });

    test('ตัวอักษรอื่นยังเป็นตัวพิมพ์เล็ก', () => {
      expect(capitalize('hELLO')).toBe('Hello');
    });
  });

  describe('truncate', () => {
    test('ตัดข้อความที่ยาวเกิน', () => {
      expect(truncate('Hello World', 5)).toBe('Hello...');
    });

    test('ข้อความที่สั้นกว่า limit ไม่เปลี่ยน', () => {
      expect(truncate('Hi', 5)).toBe('Hi');
    });

    test('ใช้ suffix ที่กำหนดเอง', () => {
      expect(truncate('Hello World', 5, ' [ดูเพิ่ม]')).toBe('Hello [ดูเพิ่ม]');
    });
  });

  describe('slugify', () => {
    test('แปลง string เป็น slug', () => {
      expect(slugify('Hello World')).toBe('hello-world');
    });

    test('ลบ special characters', () => {
      expect(slugify('Hello! World?')).toBe('hello-world');
    });

    test('แปลงหลาย spaces เป็น dash เดียว', () => {
      expect(slugify('Hello   World')).toBe('hello-world');
    });
  });

  describe('countWords', () => {
    test('นับจำนวนคำ', () => {
      expect(countWords('hello world foo')).toBe(3);
    });

    test('string ว่างมี 0 คำ', () => {
      expect(countWords('')).toBe(0);
    });

    test('whitespace เกินไม่นับเป็นคำ', () => {
      expect(countWords('  hello   world  ')).toBe(2);
    });
  });

  describe('extractEmails', () => {
    test('แยก email จาก string', () => {
      const text = 'ติดต่อได้ที่ user@example.com หรือ admin@site.org';
      expect(extractEmails(text)).toEqual(['user@example.com', 'admin@site.org']);
    });

    test('คืนค่า array ว่างถ้าไม่มี email', () => {
      expect(extractEmails('ไม่มี email')).toEqual([]);
    });
  });
});
```

```typescript
// string-utils.ts
export function capitalize(str: string): string {
  if (!str) return str;
  return str.charAt(0).toUpperCase() + str.slice(1).toLowerCase();
}

export function truncate(
  str: string,
  maxLength: number,
  suffix: string = '...'
): string {
  if (str.length <= maxLength) return str;
  return str.slice(0, maxLength) + suffix;
}

export function slugify(str: string): string {
  return str
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-')
    .trim();
}

export function countWords(str: string): number {
  const trimmed = str.trim();
  if (!trimmed) return 0;
  return trimmed.split(/\s+/).length;
}

export function extractEmails(text: string): string[] {
  const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;
  return text.match(emailRegex) ?? [];
}
```

---

## 4. TDD สำหรับ React Components

### 4.1 Component พื้นฐาน

```typescript
// LoginForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  const mockOnSubmit = jest.fn();

  beforeEach(() => {
    mockOnSubmit.mockClear();
  });

  test('แสดง form fields', () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    expect(screen.getByLabelText('อีเมล')).toBeInTheDocument();
    expect(screen.getByLabelText('รหัสผ่าน')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'เข้าสู่ระบบ' })).toBeInTheDocument();
  });

  test('แสดง error เมื่อ email ว่าง', async () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    fireEvent.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    expect(await screen.findByText('กรุณากรอกอีเมล')).toBeInTheDocument();
  });

  test('แสดง error เมื่อ email ไม่ถูกต้อง', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText('อีเมล'), 'not-an-email');
    await user.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    expect(await screen.findByText('รูปแบบอีเมลไม่ถูกต้อง')).toBeInTheDocument();
  });

  test('แสดง error เมื่อ password ว่าง', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText('อีเมล'), 'valid@email.com');
    await user.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    expect(await screen.findByText('กรุณากรอกรหัสผ่าน')).toBeInTheDocument();
  });

  test('เรียก onSubmit เมื่อ form ถูกต้อง', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByLabelText('อีเมล'), 'test@example.com');
    await user.type(screen.getByLabelText('รหัสผ่าน'), 'password123');
    await user.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123'
      });
    });
  });

  test('แสดง loading state เมื่อ submit', async () => {
    const slowOnSubmit = jest.fn(() => new Promise(resolve => setTimeout(resolve, 1000)));
    const user = userEvent.setup();
    
    render(<LoginForm onSubmit={slowOnSubmit} />);
    
    await user.type(screen.getByLabelText('อีเมล'), 'test@example.com');
    await user.type(screen.getByLabelText('รหัสผ่าน'), 'password123');
    await user.click(screen.getByRole('button', { name: 'เข้าสู่ระบบ' }));
    
    expect(screen.getByRole('button', { name: 'กำลังเข้าสู่ระบบ...' })).toBeDisabled();
  });
});
```

```typescript
// LoginForm.tsx
import React, { useState } from 'react';

interface LoginFormProps {
  onSubmit: (data: { email: string; password: string }) => Promise<void> | void;
}

interface FormErrors {
  email?: string;
  password?: string;
}

export function LoginForm({ onSubmit }: LoginFormProps) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<FormErrors>({});
  const [isLoading, setIsLoading] = useState(false);

  const validate = (): FormErrors => {
    const newErrors: FormErrors = {};

    if (!email) {
      newErrors.email = 'กรุณากรอกอีเมล';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = 'รูปแบบอีเมลไม่ถูกต้อง';
    }

    if (!password) {
      newErrors.password = 'กรุณากรอกรหัสผ่าน';
    }

    return newErrors;
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    const validationErrors = validate();
    if (Object.keys(validationErrors).length > 0) {
      setErrors(validationErrors);
      return;
    }

    setIsLoading(true);
    try {
      await onSubmit({ email, password });
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="email">อีเมล</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
        />
        {errors.email && <span role="alert">{errors.email}</span>}
      </div>

      <div>
        <label htmlFor="password">รหัสผ่าน</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={e => setPassword(e.target.value)}
        />
        {errors.password && <span role="alert">{errors.password}</span>}
      </div>

      <button type="submit" disabled={isLoading}>
        {isLoading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
      </button>
    </form>
  );
}
```

---

## 5. TDD สำหรับ APIs

### 5.1 การสร้าง REST API ด้วย TDD

```typescript
// user-api.test.ts
import request from 'supertest';
import { createApp } from './app';
import { InMemoryUserRepository } from './repositories/user-repository';

describe('User API', () => {
  let app: Express;
  let userRepo: InMemoryUserRepository;

  beforeEach(() => {
    userRepo = new InMemoryUserRepository();
    app = createApp({ userRepo });
  });

  describe('GET /api/users', () => {
    test('ควรคืน empty array เมื่อไม่มีผู้ใช้', async () => {
      const response = await request(app).get('/api/users');
      
      expect(response.status).toBe(200);
      expect(response.body).toEqual({ users: [], total: 0 });
    });

    test('ควรคืนรายการผู้ใช้', async () => {
      await userRepo.save({ id: '1', name: 'สมชาย', email: 'somchai@test.com', role: 'user' });
      await userRepo.save({ id: '2', name: 'สมหญิง', email: 'somying@test.com', role: 'user' });
      
      const response = await request(app).get('/api/users');
      
      expect(response.status).toBe(200);
      expect(response.body.total).toBe(2);
      expect(response.body.users).toHaveLength(2);
    });
  });

  describe('GET /api/users/:id', () => {
    test('ควรคืนผู้ใช้เมื่อพบ', async () => {
      await userRepo.save({ id: '1', name: 'สมชาย', email: 'somchai@test.com', role: 'user' });
      
      const response = await request(app).get('/api/users/1');
      
      expect(response.status).toBe(200);
      expect(response.body.name).toBe('สมชาย');
    });

    test('ควรคืน 404 เมื่อไม่พบ', async () => {
      const response = await request(app).get('/api/users/999');
      
      expect(response.status).toBe(404);
      expect(response.body.error).toBeDefined();
    });
  });

  describe('POST /api/users', () => {
    test('ควรสร้างผู้ใช้ใหม่', async () => {
      const newUser = {
        name: 'ผู้ใช้ใหม่',
        email: 'newuser@test.com'
      };
      
      const response = await request(app)
        .post('/api/users')
        .send(newUser);
      
      expect(response.status).toBe(201);
      expect(response.body.id).toBeDefined();
      expect(response.body.name).toBe('ผู้ใช้ใหม่');
    });

    test('ควรคืน 400 เมื่อข้อมูลไม่ครบ', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({ name: 'ไม่มี email' });
      
      expect(response.status).toBe(400);
    });

    test('ควรคืน 409 เมื่อ email ซ้ำ', async () => {
      await userRepo.save({ id: '1', name: 'เดิม', email: 'duplicate@test.com', role: 'user' });
      
      const response = await request(app)
        .post('/api/users')
        .send({ name: 'ใหม่', email: 'duplicate@test.com' });
      
      expect(response.status).toBe(409);
    });
  });

  describe('PUT /api/users/:id', () => {
    test('ควรอัปเดตข้อมูลผู้ใช้', async () => {
      await userRepo.save({ id: '1', name: 'เดิม', email: 'user@test.com', role: 'user' });
      
      const response = await request(app)
        .put('/api/users/1')
        .send({ name: 'ใหม่' });
      
      expect(response.status).toBe(200);
      expect(response.body.name).toBe('ใหม่');
    });
  });

  describe('DELETE /api/users/:id', () => {
    test('ควรลบผู้ใช้', async () => {
      await userRepo.save({ id: '1', name: 'จะถูกลบ', email: 'delete@test.com', role: 'user' });
      
      const response = await request(app).delete('/api/users/1');
      
      expect(response.status).toBe(204);
    });
  });
});
```

```typescript
// app.ts
import express, { Application } from 'express';
import { v4 as uuidv4 } from 'uuid';

interface AppDependencies {
  userRepo: UserRepository;
}

export function createApp({ userRepo }: AppDependencies): Application {
  const app = express();
  app.use(express.json());

  // GET /api/users
  app.get('/api/users', async (req, res) => {
    try {
      const users = await userRepo.findAll();
      res.json({ users, total: users.length });
    } catch (error) {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // GET /api/users/:id
  app.get('/api/users/:id', async (req, res) => {
    try {
      const user = await userRepo.findById(req.params.id);
      if (!user) {
        return res.status(404).json({ error: 'ไม่พบผู้ใช้' });
      }
      res.json(user);
    } catch (error) {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // POST /api/users
  app.post('/api/users', async (req, res) => {
    try {
      const { name, email } = req.body;
      
      if (!name || !email) {
        return res.status(400).json({ error: 'name และ email จำเป็น' });
      }

      const existing = await userRepo.findByEmail(email);
      if (existing) {
        return res.status(409).json({ error: 'Email นี้ถูกใช้แล้ว' });
      }

      const newUser: User = {
        id: uuidv4(),
        name,
        email,
        role: 'user'
      };

      await userRepo.save(newUser);
      res.status(201).json(newUser);
    } catch (error) {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // PUT /api/users/:id
  app.put('/api/users/:id', async (req, res) => {
    try {
      const user = await userRepo.findById(req.params.id);
      if (!user) {
        return res.status(404).json({ error: 'ไม่พบผู้ใช้' });
      }

      const updated = { ...user, ...req.body, id: user.id };
      await userRepo.save(updated);
      res.json(updated);
    } catch (error) {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // DELETE /api/users/:id
  app.delete('/api/users/:id', async (req, res) => {
    try {
      await userRepo.delete(req.params.id);
      res.status(204).send();
    } catch (error) {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  return app;
}
```

---

## 6. Outside-In TDD

Outside-In TDD เริ่มจากการเขียน test ระดับสูง (acceptance test) แล้วค่อย drill down ลงไปเขียน unit tests

### 6.1 ตัวอย่าง Outside-In TDD

```typescript
// acceptance.test.ts - เริ่มจากระดับบน
describe('Feature: ผู้ใช้สามารถสมัครสมาชิกได้', () => {
  test('สมัครสมาชิกสำเร็จ', async () => {
    // Arrange
    const userRepo = new InMemoryUserRepository();
    const emailService = { sendWelcomeEmail: jest.fn() };
    const registerUseCase = new RegisterUserUseCase(userRepo, emailService);

    // Act
    const result = await registerUseCase.execute({
      name: 'สมชาย ใจดี',
      email: 'somchai@example.com',
      password: 'Password1!'
    });

    // Assert
    expect(result.success).toBe(true);
    expect(result.user?.name).toBe('สมชาย ใจดี');
    expect(emailService.sendWelcomeEmail).toHaveBeenCalledWith(
      'somchai@example.com',
      'สมชาย ใจดี'
    );
  });

  test('สมัครสมาชิกไม่สำเร็จเมื่อ email ซ้ำ', async () => {
    const userRepo = new InMemoryUserRepository();
    await userRepo.save({
      id: '1',
      name: 'เดิม',
      email: 'existing@example.com',
      role: 'user'
    });

    const emailService = { sendWelcomeEmail: jest.fn() };
    const registerUseCase = new RegisterUserUseCase(userRepo, emailService);

    const result = await registerUseCase.execute({
      name: 'ใหม่',
      email: 'existing@example.com',
      password: 'Password1!'
    });

    expect(result.success).toBe(false);
    expect(result.error).toBe('Email นี้ถูกใช้แล้ว');
    expect(emailService.sendWelcomeEmail).not.toHaveBeenCalled();
  });
});
```

```typescript
// register-user-use-case.ts
interface RegisterUserInput {
  name: string;
  email: string;
  password: string;
}

interface RegisterUserResult {
  success: boolean;
  user?: User;
  error?: string;
}

interface EmailService {
  sendWelcomeEmail(email: string, name: string): Promise<void>;
}

class RegisterUserUseCase {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService
  ) {}

  async execute(input: RegisterUserInput): Promise<RegisterUserResult> {
    // Validation
    const validationError = this.validate(input);
    if (validationError) {
      return { success: false, error: validationError };
    }

    // Check duplicate email
    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) {
      return { success: false, error: 'Email นี้ถูกใช้แล้ว' };
    }

    // Create user
    const user: User = {
      id: uuidv4(),
      name: input.name,
      email: input.email,
      role: 'user'
    };

    await this.userRepo.save(user);
    await this.emailService.sendWelcomeEmail(user.email, user.name);

    return { success: true, user };
  }

  private validate(input: RegisterUserInput): string | null {
    if (!input.name || input.name.length < 2) {
      return 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
    }

    if (!input.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input.email)) {
      return 'อีเมลไม่ถูกต้อง';
    }

    if (!input.password || input.password.length < 8) {
      return 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
    }

    return null;
  }
}
```

---

## 7. Complete TDD Workflow ตัวอย่าง

### 7.1 สร้าง Task Management System ด้วย TDD

**Iteration 1: สร้าง task**

```typescript
// task.test.ts - Iteration 1
describe('Task', () => {
  test('สร้าง task ใหม่ได้', () => {
    const task = new Task('ทำการบ้าน TypeScript');
    
    expect(task.title).toBe('ทำการบ้าน TypeScript');
    expect(task.status).toBe('todo');
    expect(task.id).toBeDefined();
    expect(task.createdAt).toBeDefined();
  });
});
```

```typescript
// task.ts - Iteration 1
import { v4 as uuidv4 } from 'uuid';

type TaskStatus = 'todo' | 'in-progress' | 'done';

class Task {
  readonly id: string;
  readonly createdAt: Date;
  title: string;
  status: TaskStatus;

  constructor(title: string) {
    this.id = uuidv4();
    this.title = title;
    this.status = 'todo';
    this.createdAt = new Date();
  }
}
```

**Iteration 2: เปลี่ยน status**

```typescript
// task.test.ts - Iteration 2
describe('Task status transitions', () => {
  test('เริ่ม task ได้', () => {
    const task = new Task('ทำงาน');
    task.start();
    expect(task.status).toBe('in-progress');
  });

  test('เสร็จ task ได้', () => {
    const task = new Task('ทำงาน');
    task.start();
    task.complete();
    expect(task.status).toBe('done');
  });

  test('ไม่สามารถ complete task ที่ยังไม่ได้ start', () => {
    const task = new Task('ทำงาน');
    expect(() => task.complete()).toThrow('ต้อง start task ก่อน');
  });

  test('ไม่สามารถ start task ที่ done แล้ว', () => {
    const task = new Task('ทำงาน');
    task.start();
    task.complete();
    expect(() => task.start()).toThrow('task เสร็จแล้ว');
  });
});
```

```typescript
// task.ts - Iteration 2
class Task {
  readonly id: string;
  readonly createdAt: Date;
  title: string;
  status: TaskStatus;
  completedAt?: Date;

  constructor(title: string) {
    this.id = uuidv4();
    this.title = title;
    this.status = 'todo';
    this.createdAt = new Date();
  }

  start(): void {
    if (this.status === 'done') {
      throw new Error('task เสร็จแล้ว');
    }
    if (this.status === 'in-progress') {
      throw new Error('task กำลังดำเนินการอยู่');
    }
    this.status = 'in-progress';
  }

  complete(): void {
    if (this.status !== 'in-progress') {
      throw new Error('ต้อง start task ก่อน');
    }
    this.status = 'done';
    this.completedAt = new Date();
  }
}
```

**Iteration 3: TaskList**

```typescript
// task-list.test.ts
describe('TaskList', () => {
  let taskList: TaskList;

  beforeEach(() => {
    taskList = new TaskList();
  });

  test('เพิ่ม task ได้', () => {
    taskList.addTask('งานที่ 1');
    expect(taskList.getAll()).toHaveLength(1);
  });

  test('กรอง task ตาม status', () => {
    const task1 = taskList.addTask('งาน 1');
    const task2 = taskList.addTask('งาน 2');
    taskList.addTask('งาน 3');

    task1.start();
    task2.start();
    task2.complete();

    expect(taskList.getByStatus('todo')).toHaveLength(1);
    expect(taskList.getByStatus('in-progress')).toHaveLength(1);
    expect(taskList.getByStatus('done')).toHaveLength(1);
  });

  test('ลบ task ได้', () => {
    const task = taskList.addTask('จะถูกลบ');
    taskList.removeTask(task.id);
    expect(taskList.getAll()).toHaveLength(0);
  });

  test('นับจำนวน task ตาม status', () => {
    const task = taskList.addTask('งาน');
    task.start();

    const counts = taskList.getCounts();
    expect(counts.todo).toBe(0);
    expect(counts.inProgress).toBe(1);
    expect(counts.done).toBe(0);
  });
});
```

```typescript
// task-list.ts
interface TaskCounts {
  todo: number;
  inProgress: number;
  done: number;
}

class TaskList {
  private tasks: Map<string, Task> = new Map();

  addTask(title: string): Task {
    const task = new Task(title);
    this.tasks.set(task.id, task);
    return task;
  }

  removeTask(id: string): void {
    this.tasks.delete(id);
  }

  getAll(): Task[] {
    return Array.from(this.tasks.values());
  }

  getByStatus(status: TaskStatus): Task[] {
    return this.getAll().filter(task => task.status === status);
  }

  getCounts(): TaskCounts {
    const all = this.getAll();
    return {
      todo: all.filter(t => t.status === 'todo').length,
      inProgress: all.filter(t => t.status === 'in-progress').length,
      done: all.filter(t => t.status === 'done').length
    };
  }
}
```

---

## 8. TDD สำหรับ Data Structures

### 8.1 Stack ด้วย TDD

```typescript
// stack.test.ts
describe('Stack<T>', () => {
  test('stack ใหม่ว่าง', () => {
    const stack = new Stack<number>();
    expect(stack.isEmpty()).toBe(true);
    expect(stack.size()).toBe(0);
  });

  test('push เพิ่ม element', () => {
    const stack = new Stack<number>();
    stack.push(1);
    stack.push(2);
    expect(stack.size()).toBe(2);
  });

  test('pop คืนค่าและลบ element บนสุด', () => {
    const stack = new Stack<number>();
    stack.push(1);
    stack.push(2);
    
    expect(stack.pop()).toBe(2);
    expect(stack.size()).toBe(1);
  });

  test('peek คืนค่า element บนสุดโดยไม่ลบ', () => {
    const stack = new Stack<string>();
    stack.push('a');
    stack.push('b');
    
    expect(stack.peek()).toBe('b');
    expect(stack.size()).toBe(2);
  });

  test('pop บน empty stack โยน error', () => {
    const stack = new Stack<number>();
    expect(() => stack.pop()).toThrow('Stack is empty');
  });

  test('ทำงานเป็น LIFO', () => {
    const stack = new Stack<number>();
    stack.push(1);
    stack.push(2);
    stack.push(3);
    
    expect(stack.pop()).toBe(3);
    expect(stack.pop()).toBe(2);
    expect(stack.pop()).toBe(1);
  });
});
```

```typescript
// stack.ts
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T {
    if (this.isEmpty()) {
      throw new Error('Stack is empty');
    }
    return this.items.pop()!;
  }

  peek(): T {
    if (this.isEmpty()) {
      throw new Error('Stack is empty');
    }
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }

  clear(): void {
    this.items = [];
  }

  toArray(): T[] {
    return [...this.items].reverse();
  }
}
```

---

## 9. Common TDD Pitfalls

### 9.1 ปัญหาที่พบบ่อยใน TDD

```typescript
// ❌ Anti-pattern: Test ที่ขึ้นกับ implementation details
describe('BAD: Testing Implementation', () => {
  test('ใช้ private method โดยตรง - ไม่ดี', () => {
    const service = new UserService(new InMemoryUserRepository());
    // @ts-ignore accessing private method
    const result = service._validateEmail('test@example.com');
    expect(result).toBe(true);
  });
});

// ✅ Good: Test behavior ไม่ใช่ implementation
describe('GOOD: Testing Behavior', () => {
  test('ควร reject user ที่มี email ไม่ถูกต้อง', async () => {
    const service = new UserService(new InMemoryUserRepository());
    await expect(
      service.createUser('ชื่อ', 'invalid-email')
    ).rejects.toThrow();
  });
});
```

```typescript
// ❌ Anti-pattern: Test ที่ซับซ้อนเกินไป
test('BAD: ทำหลายอย่างในคำเดียว', async () => {
  const repo = new InMemoryUserRepository();
  const service = new UserService(repo);
  
  // ทดสอบหลายอย่างในคำเดียว
  const user = await service.createUser('สมชาย', 'somchai@test.com');
  expect(user.id).toBeDefined();
  await service.updateUser(user.id, { name: 'สมหญิง' });
  const updated = await service.getUser(user.id);
  expect(updated.name).toBe('สมหญิง');
  await service.deleteUser(user.id);
  await expect(service.getUser(user.id)).rejects.toThrow();
});

// ✅ Good: แยก test
describe('GOOD: แยก concerns', () => {
  test('สร้างผู้ใช้ได้', async () => {
    const user = await service.createUser('สมชาย', 'somchai@test.com');
    expect(user.id).toBeDefined();
  });

  test('อัปเดตผู้ใช้ได้', async () => {
    const user = await createTestUser();
    await service.updateUser(user.id, { name: 'สมหญิง' });
    const updated = await service.getUser(user.id);
    expect(updated.name).toBe('สมหญิง');
  });
});
```

---

## 10. TDD Best Practices

```typescript
// 1. Test ควรอ่านเข้าใจได้ง่าย
describe('OrderService', () => {
  describe('เมื่อผู้ใช้สั่งซื้อสินค้า', () => {
    describe('และสินค้ามีสต็อก', () => {
      test('ควรสร้างคำสั่งซื้อสำเร็จ', async () => {
        // Given (Arrange)
        const product = createProductWithStock(10);
        const user = createTestUser();
        
        // When (Act)
        const order = await orderService.createOrder(user.id, [
          { productId: product.id, quantity: 2 }
        ]);
        
        // Then (Assert)
        expect(order.status).toBe('confirmed');
        expect(order.items).toHaveLength(1);
      });
    });

    describe('และสินค้าไม่มีสต็อก', () => {
      test('ควรปฏิเสธคำสั่งซื้อ', async () => {
        // Given
        const product = createProductWithStock(0);
        const user = createTestUser();
        
        // When & Then
        await expect(
          orderService.createOrder(user.id, [
            { productId: product.id, quantity: 1 }
          ])
        ).rejects.toThrow('สินค้าหมดสต็อก');
      });
    });
  });
});
```

---

## 11. Integration ของ TDD กับ CI/CD

```yaml
# .github/workflows/tdd-ci.yml
name: TDD CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm test -- --coverage
      
      - name: Check coverage threshold
        run: |
          npm test -- --coverage --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
      
      - name: Upload coverage report
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: coverage/
```

---

## สรุปบทที่ 62

ในบทนี้เราได้เรียนรู้:

1. **วงจร Red-Green-Refactor** - หลักการพื้นฐานของ TDD
2. **TDD สำหรับ Classes** - การสร้าง Shopping Cart
3. **TDD สำหรับ Functions** - String utilities
4. **TDD สำหรับ React Components** - LoginForm
5. **TDD สำหรับ APIs** - REST API
6. **Outside-In TDD** - เริ่มจาก acceptance tests
7. **Complete TDD Workflow** - Task Management System
8. **Data Structures ด้วย TDD** - Stack
9. **TDD Pitfalls** - ข้อผิดพลาดที่ควรหลีกเลี่ยง
10. **TDD Best Practices** - แนวทางที่ดี

---

## แบบฝึกหัด

1. สร้าง LinkedList ด้วย TDD
2. ใช้ TDD สร้าง authentication service
3. เขียน tests สำหรับ React form ที่ซับซ้อน
4. สร้าง REST API สำหรับ blog ด้วย TDD
5. Implement outside-in TDD สำหรับ payment flow

---

*ต่อไป: ส่วนที่ 63 - E2E Testing with Playwright*
