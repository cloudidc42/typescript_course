# ตอนที่ 66: RxJS กับ TypeScript

## บทนำ

RxJS (Reactive Extensions for JavaScript) คือไลบรารีสำหรับการเขียนโปรแกรมแบบ Reactive โดยใช้ Observable sequences เพื่อจัดการกับข้อมูลแบบ asynchronous และ event-based เมื่อรวมกับ TypeScript จะทำให้โค้ดมีความปลอดภัยด้านชนิดข้อมูลและอ่านง่ายขึ้นมาก

## 66.1 ภาพรวม RxJS กับ TypeScript

### การติดตั้ง

```bash
npm install rxjs
npm install --save-dev @types/node
```

### แนวคิดพื้นฐาน

```typescript
import { Observable, of, from, interval } from 'rxjs';
import { map, filter, take } from 'rxjs/operators';

// Observable คือ stream ของข้อมูลที่สามารถ subscribe ได้
const numbers$: Observable<number> = of(1, 2, 3, 4, 5);

// การ subscribe เพื่อรับข้อมูล
numbers$.subscribe({
  next: (value: number) => console.log('ค่า:', value),
  error: (err: Error) => console.error('เกิดข้อผิดพลาด:', err),
  complete: () => console.log('เสร็จสิ้น')
});

// ผลลัพธ์:
// ค่า: 1
// ค่า: 2
// ค่า: 3
// ค่า: 4
// ค่า: 5
// เสร็จสิ้น
```

### ประโยชน์ของ RxJS กับ TypeScript

```typescript
import { Observable, Subject, BehaviorSubject } from 'rxjs';
import { map, filter, debounceTime, distinctUntilChanged } from 'rxjs/operators';

// TypeScript ช่วยให้เราระบุชนิดข้อมูลของ Observable
interface User {
  id: number;
  name: string;
  email: string;
}

// Observable ที่มีชนิดข้อมูลชัดเจน
const users$: Observable<User[]> = new Observable<User[]>(subscriber => {
  subscriber.next([
    { id: 1, name: 'สมชาย', email: 'somchai@example.com' },
    { id: 2, name: 'สมหญิง', email: 'somying@example.com' }
  ]);
  subscriber.complete();
});

// การใช้ map กับชนิดข้อมูล
const userNames$: Observable<string[]> = users$.pipe(
  map((users: User[]) => users.map(u => u.name))
);

userNames$.subscribe(names => console.log('ชื่อผู้ใช้:', names));
```

## 66.2 ชนิดข้อมูลของ Observable

### Observable พื้นฐาน

```typescript
import { Observable, Subscriber, Subscription } from 'rxjs';

// การสร้าง Observable แบบ generic
function createNumberStream(start: number, end: number): Observable<number> {
  return new Observable<number>((subscriber: Subscriber<number>) => {
    for (let i = start; i <= end; i++) {
      subscriber.next(i);
    }
    subscriber.complete();
    
    // Cleanup function
    return () => {
      console.log('Unsubscribed จาก number stream');
    };
  });
}

const stream$ = createNumberStream(1, 5);
const subscription: Subscription = stream$.subscribe({
  next: (n) => console.log(n),
  complete: () => console.log('เสร็จ')
});

// ยกเลิก subscription
subscription.unsubscribe();
```

### Generic Types

```typescript
import { Observable, of } from 'rxjs';
import { map } from 'rxjs/operators';

// Observable<T> - T คือชนิดข้อมูลที่ emit
const stringStream$: Observable<string> = of('Hello', 'World', 'TypeScript');
const numberStream$: Observable<number> = of(1, 2, 3, 4, 5);
const booleanStream$: Observable<boolean> = of(true, false, true);

// Generic function สำหรับ transform
function transform<T, R>(
  source$: Observable<T>, 
  transformer: (value: T) => R
): Observable<R> {
  return source$.pipe(map(transformer));
}

const lengths$ = transform(stringStream$, (s: string) => s.length);
lengths$.subscribe(len => console.log('ความยาว:', len));

// ชนิดข้อมูลแบบซับซ้อน
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

const apiResponse$: Observable<ApiResponse<User[]>> = of({
  data: [{ id: 1, name: 'สมชาย', email: 'test@example.com' }],
  status: 200,
  message: 'สำเร็จ'
});

apiResponse$.pipe(
  map((response: ApiResponse<User[]>) => response.data)
).subscribe(users => console.log('ผู้ใช้:', users));
```

## 66.3 Subject และ Variants

### Subject พื้นฐาน

```typescript
import { Subject } from 'rxjs';

// Subject ทำหน้าที่ทั้ง Observable และ Observer
const subject$: Subject<string> = new Subject<string>();

// Subscriber 1
subject$.subscribe(value => console.log('Subscriber 1:', value));

// Subscriber 2
subject$.subscribe(value => console.log('Subscriber 2:', value));

// emit ค่า
subject$.next('สวัสดี');
subject$.next('TypeScript');
subject$.complete();

// ผลลัพธ์:
// Subscriber 1: สวัสดี
// Subscriber 2: สวัสดี
// Subscriber 1: TypeScript
// Subscriber 2: TypeScript
```

### BehaviorSubject

```typescript
import { BehaviorSubject } from 'rxjs';

// BehaviorSubject เก็บค่าล่าสุดและส่งให้ subscriber ใหม่ทันที
const currentUser$: BehaviorSubject<User | null> = new BehaviorSubject<User | null>(null);

// ดูค่าปัจจุบัน
console.log('ผู้ใช้ปัจจุบัน:', currentUser$.getValue()); // null

// เปลี่ยนค่า
currentUser$.next({ id: 1, name: 'สมชาย', email: 'somchai@example.com' });

// Subscriber ใหม่จะได้ค่าล่าสุดทันที
currentUser$.subscribe(user => {
  if (user) {
    console.log('ผู้ใช้ที่ login:', user.name);
  } else {
    console.log('ยังไม่ได้ login');
  }
});

// State management ด้วย BehaviorSubject
interface AppState {
  loading: boolean;
  users: User[];
  error: string | null;
}

const initialState: AppState = {
  loading: false,
  users: [],
  error: null
};

class StateService {
  private state$ = new BehaviorSubject<AppState>(initialState);
  
  getState(): Observable<AppState> {
    return this.state$.asObservable();
  }
  
  setLoading(loading: boolean): void {
    this.state$.next({ ...this.state$.getValue(), loading });
  }
  
  setUsers(users: User[]): void {
    this.state$.next({ ...this.state$.getValue(), users, loading: false });
  }
  
  setError(error: string): void {
    this.state$.next({ ...this.state$.getValue(), error, loading: false });
  }
}

const stateService = new StateService();
stateService.getState().subscribe(state => {
  console.log('State:', state);
});
```

### ReplaySubject

```typescript
import { ReplaySubject } from 'rxjs';

// ReplaySubject จำค่าที่ผ่านมาและส่งให้ subscriber ใหม่
const replay$ = new ReplaySubject<number>(3); // จำ 3 ค่าล่าสุด

replay$.next(1);
replay$.next(2);
replay$.next(3);
replay$.next(4);
replay$.next(5);

// Subscriber ใหม่จะได้รับค่า 3, 4, 5 (3 ค่าล่าสุด)
replay$.subscribe(value => console.log('Replay:', value));

// ReplaySubject พร้อม window time
const replayWithTime$ = new ReplaySubject<string>(10, 2000); // จำ 10 ค่า ภายใน 2 วินาที

// ใช้สำหรับ logging หรือ history
class ActivityLog {
  private log$ = new ReplaySubject<string>(100); // จำ 100 actions ล่าสุด
  
  logAction(action: string): void {
    const timestamp = new Date().toISOString();
    this.log$.next(`[${timestamp}] ${action}`);
  }
  
  getRecentHistory(): Observable<string> {
    return this.log$.asObservable();
  }
}

const activityLog = new ActivityLog();
activityLog.logAction('เข้าสู่ระบบ');
activityLog.logAction('ดูรายการสินค้า');
activityLog.logAction('เพิ่มสินค้าในตะกร้า');

// ผู้ใช้ใหม่ดูประวัติย้อนหลัง
activityLog.getRecentHistory().subscribe(log => console.log(log));
```

### AsyncSubject

```typescript
import { AsyncSubject } from 'rxjs';

// AsyncSubject ส่งเฉพาะค่าสุดท้ายเมื่อ complete
const async$ = new AsyncSubject<number>();

async$.subscribe(value => console.log('Async:', value));

async$.next(1);
async$.next(2);
async$.next(3); // ค่านี้จะถูกส่งเมื่อ complete

async$.complete(); // ส่งค่า 3 ให้ subscribers

// ผลลัพธ์: Async: 3

// ใช้กับ HTTP requests ที่ต้องการเฉพาะผลลัพธ์สุดท้าย
class CacheService {
  private cache = new Map<string, AsyncSubject<any>>();
  
  get<T>(key: string, fetchFn: () => Promise<T>): Observable<T> {
    if (!this.cache.has(key)) {
      const subject = new AsyncSubject<T>();
      this.cache.set(key, subject);
      
      fetchFn().then(data => {
        subject.next(data);
        subject.complete();
      }).catch(err => {
        subject.error(err);
      });
    }
    
    return this.cache.get(key)!.asObservable();
  }
}
```

## 66.4 Operators

### Transformation Operators

```typescript
import { of, from, Observable } from 'rxjs';
import { map, flatMap, mergeMap, switchMap, concatMap, exhaustMap } from 'rxjs/operators';

// map - แปลงค่า
const doubled$ = of(1, 2, 3).pipe(
  map((n: number) => n * 2)
);
doubled$.subscribe(n => console.log(n)); // 2, 4, 6

// แปลงชนิดข้อมูล
interface Product {
  id: number;
  name: string;
  price: number;
}

interface ProductDisplay {
  id: number;
  displayName: string;
  formattedPrice: string;
}

const products$: Observable<Product[]> = of([
  { id: 1, name: 'คอมพิวเตอร์', price: 25000 },
  { id: 2, name: 'โทรศัพท์', price: 15000 }
]);

const displayProducts$: Observable<ProductDisplay[]> = products$.pipe(
  map((products: Product[]) => products.map(p => ({
    id: p.id,
    displayName: `สินค้า: ${p.name}`,
    formattedPrice: `฿${p.price.toLocaleString()}`
  })))
);

displayProducts$.subscribe(products => console.log(products));
```

### mergeMap และ switchMap

```typescript
import { of, interval, Observable } from 'rxjs';
import { mergeMap, switchMap, take, delay } from 'rxjs/operators';

// mergeMap - รวม inner observables แบบ parallel
const userIds$ = of(1, 2, 3);

function getUser(id: number): Observable<User> {
  return of({ id, name: `User ${id}`, email: `user${id}@example.com` }).pipe(
    delay(Math.random() * 1000) // จำลอง API delay
  );
}

// mergeMap รัน requests พร้อมกัน
userIds$.pipe(
  mergeMap((id: number) => getUser(id))
).subscribe(user => console.log('ผู้ใช้:', user));

// switchMap - ยกเลิก previous inner observable เมื่อมีค่าใหม่
const searchTerm$: Subject<string> = new Subject<string>();

function searchUsers(term: string): Observable<User[]> {
  return of([{ id: 1, name: term, email: `${term}@example.com` }]).pipe(
    delay(300)
  );
}

// switchMap เหมาะกับ search - ยกเลิก request เก่าเมื่อพิมพ์ใหม่
const searchResults$ = searchTerm$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap((term: string) => searchUsers(term))
);

searchResults$.subscribe(users => console.log('ผลการค้นหา:', users));

// ทดสอบ
searchTerm$.next('ส');
searchTerm$.next('สม');
searchTerm$.next('สมชาย'); // request นี้จะถูกส่งเพราะ debounce
```

### concatMap และ exhaustMap

```typescript
import { of, Subject } from 'rxjs';
import { concatMap, exhaustMap, delay } from 'rxjs/operators';

// concatMap - รอให้ previous inner observable เสร็จก่อน
const orderIds$ = of(1, 2, 3);

function processOrder(id: number): Observable<string> {
  return of(`Order ${id} processed`).pipe(delay(1000));
}

// concatMap รับประกัน order ของ requests
orderIds$.pipe(
  concatMap((id: number) => processOrder(id))
).subscribe(result => console.log(result));

// exhaustMap - ไม่รับ requests ใหม่จนกว่า current จะเสร็จ
const loginButton$: Subject<void> = new Subject<void>();

function performLogin(): Observable<string> {
  return of('Login successful').pipe(delay(2000));
}

// exhaustMap เหมาะกับ login button - ป้องกันการ click หลายครั้ง
loginButton$.pipe(
  exhaustMap(() => performLogin())
).subscribe(result => console.log(result));
```

### Filtering Operators

```typescript
import { of, interval, Subject } from 'rxjs';
import { filter, take, takeUntil, skip, skipWhile, takeWhile, 
         distinctUntilChanged, debounceTime, throttleTime, first, last } from 'rxjs/operators';

// filter - กรองข้อมูล
const numbers$ = of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// เลขคู่เท่านั้น
const evens$ = numbers$.pipe(
  filter((n: number) => n % 2 === 0)
);
evens$.subscribe(n => console.log('เลขคู่:', n));

// filter กับ object types
interface Order {
  id: number;
  status: 'pending' | 'processing' | 'completed' | 'cancelled';
  total: number;
}

const orders$: Observable<Order> = of(
  { id: 1, status: 'pending', total: 1500 },
  { id: 2, status: 'completed', total: 2500 },
  { id: 3, status: 'cancelled', total: 500 },
  { id: 4, status: 'completed', total: 3000 }
);

// กรองเฉพาะ completed orders
const completedOrders$ = orders$.pipe(
  filter((order: Order): order is Order => order.status === 'completed')
);

completedOrders$.subscribe(order => console.log('คำสั่งซื้อที่เสร็จ:', order));

// take - รับแค่ N ค่าแรก
const first3$ = interval(1000).pipe(take(3));
first3$.subscribe(n => console.log('ค่า:', n));

// takeUntil - หยุดเมื่อ observable อื่น emit
const stopSignal$: Subject<void> = new Subject<void>();
const timer$ = interval(500).pipe(takeUntil(stopSignal$));

timer$.subscribe(n => console.log('timer:', n));

setTimeout(() => {
  stopSignal$.next();
  console.log('หยุด timer แล้ว');
}, 2500);

// debounceTime - รอให้หยุดก่อน
const input$: Subject<string> = new Subject<string>();
const debouncedInput$ = input$.pipe(
  debounceTime(500),
  distinctUntilChanged()
);

debouncedInput$.subscribe(value => console.log('Input:', value));
```

### Combination Operators

```typescript
import { merge, concat, zip, combineLatest, forkJoin, of, interval } from 'rxjs';
import { take, map, delay } from 'rxjs/operators';

// merge - รวม observables แบบ parallel
const stream1$ = of(1, 2, 3);
const stream2$ = of('a', 'b', 'c');

// ต้องระบุชนิดข้อมูลที่รวมกัน
const merged$: Observable<number | string> = merge(stream1$, stream2$);
merged$.subscribe(v => console.log('Merged:', v));

// combineLatest - รวมค่าล่าสุดจากทุก observables
interface UserSettings {
  theme: string;
  language: string;
}

const theme$: BehaviorSubject<string> = new BehaviorSubject<string>('light');
const language$: BehaviorSubject<string> = new BehaviorSubject<string>('th');

const settings$: Observable<UserSettings> = combineLatest([theme$, language$]).pipe(
  map(([theme, language]: [string, string]) => ({ theme, language }))
);

settings$.subscribe(settings => console.log('การตั้งค่า:', settings));

theme$.next('dark'); // จะ trigger settings$ ด้วย
language$.next('en'); // จะ trigger settings$ ด้วย

// forkJoin - รอให้ทุก observables เสร็จ
function fetchUser(id: number): Observable<User> {
  return of({ id, name: `User ${id}`, email: `user${id}@example.com` }).pipe(delay(500));
}

function fetchUserOrders(userId: number): Observable<Order[]> {
  return of([{ id: 1, status: 'completed' as const, total: 1500 }]).pipe(delay(300));
}

// รอให้ทั้ง user และ orders ดึงมาแล้วค่อย process
forkJoin({
  user: fetchUser(1),
  orders: fetchUserOrders(1)
}).subscribe(({ user, orders }) => {
  console.log(`${user.name} มี ${orders.length} คำสั่งซื้อ`);
});

// zip - จับคู่ค่าตำแหน่งเดียวกัน
const names$ = of('สมชาย', 'สมหญิง', 'สมศักดิ์');
const ages$ = of(25, 30, 35);

const people$: Observable<[string, number]> = zip(names$, ages$);
people$.subscribe(([name, age]) => console.log(`${name}: ${age} ปี`));
```

## 66.5 การสร้าง Observable

### Observable Factories

```typescript
import { Observable, of, from, interval, timer, range, EMPTY, NEVER, throwError } from 'rxjs';

// of - สร้างจากค่าโดยตรง
const of$ = of<number>(1, 2, 3);

// from - สร้างจาก array, Promise, iterable
const fromArray$ = from<number>([1, 2, 3, 4, 5]);
const fromPromise$ = from<string>(Promise.resolve('Hello from Promise'));
const fromIterable$ = from<number>(new Set([1, 2, 3, 2, 1])); // จะได้ 1, 2, 3

// interval - emit ทุก N milliseconds
const everySecond$ = interval(1000); // Observable<number>

// timer - emit หลังจาก N milliseconds (optionally ทุก N ms)
const afterDelay$ = timer(2000); // emit หลัง 2 วินาที
const withInterval$ = timer(2000, 1000); // รอ 2 วินาที แล้ว emit ทุก 1 วินาที

// range
const range$ = range(1, 10); // 1 ถึง 10

// EMPTY - ไม่ emit อะไรเลย แล้ว complete ทันที
const empty$ = EMPTY;

// NEVER - ไม่ emit และไม่ complete
const never$ = NEVER;

// throwError - emit error ทันที
const error$ = throwError(() => new Error('เกิดข้อผิดพลาด'));

// Custom Observable Factory
function createHttpGet<T>(url: string): Observable<T> {
  return new Observable<T>(subscriber => {
    const controller = new AbortController();
    
    fetch(url, { signal: controller.signal })
      .then(response => {
        if (!response.ok) {
          throw new Error(`HTTP error: ${response.status}`);
        }
        return response.json();
      })
      .then(data => {
        subscriber.next(data as T);
        subscriber.complete();
      })
      .catch(error => {
        subscriber.error(error);
      });
    
    // Cleanup - ยกเลิก request เมื่อ unsubscribe
    return () => controller.abort();
  });
}

interface Post {
  id: number;
  title: string;
  body: string;
}

const posts$ = createHttpGet<Post[]>('https://jsonplaceholder.typicode.com/posts');
posts$.subscribe({
  next: posts => console.log('Posts:', posts.length),
  error: err => console.error('Error:', err)
});
```

### การสร้างจาก Events

```typescript
import { fromEvent, Observable } from 'rxjs';
import { map, debounceTime, distinctUntilChanged } from 'rxjs/operators';

// fromEvent - สร้างจาก DOM events
// (ในสภาพแวดล้อม browser)
function createSearchObservable(inputElement: HTMLInputElement): Observable<string> {
  return fromEvent<InputEvent>(inputElement, 'input').pipe(
    map((event: InputEvent) => (event.target as HTMLInputElement).value),
    debounceTime(300),
    distinctUntilChanged()
  );
}

// จำลองในสภาพแวดล้อม Node.js
import { EventEmitter } from 'events';

function fromNodeEvent<T>(emitter: EventEmitter, eventName: string): Observable<T> {
  return new Observable<T>(subscriber => {
    const handler = (data: T) => subscriber.next(data);
    const errorHandler = (error: Error) => subscriber.error(error);
    
    emitter.on(eventName, handler);
    emitter.on('error', errorHandler);
    
    return () => {
      emitter.off(eventName, handler);
      emitter.off('error', errorHandler);
    };
  });
}

const emitter = new EventEmitter();
const messages$ = fromNodeEvent<string>(emitter, 'message');

messages$.subscribe(msg => console.log('ข้อความ:', msg));

emitter.emit('message', 'สวัสดี TypeScript!');
emitter.emit('message', 'Hello RxJS!');
```

## 66.6 การจัดการ Error

### catchError

```typescript
import { Observable, of, throwError } from 'rxjs';
import { catchError, retry, retryWhen, delay, take } from 'rxjs/operators';

function fetchData(shouldFail: boolean): Observable<string> {
  if (shouldFail) {
    return throwError(() => new Error('ดึงข้อมูลไม่สำเร็จ'));
  }
  return of('ข้อมูลที่ดึงมาสำเร็จ');
}

// catchError - จัดการ error และ return observable อื่น
fetchData(true).pipe(
  catchError((error: Error) => {
    console.error('จัดการ error:', error.message);
    return of('ข้อมูล default'); // return fallback value
  })
).subscribe(data => console.log(data));

// retry - ลองใหม่ N ครั้ง
let attempts = 0;

function unreliableRequest(): Observable<string> {
  return new Observable<string>(subscriber => {
    attempts++;
    console.log(`พยายามครั้งที่ ${attempts}`);
    
    if (attempts < 3) {
      subscriber.error(new Error('Connection failed'));
    } else {
      subscriber.next('สำเร็จ!');
      subscriber.complete();
    }
  });
}

unreliableRequest().pipe(
  retry(3) // ลองใหม่สูงสุด 3 ครั้ง
).subscribe({
  next: result => console.log(result),
  error: err => console.error('ล้มเหลวหมดแล้ว:', err.message)
});

// Custom retry with backoff
function retryWithBackoff<T>(
  source$: Observable<T>, 
  maxRetries: number = 3,
  backoffMs: number = 1000
): Observable<T> {
  return source$.pipe(
    retryWhen(errors$ => 
      errors$.pipe(
        map((error, index) => {
          if (index >= maxRetries) {
            throw error;
          }
          const delayMs = backoffMs * Math.pow(2, index);
          console.log(`ลองใหม่หลังจาก ${delayMs}ms...`);
          return delayMs;
        }),
        mergeMap(ms => timer(ms))
      )
    )
  );
}
```

### finalize และ tap

```typescript
import { of, throwError } from 'rxjs';
import { finalize, tap, catchError } from 'rxjs/operators';

// tap - ทำ side effects โดยไม่เปลี่ยนค่า
const withLogging$ = of(1, 2, 3).pipe(
  tap(value => console.log('ก่อน map:', value)),
  map(n => n * 2),
  tap(value => console.log('หลัง map:', value))
);

withLogging$.subscribe(n => console.log('ค่าสุดท้าย:', n));

// finalize - ทำงานเมื่อ observable เสร็จหรือ error
function loadingExample(): Observable<string> {
  return of('ข้อมูล').pipe(
    tap(() => console.log('เริ่มโหลด...')),
    delay(1000),
    finalize(() => console.log('หยุดโหลด (ไม่ว่าจะสำเร็จหรือไม่)'))
  );
}

loadingExample().subscribe({
  next: data => console.log('ได้รับ:', data),
  complete: () => console.log('เสร็จ')
});
```

## 66.7 Hot vs Cold Observables

### Cold Observables

```typescript
import { Observable } from 'rxjs';

// Cold Observable - เริ่มใหม่สำหรับแต่ละ subscriber
const coldObservable$ = new Observable<number>(subscriber => {
  console.log('เริ่ม cold observable');
  subscriber.next(Math.random()); // ค่าต่างกันสำหรับแต่ละ subscriber
  subscriber.next(Math.random());
  subscriber.complete();
});

console.log('Subscriber 1:');
coldObservable$.subscribe(n => console.log(n)); // ค่า random ของ subscriber 1

console.log('Subscriber 2:');
coldObservable$.subscribe(n => console.log(n)); // ค่า random ต่างกัน

// HTTP requests ก็เป็น cold observable
// แต่ละ subscriber จะส่ง request ใหม่
```

### Hot Observables

```typescript
import { Subject, share, shareReplay, publish, refCount } from 'rxjs';
import { tap } from 'rxjs/operators';

// Hot Observable - แชร์ stream เดิมกับทุก subscribers
const hotSubject$ = new Subject<number>();

// ทั้งสอง subscriber ได้รับค่าเดียวกัน
hotSubject$.subscribe(n => console.log('Hot Subscriber 1:', n));
hotSubject$.subscribe(n => console.log('Hot Subscriber 2:', n));

hotSubject$.next(1); // ทั้งสองจะได้รับ 1
hotSubject$.next(2); // ทั้งสองจะได้รับ 2

// แปลง Cold เป็น Hot ด้วย share
const sharedCold$ = coldObservable$.pipe(
  tap(() => console.log('โค้ดนี้รันแค่ครั้งเดียว')),
  share() // แชร์กัน
);

sharedCold$.subscribe(n => console.log('Shared 1:', n));
sharedCold$.subscribe(n => console.log('Shared 2:', n));

// shareReplay - แชร์และ replay ค่าสุดท้ายให้ subscribers ใหม่
const sharedReplayed$ = coldObservable$.pipe(
  shareReplay(1) // replay 1 ค่าล่าสุด
);

sharedReplayed$.subscribe(n => console.log('Replayed 1:', n));

setTimeout(() => {
  // subscriber ใหม่ได้รับค่าล่าสุดที่ replay
  sharedReplayed$.subscribe(n => console.log('Replayed 2 (late):', n));
}, 2000);
```

## 66.8 Subjects เป็น Event Bus

### Event Bus Pattern

```typescript
import { Subject, Observable } from 'rxjs';
import { filter, map } from 'rxjs/operators';

// Event types
interface AppEvent {
  type: string;
  payload: any;
}

interface UserLoggedInEvent extends AppEvent {
  type: 'USER_LOGGED_IN';
  payload: User;
}

interface UserLoggedOutEvent extends AppEvent {
  type: 'USER_LOGGED_OUT';
  payload: { userId: number };
}

interface OrderCreatedEvent extends AppEvent {
  type: 'ORDER_CREATED';
  payload: Order;
}

type DomainEvent = UserLoggedInEvent | UserLoggedOutEvent | OrderCreatedEvent;

// Event Bus Service
class EventBus {
  private eventSubject$ = new Subject<DomainEvent>();
  
  emit<T extends DomainEvent>(event: T): void {
    this.eventSubject$.next(event);
  }
  
  on<T extends DomainEvent>(eventType: T['type']): Observable<T> {
    return this.eventSubject$.pipe(
      filter((event): event is T => event.type === eventType)
    );
  }
  
  // รับ events ทั้งหมด
  all(): Observable<DomainEvent> {
    return this.eventSubject$.asObservable();
  }
}

// การใช้งาน
const eventBus = new EventBus();

// Subscribe to specific events
eventBus.on<UserLoggedInEvent>('USER_LOGGED_IN').subscribe(event => {
  console.log(`${event.payload.name} เข้าสู่ระบบแล้ว`);
});

eventBus.on<OrderCreatedEvent>('ORDER_CREATED').subscribe(event => {
  console.log(`คำสั่งซื้อใหม่: ${event.payload.id}, ยอด: ${event.payload.total}`);
});

// Emit events
eventBus.emit({
  type: 'USER_LOGGED_IN',
  payload: { id: 1, name: 'สมชาย', email: 'somchai@example.com' }
});

eventBus.emit({
  type: 'ORDER_CREATED',
  payload: { id: 100, status: 'pending', total: 2500 }
});
```

### Store Pattern

```typescript
import { BehaviorSubject, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

// Generic Store
class Store<T extends object> {
  private state$: BehaviorSubject<T>;
  
  constructor(initialState: T) {
    this.state$ = new BehaviorSubject<T>(initialState);
  }
  
  getState(): T {
    return this.state$.getValue();
  }
  
  select<K>(selector: (state: T) => K): Observable<K> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()
    );
  }
  
  setState(newState: Partial<T>): void {
    this.state$.next({ ...this.state$.getValue(), ...newState });
  }
  
  reduce(reducer: (state: T) => T): void {
    this.state$.next(reducer(this.state$.getValue()));
  }
}

// ใช้งาน
interface CartState {
  items: CartItem[];
  total: number;
  loading: boolean;
}

interface CartItem {
  productId: number;
  name: string;
  price: number;
  quantity: number;
}

const cartStore = new Store<CartState>({
  items: [],
  total: 0,
  loading: false
});

// Select specific parts of state
const cartItems$ = cartStore.select(state => state.items);
const cartTotal$ = cartStore.select(state => state.total);

cartItems$.subscribe(items => console.log('รายการ:', items.length, 'ชิ้น'));
cartTotal$.subscribe(total => console.log('ยอดรวม:', total));

// Update state
cartStore.reduce(state => ({
  ...state,
  items: [...state.items, { productId: 1, name: 'สินค้า A', price: 100, quantity: 1 }],
  total: state.total + 100
}));
```

## 66.9 Real-World Patterns

### HTTP Polling

```typescript
import { interval, Observable, Subject } from 'rxjs';
import { switchMap, takeUntil, share } from 'rxjs/operators';

function createPollingObservable<T>(
  fetchFn: () => Observable<T>,
  intervalMs: number = 5000
): Observable<T> {
  return interval(intervalMs).pipe(
    switchMap(() => fetchFn()),
    share() // แชร์กับทุก subscribers
  );
}

// ใช้กับ API polling
function fetchCurrentPrice(symbol: string): Observable<number> {
  return of(Math.random() * 100 + 50).pipe(delay(200));
}

const stopPolling$ = new Subject<void>();

const price$ = createPollingObservable(
  () => fetchCurrentPrice('BTC'),
  3000
).pipe(takeUntil(stopPolling$));

price$.subscribe(price => console.log(`ราคา: $${price.toFixed(2)}`));

// หยุด polling หลัง 15 วินาที
setTimeout(() => {
  stopPolling$.next();
  console.log('หยุด polling แล้ว');
}, 15000);
```

### Optimistic Updates

```typescript
import { Subject, Observable, of } from 'rxjs';
import { switchMap, catchError, map } from 'rxjs/operators';

interface TodoItem {
  id: number;
  text: string;
  completed: boolean;
}

class TodoService {
  private todos$ = new BehaviorSubject<TodoItem[]>([]);
  
  getTodos(): Observable<TodoItem[]> {
    return this.todos$.asObservable();
  }
  
  toggleTodo(id: number): Observable<TodoItem[]> {
    const currentTodos = this.todos$.getValue();
    
    // Optimistic update - update UI ก่อน
    const optimisticTodos = currentTodos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    );
    this.todos$.next(optimisticTodos);
    
    // ส่ง request ไป server
    return this.saveToServer(id).pipe(
      map(() => optimisticTodos), // สำเร็จ
      catchError(error => {
        // Rollback เมื่อ error
        console.error('เกิดข้อผิดพลาด, rollback:', error);
        this.todos$.next(currentTodos);
        return of(currentTodos);
      })
    );
  }
  
  private saveToServer(id: number): Observable<void> {
    return of(undefined).pipe(delay(1000));
  }
}
```

### WebSocket Integration

```typescript
import { Observable, Subject, BehaviorSubject } from 'rxjs';
import { retry, delay, filter } from 'rxjs/operators';

interface WebSocketMessage<T = any> {
  type: string;
  data: T;
  timestamp: number;
}

class WebSocketService {
  private socket: WebSocket | null = null;
  private messageSubject$ = new Subject<WebSocketMessage>();
  private connectionStatus$ = new BehaviorSubject<boolean>(false);
  
  connect(url: string): Observable<boolean> {
    this.socket = new WebSocket(url);
    
    return new Observable<boolean>(subscriber => {
      this.socket!.onopen = () => {
        console.log('WebSocket connected');
        this.connectionStatus$.next(true);
        subscriber.next(true);
      };
      
      this.socket!.onmessage = (event: MessageEvent) => {
        try {
          const message: WebSocketMessage = JSON.parse(event.data);
          this.messageSubject$.next(message);
        } catch (error) {
          console.error('Parse error:', error);
        }
      };
      
      this.socket!.onerror = (error) => {
        subscriber.error(error);
      };
      
      this.socket!.onclose = () => {
        this.connectionStatus$.next(false);
        subscriber.complete();
      };
      
      return () => this.socket?.close();
    });
  }
  
  send<T>(type: string, data: T): void {
    if (this.socket?.readyState === WebSocket.OPEN) {
      this.socket.send(JSON.stringify({
        type,
        data,
        timestamp: Date.now()
      }));
    }
  }
  
  on<T>(messageType: string): Observable<T> {
    return this.messageSubject$.pipe(
      filter(msg => msg.type === messageType),
      map(msg => msg.data as T)
    );
  }
  
  isConnected(): Observable<boolean> {
    return this.connectionStatus$.asObservable();
  }
}
```

## 66.10 การใช้ Angular

### Angular Service with RxJS

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject, throwError } from 'rxjs';
import { map, catchError, tap, shareReplay } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private apiUrl = 'https://api.example.com';
  private currentUser$ = new BehaviorSubject<User | null>(null);
  
  constructor(private http: HttpClient) {}
  
  // HTTP request พร้อม typing
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`).pipe(
      catchError(error => {
        console.error('Error fetching users:', error);
        return throwError(() => error);
      })
    );
  }
  
  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/users/${id}`).pipe(
      shareReplay(1) // cache ผลลัพธ์
    );
  }
  
  login(credentials: { email: string; password: string }): Observable<User> {
    return this.http.post<User>(`${this.apiUrl}/auth/login`, credentials).pipe(
      tap(user => {
        this.currentUser$.next(user);
        localStorage.setItem('user', JSON.stringify(user));
      })
    );
  }
  
  getCurrentUser(): Observable<User | null> {
    return this.currentUser$.asObservable();
  }
  
  logout(): void {
    this.currentUser$.next(null);
    localStorage.removeItem('user');
  }
}
```

### Angular Component

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject, Observable, combineLatest } from 'rxjs';
import { takeUntil, switchMap, startWith } from 'rxjs/operators';
import { FormControl } from '@angular/forms';

@Component({
  selector: 'app-users',
  template: `
    <input [formControl]="searchControl" placeholder="ค้นหาผู้ใช้">
    <div *ngFor="let user of filteredUsers$ | async">
      {{ user.name }}
    </div>
  `
})
export class UsersComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  searchControl = new FormControl('');
  filteredUsers$!: Observable<User[]>;
  
  constructor(private userService: UserService) {}
  
  ngOnInit(): void {
    const search$ = this.searchControl.valueChanges.pipe(
      startWith(''),
      debounceTime(300),
      distinctUntilChanged()
    );
    
    this.filteredUsers$ = combineLatest([
      this.userService.getUsers(),
      search$
    ]).pipe(
      map(([users, search]: [User[], string | null]) => {
        if (!search) return users;
        return users.filter(u => 
          u.name.toLowerCase().includes(search.toLowerCase())
        );
      }),
      takeUntil(this.destroy$)
    );
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 66.11 การทดสอบด้วย Marble Diagrams

### Marble Testing

```typescript
import { TestScheduler } from 'rxjs/testing';
import { map, filter, debounceTime } from 'rxjs/operators';

describe('RxJS Marble Tests', () => {
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should map values correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c--|', { a: 1, b: 2, c: 3 });
      const expected =    '--x--y--z--|';
      
      const result$ = source$.pipe(map(n => n * 2));
      
      expectObservable(result$).toBe(expected, { x: 2, y: 4, z: 6 });
    });
  });
  
  it('should filter values correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--1--2--3--4--|', {
        '1': 1, '2': 2, '3': 3, '4': 4
      });
      const expected =    '-----x-----y--|';
      
      const result$ = source$.pipe(filter(n => n % 2 === 0));
      
      expectObservable(result$).toBe(expected, { x: 2, y: 4 });
    });
  });
  
  it('should debounce correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a-b-c--------d---|', {
        a: 'a', b: 'b', c: 'c', d: 'd'
      });
      const expected =    '-------------c-----d---|';
      
      const result$ = source$.pipe(debounceTime(5, testScheduler));
      
      expectObservable(result$).toBe(expected, { c: 'c', d: 'd' });
    });
  });
  
  it('should handle errors', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--#', { a: 1 }, new Error('test error'));
      const expected =    '--a--(b|)';
      
      const result$ = source$.pipe(
        catchError(() => of(99))
      );
      
      expectObservable(result$).toBe(expected, { a: 1, b: 99 });
    });
  });
});
```

## 66.12 Advanced Patterns

### Custom Operators

```typescript
import { Observable, OperatorFunction, MonoTypeOperatorFunction } from 'rxjs';
import { pipe, tap, catchError, of, retry } from 'rxjs';
import { map } from 'rxjs/operators';

// Custom operator สำหรับ logging
function logWithTimestamp<T>(label: string): MonoTypeOperatorFunction<T> {
  return (source$: Observable<T>): Observable<T> => {
    return source$.pipe(
      tap(value => {
        console.log(`[${new Date().toISOString()}] ${label}:`, value);
      })
    );
  };
}

// Custom operator สำหรับ retry with backoff
function retryWithExponentialBackoff<T>(
  maxRetries: number = 3,
  baseDelay: number = 1000
): MonoTypeOperatorFunction<T> {
  return (source$: Observable<T>): Observable<T> => {
    return source$.pipe(
      retryWhen(errors$ =>
        errors$.pipe(
          mergeMap((error, index) => {
            if (index >= maxRetries) {
              return throwError(() => error);
            }
            const delay = baseDelay * Math.pow(2, index);
            console.log(`ลองใหม่ครั้งที่ ${index + 1} หลัง ${delay}ms`);
            return timer(delay);
          })
        )
      )
    );
  };
}

// Custom operator สำหรับ pagination
function paginate<T>(pageSize: number): OperatorFunction<T[], T[]> {
  return (source$: Observable<T[]>): Observable<T[]> => {
    return source$.pipe(
      map((items: T[]) => items.slice(0, pageSize))
    );
  };
}

// การใช้งาน custom operators
const dataStream$ = fetchData(false).pipe(
  logWithTimestamp<string>('fetchData'),
  retryWithExponentialBackoff<string>(3, 500)
);

const pagedUsers$ = getUsers().pipe(
  paginate<User>(10)
);
```

### Resource Management

```typescript
import { Observable, using } from 'rxjs';

// using - จัดการ resources อัตโนมัติ
function createDatabaseConnection(): { query: (sql: string) => Observable<any>; close: () => void } {
  console.log('เปิด database connection');
  
  return {
    query: (sql: string) => of([{ id: 1, result: sql }]),
    close: () => console.log('ปิด database connection')
  };
}

const dbStream$ = using(
  () => createDatabaseConnection(),
  (connection) => connection.query('SELECT * FROM users')
);

dbStream$.subscribe({
  next: results => console.log('ผลลัพธ์:', results),
  complete: () => console.log('เสร็จ - connection จะถูกปิดอัตโนมัติ')
});
```

## สรุป

RxJS กับ TypeScript เป็นการรวมกันที่ทรงพลังมาก โดย:

1. **Observable types** ช่วยให้ทราบชนิดข้อมูลที่ stream ส่งมา
2. **Subject variants** เหมาะกับงานต่างๆ: BehaviorSubject สำหรับ state, ReplaySubject สำหรับ history
3. **Operators** ช่วยแปลงและจัดการข้อมูลอย่างชัดเจน
4. **Error handling** ด้วย catchError, retry ทำให้โค้ดแข็งแกร่ง
5. **Hot/Cold observables** เข้าใจได้ช่วยจัดการ resources ได้ดีขึ้น
6. **Custom operators** ช่วยให้โค้ด reusable และอ่านง่าย
7. **Marble testing** ช่วยทดสอบ async code ได้อย่างมีประสิทธิภาพ

การเขียน RxJS กับ TypeScript อย่างถูกต้องจะทำให้แอปพลิเคชันของเราสามารถจัดการกับ async operations, user events, และ data streams ได้อย่างมีประสิทธิภาพและปลอดภัย
