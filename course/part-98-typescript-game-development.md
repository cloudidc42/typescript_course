# Part 98: Game Development กับ TypeScript

## บทนำ

TypeScript เหมาะอย่างยิ่งสำหรับการพัฒนาเกม เพราะ type system ช่วยจัดการ game state ที่ซับซ้อน, entity systems, และ game logic ได้อย่างปลอดภัย บทนี้จะครอบคลุมตั้งแต่ Canvas API พื้นฐาน ไปจนถึงการสร้างเกมสมบูรณ์ด้วย ECS pattern

---

## 98.1 TypeScript สำหรับ Browser Games

### การตั้งค่าโปรเจกต์

```bash
mkdir ts-game && cd ts-game
npm init -y
npm install -D typescript webpack webpack-cli ts-loader html-webpack-plugin
npm install -D @types/node
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "sourceMap": true
  },
  "include": ["src/**/*"]
}
```

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>TypeScript Game</title>
  <style>
    body { margin: 0; background: #1a1a2e; display: flex; justify-content: center; align-items: center; height: 100vh; }
    canvas { border: 2px solid #e94560; }
  </style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>
  <script type="module" src="dist/main.js"></script>
</body>
</html>
```

---

## 98.2 Canvas API กับ TypeScript

```typescript
// src/canvas/canvas-manager.ts

export interface CanvasConfig {
  width: number;
  height: number;
  backgroundColor?: string;
}

export class CanvasManager {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private config: Required<CanvasConfig>;

  constructor(canvasId: string, config: CanvasConfig) {
    this.config = {
      backgroundColor: '#000000',
      ...config,
    };

    const canvas = document.getElementById(canvasId) as HTMLCanvasElement | null;
    if (!canvas) {
      throw new Error(`Canvas element with id '${canvasId}' not found`);
    }

    const ctx = canvas.getContext('2d');
    if (!ctx) {
      throw new Error('Failed to get 2D rendering context');
    }

    this.canvas = canvas;
    this.ctx = ctx;

    this.resize(config.width, config.height);
  }

  resize(width: number, height: number): void {
    this.canvas.width = width;
    this.canvas.height = height;
    this.config.width = width;
    this.config.height = height;
  }

  clear(): void {
    this.ctx.fillStyle = this.config.backgroundColor;
    this.ctx.fillRect(0, 0, this.config.width, this.config.height);
  }

  getContext(): CanvasRenderingContext2D {
    return this.ctx;
  }

  getCanvas(): HTMLCanvasElement {
    return this.canvas;
  }

  get width(): number {
    return this.config.width;
  }

  get height(): number {
    return this.config.height;
  }

  // Helper drawing methods
  drawRect(
    x: number,
    y: number,
    width: number,
    height: number,
    color: string
  ): void {
    this.ctx.fillStyle = color;
    this.ctx.fillRect(x, y, width, height);
  }

  drawCircle(
    x: number,
    y: number,
    radius: number,
    color: string,
    fill = true
  ): void {
    this.ctx.beginPath();
    this.ctx.arc(x, y, radius, 0, Math.PI * 2);

    if (fill) {
      this.ctx.fillStyle = color;
      this.ctx.fill();
    } else {
      this.ctx.strokeStyle = color;
      this.ctx.stroke();
    }
  }

  drawText(
    text: string,
    x: number,
    y: number,
    options: {
      color?: string;
      font?: string;
      align?: CanvasTextAlign;
      baseline?: CanvasTextBaseline;
    } = {}
  ): void {
    const { color = '#ffffff', font = '16px Arial', align = 'left', baseline = 'top' } = options;

    this.ctx.fillStyle = color;
    this.ctx.font = font;
    this.ctx.textAlign = align;
    this.ctx.textBaseline = baseline;
    this.ctx.fillText(text, x, y);
  }

  drawLine(
    x1: number,
    y1: number,
    x2: number,
    y2: number,
    color: string,
    width = 1
  ): void {
    this.ctx.beginPath();
    this.ctx.strokeStyle = color;
    this.ctx.lineWidth = width;
    this.ctx.moveTo(x1, y1);
    this.ctx.lineTo(x2, y2);
    this.ctx.stroke();
  }

  // Save/restore state
  save(): void {
    this.ctx.save();
  }

  restore(): void {
    this.ctx.restore();
  }

  // Transform
  translate(x: number, y: number): void {
    this.ctx.translate(x, y);
  }

  rotate(angle: number): void {
    this.ctx.rotate(angle);
  }

  scale(x: number, y: number): void {
    this.ctx.scale(x, y);
  }
}
```

---

## 98.3 Vector Math กับ TypeScript

```typescript
// src/math/vector2.ts

export class Vector2 {
  constructor(public x: number = 0, public y: number = 0) {}

  // Static factory methods
  static zero(): Vector2 {
    return new Vector2(0, 0);
  }

  static one(): Vector2 {
    return new Vector2(1, 1);
  }

  static up(): Vector2 {
    return new Vector2(0, -1);
  }

  static down(): Vector2 {
    return new Vector2(0, 1);
  }

  static left(): Vector2 {
    return new Vector2(-1, 0);
  }

  static right(): Vector2 {
    return new Vector2(1, 0);
  }

  static fromAngle(angleRadians: number, magnitude = 1): Vector2 {
    return new Vector2(
      Math.cos(angleRadians) * magnitude,
      Math.sin(angleRadians) * magnitude
    );
  }

  // Operations
  add(other: Vector2): Vector2 {
    return new Vector2(this.x + other.x, this.y + other.y);
  }

  subtract(other: Vector2): Vector2 {
    return new Vector2(this.x - other.x, this.y - other.y);
  }

  multiply(scalar: number): Vector2 {
    return new Vector2(this.x * scalar, this.y * scalar);
  }

  divide(scalar: number): Vector2 {
    if (scalar === 0) throw new Error('Division by zero');
    return new Vector2(this.x / scalar, this.y / scalar);
  }

  dot(other: Vector2): number {
    return this.x * other.x + this.y * other.y;
  }

  cross(other: Vector2): number {
    return this.x * other.y - this.y * other.x;
  }

  // Properties
  get magnitude(): number {
    return Math.sqrt(this.x * this.x + this.y * this.y);
  }

  get magnitudeSquared(): number {
    return this.x * this.x + this.y * this.y;
  }

  get angle(): number {
    return Math.atan2(this.y, this.x);
  }

  normalize(): Vector2 {
    const mag = this.magnitude;
    if (mag === 0) return Vector2.zero();
    return this.divide(mag);
  }

  // Distance
  distanceTo(other: Vector2): number {
    return this.subtract(other).magnitude;
  }

  distanceToSquared(other: Vector2): number {
    return this.subtract(other).magnitudeSquared;
  }

  // Interpolation
  lerp(target: Vector2, t: number): Vector2 {
    const clampedT = Math.max(0, Math.min(1, t));
    return new Vector2(
      this.x + (target.x - this.x) * clampedT,
      this.y + (target.y - this.y) * clampedT
    );
  }

  // Rotation
  rotate(angleRadians: number): Vector2 {
    const cos = Math.cos(angleRadians);
    const sin = Math.sin(angleRadians);
    return new Vector2(
      this.x * cos - this.y * sin,
      this.x * sin + this.y * cos
    );
  }

  // Clamp
  clamp(min: Vector2, max: Vector2): Vector2 {
    return new Vector2(
      Math.max(min.x, Math.min(max.x, this.x)),
      Math.max(min.y, Math.min(max.y, this.y))
    );
  }

  // Utilities
  clone(): Vector2 {
    return new Vector2(this.x, this.y);
  }

  equals(other: Vector2, epsilon = 0.001): boolean {
    return (
      Math.abs(this.x - other.x) < epsilon &&
      Math.abs(this.y - other.y) < epsilon
    );
  }

  toString(): string {
    return `Vector2(${this.x.toFixed(2)}, ${this.y.toFixed(2)})`;
  }
}
```

---

## 98.4 Game Loop Implementation

```typescript
// src/core/game-loop.ts

export interface GameLoopCallback {
  update(deltaTime: number): void;
  render(): void;
}

export interface GameLoopConfig {
  targetFPS?: number;
  maxDeltaTime?: number;
}

export class GameLoop {
  private animationFrameId: number | null = null;
  private lastTime = 0;
  private isRunning = false;
  private fpsCounter = 0;
  private fpsTimer = 0;
  private currentFPS = 0;
  private readonly targetFPS: number;
  private readonly maxDeltaTime: number;
  private readonly targetFrameTime: number;

  constructor(
    private callback: GameLoopCallback,
    config: GameLoopConfig = {}
  ) {
    this.targetFPS = config.targetFPS ?? 60;
    this.maxDeltaTime = config.maxDeltaTime ?? 0.1; // Max 100ms
    this.targetFrameTime = 1000 / this.targetFPS;
  }

  start(): void {
    if (this.isRunning) return;
    this.isRunning = true;
    this.lastTime = performance.now();
    this.tick(this.lastTime);
  }

  stop(): void {
    this.isRunning = false;
    if (this.animationFrameId !== null) {
      cancelAnimationFrame(this.animationFrameId);
      this.animationFrameId = null;
    }
  }

  private tick = (currentTime: number): void => {
    if (!this.isRunning) return;

    this.animationFrameId = requestAnimationFrame(this.tick);

    const rawDeltaTime = (currentTime - this.lastTime) / 1000;
    const deltaTime = Math.min(rawDeltaTime, this.maxDeltaTime);
    this.lastTime = currentTime;

    // FPS Counter
    this.fpsCounter++;
    this.fpsTimer += rawDeltaTime;
    if (this.fpsTimer >= 1) {
      this.currentFPS = this.fpsCounter;
      this.fpsCounter = 0;
      this.fpsTimer -= 1;
    }

    this.callback.update(deltaTime);
    this.callback.render();
  };

  getFPS(): number {
    return this.currentFPS;
  }

  isActive(): boolean {
    return this.isRunning;
  }
}

// Fixed timestep game loop
export class FixedTimestepLoop {
  private isRunning = false;
  private animationFrameId: number | null = null;
  private lastTime = 0;
  private accumulator = 0;
  private readonly fixedDeltaTime: number;

  constructor(
    private onFixedUpdate: (deltaTime: number) => void,
    private onRender: (interpolation: number) => void,
    private ticksPerSecond = 60
  ) {
    this.fixedDeltaTime = 1 / ticksPerSecond;
  }

  start(): void {
    if (this.isRunning) return;
    this.isRunning = true;
    this.lastTime = performance.now();
    this.animationFrameId = requestAnimationFrame(this.tick);
  }

  stop(): void {
    this.isRunning = false;
    if (this.animationFrameId !== null) {
      cancelAnimationFrame(this.animationFrameId);
    }
  }

  private tick = (currentTime: number): void => {
    if (!this.isRunning) return;

    const frameTime = Math.min((currentTime - this.lastTime) / 1000, 0.25);
    this.lastTime = currentTime;
    this.accumulator += frameTime;

    // Fixed updates
    while (this.accumulator >= this.fixedDeltaTime) {
      this.onFixedUpdate(this.fixedDeltaTime);
      this.accumulator -= this.fixedDeltaTime;
    }

    // Render with interpolation
    const interpolation = this.accumulator / this.fixedDeltaTime;
    this.onRender(interpolation);

    this.animationFrameId = requestAnimationFrame(this.tick);
  };
}
```

---

## 98.5 Entity-Component-System (ECS)

```typescript
// src/ecs/types.ts
export type EntityId = number;
export type ComponentType = string;

// Base component interface
export interface Component {
  type: ComponentType;
}

// Common components
export interface TransformComponent extends Component {
  type: 'transform';
  position: { x: number; y: number };
  rotation: number;
  scale: { x: number; y: number };
}

export interface VelocityComponent extends Component {
  type: 'velocity';
  x: number;
  y: number;
  maxSpeed?: number;
}

export interface RenderComponent extends Component {
  type: 'render';
  shape: 'rect' | 'circle' | 'sprite';
  color: string;
  width: number;
  height: number;
  visible: boolean;
  layer: number;
}

export interface ColliderComponent extends Component {
  type: 'collider';
  width: number;
  height: number;
  isTrigger: boolean;
  tag: string;
}

export interface HealthComponent extends Component {
  type: 'health';
  current: number;
  max: number;
  invincible: boolean;
  invincibleTimer: number;
}

export interface TagComponent extends Component {
  type: 'tag';
  tags: Set<string>;
}

// Component union type
export type GameComponent =
  | TransformComponent
  | VelocityComponent
  | RenderComponent
  | ColliderComponent
  | HealthComponent
  | TagComponent;
```

```typescript
// src/ecs/world.ts
import type { EntityId, Component, ComponentType, GameComponent } from './types';

export class World {
  private nextEntityId: EntityId = 1;
  private entities = new Set<EntityId>();
  private components = new Map<ComponentType, Map<EntityId, Component>>();
  private deletedEntities = new Set<EntityId>();

  // Entity management
  createEntity(): EntityId {
    const id = this.nextEntityId++;
    this.entities.add(id);
    return id;
  }

  destroyEntity(entityId: EntityId): void {
    this.deletedEntities.add(entityId);
  }

  processDestructions(): void {
    for (const entityId of this.deletedEntities) {
      this.entities.delete(entityId);
      for (const componentMap of this.components.values()) {
        componentMap.delete(entityId);
      }
    }
    this.deletedEntities.clear();
  }

  isAlive(entityId: EntityId): boolean {
    return this.entities.has(entityId) && !this.deletedEntities.has(entityId);
  }

  // Component management
  addComponent<T extends Component>(entityId: EntityId, component: T): void {
    if (!this.components.has(component.type)) {
      this.components.set(component.type, new Map());
    }
    this.components.get(component.type)!.set(entityId, component);
  }

  getComponent<T extends Component>(
    entityId: EntityId,
    componentType: ComponentType
  ): T | undefined {
    return this.components.get(componentType)?.get(entityId) as T | undefined;
  }

  hasComponent(entityId: EntityId, componentType: ComponentType): boolean {
    return this.components.get(componentType)?.has(entityId) ?? false;
  }

  removeComponent(entityId: EntityId, componentType: ComponentType): void {
    this.components.get(componentType)?.delete(entityId);
  }

  // Query entities with specific components
  query(...componentTypes: ComponentType[]): EntityId[] {
    if (componentTypes.length === 0) return Array.from(this.entities);

    const [first, ...rest] = componentTypes;
    if (!first) return [];

    const firstMap = this.components.get(first);
    if (!firstMap) return [];

    return Array.from(firstMap.keys()).filter(
      (entityId) =>
        this.isAlive(entityId) &&
        rest.every((type) => this.hasComponent(entityId, type))
    );
  }

  // Get all entities
  getEntities(): EntityId[] {
    return Array.from(this.entities).filter((id) => !this.deletedEntities.has(id));
  }
}
```

```typescript
// src/ecs/systems.ts
import type { World } from './world';
import type {
  TransformComponent,
  VelocityComponent,
  RenderComponent,
  ColliderComponent,
  HealthComponent,
} from './types';
import type { CanvasManager } from '../canvas/canvas-manager';

// System interface
export interface System {
  update(world: World, deltaTime: number): void;
}

// Movement System
export class MovementSystem implements System {
  update(world: World, deltaTime: number): void {
    const entities = world.query('transform', 'velocity');

    for (const entityId of entities) {
      const transform = world.getComponent<TransformComponent>(entityId, 'transform')!;
      const velocity = world.getComponent<VelocityComponent>(entityId, 'velocity')!;

      // Apply velocity
      transform.position.x += velocity.x * deltaTime;
      transform.position.y += velocity.y * deltaTime;

      // Apply max speed
      if (velocity.maxSpeed !== undefined) {
        const speed = Math.sqrt(velocity.x * velocity.x + velocity.y * velocity.y);
        if (speed > velocity.maxSpeed) {
          const scale = velocity.maxSpeed / speed;
          velocity.x *= scale;
          velocity.y *= scale;
        }
      }
    }
  }
}

// Render System
export class RenderSystem implements System {
  constructor(private canvas: CanvasManager) {}

  update(world: World, _deltaTime: number): void {
    const entities = world
      .query('transform', 'render')
      .filter((id) => {
        const render = world.getComponent<RenderComponent>(id, 'render');
        return render?.visible ?? false;
      })
      .sort((a, b) => {
        const layerA = world.getComponent<RenderComponent>(a, 'render')?.layer ?? 0;
        const layerB = world.getComponent<RenderComponent>(b, 'render')?.layer ?? 0;
        return layerA - layerB;
      });

    for (const entityId of entities) {
      const transform = world.getComponent<TransformComponent>(entityId, 'transform')!;
      const render = world.getComponent<RenderComponent>(entityId, 'render')!;
      const ctx = this.canvas.getContext();

      ctx.save();
      ctx.translate(transform.position.x, transform.position.y);
      ctx.rotate(transform.rotation);
      ctx.scale(transform.scale.x, transform.scale.y);

      if (render.shape === 'rect') {
        ctx.fillStyle = render.color;
        ctx.fillRect(-render.width / 2, -render.height / 2, render.width, render.height);
      } else if (render.shape === 'circle') {
        ctx.beginPath();
        ctx.arc(0, 0, render.width / 2, 0, Math.PI * 2);
        ctx.fillStyle = render.color;
        ctx.fill();
      }

      ctx.restore();
    }
  }
}

// Collision System
export class CollisionSystem implements System {
  private collisionEvents: Array<{ a: number; b: number }> = [];

  update(world: World, _deltaTime: number): void {
    this.collisionEvents = [];
    const entities = world.query('transform', 'collider');

    for (let i = 0; i < entities.length; i++) {
      for (let j = i + 1; j < entities.length; j++) {
        const entityA = entities[i]!;
        const entityB = entities[j]!;

        if (this.checkCollision(world, entityA, entityB)) {
          this.collisionEvents.push({ a: entityA, b: entityB });
        }
      }
    }
  }

  private checkCollision(world: World, idA: number, idB: number): boolean {
    const transformA = world.getComponent<TransformComponent>(idA, 'transform')!;
    const colliderA = world.getComponent<ColliderComponent>(idA, 'collider')!;
    const transformB = world.getComponent<TransformComponent>(idB, 'transform')!;
    const colliderB = world.getComponent<ColliderComponent>(idB, 'collider')!;

    // AABB collision check
    const ax = transformA.position.x - colliderA.width / 2;
    const ay = transformA.position.y - colliderA.height / 2;
    const bx = transformB.position.x - colliderB.width / 2;
    const by = transformB.position.y - colliderB.height / 2;

    return (
      ax < bx + colliderB.width &&
      ax + colliderA.width > bx &&
      ay < by + colliderB.height &&
      ay + colliderA.height > by
    );
  }

  getCollisionEvents(): Array<{ a: number; b: number }> {
    return this.collisionEvents;
  }
}

// Health System
export class HealthSystem implements System {
  private deathEvents: number[] = [];

  update(world: World, deltaTime: number): void {
    this.deathEvents = [];
    const entities = world.query('health');

    for (const entityId of entities) {
      const health = world.getComponent<HealthComponent>(entityId, 'health')!;

      // Update invincibility timer
      if (health.invincible) {
        health.invincibleTimer -= deltaTime;
        if (health.invincibleTimer <= 0) {
          health.invincible = false;
          health.invincibleTimer = 0;
        }
      }

      // Check death
      if (health.current <= 0) {
        this.deathEvents.push(entityId);
      }
    }
  }

  getDeathEvents(): number[] {
    return this.deathEvents;
  }
}
```

---

## 98.6 Input Handling กับ Types

```typescript
// src/input/input-manager.ts

export type KeyCode = string;
export type MouseButton = 0 | 1 | 2;

export interface InputState {
  keys: Map<KeyCode, boolean>;
  keysPressed: Set<KeyCode>;
  keysReleased: Set<KeyCode>;
  mousePosition: { x: number; y: number };
  mouseButtons: Map<MouseButton, boolean>;
  mouseButtonsPressed: Set<MouseButton>;
  mouseWheel: number;
}

export class InputManager {
  private state: InputState = {
    keys: new Map(),
    keysPressed: new Set(),
    keysReleased: new Set(),
    mousePosition: { x: 0, y: 0 },
    mouseButtons: new Map(),
    mouseButtonsPressed: new Set(),
    mouseWheel: 0,
  };

  private canvas: HTMLCanvasElement;

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    // Keyboard
    window.addEventListener('keydown', this.onKeyDown);
    window.addEventListener('keyup', this.onKeyUp);

    // Mouse
    this.canvas.addEventListener('mousemove', this.onMouseMove);
    this.canvas.addEventListener('mousedown', this.onMouseDown);
    this.canvas.addEventListener('mouseup', this.onMouseUp);
    this.canvas.addEventListener('wheel', this.onWheel);

    // Touch (mobile)
    this.canvas.addEventListener('touchstart', this.onTouchStart, { passive: true });
    this.canvas.addEventListener('touchend', this.onTouchEnd, { passive: true });
    this.canvas.addEventListener('touchmove', this.onTouchMove, { passive: true });
  }

  private onKeyDown = (event: KeyboardEvent): void => {
    if (!this.state.keys.get(event.code)) {
      this.state.keysPressed.add(event.code);
    }
    this.state.keys.set(event.code, true);
  };

  private onKeyUp = (event: KeyboardEvent): void => {
    this.state.keys.set(event.code, false);
    this.state.keysReleased.add(event.code);
  };

  private onMouseMove = (event: MouseEvent): void => {
    const rect = this.canvas.getBoundingClientRect();
    this.state.mousePosition = {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top,
    };
  };

  private onMouseDown = (event: MouseEvent): void => {
    const button = event.button as MouseButton;
    if (!this.state.mouseButtons.get(button)) {
      this.state.mouseButtonsPressed.add(button);
    }
    this.state.mouseButtons.set(button, true);
  };

  private onMouseUp = (event: MouseEvent): void => {
    this.state.mouseButtons.set(event.button as MouseButton, false);
  };

  private onWheel = (event: WheelEvent): void => {
    this.state.mouseWheel += event.deltaY;
  };

  private onTouchStart = (event: TouchEvent): void => {
    const touch = event.touches[0];
    if (touch) {
      const rect = this.canvas.getBoundingClientRect();
      this.state.mousePosition = {
        x: touch.clientX - rect.left,
        y: touch.clientY - rect.top,
      };
      this.state.mouseButtonsPressed.add(0);
      this.state.mouseButtons.set(0, true);
    }
  };

  private onTouchEnd = (): void => {
    this.state.mouseButtons.set(0, false);
  };

  private onTouchMove = (event: TouchEvent): void => {
    const touch = event.touches[0];
    if (touch) {
      const rect = this.canvas.getBoundingClientRect();
      this.state.mousePosition = {
        x: touch.clientX - rect.left,
        y: touch.clientY - rect.top,
      };
    }
  };

  // Query methods
  isKeyDown(keyCode: KeyCode): boolean {
    return this.state.keys.get(keyCode) ?? false;
  }

  isKeyPressed(keyCode: KeyCode): boolean {
    return this.state.keysPressed.has(keyCode);
  }

  isKeyReleased(keyCode: KeyCode): boolean {
    return this.state.keysReleased.has(keyCode);
  }

  isMouseButtonDown(button: MouseButton = 0): boolean {
    return this.state.mouseButtons.get(button) ?? false;
  }

  isMouseButtonPressed(button: MouseButton = 0): boolean {
    return this.state.mouseButtonsPressed.has(button);
  }

  getMousePosition(): { x: number; y: number } {
    return { ...this.state.mousePosition };
  }

  getMouseWheel(): number {
    return this.state.mouseWheel;
  }

  // Clear per-frame state
  endFrame(): void {
    this.state.keysPressed.clear();
    this.state.keysReleased.clear();
    this.state.mouseButtonsPressed.clear();
    this.state.mouseWheel = 0;
  }

  destroy(): void {
    window.removeEventListener('keydown', this.onKeyDown);
    window.removeEventListener('keyup', this.onKeyUp);
    this.canvas.removeEventListener('mousemove', this.onMouseMove);
    this.canvas.removeEventListener('mousedown', this.onMouseDown);
    this.canvas.removeEventListener('mouseup', this.onMouseUp);
  }
}
```

---

## 98.7 State Machine สำหรับ Game States

```typescript
// src/state/state-machine.ts

export interface GameState {
  name: string;
  onEnter?(previousState?: string): void;
  onExit?(nextState?: string): void;
  update(deltaTime: number): void;
  render(): void;
}

export interface StateTransition {
  from: string;
  to: string;
  condition?: () => boolean;
  onTransition?: () => void;
}

export class StateMachine {
  private states = new Map<string, GameState>();
  private currentState: GameState | null = null;
  private transitions: StateTransition[] = [];
  private history: string[] = [];

  addState(state: GameState): this {
    this.states.set(state.name, state);
    return this;
  }

  addTransition(transition: StateTransition): this {
    this.transitions.push(transition);
    return this;
  }

  setState(stateName: string): void {
    const newState = this.states.get(stateName);
    if (!newState) {
      throw new Error(`State '${stateName}' not found`);
    }

    const previousStateName = this.currentState?.name;

    this.currentState?.onExit?.(stateName);

    if (previousStateName) {
      this.history.push(previousStateName);
    }

    this.currentState = newState;
    this.currentState.onEnter?.(previousStateName);
  }

  goBack(): void {
    const previousState = this.history.pop();
    if (previousState) {
      this.setState(previousState);
    }
  }

  getCurrentState(): GameState | null {
    return this.currentState;
  }

  getCurrentStateName(): string | null {
    return this.currentState?.name ?? null;
  }

  checkTransitions(): void {
    if (!this.currentState) return;

    for (const transition of this.transitions) {
      if (
        transition.from === this.currentState.name &&
        (!transition.condition || transition.condition())
      ) {
        transition.onTransition?.();
        this.setState(transition.to);
        break;
      }
    }
  }

  update(deltaTime: number): void {
    this.checkTransitions();
    this.currentState?.update(deltaTime);
  }

  render(): void {
    this.currentState?.render();
  }
}

// ตัวอย่าง Game States
export function createMainMenuState(canvas: CanvasManager, machine: StateMachine): GameState {
  return {
    name: 'main-menu',
    onEnter() {
      console.log('Entering main menu');
    },
    update(_deltaTime: number): void {
      // Handle menu input
    },
    render(): void {
      canvas.clear();
      canvas.drawText('SPACE SHOOTER', canvas.width / 2, canvas.height / 3, {
        color: '#e94560',
        font: 'bold 48px Arial',
        align: 'center',
      });
      canvas.drawText('Press SPACE to start', canvas.width / 2, canvas.height / 2, {
        color: '#ffffff',
        font: '24px Arial',
        align: 'center',
      });
    },
  };
}

// type annotation fix
import type { CanvasManager } from '../canvas/canvas-manager';
```

---

## 98.8 Asset Loading กับ Types

```typescript
// src/assets/asset-loader.ts

export interface Asset<T> {
  key: string;
  data: T;
  loaded: boolean;
}

export interface LoadProgress {
  loaded: number;
  total: number;
  percentage: number;
  currentAsset: string;
}

export class AssetLoader {
  private images = new Map<string, HTMLImageElement>();
  private sounds = new Map<string, AudioBuffer>();
  private json = new Map<string, unknown>();
  private audioContext: AudioContext | null = null;

  private loadingQueue: Array<() => Promise<void>> = [];
  private totalAssets = 0;
  private loadedAssets = 0;
  private onProgress?: (progress: LoadProgress) => void;

  constructor(onProgress?: (progress: LoadProgress) => void) {
    this.onProgress = onProgress;
  }

  addImage(key: string, url: string): this {
    this.totalAssets++;
    this.loadingQueue.push(async () => {
      this.images.set(key, await this.loadImage(url));
      this.reportProgress(key);
    });
    return this;
  }

  addJSON<T>(key: string, url: string): this {
    this.totalAssets++;
    this.loadingQueue.push(async () => {
      const response = await fetch(url);
      const data = (await response.json()) as T;
      this.json.set(key, data);
      this.reportProgress(key);
    });
    return this;
  }

  async loadAll(): Promise<void> {
    const results = await Promise.allSettled(
      this.loadingQueue.map((task) => task())
    );

    const failed = results.filter((r) => r.status === 'rejected');
    if (failed.length > 0) {
      console.error(`Failed to load ${failed.length} assets`);
    }

    this.loadingQueue = [];
  }

  private loadImage(url: string): Promise<HTMLImageElement> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve(img);
      img.onerror = () => reject(new Error(`Failed to load image: ${url}`));
      img.src = url;
    });
  }

  private reportProgress(assetKey: string): void {
    this.loadedAssets++;
    this.onProgress?.({
      loaded: this.loadedAssets,
      total: this.totalAssets,
      percentage: (this.loadedAssets / this.totalAssets) * 100,
      currentAsset: assetKey,
    });
  }

  getImage(key: string): HTMLImageElement {
    const image = this.images.get(key);
    if (!image) throw new Error(`Image '${key}' not loaded`);
    return image;
  }

  getJSON<T>(key: string): T {
    const data = this.json.get(key);
    if (!data) throw new Error(`JSON '${key}' not loaded`);
    return data as T;
  }

  isLoaded(): boolean {
    return this.loadedAssets === this.totalAssets;
  }
}
```

---

## 98.9 Complete Space Shooter Game

```typescript
// src/games/space-shooter.ts
import { World } from '../ecs/world';
import { CanvasManager } from '../canvas/canvas-manager';
import { GameLoop } from '../core/game-loop';
import { InputManager } from '../input/input-manager';
import {
  MovementSystem,
  RenderSystem,
  CollisionSystem,
  HealthSystem,
} from '../ecs/systems';
import type {
  TransformComponent,
  VelocityComponent,
  RenderComponent,
  ColliderComponent,
  HealthComponent,
  TagComponent,
} from '../ecs/types';

export class SpaceShooter {
  private world: World;
  private canvas: CanvasManager;
  private loop: GameLoop;
  private input: InputManager;
  private movementSystem: MovementSystem;
  private renderSystem: RenderSystem;
  private collisionSystem: CollisionSystem;
  private healthSystem: HealthSystem;

  private playerEntity = 0;
  private score = 0;
  private level = 1;
  private gameOver = false;

  private enemySpawnTimer = 0;
  private enemySpawnInterval = 2.0;
  private bulletCooldown = 0;
  private readonly BULLET_COOLDOWN = 0.2;

  constructor(canvasId: string) {
    this.canvas = new CanvasManager(canvasId, {
      width: 800,
      height: 600,
      backgroundColor: '#0a0a1a',
    });

    this.world = new World();
    this.input = new InputManager(this.canvas.getCanvas());

    this.movementSystem = new MovementSystem();
    this.renderSystem = new RenderSystem(this.canvas);
    this.collisionSystem = new CollisionSystem();
    this.healthSystem = new HealthSystem();

    this.loop = new GameLoop(
      {
        update: this.update.bind(this),
        render: this.render.bind(this),
      },
      { targetFPS: 60 }
    );

    this.createPlayer();
    this.createStars();
  }

  private createPlayer(): void {
    this.playerEntity = this.world.createEntity();

    this.world.addComponent<TransformComponent>(this.playerEntity, {
      type: 'transform',
      position: { x: this.canvas.width / 2, y: this.canvas.height - 80 },
      rotation: 0,
      scale: { x: 1, y: 1 },
    });

    this.world.addComponent<VelocityComponent>(this.playerEntity, {
      type: 'velocity',
      x: 0,
      y: 0,
      maxSpeed: 300,
    });

    this.world.addComponent<RenderComponent>(this.playerEntity, {
      type: 'render',
      shape: 'rect',
      color: '#00ff88',
      width: 32,
      height: 40,
      visible: true,
      layer: 10,
    });

    this.world.addComponent<ColliderComponent>(this.playerEntity, {
      type: 'collider',
      width: 28,
      height: 36,
      isTrigger: false,
      tag: 'player',
    });

    this.world.addComponent<HealthComponent>(this.playerEntity, {
      type: 'health',
      current: 3,
      max: 3,
      invincible: false,
      invincibleTimer: 0,
    });

    this.world.addComponent<TagComponent>(this.playerEntity, {
      type: 'tag',
      tags: new Set(['player']),
    });
  }

  private createStars(): void {
    for (let i = 0; i < 100; i++) {
      const starEntity = this.world.createEntity();
      const brightness = Math.random() * 0.8 + 0.2;
      const size = Math.random() * 2 + 1;

      this.world.addComponent<TransformComponent>(starEntity, {
        type: 'transform',
        position: {
          x: Math.random() * this.canvas.width,
          y: Math.random() * this.canvas.height,
        },
        rotation: 0,
        scale: { x: 1, y: 1 },
      });

      this.world.addComponent<VelocityComponent>(starEntity, {
        type: 'velocity',
        x: 0,
        y: 20 + Math.random() * 30,
      });

      this.world.addComponent<RenderComponent>(starEntity, {
        type: 'render',
        shape: 'circle',
        color: `rgba(255, 255, 255, ${brightness})`,
        width: size * 2,
        height: size * 2,
        visible: true,
        layer: 0,
      });

      this.world.addComponent<TagComponent>(starEntity, {
        type: 'tag',
        tags: new Set(['star']),
      });
    }
  }

  private spawnEnemy(): void {
    const enemyEntity = this.world.createEntity();
    const x = Math.random() * (this.canvas.width - 40) + 20;

    this.world.addComponent<TransformComponent>(enemyEntity, {
      type: 'transform',
      position: { x, y: -20 },
      rotation: Math.PI,
      scale: { x: 1, y: 1 },
    });

    this.world.addComponent<VelocityComponent>(enemyEntity, {
      type: 'velocity',
      x: (Math.random() - 0.5) * 50,
      y: 100 + this.level * 20,
    });

    this.world.addComponent<RenderComponent>(enemyEntity, {
      type: 'render',
      shape: 'rect',
      color: '#e94560',
      width: 30,
      height: 36,
      visible: true,
      layer: 5,
    });

    this.world.addComponent<ColliderComponent>(enemyEntity, {
      type: 'collider',
      width: 26,
      height: 32,
      isTrigger: false,
      tag: 'enemy',
    });

    this.world.addComponent<HealthComponent>(enemyEntity, {
      type: 'health',
      current: 1,
      max: 1,
      invincible: false,
      invincibleTimer: 0,
    });

    this.world.addComponent<TagComponent>(enemyEntity, {
      type: 'tag',
      tags: new Set(['enemy']),
    });
  }

  private fireBullet(): void {
    const playerTransform = this.world.getComponent<TransformComponent>(
      this.playerEntity,
      'transform'
    );
    if (!playerTransform) return;

    const bulletEntity = this.world.createEntity();

    this.world.addComponent<TransformComponent>(bulletEntity, {
      type: 'transform',
      position: { ...playerTransform.position },
      rotation: 0,
      scale: { x: 1, y: 1 },
    });

    this.world.addComponent<VelocityComponent>(bulletEntity, {
      type: 'velocity',
      x: 0,
      y: -500,
    });

    this.world.addComponent<RenderComponent>(bulletEntity, {
      type: 'render',
      shape: 'rect',
      color: '#ffff00',
      width: 4,
      height: 12,
      visible: true,
      layer: 8,
    });

    this.world.addComponent<ColliderComponent>(bulletEntity, {
      type: 'collider',
      width: 4,
      height: 12,
      isTrigger: true,
      tag: 'bullet',
    });

    this.world.addComponent<TagComponent>(bulletEntity, {
      type: 'tag',
      tags: new Set(['bullet', 'player-bullet']),
    });
  }

  private handleInput(deltaTime: number): void {
    const velocity = this.world.getComponent<VelocityComponent>(
      this.playerEntity,
      'velocity'
    );
    if (!velocity) return;

    const speed = 300;
    velocity.x = 0;
    velocity.y = 0;

    if (this.input.isKeyDown('ArrowLeft') || this.input.isKeyDown('KeyA')) {
      velocity.x = -speed;
    }
    if (this.input.isKeyDown('ArrowRight') || this.input.isKeyDown('KeyD')) {
      velocity.x = speed;
    }
    if (this.input.isKeyDown('ArrowUp') || this.input.isKeyDown('KeyW')) {
      velocity.y = -speed;
    }
    if (this.input.isKeyDown('ArrowDown') || this.input.isKeyDown('KeyS')) {
      velocity.y = speed;
    }

    // Fire
    this.bulletCooldown -= deltaTime;
    if (
      (this.input.isKeyDown('Space') || this.input.isMouseButtonDown(0)) &&
      this.bulletCooldown <= 0
    ) {
      this.fireBullet();
      this.bulletCooldown = this.BULLET_COOLDOWN;
    }
  }

  private boundPlayer(): void {
    const transform = this.world.getComponent<TransformComponent>(
      this.playerEntity,
      'transform'
    );
    if (!transform) return;

    const margin = 20;
    transform.position.x = Math.max(
      margin,
      Math.min(this.canvas.width - margin, transform.position.x)
    );
    transform.position.y = Math.max(
      margin,
      Math.min(this.canvas.height - margin, transform.position.y)
    );
  }

  private wrapStars(): void {
    const starEntities = this.world.query('transform', 'tag').filter((id) => {
      const tag = this.world.getComponent<TagComponent>(id, 'tag');
      return tag?.tags.has('star') ?? false;
    });

    for (const starId of starEntities) {
      const transform = this.world.getComponent<TransformComponent>(starId, 'transform')!;
      if (transform.position.y > this.canvas.height + 10) {
        transform.position.y = -10;
        transform.position.x = Math.random() * this.canvas.width;
      }
    }
  }

  private cleanupOffScreenEntities(): void {
    const entities = this.world.query('transform', 'tag');

    for (const entityId of entities) {
      const tag = this.world.getComponent<TagComponent>(entityId, 'tag');
      if (!tag?.tags.has('bullet') && !tag?.tags.has('enemy')) continue;

      const transform = this.world.getComponent<TransformComponent>(entityId, 'transform')!;
      const pos = transform.position;

      if (
        pos.y < -50 ||
        pos.y > this.canvas.height + 50 ||
        pos.x < -50 ||
        pos.x > this.canvas.width + 50
      ) {
        this.world.destroyEntity(entityId);
      }
    }
  }

  private handleCollisions(): void {
    const collisions = this.collisionSystem.getCollisionEvents();

    for (const collision of collisions) {
      const tagA = this.world.getComponent<TagComponent>(collision.a, 'tag');
      const tagB = this.world.getComponent<TagComponent>(collision.b, 'tag');

      if (!tagA || !tagB) continue;

      // Bullet hits enemy
      const bulletHitsEnemy = (
        tagA.tags.has('player-bullet') && tagB.tags.has('enemy')
      ) || (
        tagB.tags.has('player-bullet') && tagA.tags.has('enemy')
      );

      if (bulletHitsEnemy) {
        const bulletId = tagA.tags.has('player-bullet') ? collision.a : collision.b;
        const enemyId = tagA.tags.has('enemy') ? collision.a : collision.b;

        this.world.destroyEntity(bulletId);
        this.world.destroyEntity(enemyId);
        this.score += 100;
        console.log(`Score: ${this.score}`);
      }

      // Enemy hits player
      const enemyHitsPlayer = (
        tagA.tags.has('enemy') && tagB.tags.has('player')
      ) || (
        tagB.tags.has('enemy') && tagA.tags.has('player')
      );

      if (enemyHitsPlayer) {
        const enemyId = tagA.tags.has('enemy') ? collision.a : collision.b;
        const health = this.world.getComponent<HealthComponent>(
          this.playerEntity,
          'health'
        );

        if (health && !health.invincible) {
          health.current--;
          health.invincible = true;
          health.invincibleTimer = 2.0;
          this.world.destroyEntity(enemyId);

          if (health.current <= 0) {
            this.gameOver = true;
          }
        }
      }
    }
  }

  private update(deltaTime: number): void {
    if (this.gameOver) return;

    this.handleInput(deltaTime);
    this.movementSystem.update(this.world, deltaTime);
    this.collisionSystem.update(this.world, deltaTime);
    this.healthSystem.update(this.world, deltaTime);

    this.handleCollisions();
    this.boundPlayer();
    this.wrapStars();
    this.cleanupOffScreenEntities();

    // Spawn enemies
    this.enemySpawnTimer += deltaTime;
    if (this.enemySpawnTimer >= this.enemySpawnInterval) {
      this.spawnEnemy();
      this.enemySpawnTimer = 0;
      this.enemySpawnInterval = Math.max(0.5, this.enemySpawnInterval - 0.01);
    }

    // Level up
    if (this.score > this.level * 1000) {
      this.level++;
      console.log(`Level up! Level ${this.level}`);
    }

    this.world.processDestructions();
    this.input.endFrame();
  }

  private render(): void {
    this.canvas.clear();
    this.renderSystem.update(this.world, 0);

    // HUD
    const health = this.world.getComponent<HealthComponent>(this.playerEntity, 'health');
    const hearts = health ? '❤️'.repeat(health.current) : '';

    this.canvas.drawText(`Score: ${this.score}`, 10, 10, {
      color: '#ffffff',
      font: '20px Arial',
    });

    this.canvas.drawText(`Level: ${this.level}`, 10, 35, {
      color: '#ffff00',
      font: '20px Arial',
    });

    this.canvas.drawText(hearts, this.canvas.width - 100, 10, {
      font: '20px Arial',
    });

    this.canvas.drawText(`FPS: ${this.loop.getFPS()}`, this.canvas.width - 80, this.canvas.height - 25, {
      color: '#888888',
      font: '14px monospace',
    });

    if (this.gameOver) {
      this.canvas.drawText('GAME OVER', this.canvas.width / 2, this.canvas.height / 2 - 30, {
        color: '#e94560',
        font: 'bold 48px Arial',
        align: 'center',
      });
      this.canvas.drawText(`Final Score: ${this.score}`, this.canvas.width / 2, this.canvas.height / 2 + 20, {
        color: '#ffffff',
        font: '24px Arial',
        align: 'center',
      });
    }
  }

  start(): void {
    this.loop.start();
  }

  stop(): void {
    this.loop.stop();
    this.input.destroy();
  }
}

// Entry point
const game = new SpaceShooter('gameCanvas');
game.start();
```

---

## 98.10 Phaser.js กับ TypeScript

```typescript
// src/phaser/phaser-game.ts
// npm install phaser
// npm install -D @types/node

import Phaser from 'phaser';

// Type-safe scene configuration
interface PlayerData {
  score: number;
  lives: number;
  level: number;
}

class MainMenuScene extends Phaser.Scene {
  constructor() {
    super({ key: 'MainMenu' });
  }

  preload(): void {
    this.load.setBaseURL('/assets');
    // this.load.image('logo', 'logo.png');
    // this.load.audio('menuMusic', 'menu.mp3');
  }

  create(): void {
    const { width, height } = this.cameras.main;

    this.add.text(width / 2, height / 3, 'MY PHASER GAME', {
      fontSize: '48px',
      color: '#e94560',
      fontStyle: 'bold',
    }).setOrigin(0.5);

    const startButton = this.add.text(width / 2, height / 2, 'START GAME', {
      fontSize: '32px',
      color: '#ffffff',
      backgroundColor: '#333333',
      padding: { x: 20, y: 10 },
    })
      .setOrigin(0.5)
      .setInteractive({ useHandCursor: true });

    startButton
      .on('pointerover', () => startButton.setStyle({ color: '#ffff00' }))
      .on('pointerout', () => startButton.setStyle({ color: '#ffffff' }))
      .on('pointerdown', () => {
        this.scene.start('Game', { score: 0, lives: 3, level: 1 } as PlayerData);
      });
  }
}

class GameScene extends Phaser.Scene {
  private player!: Phaser.Physics.Arcade.Sprite;
  private enemies!: Phaser.Physics.Arcade.Group;
  private bullets!: Phaser.Physics.Arcade.Group;
  private cursors!: Phaser.Types.Input.Keyboard.CursorKeys;
  private fireKey!: Phaser.Input.Keyboard.Key;
  private scoreText!: Phaser.GameObjects.Text;
  private playerData!: PlayerData;
  private fireCooldown = 0;

  constructor() {
    super({ key: 'Game' });
  }

  create(data: PlayerData): void {
    this.playerData = data;

    // Background
    this.add.rectangle(0, 0, 800, 600, 0x0a0a1a)
      .setOrigin(0, 0);

    // Physics groups
    this.enemies = this.physics.add.group();
    this.bullets = this.physics.add.group({
      maxSize: 50,
      runChildUpdate: true,
    });

    // Player
    this.player = this.physics.add.sprite(400, 520, 'player');
    this.player.setCollideWorldBounds(true);

    // Input
    this.cursors = this.input.keyboard!.createCursorKeys();
    this.fireKey = this.input.keyboard!.addKey(Phaser.Input.Keyboard.KeyCodes.SPACE);

    // Collisions
    this.physics.add.overlap(
      this.bullets,
      this.enemies,
      this.onBulletHitEnemy as Phaser.Types.Physics.Arcade.ArcadePhysicsCallback,
      undefined,
      this
    );

    this.physics.add.overlap(
      this.player,
      this.enemies,
      this.onPlayerHitEnemy as Phaser.Types.Physics.Arcade.ArcadePhysicsCallback,
      undefined,
      this
    );

    // Score
    this.scoreText = this.add.text(10, 10, `Score: ${this.playerData.score}`, {
      fontSize: '20px',
      color: '#ffffff',
    });

    // Enemy spawner
    this.time.addEvent({
      delay: 1000,
      callback: this.spawnEnemy,
      callbackScope: this,
      loop: true,
    });
  }

  private spawnEnemy(): void {
    const x = Phaser.Math.Between(50, 750);
    const enemy = this.enemies.create(x, -20, 'enemy') as Phaser.Physics.Arcade.Sprite;
    
    if (enemy) {
      enemy.setVelocityY(100 + this.playerData.level * 20);
    }
  }

  private onBulletHitEnemy(
    bullet: Phaser.Types.Physics.Arcade.GameObjectWithBody,
    enemy: Phaser.Types.Physics.Arcade.GameObjectWithBody
  ): void {
    (bullet as Phaser.Physics.Arcade.Sprite).destroy();
    (enemy as Phaser.Physics.Arcade.Sprite).destroy();
    this.playerData.score += 100;
    this.scoreText.setText(`Score: ${this.playerData.score}`);
  }

  private onPlayerHitEnemy(
    _player: Phaser.Types.Physics.Arcade.GameObjectWithBody,
    enemy: Phaser.Types.Physics.Arcade.GameObjectWithBody
  ): void {
    (enemy as Phaser.Physics.Arcade.Sprite).destroy();
    this.playerData.lives--;

    if (this.playerData.lives <= 0) {
      this.scene.start('GameOver', this.playerData);
    }
  }

  update(time: number, _delta: number): void {
    // Movement
    if (this.cursors.left.isDown) {
      this.player.setVelocityX(-300);
    } else if (this.cursors.right.isDown) {
      this.player.setVelocityX(300);
    } else {
      this.player.setVelocityX(0);
    }

    // Fire
    if (Phaser.Input.Keyboard.JustDown(this.fireKey)) {
      this.fireBullet();
    }

    // Cleanup off-screen enemies
    this.enemies.getChildren().forEach((enemy) => {
      const sprite = enemy as Phaser.Physics.Arcade.Sprite;
      if (sprite.y > 650) {
        sprite.destroy();
      }
    });
  }

  private fireBullet(): void {
    const bullet = this.bullets.get(
      this.player.x,
      this.player.y - 20,
      'bullet'
    ) as Phaser.Physics.Arcade.Sprite | null;

    if (bullet) {
      bullet.setActive(true);
      bullet.setVisible(true);
      bullet.setVelocityY(-400);
    }
  }
}

// Game configuration
const config: Phaser.Types.Core.GameConfig = {
  type: Phaser.AUTO,
  width: 800,
  height: 600,
  backgroundColor: '#0a0a1a',
  physics: {
    default: 'arcade',
    arcade: {
      gravity: { x: 0, y: 0 },
      debug: false,
    },
  },
  scene: [MainMenuScene, GameScene],
};

new Phaser.Game(config);
```

---

## บทสรุป Part 98

ในบทนี้เราได้เรียนรู้:

1. **Canvas API** - การวาดและ render กับ TypeScript
2. **Vector Math** - Vector2 class สำหรับ game physics
3. **Game Loop** - Fixed/variable timestep loops
4. **Entity-Component-System (ECS)** - Pattern สำหรับ game architecture
5. **Input Handling** - Keyboard, mouse, touch
6. **State Machine** - จัดการ game states
7. **Asset Loading** - โหลดและจัดการ assets
8. **Complete Game** - Space Shooter ที่ทำงานได้จริง
9. **Phaser.js** - Framework สำหรับ game development

TypeScript ช่วยให้การพัฒนาเกมมีความปลอดภัยมากขึ้น โดยเฉพาะเมื่อ codebase ใหญ่ขึ้น ECS pattern และ type system ทำงานร่วมกันได้ดีมาก

---

## แบบฝึกหัด

1. เพิ่ม power-ups ให้กับ Space Shooter โดยใช้ ECS components
2. สร้าง Puzzle game แบบ Match-3 ด้วย TypeScript
3. สร้าง platform game ด้วย Phaser.js
4. Implement save/load system ด้วย localStorage
5. สร้าง multiplayer game ด้วย WebSockets
