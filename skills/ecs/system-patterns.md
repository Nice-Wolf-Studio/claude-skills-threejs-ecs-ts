---
name: ecs-system-patterns
description: Common system patterns in ECS including update systems, reactive systems, interval systems, and system priorities
---

# ECS System Patterns

## When to Use

Use this skill when:
- Implementing game logic systems
- Designing system execution order
- Optimizing system performance
- Creating reactive or event-driven systems
- Managing system dependencies
- Building modular game mechanics

## Core Principles

1. **Single Responsibility**: Each system does one thing well
2. **Component-Driven**: Systems operate on components, not entities directly
3. **Stateless When Possible**: Systems should be stateless or minimally stateful
4. **Efficient Queries**: Cache queries and iterate efficiently
5. **Priority-Based Execution**: Control system execution order
6. **Performance-Aware**: Profile and optimize hot paths

## System Types

### 1. Update Systems (Every Frame)

```typescript
// systems/UpdateSystemBase.ts
import { Entity } from '../core/Entity';
import { World } from '../core/World';

export interface System {
  priority: number;
  enabled: boolean;
  update(world: World, deltaTime: number): void;
  cleanup?(): void;
}

export abstract class UpdateSystem implements System {
  public priority = 0;
  public enabled = true;

  abstract update(world: World, deltaTime: number): void;

  cleanup?(): void {
    // Override if needed
  }
}

// Example: Movement system
import { Transform } from '../components/Transform';
import { Velocity } from '../components/Velocity';

export class MovementSystem extends UpdateSystem {
  priority = 10; // Run early

  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, Velocity]>([Transform, Velocity]);

    for (const entity of entities) {
      const [transform, velocity] = entity.getComponents(Transform, Velocity);

      transform.position.x += velocity.x * deltaTime;
      transform.position.y += velocity.y * deltaTime;
      transform.position.z += velocity.z * deltaTime;

      // Apply drag
      velocity.x *= 0.98;
      velocity.y *= 0.98;
      velocity.z *= 0.98;
    }
  }
}
```

### 2. Reactive Systems (Event-Driven)

```typescript
// systems/ReactiveSystemBase.ts
import { World } from '../core/World';
import { Entity } from '../core/Entity';

export interface ComponentEvent {
  entity: Entity;
  component: any;
  type: 'added' | 'removed' | 'changed';
}

export abstract class ReactiveSystem implements System {
  public priority = 0;
  public enabled = true;
  protected events: ComponentEvent[] = [];

  update(world: World, deltaTime: number): void {
    // Process accumulated events
    for (const event of this.events) {
      this.onComponentEvent(event);
    }
    this.events = [];
  }

  onComponentAdded(entity: Entity, component: any): void {
    this.events.push({ entity, component, type: 'added' });
  }

  onComponentRemoved(entity: Entity, component: any): void {
    this.events.push({ entity, component, type: 'removed' });
  }

  onComponentChanged(entity: Entity, component: any): void {
    this.events.push({ entity, component, type: 'changed' });
  }

  protected abstract onComponentEvent(event: ComponentEvent): void;
}

// Example: Animation trigger system
import { AnimationController } from '../components/AnimationController';
import { Velocity } from '../components/Velocity';

export class AnimationTriggerSystem extends ReactiveSystem {
  priority = 20;

  protected onComponentEvent(event: ComponentEvent): void {
    if (event.type === 'changed' && event.component instanceof Velocity) {
      const animator = event.entity.getComponent(AnimationController);
      if (!animator) return;

      const velocity = event.component as Velocity;
      const speed = Math.sqrt(velocity.x ** 2 + velocity.y ** 2 + velocity.z ** 2);

      if (speed > 0.1) {
        animator.play('walk');
      } else {
        animator.play('idle');
      }
    }
  }
}
```

### 3. Interval Systems (Fixed Frequency)

```typescript
// systems/IntervalSystemBase.ts
export abstract class IntervalSystem implements System {
  public priority = 0;
  public enabled = true;
  protected interval: number;
  protected accumulator = 0;

  constructor(frequency: number = 1) {
    this.interval = 1 / frequency; // Convert Hz to seconds
  }

  update(world: World, deltaTime: number): void {
    this.accumulator += deltaTime;

    while (this.accumulator >= this.interval) {
      this.tick(world);
      this.accumulator -= this.interval;
    }
  }

  protected abstract tick(world: World): void;
}

// Example: AI decision system (updates 10 times per second)
import { AIController } from '../components/AIController';
import { Transform } from '../components/Transform';

export class AIDecisionSystem extends IntervalSystem {
  priority = 30;

  constructor() {
    super(10); // 10 Hz (10 times per second)
  }

  protected tick(world: World): void {
    const entities = world.query<[AIController, Transform]>([AIController, Transform]);

    for (const entity of entities) {
      const [ai, transform] = entity.getComponents(AIController, Transform);

      // Expensive AI calculations
      const target = this.findNearestTarget(world, transform.position);
      if (target) {
        ai.currentTarget = target;
        ai.state = 'chase';
      } else {
        ai.currentTarget = null;
        ai.state = 'patrol';
      }
    }
  }

  private findNearestTarget(world: World, position: Vector3): Entity | null {
    // ... search logic
    return null;
  }
}
```

### 4. Cleanup Systems (End of Frame)

```typescript
// systems/CleanupSystemBase.ts
export abstract class CleanupSystem implements System {
  public priority = 1000; // Run last
  public enabled = true;

  abstract update(world: World, deltaTime: number): void;
}

// Example: Remove dead entities
import { Health } from '../components/Health';
import { Lifetime } from '../components/Lifetime';

export class EntityCleanupSystem extends CleanupSystem {
  update(world: World, deltaTime: number): void {
    // Remove entities with 0 health
    const deadEntities = world.query<[Health]>([Health]).filter((entity) => {
      const health = entity.getComponent(Health);
      return health.current <= 0;
    });

    for (const entity of deadEntities) {
      world.destroyEntity(entity);
    }

    // Remove entities past their lifetime
    const expiredEntities = world.query<[Lifetime]>([Lifetime]);

    for (const entity of expiredEntities) {
      const lifetime = entity.getComponent(Lifetime);
      lifetime.age += deltaTime;

      if (lifetime.age >= lifetime.maxAge) {
        world.destroyEntity(entity);
      }
    }
  }
}
```

### 5. System Manager

```typescript
// core/SystemManager.ts
import { System } from '../systems/UpdateSystemBase';
import { World } from './World';

export class SystemManager {
  private systems: System[] = [];
  private systemsByType = new Map<string, System>();

  add(system: System): void {
    this.systems.push(system);
    this.systemsByType.set(system.constructor.name, system);

    // Sort by priority (lower numbers run first)
    this.systems.sort((a, b) => a.priority - b.priority);
  }

  remove(systemType: new () => System): void {
    const name = systemType.name;
    const system = this.systemsByType.get(name);

    if (system) {
      const index = this.systems.indexOf(system);
      if (index !== -1) {
        this.systems.splice(index, 1);
      }
      this.systemsByType.delete(name);

      if (system.cleanup) {
        system.cleanup();
      }
    }
  }

  get<T extends System>(systemType: new () => T): T | undefined {
    return this.systemsByType.get(systemType.name) as T;
  }

  update(world: World, deltaTime: number): void {
    for (const system of this.systems) {
      if (!system.enabled) continue;

      system.update(world, deltaTime);
    }
  }

  cleanup(): void {
    for (const system of this.systems) {
      if (system.cleanup) {
        system.cleanup();
      }
    }
    this.systems = [];
    this.systemsByType.clear();
  }
}
```

## Common System Patterns

### Pattern 1: System Groups

```typescript
// systems/SystemGroup.ts
export class SystemGroup implements System {
  public priority = 0;
  public enabled = true;
  private systems: System[] = [];

  constructor(systems: System[]) {
    this.systems = systems;
  }

  add(system: System): void {
    this.systems.push(system);
    this.systems.sort((a, b) => a.priority - b.priority);
  }

  update(world: World, deltaTime: number): void {
    if (!this.enabled) return;

    for (const system of this.systems) {
      if (system.enabled) {
        system.update(world, deltaTime);
      }
    }
  }

  cleanup(): void {
    for (const system of this.systems) {
      if (system.cleanup) {
        system.cleanup();
      }
    }
  }
}

// Usage: Group related systems
const physicsGroup = new SystemGroup([
  new CollisionDetectionSystem(),
  new PhysicsSystem(),
  new CollisionResponseSystem(),
]);
physicsGroup.priority = 5;
systemManager.add(physicsGroup);
```

### Pattern 2: Conditional Systems

```typescript
// systems/ConditionalSystem.ts
export class ConditionalSystem implements System {
  public priority = 0;
  public enabled = true;
  private innerSystem: System;
  private condition: () => boolean;

  constructor(system: System, condition: () => boolean) {
    this.innerSystem = system;
    this.condition = condition;
  }

  update(world: World, deltaTime: number): void {
    if (this.condition()) {
      this.innerSystem.update(world, deltaTime);
    }
  }

  cleanup(): void {
    if (this.innerSystem.cleanup) {
      this.innerSystem.cleanup();
    }
  }
}

// Usage: Only run AI when game is not paused
const aiSystem = new ConditionalSystem(
  new AIDecisionSystem(),
  () => !game.isPaused
);
```

### Pattern 3: Parallel Systems

```typescript
// systems/ParallelSystem.ts
export class ParallelSystem implements System {
  public priority = 0;
  public enabled = true;
  private systems: System[] = [];

  constructor(systems: System[]) {
    this.systems = systems;
  }

  async update(world: World, deltaTime: number): Promise<void> {
    if (!this.enabled) return;

    // Run all systems in parallel
    const promises = this.systems
      .filter((s) => s.enabled)
      .map((s) => Promise.resolve(s.update(world, deltaTime)));

    await Promise.all(promises);
  }

  cleanup(): void {
    for (const system of this.systems) {
      if (system.cleanup) {
        system.cleanup();
      }
    }
  }
}

// Usage: Run independent systems in parallel
const parallelSystems = new ParallelSystem([
  new ParticleSystem(),
  new SoundSystem(),
  new DebugRenderSystem(),
]);
```

## Usage Examples

```typescript
// Example: Setting up a game with systems
import { World } from './core/World';
import { SystemManager } from './core/SystemManager';

// Create world and system manager
const world = new World();
const systemManager = new SystemManager();

// Add systems in priority order
systemManager.add(new InputSystem()); // Priority 0 (first)
systemManager.add(new AIDecisionSystem()); // Priority 10
systemManager.add(new MovementSystem()); // Priority 20
systemManager.add(new CollisionSystem()); // Priority 30
systemManager.add(new AnimationSystem()); // Priority 40
systemManager.add(new RenderSystem()); // Priority 50
systemManager.add(new EntityCleanupSystem()); // Priority 1000 (last)

// Game loop
function gameLoop(currentTime: number): void {
  const deltaTime = (currentTime - lastTime) / 1000;
  lastTime = currentTime;

  // Update all systems
  systemManager.update(world, deltaTime);

  requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);

// Example: Dynamic system management
function togglePhysics(enabled: boolean): void {
  const physicsSystem = systemManager.get(PhysicsSystem);
  if (physicsSystem) {
    physicsSystem.enabled = enabled;
  }
}

// Example: System dependencies
class DependentSystem extends UpdateSystem {
  private requiredSystem: PhysicsSystem;

  constructor(systemManager: SystemManager) {
    super();
    this.requiredSystem = systemManager.get(PhysicsSystem)!;

    if (!this.requiredSystem) {
      throw new Error('DependentSystem requires PhysicsSystem');
    }
  }

  update(world: World, deltaTime: number): void {
    // Access data from required system
    const collisions = this.requiredSystem.getCollisions();
    // ...
  }
}
```

## System Priority Guidelines

```typescript
// Recommended priority ranges
export enum SystemPriority {
  // Input (0-9)
  Input = 0,

  // AI/Logic (10-19)
  AI = 10,
  Logic = 15,

  // Physics (20-39)
  Movement = 20,
  Collision = 30,
  PhysicsResponse = 35,

  // Animation (40-49)
  Animation = 40,
  Particles = 45,

  // Rendering (50-99)
  Render = 50,
  PostProcess = 60,
  UI = 70,

  // Cleanup (1000+)
  Cleanup = 1000,
}

// Usage
export class MySystem extends UpdateSystem {
  priority = SystemPriority.Movement;
}
```

## Checklist

- [ ] Choose appropriate system type (update, reactive, interval, cleanup)
- [ ] Set correct priority for execution order
- [ ] Implement efficient component queries
- [ ] Handle system dependencies properly
- [ ] Test system with different deltaTime values
- [ ] Profile system performance
- [ ] Add enable/disable functionality
- [ ] Implement cleanup if needed
- [ ] Document system purpose and requirements
- [ ] Consider parallelization opportunities

## Common Pitfalls

1. **Wrong execution order**: Systems run in wrong priority
2. **Tight coupling**: Systems depend on each other's internal state
3. **Heavy update systems**: Expensive operations every frame
4. **No cleanup**: Systems leak resources
5. **Stateful systems**: Hard to test and maintain
6. **Skipping deltaTime**: Fixed updates break on slow frames
7. **Query caching**: Creating queries every frame

## Performance Tips

### System Optimization
- Cache component queries outside update loop
- Use interval systems for expensive operations
- Batch similar operations together
- Skip disabled entities early
- Use system groups for related logic

### Profiling Systems
```typescript
export class ProfiledSystem implements System {
  public priority = 0;
  public enabled = true;
  private innerSystem: System;
  public lastUpdateTime = 0;

  constructor(system: System) {
    this.innerSystem = system;
  }

  update(world: World, deltaTime: number): void {
    const start = performance.now();
    this.innerSystem.update(world, deltaTime);
    this.lastUpdateTime = performance.now() - start;
  }
}

// Usage
const profiledAI = new ProfiledSystem(new AISystem());
console.log(`AI took ${profiledAI.lastUpdateTime}ms`);
```

## Related Skills

- `ecs-architecture` - Overall ECS structure
- `ecs-component-patterns` - Component design
- `ecs-queries` - Efficient entity queries
- `ecs-events` - Event-driven architecture
- `ecs-performance` - ECS optimization

## References

- Entity Component System patterns
- Game Programming Patterns (Robert Nystrom)
- Data-Oriented Design principles
