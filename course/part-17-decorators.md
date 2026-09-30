# ตอนที่ 17: Decorators ใน TypeScript

## บทนำ

Decorators เป็น feature ที่ทรงพลังใน TypeScript ที่ช่วยให้เราเพิ่มพฤติกรรมหรือ metadata ให้กับ class, method, property, หรือ parameter โดยไม่ต้องแก้ไข code เดิม ใช้หลักการ Aspect-Oriented Programming (AOP)

---

## 1. การเปิดใช้งาน Decorators

### 1.1 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "lib": ["ES2017", "DOM"]
  }
}
```

### 1.2 Decorators คืออะไร

Decorator คือ function พิเศษที่ใช้กับ `@` syntax:

```typescript
// Decorator function พื้นฐาน
function log(target: any, key: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  
  descriptor.value = function(...args: any[]) {
    console.log(`เรียกใช้ ${key} กับ args:`, args);
    const result = originalMethod.apply(this, args);
    console.log(`${key} คืนค่า:`, result);
    return result;
  };
  
  return descriptor;
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(5, 3);
// เรียกใช้ add กับ args: [5, 3]
// add คืนค่า: 8
```

---

## 2. Class Decorators

### 2.1 Class Decorator พื้นฐาน

```typescript
// Class decorator รับ constructor เป็น argument
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class BankAccount {
  private balance: number;
  
  constructor(initialBalance: number) {
    this.balance = initialBalance;
  }
  
  deposit(amount: number): void {
    this.balance += amount;
  }
  
  withdraw(amount: number): boolean {
    if (amount > this.balance) return false;
    this.balance -= amount;
    return true;
  }
  
  getBalance(): number {
    return this.balance;
  }
}
```

### 2.2 Class Decorator ที่แก้ไข Constructor

```typescript
function withTimestamp<T extends { new(...args: any[]): {} }>(constructor: T) {
  return class extends constructor {
    createdAt = new Date();
    updatedAt = new Date();
  };
}

@withTimestamp
class User {
  constructor(
    public name: string,
    public email: string
  ) {}
}

const user = new User("สมชาย", "somchai@example.com");
console.log((user as any).createdAt); // เวลาปัจจุบัน
```

### 2.3 Decorator Factory สำหรับ Class

```typescript
function Entity(tableName: string) {
  return function<T extends { new(...args: any[]): {} }>(constructor: T) {
    return class extends constructor {
      static tableName = tableName;
      
      static getTableName(): string {
        return tableName;
      }
    };
  };
}

@Entity('users')
class User {
  constructor(
    public id: number,
    public name: string
  ) {}
}

@Entity('products')
class Product {
  constructor(
    public id: number,
    public name: string,
    public price: number
  ) {}
}

console.log((User as any).tableName);    // 'users'
console.log((Product as any).tableName); // 'products'
```

### 2.4 Class Decorator สำหรับ Singleton Pattern

```typescript
function Singleton<T extends { new(...args: any[]): {} }>(constructor: T) {
  let instance: InstanceType<T>;
  
  return class extends constructor {
    constructor(...args: any[]) {
      if (instance) {
        return instance;
      }
      super(...args);
      instance = this as InstanceType<T>;
    }
  } as any;
}

@Singleton
class DatabaseConnection {
  private connectionId: string;
  
  constructor(public connectionString: string) {
    this.connectionId = Math.random().toString(36).substr(2, 9);
    console.log(`สร้าง connection: ${this.connectionId}`);
  }
  
  query(sql: string): Promise<any[]> {
    return Promise.resolve([]);
  }
}

const db1 = new DatabaseConnection('postgresql://localhost/mydb');
const db2 = new DatabaseConnection('postgresql://localhost/mydb');

console.log(db1 === db2); // true - เป็น instance เดียวกัน
```

### 2.5 Component Decorator (React-style)

```typescript
interface ComponentOptions {
  selector: string;
  template: string;
  styles?: string[];
}

const componentRegistry = new Map<string, any>();

function Component(options: ComponentOptions) {
  return function(constructor: Function) {
    (constructor as any).__componentOptions = options;
    componentRegistry.set(options.selector, constructor);
    
    console.log(`ลงทะเบียน component: ${options.selector}`);
  };
}

@Component({
  selector: 'app-header',
  template: '<header>{{title}}</header>',
  styles: ['header { background: #333; }']
})
class HeaderComponent {
  title = 'My App';
  
  render(): string {
    const options = (HeaderComponent as any).__componentOptions;
    return options.template.replace('{{title}}', this.title);
  }
}

@Component({
  selector: 'app-footer',
  template: '<footer>{{copyright}}</footer>'
})
class FooterComponent {
  copyright = '© 2024 My App';
}
```

---

## 3. Method Decorators

### 3.1 Method Decorator พื้นฐาน

```typescript
function readonly(target: any, propertyKey: string, descriptor: PropertyDescriptor): PropertyDescriptor {
  descriptor.writable = false;
  return descriptor;
}

function enumerable(value: boolean) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    descriptor.enumerable = value;
    return descriptor;
  };
}

class Person {
  constructor(public firstName: string, public lastName: string) {}
  
  @readonly
  greet(): string {
    return `สวัสดี ฉันชื่อ ${this.firstName} ${this.lastName}`;
  }
  
  @enumerable(false)
  privateMethod(): void {
    console.log('method นี้ไม่ปรากฏใน for...in');
  }
}
```

### 3.2 @Log Decorator

```typescript
interface LogOptions {
  level?: 'debug' | 'info' | 'warn' | 'error';
  prefix?: string;
  logArgs?: boolean;
  logResult?: boolean;
  logTime?: boolean;
}

function Log(options: LogOptions = {}) {
  const {
    level = 'info',
    prefix = '',
    logArgs = true,
    logResult = true,
    logTime = false
  } = options;
  
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      const className = target.constructor.name;
      const fullName = `${prefix}${className}.${methodName}`;
      
      if (logArgs) {
        console[level](`[LOG] ${fullName} - เรียกด้วย:`, args);
      } else {
        console[level](`[LOG] ${fullName} - เรียกใช้`);
      }
      
      const startTime = Date.now();
      
      try {
        const result = await originalMethod.apply(this, args);
        
        if (logTime) {
          console[level](`[LOG] ${fullName} - ใช้เวลา: ${Date.now() - startTime}ms`);
        }
        
        if (logResult) {
          console[level](`[LOG] ${fullName} - คืนค่า:`, result);
        }
        
        return result;
      } catch (error) {
        console.error(`[LOG ERROR] ${fullName} - เกิดข้อผิดพลาด:`, error);
        throw error;
      }
    };
    
    return descriptor;
  };
}

class UserService {
  @Log({ logTime: true, level: 'info' })
  async getUser(id: number): Promise<{ id: number; name: string }> {
    // จำลองการดึงข้อมูล
    await new Promise(resolve => setTimeout(resolve, 100));
    return { id, name: 'สมชาย' };
  }
  
  @Log({ logArgs: false, logResult: false })
  async deleteUser(id: number): Promise<void> {
    console.log(`ลบผู้ใช้ ${id}`);
  }
}
```

### 3.3 @Cache Decorator

```typescript
interface CacheOptions {
  ttl?: number; // milliseconds
  key?: (...args: any[]) => string;
}

function Cache(options: CacheOptions = {}) {
  const { ttl = 60000, key } = options;
  const cache = new Map<string, { value: any; expiry: number }>();
  
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      const cacheKey = key ? key(...args) : JSON.stringify(args);
      const cached = cache.get(cacheKey);
      
      if (cached && Date.now() < cached.expiry) {
        console.log(`[CACHE HIT] ${methodName}(${cacheKey})`);
        return cached.value;
      }
      
      console.log(`[CACHE MISS] ${methodName}(${cacheKey})`);
      const result = await originalMethod.apply(this, args);
      
      cache.set(cacheKey, {
        value: result,
        expiry: Date.now() + ttl
      });
      
      return result;
    };
    
    return descriptor;
  };
}

class ProductService {
  @Cache({ ttl: 5 * 60 * 1000 }) // cache 5 นาที
  async getProduct(id: string): Promise<any> {
    console.log(`ดึงข้อมูล product ${id} จาก database`);
    await new Promise(resolve => setTimeout(resolve, 500));
    return { id, name: `Product ${id}`, price: 100 };
  }
  
  @Cache({
    ttl: 30000,
    key: (category, page) => `${category}-${page}`
  })
  async getProductsByCategory(category: string, page: number = 1): Promise<any[]> {
    return [];
  }
}
```

### 3.4 @Retry Decorator

```typescript
function Retry(maxAttempts: number = 3, delay: number = 1000) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      let lastError: Error;
      
      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error as Error;
          console.warn(`[RETRY] ${methodName} ครั้งที่ ${attempt}/${maxAttempts} ล้มเหลว:`, error);
          
          if (attempt < maxAttempts) {
            await new Promise(resolve => setTimeout(resolve, delay * attempt));
          }
        }
      }
      
      throw lastError!;
    };
    
    return descriptor;
  };
}

class ApiService {
  @Retry(3, 1000)
  async fetchData(url: string): Promise<any> {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return response.json();
  }
}
```

### 3.5 @Throttle และ @Debounce Decorator

```typescript
function Throttle(delay: number) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    let lastCall = 0;
    const originalMethod = descriptor.value;
    
    descriptor.value = function(...args: any[]) {
      const now = Date.now();
      if (now - lastCall >= delay) {
        lastCall = now;
        return originalMethod.apply(this, args);
      }
      console.log(`[THROTTLE] ${methodName} ถูก throttle`);
    };
    
    return descriptor;
  };
}

function Debounce(delay: number) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    let timeoutId: ReturnType<typeof setTimeout>;
    const originalMethod = descriptor.value;
    
    descriptor.value = function(...args: any[]) {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => {
        originalMethod.apply(this, args);
      }, delay);
    };
    
    return descriptor;
  };
}

class SearchService {
  @Debounce(300)
  search(query: string): void {
    console.log(`ค้นหา: ${query}`);
    // เรียก API
  }
  
  @Throttle(1000)
  trackEvent(event: string): void {
    console.log(`Track event: ${event}`);
  }
}
```

---

## 4. Property Decorators

### 4.1 Property Decorator พื้นฐาน

```typescript
function required(target: any, propertyKey: string): void {
  let value: any;
  
  const getter = function() {
    return value;
  };
  
  const setter = function(newValue: any) {
    if (newValue === null || newValue === undefined || newValue === '') {
      throw new Error(`${propertyKey} is required`);
    }
    value = newValue;
  };
  
  Object.defineProperty(target, propertyKey, {
    get: getter,
    set: setter,
    enumerable: true,
    configurable: true
  });
}

class UserProfile {
  @required
  name!: string;
  
  @required
  email!: string;
  
  age?: number;
}

const profile = new UserProfile();
// profile.name = ''; // Error: name is required
profile.name = 'สมชาย';
profile.email = 'somchai@example.com';
```

### 4.2 @Validate Property Decorator

```typescript
type ValidatorFn = (value: any) => string | null;

const validators = new Map<object, Map<string, ValidatorFn[]>>();

function addValidator(target: any, propertyKey: string, validator: ValidatorFn) {
  if (!validators.has(target)) {
    validators.set(target, new Map());
  }
  const propValidators = validators.get(target)!;
  if (!propValidators.has(propertyKey)) {
    propValidators.set(propertyKey, []);
  }
  propValidators.get(propertyKey)!.push(validator);
}

function IsEmail(target: any, propertyKey: string): void {
  addValidator(target, propertyKey, (value: string) => {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      return `${propertyKey} ต้องเป็น email ที่ถูกต้อง`;
    }
    return null;
  });
}

function MinLength(min: number) {
  return function(target: any, propertyKey: string): void {
    addValidator(target, propertyKey, (value: string) => {
      if (!value || value.length < min) {
        return `${propertyKey} ต้องมีความยาวอย่างน้อย ${min} ตัวอักษร`;
      }
      return null;
    });
  };
}

function MaxLength(max: number) {
  return function(target: any, propertyKey: string): void {
    addValidator(target, propertyKey, (value: string) => {
      if (value && value.length > max) {
        return `${propertyKey} ต้องมีความยาวไม่เกิน ${max} ตัวอักษร`;
      }
      return null;
    });
  };
}

function Min(min: number) {
  return function(target: any, propertyKey: string): void {
    addValidator(target, propertyKey, (value: number) => {
      if (value < min) {
        return `${propertyKey} ต้องมีค่าอย่างน้อย ${min}`;
      }
      return null;
    });
  };
}

function validate(instance: any): string[] {
  const errors: string[] = [];
  const proto = Object.getPrototypeOf(instance);
  const propValidators = validators.get(proto);
  
  if (propValidators) {
    for (const [key, validatorFns] of propValidators.entries()) {
      const value = instance[key];
      for (const validator of validatorFns) {
        const error = validator(value);
        if (error) errors.push(error);
      }
    }
  }
  
  return errors;
}

class RegisterForm {
  @MinLength(3)
  @MaxLength(50)
  name!: string;
  
  @IsEmail
  email!: string;
  
  @MinLength(8)
  password!: string;
  
  @Min(18)
  age!: number;
}

const form = new RegisterForm();
form.name = 'สม';
form.email = 'invalid-email';
form.password = '12345';
form.age = 16;

const errors = validate(form);
console.log(errors);
// ['name ต้องมีความยาวอย่างน้อย 3 ตัวอักษร', 'email ต้องเป็น email ที่ถูกต้อง', ...]
```

### 4.3 @Observable Property Decorator

```typescript
type Observer<T> = (newValue: T, oldValue: T) => void;

function Observable<T>() {
  return function(target: any, propertyKey: string): void {
    const observersKey = `__observers_${propertyKey}`;
    let value: T;
    
    Object.defineProperty(target, propertyKey, {
      get() {
        return value;
      },
      set(newValue: T) {
        const oldValue = value;
        value = newValue;
        
        const observers: Observer<T>[] = this[observersKey] || [];
        observers.forEach(observer => observer(newValue, oldValue));
      },
      enumerable: true,
      configurable: true
    });
    
    // เพิ่ม method สำหรับ subscribe
    if (!target[`observe_${propertyKey}`]) {
      target[`observe_${propertyKey}`] = function(observer: Observer<T>) {
        if (!this[observersKey]) {
          this[observersKey] = [];
        }
        this[observersKey].push(observer);
        
        // return unsubscribe function
        return () => {
          this[observersKey] = this[observersKey].filter((o: Observer<T>) => o !== observer);
        };
      };
    }
  };
}

class Store {
  @Observable<number>()
  count: number = 0;
  
  @Observable<string>()
  status: string = 'idle';
}

const store = new Store();

// Subscribe to changes
const unsubscribe = (store as any).observe_count((newVal: number, oldVal: number) => {
  console.log(`count เปลี่ยนจาก ${oldVal} เป็น ${newVal}`);
});

store.count = 5;   // count เปลี่ยนจาก 0 เป็น 5
store.count = 10;  // count เปลี่ยนจาก 5 เป็น 10

unsubscribe(); // หยุด observe
store.count = 15;  // ไม่มี log
```

---

## 5. Parameter Decorators

### 5.1 Parameter Decorator พื้นฐาน

```typescript
const parameterMetadata = new Map<string, number[]>();

function Inject(serviceId: string) {
  return function(target: any, methodName: string | undefined, parameterIndex: number) {
    const key = `${target.constructor?.name || target.name}:${methodName || 'constructor'}:${parameterIndex}`;
    console.log(`@Inject('${serviceId}') บน ${key}`);
  };
}

function Body(target: any, methodName: string, parameterIndex: number): void {
  console.log(`@Body บน parameter ${parameterIndex} ของ ${methodName}`);
}

function Param(name: string) {
  return function(target: any, methodName: string, parameterIndex: number): void {
    console.log(`@Param('${name}') บน parameter ${parameterIndex} ของ ${methodName}`);
  };
}

function Query(name: string) {
  return function(target: any, methodName: string, parameterIndex: number): void {
    console.log(`@Query('${name}') บน parameter ${parameterIndex} ของ ${methodName}`);
  };
}

class UserController {
  constructor(@Inject('UserService') private userService: any) {}
  
  async getUser(@Param('id') id: string): Promise<any> {
    return { id };
  }
  
  async createUser(@Body data: any): Promise<any> {
    return data;
  }
  
  async searchUsers(@Query('name') name: string, @Query('page') page: number): Promise<any[]> {
    return [];
  }
}
```

### 5.2 Parameter Validation Decorator

```typescript
import 'reflect-metadata';

const VALIDATION_METADATA_KEY = 'validation:params';

type ParamValidation = {
  index: number;
  validator: (value: any) => boolean;
  message: string;
};

function ValidateParam(validator: (value: any) => boolean, message: string) {
  return function(target: any, methodName: string, parameterIndex: number): void {
    const validations: ParamValidation[] = 
      Reflect.getMetadata(VALIDATION_METADATA_KEY, target, methodName) || [];
    
    validations.push({ index: parameterIndex, validator, message });
    Reflect.defineMetadata(VALIDATION_METADATA_KEY, validations, target, methodName);
  };
}

function NotEmpty(target: any, methodName: string, parameterIndex: number): void {
  ValidateParam(
    (value) => value !== null && value !== undefined && value !== '',
    `Parameter ${parameterIndex} ต้องไม่ว่างเปล่า`
  )(target, methodName, parameterIndex);
}

function ValidateArgs(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
  const originalMethod = descriptor.value;
  
  descriptor.value = function(...args: any[]) {
    const validations: ParamValidation[] = 
      Reflect.getMetadata(VALIDATION_METADATA_KEY, target, methodName) || [];
    
    for (const validation of validations) {
      if (!validation.validator(args[validation.index])) {
        throw new Error(`Validation error: ${validation.message}`);
      }
    }
    
    return originalMethod.apply(this, args);
  };
  
  return descriptor;
}

class Service {
  @ValidateArgs
  process(@NotEmpty name: string, @ValidateParam((v) => v > 0, 'age ต้องมากกว่า 0') age: number): string {
    return `ประมวลผล: ${name}, ${age}`;
  }
}
```

---

## 6. Decorator Factories

### 6.1 สร้าง Decorator Factory ที่ยืดหยุ่น

```typescript
interface MethodDecoratorOptions {
  before?: (...args: any[]) => void;
  after?: (result: any) => void;
  onError?: (error: Error) => void;
  transform?: (result: any) => any;
}

function intercept(options: MethodDecoratorOptions) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      options.before?.(...args);
      
      try {
        let result = await originalMethod.apply(this, args);
        
        if (options.transform) {
          result = options.transform(result);
        }
        
        options.after?.(result);
        return result;
      } catch (error) {
        options.onError?.(error as Error);
        throw error;
      }
    };
    
    return descriptor;
  };
}

class OrderService {
  @intercept({
    before: (orderId) => console.log(`[ORDER] Processing order: ${orderId}`),
    after: (order) => console.log(`[ORDER] Processed:`, order),
    onError: (error) => console.error(`[ORDER ERROR]:`, error),
    transform: (order) => ({ ...order, processed: true })
  })
  async processOrder(orderId: string) {
    return { orderId, status: 'completed' };
  }
}
```

---

## 7. Decorator Composition

### 7.1 การใช้ Decorators หลายตัวพร้อมกัน

Decorators ถูก apply จากล่างขึ้นบน (bottom-to-top):

```typescript
function first() {
  return function(target: any, key: string, descriptor: PropertyDescriptor) {
    console.log('first(): ถูก evaluate');
    const original = descriptor.value;
    descriptor.value = function(...args: any[]) {
      console.log('first(): ก่อน call');
      const result = original.apply(this, args);
      console.log('first(): หลัง call');
      return result;
    };
    return descriptor;
  };
}

function second() {
  return function(target: any, key: string, descriptor: PropertyDescriptor) {
    console.log('second(): ถูก evaluate');
    const original = descriptor.value;
    descriptor.value = function(...args: any[]) {
      console.log('second(): ก่อน call');
      const result = original.apply(this, args);
      console.log('second(): หลัง call');
      return result;
    };
    return descriptor;
  };
}

class Demo {
  @first()
  @second()
  method() {
    console.log('method: ทำงาน');
  }
}

// Output เมื่อ class ถูกสร้าง:
// first(): ถูก evaluate
// second(): ถูก evaluate

// Output เมื่อ method ถูกเรียก:
// first(): ก่อน call
// second(): ก่อน call
// method: ทำงาน
// second(): หลัง call
// first(): หลัง call
```

### 7.2 Compose Decorators

```typescript
function compose(...decorators: MethodDecorator[]): MethodDecorator {
  return (target, key, descriptor) => {
    return decorators.reduceRight(
      (desc, decorator) => decorator(target, key, desc) || desc,
      descriptor
    );
  };
}

const withLoggingAndCache = compose(
  Log({ level: 'info' }),
  Cache({ ttl: 5000 })
);

class DataService {
  @withLoggingAndCache
  async getData(id: string) {
    return { id, data: 'example' };
  }
}
```

---

## 8. Metadata Reflection

### 8.1 reflect-metadata

```typescript
import 'reflect-metadata';

// กำหนด metadata
function Injectable(serviceId?: string) {
  return function(target: any) {
    Reflect.defineMetadata('injectable', true, target);
    if (serviceId) {
      Reflect.defineMetadata('serviceId', serviceId, target);
    }
  };
}

function Autowired(target: any, propertyKey: string): void {
  const type = Reflect.getMetadata('design:type', target, propertyKey);
  console.log(`${propertyKey} type:`, type?.name);
}

// ใช้งาน
@Injectable('UserService')
class UserService {
  getUser(id: number) {
    return { id, name: 'test' };
  }
}

class UserController {
  @Autowired
  userService!: UserService;
}

// ตรวจสอบ metadata
console.log(Reflect.getMetadata('injectable', UserService)); // true
console.log(Reflect.getMetadata('serviceId', UserService)); // 'UserService'
```

### 8.2 Type Metadata

```typescript
import 'reflect-metadata';

function logTypes(target: any, key: string): void {
  const type = Reflect.getMetadata('design:type', target, key);
  const paramTypes = Reflect.getMetadata('design:paramtypes', target, key);
  const returnType = Reflect.getMetadata('design:returntype', target, key);
  
  console.log(`Property: ${key}`);
  console.log(`Type: ${type?.name}`);
  console.log(`Param Types: ${paramTypes?.map((t: any) => t.name)}`);
  console.log(`Return Type: ${returnType?.name}`);
}

class Example {
  @logTypes
  name!: string;
  
  @logTypes
  process(input: string): number {
    return parseInt(input);
  }
}
```

---

## 9. Real-world Decorators

### 9.1 @Injectable สำหรับ Dependency Injection

```typescript
import 'reflect-metadata';

type Constructor<T = {}> = new (...args: any[]) => T;
const registry = new Map<string, Constructor>();
const singletons = new Map<string, any>();

function Injectable(id?: string) {
  return function<T>(constructor: Constructor<T>) {
    const serviceId = id || constructor.name;
    registry.set(serviceId, constructor);
    return constructor;
  };
}

function inject<T>(serviceId: string): T {
  if (singletons.has(serviceId)) {
    return singletons.get(serviceId);
  }
  
  const constructor = registry.get(serviceId);
  if (!constructor) {
    throw new Error(`Service not found: ${serviceId}`);
  }
  
  const paramTypes = Reflect.getMetadata('design:paramtypes', constructor) || [];
  const dependencies = paramTypes.map((type: Constructor) => inject(type.name));
  
  const instance = new constructor(...dependencies);
  singletons.set(serviceId, instance);
  return instance as T;
}

@Injectable()
class DatabaseService {
  async query(sql: string): Promise<any[]> {
    console.log(`Query: ${sql}`);
    return [];
  }
}

@Injectable()
class UserRepository {
  constructor(private db: DatabaseService) {}
  
  async findAll(): Promise<any[]> {
    return this.db.query('SELECT * FROM users');
  }
  
  async findById(id: number): Promise<any> {
    const results = await this.db.query(`SELECT * FROM users WHERE id = ${id}`);
    return results[0];
  }
}

@Injectable()
class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async getUser(id: number): Promise<any> {
    return this.userRepository.findById(id);
  }
  
  async getAllUsers(): Promise<any[]> {
    return this.userRepository.findAll();
  }
}
```

### 9.2 @Auth สำหรับ Authorization

```typescript
type Role = 'admin' | 'user' | 'moderator' | 'guest';

interface AuthContext {
  userId: number;
  role: Role;
  permissions: string[];
}

// จำลอง auth context
let currentUser: AuthContext | null = null;

function setCurrentUser(user: AuthContext | null) {
  currentUser = user;
}

function Authorize(roles?: Role[], permissions?: string[]) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      if (!currentUser) {
        throw new Error('Unauthorized: กรุณาเข้าสู่ระบบ');
      }
      
      if (roles && roles.length > 0) {
        if (!roles.includes(currentUser.role)) {
          throw new Error(`Forbidden: ต้องการ role ${roles.join(' หรือ ')}`);
        }
      }
      
      if (permissions && permissions.length > 0) {
        const hasPermission = permissions.every(p => currentUser!.permissions.includes(p));
        if (!hasPermission) {
          throw new Error(`Forbidden: ต้องการ permission ${permissions.join(', ')}`);
        }
      }
      
      return originalMethod.apply(this, args);
    };
    
    return descriptor;
  };
}

class AdminController {
  @Authorize(['admin'])
  async deleteUser(userId: number): Promise<void> {
    console.log(`ลบผู้ใช้ ${userId}`);
  }
  
  @Authorize(['admin', 'moderator'], ['users.read'])
  async getUsers(): Promise<any[]> {
    return [];
  }
  
  @Authorize()
  async getProfile(): Promise<any> {
    return { userId: currentUser?.userId };
  }
}

// การใช้งาน
setCurrentUser({ userId: 1, role: 'admin', permissions: ['users.read', 'users.write'] });
const controller = new AdminController();

controller.deleteUser(5).then(() => console.log('ลบสำเร็จ'));
controller.getUsers().then(users => console.log('Users:', users));
```

### 9.3 @HttpController และ @Route (NestJS-style)

```typescript
import 'reflect-metadata';

type HttpMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';

interface RouteDefinition {
  path: string;
  method: HttpMethod;
  handlerName: string;
  middlewares?: Function[];
}

const ROUTES_KEY = 'routes';
const CONTROLLER_PATH_KEY = 'controllerPath';

function Controller(basePath: string = '') {
  return function(target: any) {
    Reflect.defineMetadata(CONTROLLER_PATH_KEY, basePath, target);
  };
}

function createRouteDecorator(method: HttpMethod) {
  return function(path: string = '') {
    return function(target: any, key: string, descriptor: PropertyDescriptor) {
      const routes: RouteDefinition[] = Reflect.getMetadata(ROUTES_KEY, target) || [];
      
      routes.push({
        path,
        method,
        handlerName: key
      });
      
      Reflect.defineMetadata(ROUTES_KEY, routes, target);
      return descriptor;
    };
  };
}

const Get = createRouteDecorator('GET');
const Post = createRouteDecorator('POST');
const Put = createRouteDecorator('PUT');
const Delete = createRouteDecorator('DELETE');

@Controller('/api/users')
class UserController {
  @Get('/')
  async getAll() {
    return { users: [] };
  }
  
  @Get('/:id')
  async getById() {
    return { user: null };
  }
  
  @Post('/')
  async create() {
    return { user: { id: 1 } };
  }
  
  @Put('/:id')
  async update() {
    return { user: null };
  }
  
  @Delete('/:id')
  async delete() {
    return { success: true };
  }
}

// อ่าน routes
function getRoutes(controller: any) {
  const basePath = Reflect.getMetadata(CONTROLLER_PATH_KEY, controller);
  const routes: RouteDefinition[] = Reflect.getMetadata(ROUTES_KEY, controller.prototype) || [];
  
  return routes.map(route => ({
    ...route,
    fullPath: `${basePath}${route.path}`
  }));
}

console.log(getRoutes(UserController));
```

### 9.4 @Column และ @Table สำหรับ ORM (TypeORM-style)

```typescript
import 'reflect-metadata';

interface ColumnOptions {
  type?: 'string' | 'number' | 'boolean' | 'date';
  nullable?: boolean;
  unique?: boolean;
  length?: number;
  default?: any;
}

interface TableOptions {
  name?: string;
  schema?: string;
}

const TABLE_METADATA = 'table:metadata';
const COLUMNS_METADATA = 'columns:metadata';

function Table(options: TableOptions = {}) {
  return function(target: any) {
    Reflect.defineMetadata(TABLE_METADATA, {
      name: options.name || target.name.toLowerCase() + 's',
      schema: options.schema || 'public'
    }, target);
  };
}

function Column(options: ColumnOptions = {}) {
  return function(target: any, propertyKey: string) {
    const columns: Record<string, ColumnOptions & { name: string }> = 
      Reflect.getMetadata(COLUMNS_METADATA, target.constructor) || {};
    
    const type = Reflect.getMetadata('design:type', target, propertyKey);
    
    columns[propertyKey] = {
      name: propertyKey,
      type: options.type || (type?.name?.toLowerCase()),
      nullable: options.nullable ?? false,
      unique: options.unique ?? false,
      length: options.length,
      default: options.default
    };
    
    Reflect.defineMetadata(COLUMNS_METADATA, columns, target.constructor);
  };
}

function PrimaryColumn(target: any, propertyKey: string) {
  Column({ type: 'number', nullable: false, unique: true })(target, propertyKey);
}

@Table({ name: 'users' })
class UserEntity {
  @PrimaryColumn
  id!: number;
  
  @Column({ type: 'string', length: 100 })
  name!: string;
  
  @Column({ type: 'string', unique: true })
  email!: string;
  
  @Column({ type: 'number', nullable: true })
  age?: number;
  
  @Column({ type: 'boolean', default: true })
  isActive!: boolean;
  
  @Column({ type: 'date' })
  createdAt!: Date;
}

// อ่าน metadata
const tableInfo = Reflect.getMetadata(TABLE_METADATA, UserEntity);
const columns = Reflect.getMetadata(COLUMNS_METADATA, UserEntity);

console.log('Table:', tableInfo);
console.log('Columns:', columns);
```

---

## 10. NestJS Decorator Patterns

### 10.1 @Module Decorator

```typescript
interface ModuleOptions {
  imports?: any[];
  controllers?: any[];
  providers?: any[];
  exports?: any[];
}

function Module(options: ModuleOptions) {
  return function(target: any) {
    Reflect.defineMetadata('module:options', options, target);
  };
}

@Module({
  controllers: [UserController],
  providers: [UserService, UserRepository, DatabaseService]
})
class UserModule {}

@Module({
  imports: [UserModule],
  controllers: [],
  providers: []
})
class AppModule {}
```

### 10.2 @UseGuards, @UsePipes, @UseInterceptors

```typescript
interface Guard {
  canActivate(context: any): boolean | Promise<boolean>;
}

interface Pipe {
  transform(value: any, metadata: any): any;
}

function UseGuards(...guards: (new () => Guard)[]) {
  return function(target: any, key?: string, descriptor?: PropertyDescriptor) {
    if (descriptor) {
      // Method decorator
      Reflect.defineMetadata('guards', guards, target, key!);
    } else {
      // Class decorator
      Reflect.defineMetadata('guards', guards, target);
    }
    return descriptor;
  };
}

function UsePipes(...pipes: (new () => Pipe)[]) {
  return function(target: any, key?: string, descriptor?: PropertyDescriptor) {
    if (descriptor) {
      Reflect.defineMetadata('pipes', pipes, target, key!);
    } else {
      Reflect.defineMetadata('pipes', pipes, target);
    }
    return descriptor;
  };
}

class AuthGuard implements Guard {
  canActivate(context: any): boolean {
    const token = context.headers?.authorization;
    return !!token;
  }
}

class RolesGuard implements Guard {
  canActivate(context: any): boolean {
    const userRole = context.user?.role;
    return userRole === 'admin';
  }
}

class ValidationPipe implements Pipe {
  transform(value: any, metadata: any): any {
    // validation logic
    return value;
  }
}

@UseGuards(AuthGuard)
@Controller('/api/admin')
class AdminController {
  @UseGuards(RolesGuard)
  @Get('/users')
  async getUsers() {
    return [];
  }
  
  @UsePipes(ValidationPipe)
  @Post('/users')
  async createUser(@Body data: any) {
    return data;
  }
}
```

---

## 11. ตัวอย่าง Decorators ขั้นสูง

### 11.1 @Memoize Decorator

```typescript
function Memoize(hashFn?: (...args: any[]) => string) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const cache = new Map<string, any>();
    const originalMethod = descriptor.value;
    
    descriptor.value = function(...args: any[]) {
      const key = hashFn ? hashFn(...args) : JSON.stringify(args);
      
      if (cache.has(key)) {
        return cache.get(key);
      }
      
      const result = originalMethod.apply(this, args);
      cache.set(key, result);
      return result;
    };
    
    // เพิ่ม method สำหรับล้าง cache
    descriptor.value.clearCache = () => cache.clear();
    
    return descriptor;
  };
}

class MathUtils {
  @Memoize()
  fibonacci(n: number): number {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
  
  @Memoize((a, b) => `${a},${b}`)
  complexCalculation(a: number, b: number): number {
    console.log(`คำนวณ... ${a}, ${b}`);
    return Math.pow(a, b);
  }
}

const math = new MathUtils();
console.time('first');
math.fibonacci(40); // คำนวณครั้งแรก
console.timeEnd('first');

console.time('second');
math.fibonacci(40); // ดึงจาก cache
console.timeEnd('second');
```

### 11.2 @RateLimit Decorator

```typescript
interface RateLimitOptions {
  maxRequests: number;
  windowMs: number;
}

function RateLimit(options: RateLimitOptions) {
  const { maxRequests, windowMs } = options;
  const requestCounts = new Map<string, { count: number; resetTime: number }>();
  
  return function(target: any, methodName: string, descriptor: PropertyDescriptor): PropertyDescriptor {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      const key = `${target.constructor.name}:${methodName}`;
      const now = Date.now();
      
      const rateData = requestCounts.get(key);
      
      if (!rateData || now > rateData.resetTime) {
        requestCounts.set(key, { count: 1, resetTime: now + windowMs });
      } else if (rateData.count >= maxRequests) {
        const waitTime = Math.ceil((rateData.resetTime - now) / 1000);
        throw new Error(`Rate limit exceeded. กรุณารอ ${waitTime} วินาที`);
      } else {
        rateData.count++;
      }
      
      return originalMethod.apply(this, args);
    };
    
    return descriptor;
  };
}

class ApiController {
  @RateLimit({ maxRequests: 10, windowMs: 60000 }) // 10 requests per minute
  async getData(): Promise<any> {
    return { data: 'example' };
  }
}
```

---

## สรุป

Decorators เป็น pattern ที่ทรงพลังใน TypeScript:

1. **Class Decorators** - แก้ไขหรือเพิ่มความสามารถให้ class
2. **Method Decorators** - wrap method เพื่อเพิ่ม cross-cutting concerns
3. **Property Decorators** - ควบคุมการ get/set ของ property
4. **Parameter Decorators** - เพิ่ม metadata ให้ parameters
5. **Decorator Factories** - สร้าง configurable decorators
6. **Decorator Composition** - รวม decorators หลายตัว

การใช้ Decorators ช่วยให้:
- Code สะอาดขึ้น (separation of concerns)
- นำ code กลับมาใช้ใหม่ได้ (reusability)
- เพิ่ม metadata ให้ runtime (reflection)
- สร้าง framework-level features (DI, ORM, routing)
