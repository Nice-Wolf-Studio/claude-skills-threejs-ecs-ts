---
name: typescript-game-types
description: Type-safe patterns for game development with TypeScript, including branded types, strict null checks, and performance-focused type design
---

# TypeScript Game Types

## When to Use

Use this skill when:
- Setting up type definitions for a new game project
- Refactoring JavaScript game code to TypeScript
- Implementing type-safe APIs for game systems
- Ensuring compile-time safety for game logic

## Core Principles

1. **Branded Types**: Use nominal typing for entity IDs, resource handles, and unique identifiers
2. **Discriminated Unions**: Model game states and events with tagged unions
3. **Strict Null Safety**: Enable strict TypeScript settings and handle undefined explicitly
4. **Type Guards**: Create runtime type checking for dynamic game data
5. **Const Assertions**: Use `as const` for game constants and enums

## Implementation

### 1. Branded Types for IDs

```typescript
// Generic branded type utility
export type Branded<T, Brand extends string> = T & { readonly __brand: Brand };

// Entity and resource IDs
export type EntityId = Branded<number, 'EntityId'>;
export type TextureId = Branded<string, 'TextureId'>;
export type SoundId = Branded<string, 'SoundId'>;
export type LevelId = Branded<string, 'LevelId'>;

// Factory functions with type guards
export function createEntityId(value: number): EntityId {
  return value as EntityId;
}

export function isEntityId(value: unknown): value is EntityId {
  return typeof value === 'number';
}

// Usage
const playerId: EntityId = createEntityId(1);
const enemyId: EntityId = createEntityId(2);

// Prevents accidental ID mixing at compile time
function getEntity(id: EntityId): Entity { /* ... */ }

// ✓ Type-safe
getEntity(playerId);

// ✗ Compile error: number not assignable to EntityId
// getEntity(123);
```

### 2. Discriminated Unions for Game States

```typescript
// Game state modeling
export type GameState =
  | { type: 'loading'; progress: number }
  | { type: 'menu'; selectedOption: number }
  | { type: 'playing'; level: LevelId; score: number; timeRemaining: number }
  | { type: 'paused'; level: LevelId; score: number }
  | { type: 'gameOver'; finalScore: number; reason: 'defeated' | 'completed' | 'timeout' };

export class GameStateManager {
  private state: GameState = { type: 'loading', progress: 0 };

  setState(newState: GameState): void {
    this.state = newState;
    this.handleStateChange(newState);
  }

  getState(): GameState {
    return this.state;
  }

  private handleStateChange(state: GameState): void {
    // TypeScript narrows the type in each case
    switch (state.type) {
      case 'loading':
        console.log(`Loading: ${state.progress}%`);
        break;
      case 'menu':
        console.log(`Menu option: ${state.selectedOption}`);
        break;
      case 'playing':
        console.log(`Playing level ${state.level}, score: ${state.score}`);
        break;
      case 'paused':
        console.log(`Paused at level ${state.level}`);
        break;
      case 'gameOver':
        console.log(`Game over: ${state.reason}, score: ${state.finalScore}`);
        break;
      default:
        // Exhaustiveness check - compile error if we miss a case
        const _exhaustive: never = state;
        throw new Error(`Unhandled state: ${_exhaustive}`);
    }
  }
}
```

### 3. Event System with Type Safety

```typescript
// Event type definitions
export type GameEvent =
  | { type: 'entitySpawned'; entity: EntityId; position: Vector3 }
  | { type: 'entityDestroyed'; entity: EntityId }
  | { type: 'collision'; entityA: EntityId; entityB: EntityId; point: Vector3 }
  | { type: 'scoreChanged'; delta: number; newTotal: number }
  | { type: 'healthChanged'; entity: EntityId; oldValue: number; newValue: number }
  | { type: 'levelCompleted'; level: LevelId; time: number };

// Type-safe event listener
export type EventListener<T extends GameEvent> = (event: T) => void;

export class EventBus {
  private listeners = new Map<GameEvent['type'], Set<EventListener<any>>>();

  // Subscribe to specific event type
  on<T extends GameEvent['type']>(
    eventType: T,
    listener: EventListener<Extract<GameEvent, { type: T }>>
  ): () => void {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, new Set());
    }

    this.listeners.get(eventType)!.add(listener);

    // Return unsubscribe function
    return () => {
      this.listeners.get(eventType)?.delete(listener);
    };
  }

  emit<T extends GameEvent>(event: T): void {
    const listeners = this.listeners.get(event.type);
    if (listeners) {
      listeners.forEach(listener => listener(event));
    }
  }

  clear(): void {
    this.listeners.clear();
  }
}

// Usage
const eventBus = new EventBus();

// Type-safe subscription - TypeScript knows event structure
const unsubscribe = eventBus.on('collision', (event) => {
  // event is typed as Extract<GameEvent, { type: 'collision' }>
  console.log(`Collision at (${event.point.x}, ${event.point.y}, ${event.point.z})`);
  console.log(`Between entities: ${event.entityA} and ${event.entityB}`);
});

// Type-safe emission
eventBus.emit({
  type: 'collision',
  entityA: createEntityId(1),
  entityB: createEntityId(2),
  point: { x: 0, y: 0, z: 0 }
});
```

### 4. Component Type Safety

```typescript
// Component interface with brand
export interface Component {
  readonly __componentBrand?: never;
}

// Const assertion for component types
export const ComponentTypes = {
  Transform: 'Transform',
  Velocity: 'Velocity',
  Health: 'Health',
  Renderable: 'Renderable',
} as const;

export type ComponentType = typeof ComponentTypes[keyof typeof ComponentTypes];

// Type-safe component classes
export class Transform implements Component {
  readonly type = ComponentTypes.Transform;

  constructor(
    public x: number = 0,
    public y: number = 0,
    public z: number = 0
  ) {}
}

export class Velocity implements Component {
  readonly type = ComponentTypes.Velocity;

  constructor(
    public vx: number = 0,
    public vy: number = 0,
    public vz: number = 0
  ) {}
}

// Component registry with type mapping
export interface ComponentRegistry {
  [ComponentTypes.Transform]: Transform;
  [ComponentTypes.Velocity]: Velocity;
  [ComponentTypes.Health]: Health;
  [ComponentTypes.Renderable]: Renderable;
}

// Type-safe component access
export class TypedComponentManager {
  private components = new Map<EntityId, Partial<ComponentRegistry>>();

  add<K extends keyof ComponentRegistry>(
    entity: EntityId,
    type: K,
    component: ComponentRegistry[K]
  ): void {
    if (!this.components.has(entity)) {
      this.components.set(entity, {});
    }
    this.components.get(entity)![type] = component;
  }

  get<K extends keyof ComponentRegistry>(
    entity: EntityId,
    type: K
  ): ComponentRegistry[K] | undefined {
    return this.components.get(entity)?.[type];
  }

  has<K extends keyof ComponentRegistry>(entity: EntityId, type: K): boolean {
    return this.components.get(entity)?.[type] !== undefined;
  }
}

// Usage - fully type-safe
const manager = new TypedComponentManager();
const entity = createEntityId(1);

manager.add(entity, ComponentTypes.Transform, new Transform(1, 2, 3));

// TypeScript knows this is Transform | undefined
const transform = manager.get(entity, ComponentTypes.Transform);
if (transform) {
  console.log(transform.x, transform.y, transform.z); // All typed correctly
}
```

### 5. Vector and Math Types

```typescript
// Immutable vector types
export interface Vector2 {
  readonly x: number;
  readonly y: number;
}

export interface Vector3 {
  readonly x: number;
  readonly y: number;
  readonly z: number;
}

// Vector utilities with pure functions
export namespace VectorUtils {
  export function add(a: Vector3, b: Vector3): Vector3 {
    return { x: a.x + b.x, y: a.y + b.y, z: a.z + b.z };
  }

  export function subtract(a: Vector3, b: Vector3): Vector3 {
    return { x: a.x - b.x, y: a.y - b.y, z: a.z - b.z };
  }

  export function scale(v: Vector3, scalar: number): Vector3 {
    return { x: v.x * scalar, y: v.y * scalar, z: v.z * scalar };
  }

  export function length(v: Vector3): number {
    return Math.sqrt(v.x * v.x + v.y * v.y + v.z * v.z);
  }

  export function normalize(v: Vector3): Vector3 {
    const len = length(v);
    return len > 0 ? scale(v, 1 / len) : { x: 0, y: 0, z: 0 };
  }

  export function dot(a: Vector3, b: Vector3): number {
    return a.x * b.x + a.y * b.y + a.z * b.z;
  }
}

// Bounding box with readonly properties
export interface AABB {
  readonly min: Vector3;
  readonly max: Vector3;
}

export namespace AABBUtils {
  export function contains(box: AABB, point: Vector3): boolean {
    return (
      point.x >= box.min.x && point.x <= box.max.x &&
      point.y >= box.min.y && point.y <= box.max.y &&
      point.z >= box.min.z && point.z <= box.max.z
    );
  }

  export function intersects(a: AABB, b: AABB): boolean {
    return (
      a.min.x <= b.max.x && a.max.x >= b.min.x &&
      a.min.y <= b.max.y && a.max.y >= b.min.y &&
      a.min.z <= b.max.z && a.max.z >= b.min.z
    );
  }
}
```

### 6. Configuration with Strict Types

```typescript
// Game configuration with const assertions
export const GameConfig = {
  player: {
    speed: 5,
    jumpForce: 10,
    maxHealth: 100,
  },
  enemies: {
    goblin: {
      speed: 2,
      damage: 10,
      health: 30,
    },
    orc: {
      speed: 1.5,
      damage: 20,
      health: 50,
    },
  },
  world: {
    gravity: -9.81,
    tickRate: 60,
  },
} as const;

// Type extracted from config
export type EnemyType = keyof typeof GameConfig.enemies;

// Type-safe config access
export function getEnemyConfig(type: EnemyType) {
  return GameConfig.enemies[type];
}

// Readonly prevents mutation
// GameConfig.player.speed = 10; // Error: Cannot assign to 'speed' because it is a read-only property
```

### 7. Type Guards for Runtime Safety

```typescript
// Type guards for unknown data (e.g., from network, files)
export function isVector3(value: unknown): value is Vector3 {
  return (
    typeof value === 'object' &&
    value !== null &&
    'x' in value && typeof value.x === 'number' &&
    'y' in value && typeof value.y === 'number' &&
    'z' in value && typeof value.z === 'number'
  );
}

export function isGameEvent(value: unknown): value is GameEvent {
  if (typeof value !== 'object' || value === null || !('type' in value)) {
    return false;
  }

  const event = value as { type: unknown };
  return typeof event.type === 'string';
}

// Safe deserialization
export function deserializeVector3(data: unknown): Vector3 | null {
  if (isVector3(data)) {
    return data;
  }
  console.warn('Invalid Vector3 data:', data);
  return null;
}
```

## TypeScript Configuration

Enable strict settings in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

## Checklist

- [ ] Enable strict TypeScript settings
- [ ] Use branded types for unique identifiers
- [ ] Model states with discriminated unions
- [ ] Implement type-safe event system
- [ ] Create type guards for runtime validation
- [ ] Use const assertions for constants
- [ ] Define readonly types for immutable data
- [ ] Add exhaustiveness checks for switches
- [ ] Type all component interfaces
- [ ] Document complex generic types

## Common Pitfalls

1. **Loose typing**: Avoid `any`; use `unknown` and type guards instead
2. **Mutable config**: Use `as const` for configuration objects
3. **Missing null checks**: Enable strictNullChecks and handle undefined
4. **Type assertions**: Prefer type guards over `as` casts
5. **Implicit any**: Set noImplicitAny to catch untyped variables

## Performance Tips

- Use const assertions to enable literal type inference
- Avoid complex generic types in hot paths
- Use branded types (zero runtime cost) instead of classes for IDs
- Prefer interfaces over type aliases for object types (better error messages)
- Use readonly to enable compiler optimizations

## Related Skills

- `ecs-architecture` - ECS implementation with TypeScript
- `typescript-ecs-types` - Advanced ECS typing patterns
- `typescript-performance` - Performance optimization techniques
