# ส่วนที่ 71: โครงสร้างข้อมูลและอัลกอริทึมใน TypeScript

## บทนำ

โครงสร้างข้อมูล (Data Structures) และอัลกอริทึม (Algorithms) เป็นรากฐานสำคัญของการเขียนโปรแกรม การเข้าใจและนำไปใช้ใน TypeScript จะช่วยให้เราสร้างโปรแกรมที่มีประสิทธิภาพและบำรุงรักษาได้ง่าย

---

## 1. Stack (สแตก)

Stack เป็นโครงสร้างข้อมูลแบบ LIFO (Last In, First Out) - ข้อมูลที่เข้าหลังสุดออกก่อน

### 1.1 การ Implement Stack พื้นฐาน

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
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
    return [...this.items];
  }
}

// ตัวอย่างการใช้งาน
const stack = new Stack<number>();
stack.push(1);
stack.push(2);
stack.push(3);
console.log(stack.peek());  // 3
console.log(stack.pop());   // 3
console.log(stack.size());  // 2
```

### 1.2 Stack ด้วย Linked List

```typescript
interface StackNode<T> {
  value: T;
  next: StackNode<T> | null;
}

class LinkedStack<T> {
  private top: StackNode<T> | null = null;
  private _size: number = 0;

  push(value: T): void {
    const node: StackNode<T> = { value, next: this.top };
    this.top = node;
    this._size++;
  }

  pop(): T | undefined {
    if (!this.top) return undefined;
    const value = this.top.value;
    this.top = this.top.next;
    this._size--;
    return value;
  }

  peek(): T | undefined {
    return this.top?.value;
  }

  get size(): number {
    return this._size;
  }

  isEmpty(): boolean {
    return this._size === 0;
  }
}
```

### 1.3 ตัวอย่างการใช้ Stack - ตรวจสอบวงเล็บ

```typescript
function isBalanced(expression: string): boolean {
  const stack = new Stack<string>();
  const pairs: Record<string, string> = {
    ')': '(',
    ']': '[',
    '}': '{'
  };
  const closers = new Set(Object.keys(pairs));
  const openers = new Set(Object.values(pairs));

  for (const char of expression) {
    if (openers.has(char)) {
      stack.push(char);
    } else if (closers.has(char)) {
      if (stack.isEmpty() || stack.pop() !== pairs[char]) {
        return false;
      }
    }
  }

  return stack.isEmpty();
}

console.log(isBalanced("({[]})"));   // true
console.log(isBalanced("({[})"));    // false
console.log(isBalanced("((()))"));   // true
```

### 1.4 Stack สำหรับ Undo/Redo

```typescript
class UndoRedoManager<T> {
  private undoStack = new Stack<T>();
  private redoStack = new Stack<T>();
  private currentState: T;

  constructor(initialState: T) {
    this.currentState = initialState;
  }

  execute(newState: T): void {
    this.undoStack.push(this.currentState);
    this.redoStack.clear();
    this.currentState = newState;
  }

  undo(): T | undefined {
    const previousState = this.undoStack.pop();
    if (previousState !== undefined) {
      this.redoStack.push(this.currentState);
      this.currentState = previousState;
      return this.currentState;
    }
    return undefined;
  }

  redo(): T | undefined {
    const nextState = this.redoStack.pop();
    if (nextState !== undefined) {
      this.undoStack.push(this.currentState);
      this.currentState = nextState;
      return this.currentState;
    }
    return undefined;
  }

  getState(): T {
    return this.currentState;
  }
}

// ตัวอย่าง
const editor = new UndoRedoManager("Hello");
editor.execute("Hello World");
editor.execute("Hello TypeScript");
console.log(editor.getState());  // Hello TypeScript
editor.undo();
console.log(editor.getState());  // Hello World
editor.redo();
console.log(editor.getState());  // Hello TypeScript
```

**Time Complexity:**
- Push: O(1)
- Pop: O(1)
- Peek: O(1)
- Search: O(n)

---

## 2. Queue (คิว)

Queue เป็นโครงสร้างข้อมูลแบบ FIFO (First In, First Out) - ข้อมูลที่เข้าแรกออกก่อน

### 2.1 Queue พื้นฐาน

```typescript
class Queue<T> {
  private items: T[] = [];

  enqueue(item: T): void {
    this.items.push(item);
  }

  dequeue(): T | undefined {
    return this.items.shift();
  }

  front(): T | undefined {
    return this.items[0];
  }

  rear(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }

  toArray(): T[] {
    return [...this.items];
  }
}
```

### 2.2 Queue ด้วย Linked List (ประสิทธิภาพดีกว่า)

```typescript
interface QueueNode<T> {
  value: T;
  next: QueueNode<T> | null;
}

class LinkedQueue<T> {
  private head: QueueNode<T> | null = null;
  private tail: QueueNode<T> | null = null;
  private _size: number = 0;

  enqueue(value: T): void {
    const node: QueueNode<T> = { value, next: null };
    if (this.tail) {
      this.tail.next = node;
    }
    this.tail = node;
    if (!this.head) {
      this.head = node;
    }
    this._size++;
  }

  dequeue(): T | undefined {
    if (!this.head) return undefined;
    const value = this.head.value;
    this.head = this.head.next;
    if (!this.head) {
      this.tail = null;
    }
    this._size--;
    return value;
  }

  front(): T | undefined {
    return this.head?.value;
  }

  get size(): number {
    return this._size;
  }

  isEmpty(): boolean {
    return this._size === 0;
  }
}
```

### 2.3 Circular Queue

```typescript
class CircularQueue<T> {
  private items: (T | undefined)[];
  private head: number = 0;
  private tail: number = 0;
  private _size: number = 0;
  private capacity: number;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.items = new Array(capacity);
  }

  enqueue(item: T): boolean {
    if (this.isFull()) return false;
    this.items[this.tail] = item;
    this.tail = (this.tail + 1) % this.capacity;
    this._size++;
    return true;
  }

  dequeue(): T | undefined {
    if (this.isEmpty()) return undefined;
    const item = this.items[this.head];
    this.items[this.head] = undefined;
    this.head = (this.head + 1) % this.capacity;
    this._size--;
    return item;
  }

  isEmpty(): boolean {
    return this._size === 0;
  }

  isFull(): boolean {
    return this._size === this.capacity;
  }

  get size(): number {
    return this._size;
  }
}
```

### 2.4 Priority Queue

```typescript
interface PriorityItem<T> {
  value: T;
  priority: number;
}

class PriorityQueue<T> {
  private items: PriorityItem<T>[] = [];

  enqueue(value: T, priority: number): void {
    const item: PriorityItem<T> = { value, priority };
    let added = false;

    for (let i = 0; i < this.items.length; i++) {
      if (item.priority > this.items[i].priority) {
        this.items.splice(i, 0, item);
        added = true;
        break;
      }
    }

    if (!added) {
      this.items.push(item);
    }
  }

  dequeue(): T | undefined {
    return this.items.shift()?.value;
  }

  peek(): T | undefined {
    return this.items[0]?.value;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }
}

// ตัวอย่าง
const pq = new PriorityQueue<string>();
pq.enqueue("ภารกิจปกติ", 1);
pq.enqueue("ภารกิจด่วนมาก", 10);
pq.enqueue("ภารกิจด่วน", 5);
console.log(pq.dequeue()); // ภารกิจด่วนมาก
console.log(pq.dequeue()); // ภารกิจด่วน
```

**Time Complexity:**
- Enqueue: O(1) สำหรับ Queue ทั่วไป, O(n) สำหรับ Priority Queue
- Dequeue: O(1) สำหรับ Linked Queue, O(n) สำหรับ Array Queue

---

## 3. Linked List (ลิงก์ลิสต์)

### 3.1 Singly Linked List

```typescript
class SinglyNode<T> {
  constructor(
    public value: T,
    public next: SinglyNode<T> | null = null
  ) {}
}

class SinglyLinkedList<T> {
  private head: SinglyNode<T> | null = null;
  private _size: number = 0;

  append(value: T): void {
    const node = new SinglyNode(value);
    if (!this.head) {
      this.head = node;
    } else {
      let current = this.head;
      while (current.next) {
        current = current.next;
      }
      current.next = node;
    }
    this._size++;
  }

  prepend(value: T): void {
    const node = new SinglyNode(value, this.head);
    this.head = node;
    this._size++;
  }

  insertAt(index: number, value: T): boolean {
    if (index < 0 || index > this._size) return false;
    if (index === 0) {
      this.prepend(value);
      return true;
    }

    const node = new SinglyNode(value);
    let current = this.head!;
    for (let i = 0; i < index - 1; i++) {
      current = current.next!;
    }
    node.next = current.next;
    current.next = node;
    this._size++;
    return true;
  }

  remove(value: T): boolean {
    if (!this.head) return false;

    if (this.head.value === value) {
      this.head = this.head.next;
      this._size--;
      return true;
    }

    let current = this.head;
    while (current.next) {
      if (current.next.value === value) {
        current.next = current.next.next;
        this._size--;
        return true;
      }
      current = current.next;
    }
    return false;
  }

  removeAt(index: number): T | undefined {
    if (index < 0 || index >= this._size) return undefined;

    if (index === 0) {
      const value = this.head!.value;
      this.head = this.head!.next;
      this._size--;
      return value;
    }

    let current = this.head!;
    for (let i = 0; i < index - 1; i++) {
      current = current.next!;
    }
    const value = current.next!.value;
    current.next = current.next!.next;
    this._size--;
    return value;
  }

  find(value: T): number {
    let current = this.head;
    let index = 0;
    while (current) {
      if (current.value === value) return index;
      current = current.next;
      index++;
    }
    return -1;
  }

  get(index: number): T | undefined {
    if (index < 0 || index >= this._size) return undefined;
    let current = this.head!;
    for (let i = 0; i < index; i++) {
      current = current.next!;
    }
    return current.value;
  }

  reverse(): void {
    let prev: SinglyNode<T> | null = null;
    let current = this.head;
    while (current) {
      const next = current.next;
      current.next = prev;
      prev = current;
      current = next;
    }
    this.head = prev;
  }

  toArray(): T[] {
    const result: T[] = [];
    let current = this.head;
    while (current) {
      result.push(current.value);
      current = current.next;
    }
    return result;
  }

  get size(): number {
    return this._size;
  }
}

// ตัวอย่าง
const list = new SinglyLinkedList<number>();
list.append(1);
list.append(2);
list.append(3);
list.prepend(0);
console.log(list.toArray()); // [0, 1, 2, 3]
list.reverse();
console.log(list.toArray()); // [3, 2, 1, 0]
```

### 3.2 Doubly Linked List

```typescript
class DoublyNode<T> {
  constructor(
    public value: T,
    public prev: DoublyNode<T> | null = null,
    public next: DoublyNode<T> | null = null
  ) {}
}

class DoublyLinkedList<T> {
  private head: DoublyNode<T> | null = null;
  private tail: DoublyNode<T> | null = null;
  private _size: number = 0;

  append(value: T): void {
    const node = new DoublyNode(value, this.tail);
    if (this.tail) {
      this.tail.next = node;
    }
    this.tail = node;
    if (!this.head) {
      this.head = node;
    }
    this._size++;
  }

  prepend(value: T): void {
    const node = new DoublyNode(value, null, this.head);
    if (this.head) {
      this.head.prev = node;
    }
    this.head = node;
    if (!this.tail) {
      this.tail = node;
    }
    this._size++;
  }

  removeFromTail(): T | undefined {
    if (!this.tail) return undefined;
    const value = this.tail.value;
    if (this.tail.prev) {
      this.tail.prev.next = null;
      this.tail = this.tail.prev;
    } else {
      this.head = null;
      this.tail = null;
    }
    this._size--;
    return value;
  }

  removeFromHead(): T | undefined {
    if (!this.head) return undefined;
    const value = this.head.value;
    if (this.head.next) {
      this.head.next.prev = null;
      this.head = this.head.next;
    } else {
      this.head = null;
      this.tail = null;
    }
    this._size--;
    return value;
  }

  toArray(): T[] {
    const result: T[] = [];
    let current = this.head;
    while (current) {
      result.push(current.value);
      current = current.next;
    }
    return result;
  }

  toArrayReverse(): T[] {
    const result: T[] = [];
    let current = this.tail;
    while (current) {
      result.push(current.value);
      current = current.prev;
    }
    return result;
  }

  get size(): number {
    return this._size;
  }
}
```

### 3.3 LRU Cache ด้วย Doubly Linked List + Hash Map

```typescript
class LRUCache<K, V> {
  private capacity: number;
  private map: Map<K, DoublyNode<{ key: K; value: V }>> = new Map();
  private list = new DoublyLinkedList<{ key: K; value: V }>();

  constructor(capacity: number) {
    this.capacity = capacity;
  }

  get(key: K): V | undefined {
    const node = this.map.get(key);
    if (!node) return undefined;
    // ย้ายไปหน้าสุด (MRU position)
    this.list.remove(node.value);
    this.list.prepend(node.value);
    return node.value.value;
  }

  put(key: K, value: V): void {
    if (this.map.has(key)) {
      const node = this.map.get(key)!;
      this.list.remove(node.value);
    } else if (this.map.size >= this.capacity) {
      // ลบ LRU item (tail)
      const lruItem = this.list.removeFromTail();
      if (lruItem) {
        this.map.delete(lruItem.key);
      }
    }
    this.list.prepend({ key, value });
    this.map.set(key, this.list.getHead()!);
  }
}
```

**Time Complexity:**
- Append/Prepend: O(1)
- Remove (with reference): O(1) สำหรับ Doubly, O(n) สำหรับ Singly
- Search: O(n)
- Access by index: O(n)

---

## 4. Binary Tree (ไบนารีทรี)

### 4.1 โครงสร้าง Binary Tree

```typescript
class TreeNode<T> {
  constructor(
    public value: T,
    public left: TreeNode<T> | null = null,
    public right: TreeNode<T> | null = null
  ) {}
}

class BinaryTree<T> {
  root: TreeNode<T> | null = null;

  // Inorder Traversal (Left -> Root -> Right)
  inorder(node: TreeNode<T> | null = this.root, result: T[] = []): T[] {
    if (node) {
      this.inorder(node.left, result);
      result.push(node.value);
      this.inorder(node.right, result);
    }
    return result;
  }

  // Preorder Traversal (Root -> Left -> Right)
  preorder(node: TreeNode<T> | null = this.root, result: T[] = []): T[] {
    if (node) {
      result.push(node.value);
      this.preorder(node.left, result);
      this.preorder(node.right, result);
    }
    return result;
  }

  // Postorder Traversal (Left -> Right -> Root)
  postorder(node: TreeNode<T> | null = this.root, result: T[] = []): T[] {
    if (node) {
      this.postorder(node.left, result);
      this.postorder(node.right, result);
      result.push(node.value);
    }
    return result;
  }

  // Level Order Traversal (BFS)
  levelOrder(): T[][] {
    if (!this.root) return [];
    const result: T[][] = [];
    const queue: TreeNode<T>[] = [this.root];

    while (queue.length > 0) {
      const levelSize = queue.length;
      const level: T[] = [];

      for (let i = 0; i < levelSize; i++) {
        const node = queue.shift()!;
        level.push(node.value);
        if (node.left) queue.push(node.left);
        if (node.right) queue.push(node.right);
      }
      result.push(level);
    }
    return result;
  }

  height(node: TreeNode<T> | null = this.root): number {
    if (!node) return -1;
    return 1 + Math.max(this.height(node.left), this.height(node.right));
  }

  isBalanced(node: TreeNode<T> | null = this.root): boolean {
    if (!node) return true;
    const leftHeight = this.height(node.left);
    const rightHeight = this.height(node.right);
    return (
      Math.abs(leftHeight - rightHeight) <= 1 &&
      this.isBalanced(node.left) &&
      this.isBalanced(node.right)
    );
  }
}
```

---

## 5. Binary Search Tree (BST)

### 5.1 BST Implementation

```typescript
class BST<T> {
  root: TreeNode<T> | null = null;
  private compareFn: (a: T, b: T) => number;

  constructor(compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)) {
    this.compareFn = compareFn;
  }

  insert(value: T): void {
    this.root = this.insertNode(this.root, value);
  }

  private insertNode(node: TreeNode<T> | null, value: T): TreeNode<T> {
    if (!node) return new TreeNode(value);
    const cmp = this.compareFn(value, node.value);
    if (cmp < 0) {
      node.left = this.insertNode(node.left, value);
    } else if (cmp > 0) {
      node.right = this.insertNode(node.right, value);
    }
    return node;
  }

  search(value: T): TreeNode<T> | null {
    return this.searchNode(this.root, value);
  }

  private searchNode(node: TreeNode<T> | null, value: T): TreeNode<T> | null {
    if (!node) return null;
    const cmp = this.compareFn(value, node.value);
    if (cmp === 0) return node;
    if (cmp < 0) return this.searchNode(node.left, value);
    return this.searchNode(node.right, value);
  }

  delete(value: T): void {
    this.root = this.deleteNode(this.root, value);
  }

  private deleteNode(node: TreeNode<T> | null, value: T): TreeNode<T> | null {
    if (!node) return null;
    const cmp = this.compareFn(value, node.value);
    if (cmp < 0) {
      node.left = this.deleteNode(node.left, value);
    } else if (cmp > 0) {
      node.right = this.deleteNode(node.right, value);
    } else {
      // Node to delete found
      if (!node.left) return node.right;
      if (!node.right) return node.left;
      // Node has two children - find inorder successor
      const minRight = this.findMin(node.right)!;
      node.value = minRight.value;
      node.right = this.deleteNode(node.right, minRight.value);
    }
    return node;
  }

  findMin(node: TreeNode<T> | null = this.root): TreeNode<T> | null {
    if (!node) return null;
    while (node.left) node = node.left;
    return node;
  }

  findMax(node: TreeNode<T> | null = this.root): TreeNode<T> | null {
    if (!node) return null;
    while (node.right) node = node.right;
    return node;
  }

  inorder(): T[] {
    const result: T[] = [];
    const traverse = (node: TreeNode<T> | null) => {
      if (node) {
        traverse(node.left);
        result.push(node.value);
        traverse(node.right);
      }
    };
    traverse(this.root);
    return result;
  }
}

// ตัวอย่าง
const bst = new BST<number>();
[5, 3, 7, 1, 4, 6, 8].forEach(n => bst.insert(n));
console.log(bst.inorder()); // [1, 3, 4, 5, 6, 7, 8]
bst.delete(3);
console.log(bst.inorder()); // [1, 4, 5, 6, 7, 8]
```

**Time Complexity:**
- Insert: O(log n) เฉลี่ย, O(n) กรณีแย่สุด
- Search: O(log n) เฉลี่ย, O(n) กรณีแย่สุด
- Delete: O(log n) เฉลี่ย, O(n) กรณีแย่สุด

---

## 6. Heap (ฮีป)

### 6.1 Min Heap

```typescript
class MinHeap<T> {
  private heap: T[] = [];
  private compareFn: (a: T, b: T) => number;

  constructor(compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)) {
    this.compareFn = compareFn;
  }

  private parent(i: number): number { return Math.floor((i - 1) / 2); }
  private left(i: number): number { return 2 * i + 1; }
  private right(i: number): number { return 2 * i + 2; }

  private swap(i: number, j: number): void {
    [this.heap[i], this.heap[j]] = [this.heap[j], this.heap[i]];
  }

  insert(value: T): void {
    this.heap.push(value);
    this.bubbleUp(this.heap.length - 1);
  }

  private bubbleUp(i: number): void {
    while (i > 0) {
      const p = this.parent(i);
      if (this.compareFn(this.heap[i], this.heap[p]) < 0) {
        this.swap(i, p);
        i = p;
      } else {
        break;
      }
    }
  }

  extractMin(): T | undefined {
    if (this.heap.length === 0) return undefined;
    const min = this.heap[0];
    const last = this.heap.pop()!;
    if (this.heap.length > 0) {
      this.heap[0] = last;
      this.bubbleDown(0);
    }
    return min;
  }

  private bubbleDown(i: number): void {
    const n = this.heap.length;
    while (true) {
      let smallest = i;
      const l = this.left(i);
      const r = this.right(i);

      if (l < n && this.compareFn(this.heap[l], this.heap[smallest]) < 0) {
        smallest = l;
      }
      if (r < n && this.compareFn(this.heap[r], this.heap[smallest]) < 0) {
        smallest = r;
      }

      if (smallest !== i) {
        this.swap(i, smallest);
        i = smallest;
      } else {
        break;
      }
    }
  }

  peek(): T | undefined {
    return this.heap[0];
  }

  get size(): number {
    return this.heap.length;
  }

  isEmpty(): boolean {
    return this.heap.length === 0;
  }
}

// ตัวอย่าง
const minHeap = new MinHeap<number>();
[5, 3, 8, 1, 9, 2].forEach(n => minHeap.insert(n));
console.log(minHeap.extractMin()); // 1
console.log(minHeap.extractMin()); // 2
console.log(minHeap.extractMin()); // 3
```

### 6.2 Max Heap

```typescript
class MaxHeap<T> extends MinHeap<T> {
  constructor(compareFn?: (a: T, b: T) => number) {
    // กลับลำดับการเปรียบเทียบ
    super(compareFn ? (a, b) => -compareFn(a, b) : (a, b) => (a > b ? -1 : a < b ? 1 : 0));
  }

  extractMax(): T | undefined {
    return this.extractMin();
  }

  peekMax(): T | undefined {
    return this.peek();
  }
}

// ตัวอย่าง
const maxHeap = new MaxHeap<number>();
[5, 3, 8, 1, 9, 2].forEach(n => maxHeap.insert(n));
console.log(maxHeap.extractMax()); // 9
console.log(maxHeap.extractMax()); // 8
```

**Time Complexity:**
- Insert: O(log n)
- Extract Min/Max: O(log n)
- Peek: O(1)
- Build Heap: O(n)

---

## 7. Hash Map (แฮชแมป)

### 7.1 Hash Map Implementation

```typescript
class HashMap<K, V> {
  private buckets: Array<Array<[K, V]>>;
  private size: number = 0;
  private readonly loadFactor: number = 0.75;
  private capacity: number;

  constructor(initialCapacity: number = 16) {
    this.capacity = initialCapacity;
    this.buckets = new Array(this.capacity).fill(null).map(() => []);
  }

  private hash(key: K): number {
    const str = JSON.stringify(key);
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = (hash * 31 + str.charCodeAt(i)) % this.capacity;
    }
    return hash;
  }

  set(key: K, value: V): void {
    if (this.size / this.capacity >= this.loadFactor) {
      this.resize();
    }

    const index = this.hash(key);
    const bucket = this.buckets[index];

    for (const pair of bucket) {
      if (JSON.stringify(pair[0]) === JSON.stringify(key)) {
        pair[1] = value;
        return;
      }
    }

    bucket.push([key, value]);
    this.size++;
  }

  get(key: K): V | undefined {
    const index = this.hash(key);
    const bucket = this.buckets[index];

    for (const [k, v] of bucket) {
      if (JSON.stringify(k) === JSON.stringify(key)) {
        return v;
      }
    }
    return undefined;
  }

  has(key: K): boolean {
    return this.get(key) !== undefined;
  }

  delete(key: K): boolean {
    const index = this.hash(key);
    const bucket = this.buckets[index];

    for (let i = 0; i < bucket.length; i++) {
      if (JSON.stringify(bucket[i][0]) === JSON.stringify(key)) {
        bucket.splice(i, 1);
        this.size--;
        return true;
      }
    }
    return false;
  }

  private resize(): void {
    const oldBuckets = this.buckets;
    this.capacity *= 2;
    this.buckets = new Array(this.capacity).fill(null).map(() => []);
    this.size = 0;

    for (const bucket of oldBuckets) {
      for (const [key, value] of bucket) {
        this.set(key, value);
      }
    }
  }

  keys(): K[] {
    const result: K[] = [];
    for (const bucket of this.buckets) {
      for (const [key] of bucket) {
        result.push(key);
      }
    }
    return result;
  }

  values(): V[] {
    const result: V[] = [];
    for (const bucket of this.buckets) {
      for (const [, value] of bucket) {
        result.push(value);
      }
    }
    return result;
  }

  entries(): [K, V][] {
    const result: [K, V][] = [];
    for (const bucket of this.buckets) {
      result.push(...bucket);
    }
    return result;
  }

  getSize(): number {
    return this.size;
  }
}

// ตัวอย่าง
const map = new HashMap<string, number>();
map.set("หนึ่ง", 1);
map.set("สอง", 2);
map.set("สาม", 3);
console.log(map.get("สอง")); // 2
console.log(map.has("สี่")); // false
map.delete("หนึ่ง");
console.log(map.getSize()); // 2
```

**Time Complexity:**
- Set/Get/Delete: O(1) เฉลี่ย, O(n) กรณีแย่สุด

---

## 8. Graph (กราฟ)

### 8.1 Adjacency List Graph

```typescript
class Graph<T> {
  private adjacencyList: Map<T, Set<T>> = new Map();
  private directed: boolean;

  constructor(directed: boolean = false) {
    this.directed = directed;
  }

  addVertex(vertex: T): void {
    if (!this.adjacencyList.has(vertex)) {
      this.adjacencyList.set(vertex, new Set());
    }
  }

  addEdge(from: T, to: T, weight?: number): void {
    this.addVertex(from);
    this.addVertex(to);
    this.adjacencyList.get(from)!.add(to);
    if (!this.directed) {
      this.adjacencyList.get(to)!.add(from);
    }
  }

  removeEdge(from: T, to: T): void {
    this.adjacencyList.get(from)?.delete(to);
    if (!this.directed) {
      this.adjacencyList.get(to)?.delete(from);
    }
  }

  removeVertex(vertex: T): void {
    this.adjacencyList.delete(vertex);
    for (const neighbors of this.adjacencyList.values()) {
      neighbors.delete(vertex);
    }
  }

  getNeighbors(vertex: T): T[] {
    return Array.from(this.adjacencyList.get(vertex) ?? []);
  }

  hasEdge(from: T, to: T): boolean {
    return this.adjacencyList.get(from)?.has(to) ?? false;
  }

  getVertices(): T[] {
    return Array.from(this.adjacencyList.keys());
  }

  // BFS - Breadth First Search
  bfs(start: T): T[] {
    const visited = new Set<T>();
    const result: T[] = [];
    const queue = new Queue<T>();

    visited.add(start);
    queue.enqueue(start);

    while (!queue.isEmpty()) {
      const vertex = queue.dequeue()!;
      result.push(vertex);

      for (const neighbor of this.getNeighbors(vertex)) {
        if (!visited.has(neighbor)) {
          visited.add(neighbor);
          queue.enqueue(neighbor);
        }
      }
    }
    return result;
  }

  // DFS - Depth First Search
  dfs(start: T): T[] {
    const visited = new Set<T>();
    const result: T[] = [];

    const dfsHelper = (vertex: T) => {
      visited.add(vertex);
      result.push(vertex);
      for (const neighbor of this.getNeighbors(vertex)) {
        if (!visited.has(neighbor)) {
          dfsHelper(neighbor);
        }
      }
    };

    dfsHelper(start);
    return result;
  }

  // ตรวจสอบ Cycle
  hasCycle(): boolean {
    const visited = new Set<T>();
    const recursionStack = new Set<T>();

    const dfsCheck = (vertex: T): boolean => {
      visited.add(vertex);
      recursionStack.add(vertex);

      for (const neighbor of this.getNeighbors(vertex)) {
        if (!visited.has(neighbor)) {
          if (dfsCheck(neighbor)) return true;
        } else if (recursionStack.has(neighbor)) {
          return true;
        }
      }

      recursionStack.delete(vertex);
      return false;
    };

    for (const vertex of this.getVertices()) {
      if (!visited.has(vertex)) {
        if (dfsCheck(vertex)) return true;
      }
    }
    return false;
  }

  // Topological Sort (สำหรับ DAG)
  topologicalSort(): T[] {
    const visited = new Set<T>();
    const result: T[] = [];

    const dfs = (vertex: T) => {
      visited.add(vertex);
      for (const neighbor of this.getNeighbors(vertex)) {
        if (!visited.has(neighbor)) {
          dfs(neighbor);
        }
      }
      result.unshift(vertex);
    };

    for (const vertex of this.getVertices()) {
      if (!visited.has(vertex)) {
        dfs(vertex);
      }
    }
    return result;
  }
}
```

### 8.2 Weighted Graph สำหรับ Dijkstra

```typescript
interface WeightedEdge<T> {
  to: T;
  weight: number;
}

class WeightedGraph<T> {
  private adjacencyList: Map<T, WeightedEdge<T>[]> = new Map();

  addVertex(vertex: T): void {
    if (!this.adjacencyList.has(vertex)) {
      this.adjacencyList.set(vertex, []);
    }
  }

  addEdge(from: T, to: T, weight: number): void {
    this.addVertex(from);
    this.addVertex(to);
    this.adjacencyList.get(from)!.push({ to, weight });
    this.adjacencyList.get(to)!.push({ to: from, weight });
  }

  // Dijkstra Algorithm - หาเส้นทางสั้นที่สุด
  dijkstra(start: T): Map<T, { distance: number; previous: T | null }> {
    const distances = new Map<T, { distance: number; previous: T | null }>();
    const visited = new Set<T>();
    const pq = new PriorityQueue<T>();

    // Initialize
    for (const vertex of this.adjacencyList.keys()) {
      distances.set(vertex, {
        distance: vertex === start ? 0 : Infinity,
        previous: null
      });
    }
    pq.enqueue(start, 0);

    while (!pq.isEmpty()) {
      const current = pq.dequeue()!;
      if (visited.has(current)) continue;
      visited.add(current);

      const edges = this.adjacencyList.get(current) ?? [];
      for (const { to, weight } of edges) {
        if (visited.has(to)) continue;
        const newDist = distances.get(current)!.distance + weight;
        if (newDist < distances.get(to)!.distance) {
          distances.set(to, { distance: newDist, previous: current });
          pq.enqueue(to, -newDist); // Negative for min priority
        }
      }
    }
    return distances;
  }

  getShortestPath(start: T, end: T): T[] {
    const distances = this.dijkstra(start);
    const path: T[] = [];
    let current: T | null = end;

    while (current !== null) {
      path.unshift(current);
      current = distances.get(current)?.previous ?? null;
    }

    return path[0] === start ? path : [];
  }
}

// ตัวอย่าง
const graph = new WeightedGraph<string>();
graph.addEdge("A", "B", 4);
graph.addEdge("A", "C", 2);
graph.addEdge("B", "D", 3);
graph.addEdge("C", "D", 1);
graph.addEdge("D", "E", 5);

const path = graph.getShortestPath("A", "E");
console.log(path); // ["A", "C", "D", "E"]
```

---

## 9. Sorting Algorithms (อัลกอริทึมการเรียงลำดับ)

### 9.1 Bubble Sort

```typescript
function bubbleSort<T>(
  arr: T[],
  compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)
): T[] {
  const result = [...arr];
  const n = result.length;

  for (let i = 0; i < n - 1; i++) {
    let swapped = false;
    for (let j = 0; j < n - i - 1; j++) {
      if (compareFn(result[j], result[j + 1]) > 0) {
        [result[j], result[j + 1]] = [result[j + 1], result[j]];
        swapped = true;
      }
    }
    if (!swapped) break; // Optimization: หยุดถ้าไม่มีการสลับ
  }
  return result;
}

// Time: O(n²) | Space: O(1)
console.log(bubbleSort([64, 34, 25, 12, 22, 11, 90]));
```

### 9.2 Merge Sort

```typescript
function mergeSort<T>(
  arr: T[],
  compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)
): T[] {
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid), compareFn);
  const right = mergeSort(arr.slice(mid), compareFn);

  return merge(left, right, compareFn);
}

function merge<T>(
  left: T[],
  right: T[],
  compareFn: (a: T, b: T) => number
): T[] {
  const result: T[] = [];
  let i = 0, j = 0;

  while (i < left.length && j < right.length) {
    if (compareFn(left[i], right[j]) <= 0) {
      result.push(left[i++]);
    } else {
      result.push(right[j++]);
    }
  }

  return result.concat(left.slice(i)).concat(right.slice(j));
}

// Time: O(n log n) | Space: O(n)
console.log(mergeSort([38, 27, 43, 3, 9, 82, 10]));
```

### 9.3 Quick Sort

```typescript
function quickSort<T>(
  arr: T[],
  compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0),
  low: number = 0,
  high: number = arr.length - 1
): T[] {
  const result = [...arr];

  const partition = (arr: T[], low: number, high: number): number => {
    const pivot = arr[high];
    let i = low - 1;

    for (let j = low; j < high; j++) {
      if (compareFn(arr[j], pivot) <= 0) {
        i++;
        [arr[i], arr[j]] = [arr[j], arr[i]];
      }
    }
    [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
    return i + 1;
  };

  const sort = (arr: T[], low: number, high: number) => {
    if (low < high) {
      const pi = partition(arr, low, high);
      sort(arr, low, pi - 1);
      sort(arr, pi + 1, high);
    }
  };

  sort(result, low, high);
  return result;
}

// Time: O(n log n) เฉลี่ย, O(n²) กรณีแย่สุด | Space: O(log n)
console.log(quickSort([10, 7, 8, 9, 1, 5]));
```

### 9.4 Heap Sort

```typescript
function heapSort<T>(
  arr: T[],
  compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)
): T[] {
  const result = [...arr];
  const n = result.length;

  const heapify = (arr: T[], n: number, i: number) => {
    let largest = i;
    const left = 2 * i + 1;
    const right = 2 * i + 2;

    if (left < n && compareFn(arr[left], arr[largest]) > 0) {
      largest = left;
    }
    if (right < n && compareFn(arr[right], arr[largest]) > 0) {
      largest = right;
    }

    if (largest !== i) {
      [arr[i], arr[largest]] = [arr[largest], arr[i]];
      heapify(arr, n, largest);
    }
  };

  // Build max heap
  for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
    heapify(result, n, i);
  }

  // Extract elements
  for (let i = n - 1; i > 0; i--) {
    [result[0], result[i]] = [result[i], result[0]];
    heapify(result, i, 0);
  }

  return result;
}

// Time: O(n log n) | Space: O(1)
console.log(heapSort([12, 11, 13, 5, 6, 7]));
```

---

## 10. Searching Algorithms (อัลกอริทึมการค้นหา)

### 10.1 Binary Search

```typescript
function binarySearch<T>(
  arr: T[],
  target: T,
  compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)
): number {
  let low = 0;
  let high = arr.length - 1;

  while (low <= high) {
    const mid = Math.floor((low + high) / 2);
    const cmp = compareFn(arr[mid], target);

    if (cmp === 0) return mid;
    if (cmp < 0) low = mid + 1;
    else high = mid - 1;
  }
  return -1;
}

// Recursive version
function binarySearchRecursive<T>(
  arr: T[],
  target: T,
  low: number = 0,
  high: number = arr.length - 1,
  compareFn: (a: T, b: T) => number = (a, b) => (a < b ? -1 : a > b ? 1 : 0)
): number {
  if (low > high) return -1;

  const mid = Math.floor((low + high) / 2);
  const cmp = compareFn(arr[mid], target);

  if (cmp === 0) return mid;
  if (cmp < 0) return binarySearchRecursive(arr, target, mid + 1, high, compareFn);
  return binarySearchRecursive(arr, target, low, mid - 1, compareFn);
}

// Time: O(log n) | Space: O(1)
const sortedArr = [1, 3, 5, 7, 9, 11, 13];
console.log(binarySearch(sortedArr, 7));  // 3
console.log(binarySearch(sortedArr, 6));  // -1
```

### 10.2 BFS สำหรับหา Shortest Path

```typescript
function shortestPathBFS<T>(
  graph: Map<T, T[]>,
  start: T,
  end: T
): T[] | null {
  if (start === end) return [start];

  const visited = new Set<T>([start]);
  const queue: Array<{ node: T; path: T[] }> = [{ node: start, path: [start] }];

  while (queue.length > 0) {
    const { node, path } = queue.shift()!;

    for (const neighbor of (graph.get(node) ?? [])) {
      if (neighbor === end) {
        return [...path, neighbor];
      }

      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push({ node: neighbor, path: [...path, neighbor] });
      }
    }
  }
  return null;
}
```

---

## 11. Dynamic Programming (ไดนามิกโปรแกรมมิ่ง)

### 11.1 Fibonacci

```typescript
// Memoization (Top-Down)
function fibMemo(n: number, memo: Map<number, number> = new Map()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

// Tabulation (Bottom-Up)
function fibTable(n: number): number {
  if (n <= 1) return n;
  const dp = new Array(n + 1).fill(0);
  dp[1] = 1;
  for (let i = 2; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}

// Space Optimized
function fibOptimal(n: number): number {
  if (n <= 1) return n;
  let prev2 = 0, prev1 = 1;
  for (let i = 2; i <= n; i++) {
    const curr = prev1 + prev2;
    prev2 = prev1;
    prev1 = curr;
  }
  return prev1;
}
```

### 11.2 Knapsack Problem

```typescript
function knapsack(
  weights: number[],
  values: number[],
  capacity: number
): number {
  const n = weights.length;
  const dp: number[][] = Array.from({ length: n + 1 }, () =>
    new Array(capacity + 1).fill(0)
  );

  for (let i = 1; i <= n; i++) {
    for (let w = 0; w <= capacity; w++) {
      dp[i][w] = dp[i - 1][w];
      if (weights[i - 1] <= w) {
        dp[i][w] = Math.max(
          dp[i][w],
          dp[i - 1][w - weights[i - 1]] + values[i - 1]
        );
      }
    }
  }
  return dp[n][capacity];
}

// ตัวอย่าง
const weights = [2, 3, 4, 5];
const values = [3, 4, 5, 6];
console.log(knapsack(weights, values, 8)); // 10
```

### 11.3 Longest Common Subsequence

```typescript
function lcs(str1: string, str2: string): string {
  const m = str1.length;
  const n = str2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () =>
    new Array(n + 1).fill(0)
  );

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (str1[i - 1] === str2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }

  // Reconstruct LCS
  let result = '';
  let i = m, j = n;
  while (i > 0 && j > 0) {
    if (str1[i - 1] === str2[j - 1]) {
      result = str1[i - 1] + result;
      i--;
      j--;
    } else if (dp[i - 1][j] > dp[i][j - 1]) {
      i--;
    } else {
      j--;
    }
  }
  return result;
}

console.log(lcs("ABCBDAB", "BDCAB")); // "BCAB" หรือ "BDAB"
```

### 11.4 Coin Change

```typescript
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (coin <= i) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }

  return dp[amount] === Infinity ? -1 : dp[amount];
}

// ตัวอย่าง
console.log(coinChange([1, 5, 6, 9], 11)); // 2 (5+6)
```

### 11.5 Edit Distance

```typescript
function editDistance(str1: string, str2: string): number {
  const m = str1.length;
  const n = str2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, (_, i) =>
    Array.from({ length: n + 1 }, (_, j) => (i === 0 ? j : j === 0 ? i : 0))
  );

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (str1[i - 1] === str2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        dp[i][j] = 1 + Math.min(
          dp[i - 1][j],   // Delete
          dp[i][j - 1],   // Insert
          dp[i - 1][j - 1] // Replace
        );
      }
    }
  }

  return dp[m][n];
}

console.log(editDistance("kitten", "sitting")); // 3
```

---

## 12. Graph Algorithms เพิ่มเติม

### 12.1 Floyd-Warshall (All-Pairs Shortest Path)

```typescript
function floydWarshall(graph: number[][]): number[][] {
  const n = graph.length;
  const dist = graph.map(row => [...row]);

  // Replace 0 with Infinity for non-existing paths (except diagonal)
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      if (i !== j && dist[i][j] === 0) {
        dist[i][j] = Infinity;
      }
    }
  }

  for (let k = 0; k < n; k++) {
    for (let i = 0; i < n; i++) {
      for (let j = 0; j < n; j++) {
        if (dist[i][k] + dist[k][j] < dist[i][j]) {
          dist[i][j] = dist[i][k] + dist[k][j];
        }
      }
    }
  }

  return dist;
}
```

### 12.2 Kruskal's Algorithm (Minimum Spanning Tree)

```typescript
interface Edge {
  from: number;
  to: number;
  weight: number;
}

class UnionFind {
  private parent: number[];
  private rank: number[];

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
  }

  find(x: number): number {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]); // Path compression
    }
    return this.parent[x];
  }

  union(x: number, y: number): boolean {
    const px = this.find(x);
    const py = this.find(y);
    if (px === py) return false;

    if (this.rank[px] < this.rank[py]) {
      this.parent[px] = py;
    } else if (this.rank[px] > this.rank[py]) {
      this.parent[py] = px;
    } else {
      this.parent[py] = px;
      this.rank[px]++;
    }
    return true;
  }
}

function kruskal(n: number, edges: Edge[]): Edge[] {
  const sorted = [...edges].sort((a, b) => a.weight - b.weight);
  const uf = new UnionFind(n);
  const mst: Edge[] = [];

  for (const edge of sorted) {
    if (uf.union(edge.from, edge.to)) {
      mst.push(edge);
      if (mst.length === n - 1) break;
    }
  }

  return mst;
}
```

---

## 13. สรุปตารางความซับซ้อน

| โครงสร้างข้อมูล | Access | Search | Insert | Delete | Space |
|----------------|--------|--------|--------|--------|-------|
| Array | O(1) | O(n) | O(n) | O(n) | O(n) |
| Linked List | O(n) | O(n) | O(1) | O(1)* | O(n) |
| Stack | O(n) | O(n) | O(1) | O(1) | O(n) |
| Queue | O(n) | O(n) | O(1) | O(1) | O(n) |
| Hash Map | O(1)** | O(1)** | O(1)** | O(1)** | O(n) |
| BST | O(log n)** | O(log n)** | O(log n)** | O(log n)** | O(n) |
| Heap | O(1)*** | O(n) | O(log n) | O(log n) | O(n) |

*กรณีมี reference  
**เฉลี่ย  
***สำหรับ min/max เท่านั้น

| อัลกอริทึม | Best | Average | Worst | Space |
|-----------|------|---------|-------|-------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) |

---

## สรุป

ในบทนี้เราได้เรียนรู้โครงสร้างข้อมูลและอัลกอริทึมหลักๆ ใน TypeScript:

1. **Stack** - LIFO, ใช้ใน undo/redo, expression evaluation
2. **Queue** - FIFO, ใช้ใน BFS, task scheduling
3. **Linked List** - Dynamic size, ใช้ใน LRU cache, music playlist
4. **Binary Tree** - Hierarchical structure, ใช้ใน file system
5. **BST** - Ordered tree, ใช้ใน search/insert/delete efficiently
6. **Heap** - Priority-based, ใช้ใน priority queue, heap sort
7. **Hash Map** - Key-value pairs, O(1) average operations
8. **Graph** - Network structure, ใช้ใน routing, social networks
9. **Sorting** - Bubble, Merge, Quick, Heap sort
10. **Searching** - Binary search, BFS, DFS
11. **Dynamic Programming** - Optimization problems
12. **Graph Algorithms** - Dijkstra, Floyd-Warshall, Kruskal
