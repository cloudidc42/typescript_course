# ตอนที่ 67: Angular กับ TypeScript

## บทนำ

Angular เป็น framework ที่สร้างขึ้นด้วย TypeScript โดยกำเนิด ดังนั้นการทำงานร่วมกันจึงราบรื่นและมีประสิทธิภาพสูง Angular ใช้ประโยชน์จาก decorators, interfaces, generics และ strict typing ของ TypeScript อย่างเต็มที่

## 67.1 Angular + TypeScript: การรวมกันอย่างลึกซึ้ง

### การตั้งค่าโปรเจกต์

```bash
npm install -g @angular/cli
ng new my-app --strict
cd my-app
```

### tsconfig.json สำหรับ Angular

```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "baseUrl": "./",
    "outDir": "./dist/out-tsc",
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "sourceMap": true,
    "declaration": false,
    "downlevelIteration": true,
    "experimentalDecorators": true,
    "moduleResolution": "node",
    "importHelpers": true,
    "target": "ES2022",
    "module": "ES2022",
    "useDefineForClassFields": false,
    "lib": ["ES2022", "dom"]
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

## 67.2 Component Types

### Component พื้นฐาน

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

// Interface สำหรับ component data
interface ProductData {
  id: number;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-list">
      <h2>รายการสินค้า ({{ products.length }} ชิ้น)</h2>
      <div *ngFor="let product of products; trackBy: trackById" class="product-item">
        <h3>{{ product.name }}</h3>
        <p>ราคา: {{ product.price | currency:'THB' }}</p>
        <span [class.in-stock]="product.inStock">
          {{ product.inStock ? 'มีสินค้า' : 'สินค้าหมด' }}
        </span>
        <button (click)="addToCart(product)" [disabled]="!product.inStock">
          เพิ่มในตะกร้า
        </button>
      </div>
    </div>
  `,
  styles: [`
    .in-stock { color: green; }
    .product-item { border: 1px solid #ccc; padding: 16px; margin: 8px; }
  `]
})
export class ProductListComponent implements OnInit, OnDestroy {
  products: ProductData[] = [];
  private destroy$ = new Subject<void>();
  
  constructor(private productService: ProductService) {}
  
  ngOnInit(): void {
    this.productService.getProducts()
      .pipe(takeUntil(this.destroy$))
      .subscribe({
        next: (products: ProductData[]) => {
          this.products = products;
        },
        error: (error: Error) => {
          console.error('ไม่สามารถโหลดสินค้าได้:', error.message);
        }
      });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  trackById(index: number, product: ProductData): number {
    return product.id;
  }
  
  addToCart(product: ProductData): void {
    console.log('เพิ่ม', product.name, 'ในตะกร้า');
  }
}
```

### Standalone Components (Angular 14+)

```typescript
import { Component, signal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';

interface Task {
  id: number;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

@Component({
  selector: 'app-task-manager',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="task-manager">
      <h2>จัดการงาน</h2>
      
      <div class="stats">
        <span>ทั้งหมด: {{ totalTasks() }}</span>
        <span>เสร็จแล้ว: {{ completedTasks() }}</span>
        <span>ค้างอยู่: {{ pendingTasks() }}</span>
      </div>
      
      <input [(ngModel)]="newTaskTitle" placeholder="งานใหม่...">
      <select [(ngModel)]="newTaskPriority">
        <option value="low">ต่ำ</option>
        <option value="medium">กลาง</option>
        <option value="high">สูง</option>
      </select>
      <button (click)="addTask()">เพิ่มงาน</button>
      
      <ul>
        <li *ngFor="let task of tasks()">
          <input type="checkbox" 
                 [checked]="task.completed"
                 (change)="toggleTask(task.id)">
          <span [class.completed]="task.completed"
                [class]="'priority-' + task.priority">
            {{ task.title }}
          </span>
          <button (click)="deleteTask(task.id)">ลบ</button>
        </li>
      </ul>
    </div>
  `
})
export class TaskManagerComponent {
  // Signals (Angular 16+)
  tasks = signal<Task[]>([]);
  newTaskTitle = '';
  newTaskPriority: Task['priority'] = 'medium';
  private nextId = 1;
  
  // Computed signals
  totalTasks = computed(() => this.tasks().length);
  completedTasks = computed(() => this.tasks().filter(t => t.completed).length);
  pendingTasks = computed(() => this.tasks().filter(t => !t.completed).length);
  
  addTask(): void {
    if (!this.newTaskTitle.trim()) return;
    
    const newTask: Task = {
      id: this.nextId++,
      title: this.newTaskTitle.trim(),
      completed: false,
      priority: this.newTaskPriority
    };
    
    this.tasks.update(tasks => [...tasks, newTask]);
    this.newTaskTitle = '';
  }
  
  toggleTask(id: number): void {
    this.tasks.update(tasks =>
      tasks.map(task =>
        task.id === id ? { ...task, completed: !task.completed } : task
      )
    );
  }
  
  deleteTask(id: number): void {
    this.tasks.update(tasks => tasks.filter(task => task.id !== id));
  }
}
```

## 67.3 Service Types

### Injectable Service

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpHeaders } from '@angular/common/http';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import { map, catchError, tap, shareReplay } from 'rxjs/operators';

// API response wrapper
interface ApiResponse<T> {
  data: T;
  total: number;
  page: number;
  perPage: number;
}

// Pagination parameters
interface PaginationParams {
  page: number;
  perPage: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}

interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'moderator';
  createdAt: Date;
}

interface CreateUserDto {
  name: string;
  email: string;
  password: string;
  role?: User['role'];
}

interface UpdateUserDto {
  name?: string;
  email?: string;
  role?: User['role'];
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private readonly baseUrl = '/api/users';
  private usersCache$?: Observable<User[]>;
  
  // User state management
  private selectedUser$ = new BehaviorSubject<User | null>(null);
  
  constructor(private http: HttpClient) {}
  
  getUsers(params?: PaginationParams): Observable<ApiResponse<User[]>> {
    let httpParams = new HttpParams();
    
    if (params) {
      httpParams = httpParams
        .set('page', params.page.toString())
        .set('perPage', params.perPage.toString());
      
      if (params.sortBy) {
        httpParams = httpParams.set('sortBy', params.sortBy);
      }
      if (params.sortOrder) {
        httpParams = httpParams.set('sortOrder', params.sortOrder);
      }
    }
    
    return this.http.get<ApiResponse<User[]>>(this.baseUrl, { params: httpParams });
  }
  
  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`).pipe(
      shareReplay(1) // cache result
    );
  }
  
  createUser(dto: CreateUserDto): Observable<User> {
    const headers = new HttpHeaders({ 'Content-Type': 'application/json' });
    
    return this.http.post<User>(this.baseUrl, dto, { headers }).pipe(
      tap(newUser => {
        console.log('สร้างผู้ใช้ใหม่:', newUser.name);
        // Invalidate cache
        this.usersCache$ = undefined;
      })
    );
  }
  
  updateUser(id: number, dto: UpdateUserDto): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, dto);
  }
  
  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
  
  selectUser(user: User | null): void {
    this.selectedUser$.next(user);
  }
  
  getSelectedUser(): Observable<User | null> {
    return this.selectedUser$.asObservable();
  }
  
  // Search with type safety
  searchUsers(query: string): Observable<User[]> {
    return this.http.get<User[]>(`${this.baseUrl}/search`, {
      params: new HttpParams().set('q', query)
    });
  }
}
```

## 67.4 Dependency Injection ใน Angular

### Token-based Injection

```typescript
import { InjectionToken, inject } from '@angular/core';

// Custom injection tokens
export const API_URL = new InjectionToken<string>('API_URL');
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');
export const LOGGER = new InjectionToken<Logger>('LOGGER');

interface AppConfig {
  apiUrl: string;
  maxRetries: number;
  timeout: number;
  debugMode: boolean;
}

interface Logger {
  log(message: string): void;
  error(message: string, error?: Error): void;
  warn(message: string): void;
}

// สร้าง logger implementation
class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[INFO] ${new Date().toISOString()}: ${message}`);
  }
  
  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${new Date().toISOString()}: ${message}`, error);
  }
  
  warn(message: string): void {
    console.warn(`[WARN] ${new Date().toISOString()}: ${message}`);
  }
}

// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter([]),
    provideHttpClient(),
    { provide: API_URL, useValue: 'https://api.example.com' },
    { 
      provide: APP_CONFIG, 
      useValue: { 
        apiUrl: 'https://api.example.com',
        maxRetries: 3,
        timeout: 5000,
        debugMode: false
      } 
    },
    { provide: LOGGER, useClass: ConsoleLogger }
  ]
};

// การใช้งานใน service
@Injectable({ providedIn: 'root' })
export class DataService {
  private readonly apiUrl = inject(API_URL);
  private readonly config = inject(APP_CONFIG);
  private readonly logger = inject(LOGGER);
  
  constructor(private http: HttpClient) {
    this.logger.log(`DataService initialized with URL: ${this.apiUrl}`);
  }
  
  getData<T>(endpoint: string): Observable<T> {
    this.logger.log(`Fetching: ${this.apiUrl}/${endpoint}`);
    
    return this.http.get<T>(`${this.apiUrl}/${endpoint}`).pipe(
      retry(this.config.maxRetries),
      catchError(error => {
        this.logger.error(`Failed to fetch ${endpoint}`, error);
        return throwError(() => error);
      })
    );
  }
}
```

## 67.5 Decorators

### Component Decorators

```typescript
import { 
  Component, Input, Output, EventEmitter, 
  ViewChild, ViewChildren, ContentChild,
  ChangeDetectionStrategy, ChangeDetectorRef,
  QueryList, ElementRef, AfterViewInit
} from '@angular/core';

interface TableColumn {
  key: string;
  label: string;
  sortable?: boolean;
  type?: 'text' | 'number' | 'date' | 'boolean';
}

interface SortEvent {
  column: string;
  direction: 'asc' | 'desc';
}

@Component({
  selector: 'app-data-table',
  changeDetection: ChangeDetectionStrategy.OnPush, // Performance optimization
  template: `
    <table #tableRef>
      <thead>
        <tr>
          <th *ngFor="let col of columns" 
              (click)="col.sortable && onSort(col.key)"
              [class.sortable]="col.sortable">
            {{ col.label }}
            <span *ngIf="sortColumn === col.key">
              {{ sortDirection === 'asc' ? '↑' : '↓' }}
            </span>
          </th>
          <th>การดำเนินการ</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let row of data; index as i">
          <td *ngFor="let col of columns">
            {{ formatCell(row[col.key], col.type) }}
          </td>
          <td>
            <button (click)="edit.emit(row)">แก้ไข</button>
            <button (click)="delete.emit(row)">ลบ</button>
          </td>
        </tr>
      </tbody>
    </table>
    <p *ngIf="data.length === 0" class="empty-message">
      ไม่มีข้อมูล
    </p>
  `
})
export class DataTableComponent<T extends Record<string, any>> implements AfterViewInit {
  // Input decorators with types
  @Input({ required: true }) columns!: TableColumn[];
  @Input({ required: true }) data!: T[];
  @Input() sortColumn: string = '';
  @Input() sortDirection: 'asc' | 'desc' = 'asc';
  @Input() emptyMessage: string = 'ไม่มีข้อมูล';
  
  // Output decorators
  @Output() edit = new EventEmitter<T>();
  @Output() delete = new EventEmitter<T>();
  @Output() sort = new EventEmitter<SortEvent>();
  
  // ViewChild with type
  @ViewChild('tableRef') tableRef!: ElementRef<HTMLTableElement>;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngAfterViewInit(): void {
    console.log('Table element:', this.tableRef.nativeElement);
  }
  
  onSort(column: string): void {
    const direction: 'asc' | 'desc' = 
      this.sortColumn === column && this.sortDirection === 'asc' ? 'desc' : 'asc';
    
    this.sort.emit({ column, direction });
  }
  
  formatCell(value: any, type?: TableColumn['type']): string {
    if (value === null || value === undefined) return '-';
    
    switch (type) {
      case 'date':
        return new Date(value).toLocaleDateString('th-TH');
      case 'boolean':
        return value ? 'ใช่' : 'ไม่';
      case 'number':
        return value.toLocaleString('th-TH');
      default:
        return String(value);
    }
  }
}
```

## 67.6 Input/Output พร้อม Types

### Input/Output Pattern

```typescript
import { Component, Input, Output, EventEmitter, OnChanges, SimpleChanges } from '@angular/core';

interface PaginationConfig {
  currentPage: number;
  totalItems: number;
  itemsPerPage: number;
  maxPages?: number;
}

interface PageChangeEvent {
  page: number;
  previousPage: number;
}

@Component({
  selector: 'app-pagination',
  template: `
    <div class="pagination">
      <button (click)="goToPage(1)" [disabled]="config.currentPage === 1">
        หน้าแรก
      </button>
      <button (click)="goToPage(config.currentPage - 1)" 
              [disabled]="config.currentPage === 1">
        ก่อนหน้า
      </button>
      
      <button *ngFor="let page of visiblePages" 
              (click)="goToPage(page)"
              [class.active]="page === config.currentPage">
        {{ page }}
      </button>
      
      <button (click)="goToPage(config.currentPage + 1)"
              [disabled]="config.currentPage === totalPages">
        ถัดไป
      </button>
      <button (click)="goToPage(totalPages)" 
              [disabled]="config.currentPage === totalPages">
        หน้าสุดท้าย
      </button>
      
      <span class="info">
        หน้า {{ config.currentPage }} จาก {{ totalPages }}
        ({{ config.totalItems }} รายการ)
      </span>
    </div>
  `
})
export class PaginationComponent implements OnChanges {
  @Input({ required: true }) config!: PaginationConfig;
  @Output() pageChange = new EventEmitter<PageChangeEvent>();
  
  totalPages = 0;
  visiblePages: number[] = [];
  
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['config']) {
      this.updatePagination();
    }
  }
  
  private updatePagination(): void {
    this.totalPages = Math.ceil(this.config.totalItems / this.config.itemsPerPage);
    this.visiblePages = this.calculateVisiblePages();
  }
  
  private calculateVisiblePages(): number[] {
    const maxVisible = this.config.maxPages || 5;
    const current = this.config.currentPage;
    const total = this.totalPages;
    
    let start = Math.max(1, current - Math.floor(maxVisible / 2));
    let end = Math.min(total, start + maxVisible - 1);
    
    if (end - start + 1 < maxVisible) {
      start = Math.max(1, end - maxVisible + 1);
    }
    
    return Array.from({ length: end - start + 1 }, (_, i) => start + i);
  }
  
  goToPage(page: number): void {
    if (page < 1 || page > this.totalPages || page === this.config.currentPage) {
      return;
    }
    
    this.pageChange.emit({
      page,
      previousPage: this.config.currentPage
    });
  }
}
```

## 67.7 Template-Driven Forms

```typescript
import { Component, ViewChild } from '@angular/core';
import { NgForm, NgModel } from '@angular/forms';

interface ContactForm {
  name: string;
  email: string;
  phone: string;
  subject: string;
  message: string;
  agreeToTerms: boolean;
}

@Component({
  selector: 'app-contact-form',
  template: `
    <form #contactForm="ngForm" (ngSubmit)="onSubmit(contactForm)" novalidate>
      
      <div class="form-group">
        <label>ชื่อ-นามสกุล *</label>
        <input 
          name="name"
          [(ngModel)]="formData.name"
          #nameInput="ngModel"
          required
          minlength="2"
          maxlength="100"
          placeholder="กรอกชื่อ-นามสกุล">
        
        <div *ngIf="nameInput.invalid && (nameInput.dirty || nameInput.touched)">
          <span *ngIf="nameInput.errors?.['required']">กรุณากรอกชื่อ</span>
          <span *ngIf="nameInput.errors?.['minlength']">ชื่อต้องมีอย่างน้อย 2 ตัวอักษร</span>
        </div>
      </div>
      
      <div class="form-group">
        <label>อีเมล *</label>
        <input
          type="email"
          name="email"
          [(ngModel)]="formData.email"
          #emailInput="ngModel"
          required
          email
          placeholder="example@email.com">
        
        <div *ngIf="emailInput.invalid && (emailInput.dirty || emailInput.touched)">
          <span *ngIf="emailInput.errors?.['required']">กรุณากรอกอีเมล</span>
          <span *ngIf="emailInput.errors?.['email']">รูปแบบอีเมลไม่ถูกต้อง</span>
        </div>
      </div>
      
      <div class="form-group">
        <label>เรื่อง *</label>
        <select name="subject" [(ngModel)]="formData.subject" required>
          <option value="">-- เลือกเรื่อง --</option>
          <option value="general">ทั่วไป</option>
          <option value="support">ขอความช่วยเหลือ</option>
          <option value="billing">การชำระเงิน</option>
          <option value="feedback">ข้อเสนอแนะ</option>
        </select>
      </div>
      
      <div class="form-group">
        <label>ข้อความ *</label>
        <textarea
          name="message"
          [(ngModel)]="formData.message"
          #messageInput="ngModel"
          required
          minlength="10"
          rows="5"
          placeholder="กรอกข้อความของคุณ...">
        </textarea>
      </div>
      
      <div class="form-group">
        <label>
          <input type="checkbox" name="agreeToTerms" [(ngModel)]="formData.agreeToTerms" required>
          ฉันยอมรับเงื่อนไขการใช้งาน
        </label>
      </div>
      
      <button type="submit" [disabled]="contactForm.invalid || isSubmitting">
        {{ isSubmitting ? 'กำลังส่ง...' : 'ส่งข้อความ' }}
      </button>
    </form>
  `
})
export class ContactFormComponent {
  formData: ContactForm = {
    name: '',
    email: '',
    phone: '',
    subject: '',
    message: '',
    agreeToTerms: false
  };
  
  isSubmitting = false;
  
  onSubmit(form: NgForm): void {
    if (form.invalid) return;
    
    this.isSubmitting = true;
    
    // ส่งข้อมูล
    console.log('ส่งข้อมูล:', this.formData);
    
    setTimeout(() => {
      this.isSubmitting = false;
      form.resetForm();
      alert('ส่งข้อความเรียบร้อยแล้ว!');
    }, 1000);
  }
}
```

## 67.8 Reactive Forms

```typescript
import { Component, OnInit } from '@angular/core';
import { 
  FormBuilder, FormGroup, FormArray, FormControl,
  Validators, AbstractControl, ValidationErrors, AsyncValidatorFn
} from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';

// Custom validator
function thaiPhoneValidator(control: AbstractControl): ValidationErrors | null {
  if (!control.value) return null;
  
  const phoneRegex = /^(0[689][0-9]{8}|0[2][0-9]{7})$/;
  return phoneRegex.test(control.value) ? null : { thaiPhone: true };
}

// Async validator
function emailUniqueValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);
    
    return timer(500).pipe(
      switchMap(() => userService.checkEmailExists(control.value)),
      map(exists => exists ? { emailTaken: true } : null)
    );
  };
}

// Form value types
interface AddressForm {
  street: string;
  city: string;
  province: string;
  postalCode: string;
}

interface RegisterFormValue {
  personalInfo: {
    firstName: string;
    lastName: string;
    phone: string;
  };
  account: {
    email: string;
    password: string;
    confirmPassword: string;
  };
  addresses: AddressForm[];
}

@Component({
  selector: 'app-register',
  template: `
    <form [formGroup]="registerForm" (ngSubmit)="onSubmit()">
      
      <fieldset formGroupName="personalInfo">
        <legend>ข้อมูลส่วนตัว</legend>
        
        <input formControlName="firstName" placeholder="ชื่อ">
        <div *ngIf="getControl('personalInfo.firstName')?.errors?.['required'] && 
                    getControl('personalInfo.firstName')?.touched">
          กรุณากรอกชื่อ
        </div>
        
        <input formControlName="lastName" placeholder="นามสกุล">
        
        <input formControlName="phone" placeholder="เบอร์โทรศัพท์">
        <div *ngIf="getControl('personalInfo.phone')?.errors?.['thaiPhone']">
          รูปแบบเบอร์โทรไม่ถูกต้อง
        </div>
      </fieldset>
      
      <fieldset formGroupName="account">
        <legend>บัญชีผู้ใช้</legend>
        
        <input formControlName="email" type="email" placeholder="อีเมล">
        <div *ngIf="getControl('account.email')?.errors?.['emailTaken']">
          อีเมลนี้มีการใช้งานแล้ว
        </div>
        <div *ngIf="getControl('account.email')?.pending">
          กำลังตรวจสอบ...
        </div>
        
        <input formControlName="password" type="password" placeholder="รหัสผ่าน">
        <input formControlName="confirmPassword" type="password" placeholder="ยืนยันรหัสผ่าน">
        
        <div *ngIf="registerForm.get('account')?.errors?.['passwordMismatch']">
          รหัสผ่านไม่ตรงกัน
        </div>
      </fieldset>
      
      <div formArrayName="addresses">
        <div *ngFor="let address of addresses.controls; index as i" [formGroupName]="i">
          <h4>ที่อยู่ {{ i + 1 }}</h4>
          <input formControlName="street" placeholder="ที่อยู่">
          <input formControlName="city" placeholder="เมือง/อำเภอ">
          <input formControlName="province" placeholder="จังหวัด">
          <input formControlName="postalCode" placeholder="รหัสไปรษณีย์">
          <button type="button" (click)="removeAddress(i)">ลบที่อยู่</button>
        </div>
        <button type="button" (click)="addAddress()">เพิ่มที่อยู่</button>
      </div>
      
      <button type="submit" [disabled]="registerForm.invalid || registerForm.pending">
        ลงทะเบียน
      </button>
    </form>
  `
})
export class RegisterComponent implements OnInit {
  registerForm!: FormGroup;
  
  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {}
  
  ngOnInit(): void {
    this.registerForm = this.fb.group({
      personalInfo: this.fb.group({
        firstName: ['', [Validators.required, Validators.minLength(2)]],
        lastName: ['', [Validators.required, Validators.minLength(2)]],
        phone: ['', [Validators.required, thaiPhoneValidator]]
      }),
      account: this.fb.group({
        email: [
          '', 
          [Validators.required, Validators.email],
          [emailUniqueValidator(this.userService)]
        ],
        password: ['', [Validators.required, Validators.minLength(8)]],
        confirmPassword: ['', Validators.required]
      }, { validators: this.passwordMatchValidator }),
      addresses: this.fb.array([this.createAddressGroup()])
    });
  }
  
  get addresses(): FormArray {
    return this.registerForm.get('addresses') as FormArray;
  }
  
  createAddressGroup(): FormGroup {
    return this.fb.group({
      street: ['', Validators.required],
      city: ['', Validators.required],
      province: ['', Validators.required],
      postalCode: ['', [Validators.required, Validators.pattern(/^\d{5}$/)]]
    });
  }
  
  addAddress(): void {
    this.addresses.push(this.createAddressGroup());
  }
  
  removeAddress(index: number): void {
    this.addresses.removeAt(index);
  }
  
  getControl(path: string): AbstractControl | null {
    return this.registerForm.get(path);
  }
  
  passwordMatchValidator(group: AbstractControl): ValidationErrors | null {
    const password = group.get('password')?.value;
    const confirmPassword = group.get('confirmPassword')?.value;
    
    return password === confirmPassword ? null : { passwordMismatch: true };
  }
  
  onSubmit(): void {
    if (this.registerForm.valid) {
      const value = this.registerForm.value as RegisterFormValue;
      console.log('ลงทะเบียน:', value);
    }
  }
}
```

## 67.9 HTTP Client พร้อม Types

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpInterceptorFn, HttpRequest, HttpHandler } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map, retry } from 'rxjs/operators';

// Generic HTTP service
@Injectable({ providedIn: 'root' })
export class HttpService {
  constructor(private http: HttpClient) {}
  
  get<T>(url: string, params?: Record<string, string>): Observable<T> {
    return this.http.get<T>(url, { params }).pipe(
      catchError(this.handleError)
    );
  }
  
  post<T, B = unknown>(url: string, body: B): Observable<T> {
    return this.http.post<T>(url, body).pipe(
      catchError(this.handleError)
    );
  }
  
  put<T, B = unknown>(url: string, body: B): Observable<T> {
    return this.http.put<T>(url, body).pipe(
      catchError(this.handleError)
    );
  }
  
  patch<T, B = unknown>(url: string, body: Partial<B>): Observable<T> {
    return this.http.patch<T>(url, body).pipe(
      catchError(this.handleError)
    );
  }
  
  delete<T>(url: string): Observable<T> {
    return this.http.delete<T>(url).pipe(
      catchError(this.handleError)
    );
  }
  
  private handleError(error: any): Observable<never> {
    let errorMessage = 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ';
    
    if (error.error instanceof ErrorEvent) {
      errorMessage = `ข้อผิดพลาด: ${error.error.message}`;
    } else {
      errorMessage = `รหัสข้อผิดพลาด: ${error.status}\nข้อความ: ${error.message}`;
    }
    
    return throwError(() => new Error(errorMessage));
  }
}
```

## 67.10 Route Parameters Typing

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router, ParamMap } from '@angular/router';
import { Observable } from 'rxjs';
import { switchMap, map } from 'rxjs/operators';

interface RouteParams {
  id: string;
  category?: string;
}

interface QueryParams {
  page?: string;
  sort?: string;
  filter?: string;
}

@Component({
  selector: 'app-product-detail',
  template: `
    <div *ngIf="product$ | async as product">
      <h1>{{ product.name }}</h1>
      <p>ราคา: {{ product.price | currency:'THB' }}</p>
      <p>หมวดหมู่: {{ product.category }}</p>
    </div>
  `
})
export class ProductDetailComponent implements OnInit {
  product$!: Observable<ProductData>;
  
  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private productService: ProductService
  ) {}
  
  ngOnInit(): void {
    // ดึง route params แบบ type-safe
    this.product$ = this.route.paramMap.pipe(
      map((params: ParamMap) => {
        const id = params.get('id');
        if (!id) throw new Error('ไม่พบ ID');
        return Number(id);
      }),
      switchMap(id => this.productService.getProductById(id))
    );
    
    // ดึง query params
    this.route.queryParamMap.subscribe((params: ParamMap) => {
      const page = Number(params.get('page') || '1');
      const sort = params.get('sort') || 'name';
      console.log(`หน้า: ${page}, เรียงตาม: ${sort}`);
    });
  }
  
  navigateToCategory(category: string): void {
    this.router.navigate(['/products'], {
      queryParams: { category, page: 1 }
    });
  }
}

// Typed route data
interface RouteData {
  title: string;
  breadcrumbs: Array<{ label: string; url: string }>;
}

@Component({
  selector: 'app-page',
  template: `<h1>{{ title }}</h1>`
})
export class PageComponent implements OnInit {
  title = '';
  
  constructor(private route: ActivatedRoute) {}
  
  ngOnInit(): void {
    const data = this.route.snapshot.data as RouteData;
    this.title = data.title;
  }
}
```

## 67.11 Guards พร้อม Types

```typescript
import { inject } from '@angular/core';
import { 
  CanActivateFn, CanDeactivateFn, 
  ActivatedRouteSnapshot, RouterStateSnapshot,
  Router, UrlTree
} from '@angular/router';
import { Observable, map } from 'rxjs';

// Auth guard
export const authGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot,
  state: RouterStateSnapshot
): Observable<boolean | UrlTree> => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  return authService.isAuthenticated().pipe(
    map(isAuth => {
      if (isAuth) return true;
      return router.createUrlTree(['/login'], {
        queryParams: { returnUrl: state.url }
      });
    })
  );
};

// Role guard
export const roleGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot
): Observable<boolean | UrlTree> => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const requiredRoles = route.data['roles'] as string[];
  
  return authService.getCurrentUser().pipe(
    map(user => {
      if (!user) return router.createUrlTree(['/login']);
      
      const hasRole = requiredRoles.some(role => user.role === role);
      if (!hasRole) return router.createUrlTree(['/unauthorized']);
      
      return true;
    })
  );
};

// CanDeactivate guard
export interface CanDeactivateComponent {
  canDeactivate(): boolean | Observable<boolean>;
}

export const unsavedChangesGuard: CanDeactivateFn<CanDeactivateComponent> = (
  component: CanDeactivateComponent
): boolean | Observable<boolean> => {
  return component.canDeactivate();
};
```

## 67.12 Resolvers

```typescript
import { inject } from '@angular/core';
import { ResolveFn, ActivatedRouteSnapshot } from '@angular/router';
import { Observable, catchError, of } from 'rxjs';

interface ResolvedData {
  user: User;
  orders: Order[];
}

export const userResolver: ResolveFn<User> = (
  route: ActivatedRouteSnapshot
): Observable<User> => {
  const userService = inject(UserService);
  const id = Number(route.paramMap.get('id'));
  
  return userService.getUserById(id).pipe(
    catchError(() => of({
      id: 0, 
      name: 'ไม่พบผู้ใช้', 
      email: '', 
      role: 'user' as const, 
      createdAt: new Date()
    }))
  );
};

// App routes with resolver
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'users/:id',
    component: UserDetailComponent,
    resolve: { user: userResolver },
    canActivate: [authGuard],
    data: { roles: ['admin', 'moderator'] }
  }
];
```

## 67.13 Interceptors

```typescript
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { Observable, throwError } from 'rxjs';
import { catchError, switchMap } from 'rxjs/operators';

// Auth interceptor
export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<any> => {
  const authService = inject(AuthService);
  const token = authService.getToken();
  
  if (token) {
    req = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }
  
  return next(req).pipe(
    catchError(error => {
      if (error.status === 401) {
        return authService.refreshToken().pipe(
          switchMap(newToken => {
            const retryReq = req.clone({
              setHeaders: { Authorization: `Bearer ${newToken}` }
            });
            return next(retryReq);
          }),
          catchError(() => {
            authService.logout();
            return throwError(() => error);
          })
        );
      }
      return throwError(() => error);
    })
  );
};

// Loading interceptor
export const loadingInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<any> => {
  const loadingService = inject(LoadingService);
  
  loadingService.show();
  
  return next(req).pipe(
    finalize(() => loadingService.hide())
  );
};
```

## 67.14 Custom Pipes

```typescript
import { Pipe, PipeTransform } from '@angular/core';

// Thai date pipe
@Pipe({ name: 'thaiDate', pure: true, standalone: true })
export class ThaiDatePipe implements PipeTransform {
  private readonly thaiMonths = [
    'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
    'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
    'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'
  ];
  
  transform(
    value: Date | string | null, 
    format: 'short' | 'long' | 'buddhist' = 'long'
  ): string {
    if (!value) return '';
    
    const date = value instanceof Date ? value : new Date(value);
    
    if (isNaN(date.getTime())) return 'วันที่ไม่ถูกต้อง';
    
    const day = date.getDate();
    const month = this.thaiMonths[date.getMonth()];
    const year = date.getFullYear();
    const buddhistYear = year + 543;
    
    switch (format) {
      case 'short':
        return `${day}/${date.getMonth() + 1}/${buddhistYear}`;
      case 'buddhist':
        return `${day} ${month} ${buddhistYear}`;
      default:
        return `${day} ${month} ${year}`;
    }
  }
}

// Filter pipe
@Pipe({ name: 'filterBy', pure: false, standalone: true })
export class FilterByPipe implements PipeTransform {
  transform<T extends Record<string, any>>(
    items: T[],
    field: keyof T,
    value: any
  ): T[] {
    if (!items || !field) return items;
    return items.filter(item => item[field] === value);
  }
}

// Currency Thai pipe
@Pipe({ name: 'thaiCurrency', standalone: true })
export class ThaiCurrencyPipe implements PipeTransform {
  transform(value: number | null, showSymbol: boolean = true): string {
    if (value === null || value === undefined) return '';
    
    const formatted = value.toLocaleString('th-TH', {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2
    });
    
    return showSymbol ? `฿${formatted}` : formatted;
  }
}
```

## 67.15 NgRx State Management

```typescript
// State types
interface ProductState {
  products: ProductData[];
  selectedProduct: ProductData | null;
  loading: boolean;
  error: string | null;
  totalCount: number;
  filters: ProductFilters;
}

interface ProductFilters {
  category: string;
  minPrice: number;
  maxPrice: number;
  inStock: boolean;
}

// Actions
import { createAction, props } from '@ngrx/store';

export const loadProducts = createAction('[Products] Load Products');
export const loadProductsSuccess = createAction(
  '[Products] Load Products Success',
  props<{ products: ProductData[]; total: number }>()
);
export const loadProductsFailure = createAction(
  '[Products] Load Products Failure',
  props<{ error: string }>()
);
export const selectProduct = createAction(
  '[Products] Select Product',
  props<{ product: ProductData }>()
);
export const clearSelectedProduct = createAction('[Products] Clear Selected');
export const updateFilters = createAction(
  '[Products] Update Filters',
  props<{ filters: Partial<ProductFilters> }>()
);

// Reducer
import { createReducer, on } from '@ngrx/store';

const initialState: ProductState = {
  products: [],
  selectedProduct: null,
  loading: false,
  error: null,
  totalCount: 0,
  filters: {
    category: '',
    minPrice: 0,
    maxPrice: Infinity,
    inStock: false
  }
};

export const productReducer = createReducer(
  initialState,
  on(loadProducts, state => ({ ...state, loading: true, error: null })),
  on(loadProductsSuccess, (state, { products, total }) => ({
    ...state,
    products,
    totalCount: total,
    loading: false
  })),
  on(loadProductsFailure, (state, { error }) => ({
    ...state,
    error,
    loading: false
  })),
  on(selectProduct, (state, { product }) => ({
    ...state,
    selectedProduct: product
  })),
  on(clearSelectedProduct, state => ({
    ...state,
    selectedProduct: null
  })),
  on(updateFilters, (state, { filters }) => ({
    ...state,
    filters: { ...state.filters, ...filters }
  }))
);

// Selectors
import { createSelector, createFeatureSelector } from '@ngrx/store';

export const selectProductFeature = createFeatureSelector<ProductState>('products');

export const selectAllProducts = createSelector(
  selectProductFeature,
  state => state.products
);

export const selectLoading = createSelector(
  selectProductFeature,
  state => state.loading
);

export const selectFilteredProducts = createSelector(
  selectAllProducts,
  selectProductFeature,
  (products, state) => {
    const { filters } = state;
    return products.filter(p => {
      if (filters.category && p.category !== filters.category) return false;
      if (p.price < filters.minPrice || p.price > filters.maxPrice) return false;
      if (filters.inStock && !p.inStock) return false;
      return true;
    });
  }
);

// Effects
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { switchMap, map, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

@Injectable()
export class ProductEffects {
  loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadProducts),
      switchMap(() =>
        this.productService.getProducts().pipe(
          map(response => loadProductsSuccess({
            products: response.data,
            total: response.total
          })),
          catchError(error => of(loadProductsFailure({
            error: error.message
          })))
        )
      )
    )
  );
  
  constructor(
    private actions$: Actions,
    private productService: ProductService
  ) {}
}

// Component using NgRx
@Component({
  selector: 'app-products-ngrx',
  template: `
    <div *ngIf="loading$ | async">กำลังโหลด...</div>
    <div *ngIf="error$ | async as error" class="error">{{ error }}</div>
    
    <div *ngFor="let product of filteredProducts$ | async">
      <h3>{{ product.name }}</h3>
      <p>{{ product.price | thaiCurrency }}</p>
    </div>
  `
})
export class ProductsNgRxComponent implements OnInit {
  loading$ = this.store.select(selectLoading);
  error$ = this.store.select(selectProductFeature).pipe(
    map(state => state.error)
  );
  filteredProducts$ = this.store.select(selectFilteredProducts);
  
  constructor(private store: Store) {}
  
  ngOnInit(): void {
    this.store.dispatch(loadProducts());
  }
}
```

## สรุป

Angular กับ TypeScript ทำงานร่วมกันได้อย่างยอดเยี่ยม:

1. **Component types** ช่วยให้โค้ดอ่านง่ายและปลอดภัย
2. **Service injection** ด้วย tokens และ generics
3. **Reactive Forms** กับ typed form groups
4. **HTTP Client** พร้อม response types
5. **Guards และ Resolvers** สำหรับ routing
6. **Interceptors** สำหรับ cross-cutting concerns
7. **NgRx** สำหรับ state management แบบ predictable

การใช้ TypeScript อย่างเต็มที่ใน Angular จะทำให้ development experience ดีขึ้นมากและลด runtime errors ได้อย่างมีนัยสำคัญ
