---
name: ecs-component-patterns
description: Advanced ECS component design patterns including tag components, singleton components, shared data, and component pools
---

# ECS Component Patterns

## When to Use

Use this skill when:
- Designing components for game entities
- Optimizing component memory layout
- Implementing special component types
- Managing shared game data
- Creating reusable component libraries

## Core Principles

1. **Data-Only Components**: No logic, only data
2. **Small Components**: Single responsibility principle
3. **Composition**: Combine simple components for complex behavior
4. **Tag Components**: Empty components as flags
5. **Shared Data**: Singleton components for global state
6. **Type Safety**: TypeScript for all component definitions

## Implementation

### 1. Basic Component Types

```typescript
// components/Transform.ts
import { Component } from '../core/Component';

export class Transform implements Component {
  constructor(
    public x: number = 0,
    public y: number = 0,
    public z: number = 0,
    public rotationX: number = 0,
    public rotationY: number = 0,
    public rotationZ: number = 0,
    public scaleX: number = 1,
    public scaleY: number = 1,
    public scaleZ: number = 1
  ) {}

  setPosition(x: number, y: number, z: number): this {
    this.x = x;
    this.y = y;
    this.z = z;
    return this;
  }

  setRotation(x: number, y: number, z: number): this {
    this.rotationX = x;
    this.rotationY = y;
    this.rotationZ = z;
    return this;
  }

  setScale(x: number, y: number, z: number): this {
    this.scaleX = x;
    this.scaleY = y;
    this.scaleZ = z;
    return this;
  }

  clone(): Transform {
    return new Transform(
      this.x, this.y, this.z,
      this.rotationX, this.rotationY, this.rotationZ,
      this.scaleX, this.scaleY, this.scaleZ
    );
  }
}
```

```typescript
// components/Velocity.ts
export class Velocity implements Component {
  constructor(
    public vx: number = 0,
    public vy: number = 0,
    public vz: number = 0
  ) {}

  get magnitude(): number {
    return Math.sqrt(this.vx * this.vx + this.vy * this.vy + this.vz * this.vz);
  }

  set(x: number, y: number, z: number): this {
    this.vx = x;
    this.vy = y;
    this.vz = z;
    return this;
  }

  add(x: number, y: number, z: number): this {
    this.vx += x;
    this.vy += y;
    this.vz += z;
    return this;
  }

  scale(factor: number): this {
    this.vx *= factor;
    this.vy *= factor;
    this.vz *= factor;
    return this;
  }

  normalize(): this {
    const mag = this.magnitude;
    if (mag > 0) {
      this.scale(1 / mag);
    }
    return this;
  }
}
```

### 2. Tag Components

```typescript
// components/Tags.ts

/**
 * Tag component - no data, just marks entity
 */
export class Player implements Component {
  readonly __brand = 'Player' as const;
}

export class Enemy implements Component {
  readonly __brand = 'Enemy' as const;
}

export class Dead implements Component {
  readonly __brand = 'Dead' as const;
}

export class Selected implements Component {
  readonly __brand = 'Selected' as const;
}

export class Invincible implements Component {
  readonly __brand = 'Invincible' as const;

  constructor(public duration: number) {}
}

// Helper to check if entity has tag
export function hasTag<T extends Component>(
  componentManager: ComponentManager,
  entity: EntityId,
  tagClass: ComponentClass<T>
): boolean {
  return componentManager.has(entity, tagClass);
}
```

### 3. Singleton Components (Shared Data)

```typescript
// components/GameState.ts
export class GameState implements Component {
  private static instance?: GameState;

  score: number = 0;
  level: number = 1;
  timeElapsed: number = 0;
  isPaused: boolean = false;
  highScore: number = 0;

  private constructor() {}

  static getInstance(): GameState {
    if (!GameState.instance) {
      GameState.instance = new GameState();
    }
    return GameState.instance;
  }

  reset(): void {
    this.score = 0;
    this.level = 1;
    this.timeElapsed = 0;
    this.isPaused = false;
  }

  addScore(points: number): void {
    this.score += points;
    if (this.score > this.highScore) {
      this.highScore = this.score;
    }
  }
}
```

```typescript
// components/InputState.ts
export class InputState implements Component {
  private static instance?: InputState;

  keys = new Set<string>();
  mouseX: number = 0;
  mouseY: number = 0;
  mouseDown: boolean = false;

  touches: Array<{ id: number; x: number; y: number }> = [];

  private constructor() {}

  static getInstance(): InputState {
    if (!InputState.instance) {
      InputState.instance = new InputState();
    }
    return InputState.instance;
  }

  isKeyPressed(key: string): boolean {
    return this.keys.has(key.toLowerCase());
  }

  getAxis(positive: string, negative: string): number {
    return (this.isKeyPressed(positive) ? 1 : 0) + (this.isKeyPressed(negative) ? -1 : 0);
  }
}
```

### 4. Composite Components

```typescript
// components/Health.ts
export class Health implements Component {
  current: number;
  maximum: number;
  regenerationRate: number;
  lastDamageTime: number = 0;

  constructor(max: number, regenRate: number = 0) {
    this.current = max;
    this.maximum = max;
    this.regenerationRate = regenRate;
  }

  get isDead(): boolean {
    return this.current <= 0;
  }

  get isFullHealth(): boolean {
    return this.current >= this.maximum;
  }

  get healthPercent(): number {
    return this.current / this.maximum;
  }

  damage(amount: number, time: number): void {
    this.current = Math.max(0, this.current - amount);
    this.lastDamageTime = time;
  }

  heal(amount: number): void {
    this.current = Math.min(this.maximum, this.current + amount);
  }

  regenerate(deltaTime: number, currentTime: number): void {
    if (this.regenerationRate > 0 && !this.isFullHealth) {
      // Only regen if not damaged recently (3 seconds)
      if (currentTime - this.lastDamageTime > 3) {
        this.heal(this.regenerationRate * deltaTime);
      }
    }
  }
}
```

```typescript
// components/Stats.ts
export class Stats implements Component {
  constructor(
    public strength: number = 10,
    public defense: number = 5,
    public speed: number = 5,
    public luck: number = 5
  ) {}

  get attackPower(): number {
    return this.strength * 2 + this.luck;
  }

  get damageReduction(): number {
    return this.defense / (this.defense + 100);
  }

  get moveSpeed(): number {
    return 5 + this.speed * 0.5;
  }

  levelUp(): void {
    this.strength += 2;
    this.defense += 1;
    this.speed += 1;
    this.luck += 1;
  }
}
```

### 5. Relationship Components

```typescript
// components/Parent.ts
export class Parent implements Component {
  constructor(public children: EntityId[] = []) {}

  addChild(child: EntityId): void {
    if (!this.children.includes(child)) {
      this.children.push(child);
    }
  }

  removeChild(child: EntityId): void {
    const index = this.children.indexOf(child);
    if (index !== -1) {
      this.children.splice(index, 1);
    }
  }

  hasChild(child: EntityId): boolean {
    return this.children.includes(child);
  }
}

export class Child implements Component {
  constructor(public parent: EntityId | null = null) {}
}
```

### 6. Temporal Components

```typescript
// components/Lifetime.ts
export class Lifetime implements Component {
  elapsed: number = 0;

  constructor(public duration: number) {}

  get isExpired(): boolean {
    return this.elapsed >= this.duration;
  }

  get percentComplete(): number {
    return Math.min(this.elapsed / this.duration, 1);
  }

  update(deltaTime: number): void {
    this.elapsed += deltaTime;
  }
}

export class Cooldown implements Component {
  elapsed: number;

  constructor(public duration: number, startOnCooldown: boolean = false) {
    this.elapsed = startOnCooldown ? 0 : duration;
  }

  get isReady(): boolean {
    return this.elapsed >= this.duration;
  }

  get percentReady(): number {
    return Math.min(this.elapsed / this.duration, 1);
  }

  trigger(): void {
    this.elapsed = 0;
  }

  update(deltaTime: number): void {
    if (!this.isReady) {
      this.elapsed += deltaTime;
    }
  }
}
```

### 7. Component Pools

```typescript
// ComponentPool.ts
export class ComponentPool<T extends Component> {
  private available: T[] = [];
  private inUse = new Set<T>();
  private factory: () => T;

  constructor(factory: () => T, initialSize: number = 100) {
    this.factory = factory;

    for (let i = 0; i < initialSize; i++) {
      this.available.push(factory());
    }
  }

  acquire(): T {
    let component = this.available.pop();

    if (!component) {
      component = this.factory();
      console.warn(`ComponentPool exhausted, creating new component`);
    }

    this.inUse.add(component);
    return component;
  }

  release(component: T): void {
    if (this.inUse.has(component)) {
      this.inUse.delete(component);
      this.available.push(component);
    }
  }

  releaseAll(): void {
    this.inUse.forEach(component => {
      this.available.push(component);
    });
    this.inUse.clear();
  }

  get poolSize(): number {
    return this.available.length + this.inUse.size;
  }

  get availableCount(): number {
    return this.available.length;
  }

  get inUseCount(): number {
    return this.inUse.size;
  }
}
```

### 8. Component Registry

```typescript
// ComponentRegistry.ts
export class ComponentRegistry {
  private static pools = new Map<string, ComponentPool<any>>();

  static registerPool<T extends Component>(
    name: string,
    factory: () => T,
    size: number = 100
  ): void {
    this.pools.set(name, new ComponentPool(factory, size));
  }

  static getPool<T extends Component>(name: string): ComponentPool<T> | undefined {
    return this.pools.get(name);
  }

  static acquire<T extends Component>(name: string): T | undefined {
    const pool = this.pools.get(name);
    return pool?.acquire();
  }

  static release(name: string, component: Component): void {
    const pool = this.pools.get(name);
    pool?.release(component);
  }

  static releaseAll(): void {
    this.pools.forEach(pool => pool.releaseAll());
  }
}

// Usage
ComponentRegistry.registerPool('Transform', () => new Transform(), 1000);
ComponentRegistry.registerPool('Velocity', () => new Velocity(), 1000);

const transform = ComponentRegistry.acquire<Transform>('Transform');
// ... use component ...
ComponentRegistry.release('Transform', transform!);
```

## Usage Examples

```typescript
// Tag components
world.addComponent(player, Player, new Player());
world.addComponent(player, Invincible, new Invincible(3.0)); // 3 seconds

// Check tags
if (hasTag(componentManager, entity, Player)) {
  console.log('This is the player!');
}

// Singleton components
const gameState = GameState.getInstance();
gameState.addScore(100);

const input = InputState.getInstance();
if (input.isKeyPressed('w')) {
  // Move forward
}

// Component pooling
const transform = ComponentRegistry.acquire<Transform>('Transform');
transform.setPosition(0, 1, 0);

world.addComponent(entity, Transform, transform);

// Release when entity is destroyed
const oldTransform = world.getComponent(entity, Transform);
if (oldTransform) {
  ComponentRegistry.release('Transform', oldTransform);
}

// Temporal components
const lifetime = new Lifetime(5.0); // Die after 5 seconds
world.addComponent(entity, Lifetime, lifetime);

const cooldown = new Cooldown(1.0); // 1 second cooldown
if (cooldown.isReady) {
  // Fire weapon
  cooldown.trigger();
}
```

## Checklist

- [ ] Keep components data-only (no logic)
- [ ] Use tag components for entity flags
- [ ] Implement singleton components for global state
- [ ] Create component pools for frequently used components
- [ ] Design components with composition in mind
- [ ] Add helper methods to components (getters/setters)
- [ ] Use TypeScript for type safety
- [ ] Document component relationships
- [ ] Test component memory usage
- [ ] Profile component access patterns
- [ ] Implement component cloning where needed
- [ ] Add component validation in development

## Common Pitfalls

1. **Logic in components**: Components are data, systems are logic
2. **Large monolithic components**: Break into smaller pieces
3. **Circular dependencies**: Components should be independent
4. **Not pooling components**: Memory allocations in game loop
5. **Mutable shared references**: Components should own their data
6. **Deep component hierarchies**: Flatten where possible

## Performance Tips

- Keep components small (< 100 bytes ideal)
- Use primitive types over objects when possible
- Pool frequently created/destroyed components
- Use tag components instead of boolean flags
- Implement component cloning for fast copies
- Use TypedArrays for numeric-heavy components
- Cache component lookups in systems
- Batch component additions/removals

## Related Skills

- `ecs-architecture` - Core ECS implementation
- `ecs-system-patterns` - System design patterns
- `typescript-game-types` - Type-safe component patterns
- `ecs-performance` - Performance optimization
