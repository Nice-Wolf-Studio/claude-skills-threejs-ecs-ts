---
name: ecs-performance
description: Performance optimization for ECS including archetype optimization, memory pooling, cache-friendly data structures, and profiling techniques
---

# ECS Performance Optimization

## When to Use

Use this skill when:
- Scaling to thousands of entities
- Optimizing frame time
- Reducing memory allocations
- Improving cache coherency
- Debugging performance issues
- Targeting mobile devices

## Core Principles

1. **Data-Oriented**: Organize data for cache efficiency
2. **Minimize Allocations**: Pool and reuse objects
3. **Batch Operations**: Process similar entities together
4. **Profile First**: Measure before optimizing
5. **Lazy Evaluation**: Compute only when needed
6. **Archetype-Based**: Group entities by components

## Performance Optimization Techniques

### 1. Archetype Optimization

```typescript
// core/Archetype.ts
export interface Archetype {
  signature: string;
  entities: Entity[];
  componentData: Map<ComponentConstructor<any>, any[]>;
}

export class ArchetypeStorage {
  private archetypes = new Map<string, Archetype>();

  getArchetype(componentTypes: ComponentConstructor<any>[]): Archetype {
    const signature = this.getSignature(componentTypes);
    let archetype = this.archetypes.get(signature);

    if (!archetype) {
      archetype = {
        signature,
        entities: [],
        componentData: new Map(),
      };

      // Pre-allocate component arrays
      for (const type of componentTypes) {
        archetype.componentData.set(type, []);
      }

      this.archetypes.set(signature, archetype);
    }

    return archetype;
  }

  addEntity(entity: Entity, componentTypes: ComponentConstructor<any>[]): void {
    const archetype = this.getArchetype(componentTypes);
    archetype.entities.push(entity);

    // Add component data to arrays
    for (const type of componentTypes) {
      const component = entity.getComponent(type);
      const array = archetype.componentData.get(type)!;
      array.push(component);
    }
  }

  removeEntity(entity: Entity, componentTypes: ComponentConstructor<any>[]): void {
    const archetype = this.getArchetype(componentTypes);
    const index = archetype.entities.indexOf(entity);

    if (index !== -1) {
      // Remove from entity array
      archetype.entities.splice(index, 1);

      // Remove from component arrays
      for (const [type, array] of archetype.componentData) {
        array.splice(index, 1);
      }
    }
  }

  // Iterate cache-friendly
  iterateArchetype<T extends any[]>(
    componentTypes: ComponentConstructor<any>[],
    callback: (components: T, entity: Entity) => void
  ): void {
    const archetype = this.getArchetype(componentTypes);
    const componentArrays = componentTypes.map((type) =>
      archetype.componentData.get(type)
    );

    const length = archetype.entities.length;

    for (let i = 0; i < length; i++) {
      const components = componentArrays.map((arr) => arr![i]) as T;
      callback(components, archetype.entities[i]);
    }
  }

  private getSignature(componentTypes: ComponentConstructor<any>[]): string {
    return componentTypes
      .map((t) => t.name)
      .sort()
      .join(':');
  }
}
```

### 2. Component Pooling

```typescript
// core/ComponentPool.ts
export class ComponentPool<T> {
  private pool: T[] = [];
  private factory: () => T;
  private reset: (component: T) => void;

  constructor(factory: () => T, reset: (component: T) => void) {
    this.factory = factory;
    this.reset = reset;
  }

  acquire(): T {
    if (this.pool.length > 0) {
      return this.pool.pop()!;
    }
    return this.factory();
  }

  release(component: T): void {
    this.reset(component);
    this.pool.push(component);
  }

  prewarm(count: number): void {
    for (let i = 0; i < count; i++) {
      this.pool.push(this.factory());
    }
  }

  clear(): void {
    this.pool = [];
  }

  getPoolSize(): number {
    return this.pool.length;
  }
}

// Example usage
const velocityPool = new ComponentPool<Velocity>(
  () => new Velocity(),
  (v) => {
    v.x = 0;
    v.y = 0;
    v.z = 0;
  }
);

// Prewarm pool
velocityPool.prewarm(1000);

// Acquire from pool
const velocity = velocityPool.acquire();
entity.addComponent(velocity);

// Release back to pool
entity.removeComponent(Velocity);
velocityPool.release(velocity);
```

### 3. Entity Pooling

```typescript
// core/EntityPool.ts
export class EntityPool {
  private pool: Entity[] = [];
  private world: World;

  constructor(world: World) {
    this.world = world;
  }

  acquire(): Entity {
    if (this.pool.length > 0) {
      const entity = this.pool.pop()!;
      entity.active = true;
      return entity;
    }
    return this.world.createEntity();
  }

  release(entity: Entity): void {
    // Remove all components
    entity.removeAllComponents();
    entity.active = false;
    this.pool.push(entity);
  }

  prewarm(count: number): void {
    for (let i = 0; i < count; i++) {
      const entity = this.world.createEntity();
      entity.active = false;
      this.pool.push(entity);
    }
  }

  clear(): void {
    for (const entity of this.pool) {
      this.world.destroyEntity(entity);
    }
    this.pool = [];
  }
}

// Usage
const bulletPool = new EntityPool(world);
bulletPool.prewarm(100); // Pre-create 100 bullets

// Spawn bullet
function spawnBullet(position: Vector3): void {
  const bullet = bulletPool.acquire();
  bullet.addComponent(new Transform(position));
  bullet.addComponent(new Velocity(0, 0, -10));
  bullet.addComponent(new Lifetime(5));
}

// Destroy bullet (return to pool)
function destroyBullet(bullet: Entity): void {
  bulletPool.release(bullet);
}
```

### 4. Packed Array Storage

```typescript
// core/PackedArray.ts
export class PackedArray<T> {
  private dense: T[] = [];
  private sparse: number[] = [];
  private indices: number[] = [];

  add(index: number, value: T): void {
    const denseIndex = this.dense.length;
    this.dense.push(value);
    this.indices.push(index);
    this.sparse[index] = denseIndex;
  }

  remove(index: number): void {
    const denseIndex = this.sparse[index];
    if (denseIndex === undefined) return;

    // Swap with last element
    const lastIndex = this.dense.length - 1;
    if (denseIndex !== lastIndex) {
      const lastElement = this.dense[lastIndex];
      const lastSparseIndex = this.indices[lastIndex];

      this.dense[denseIndex] = lastElement;
      this.indices[denseIndex] = lastSparseIndex;
      this.sparse[lastSparseIndex] = denseIndex;
    }

    // Remove last element
    this.dense.pop();
    this.indices.pop();
    delete this.sparse[index];
  }

  get(index: number): T | undefined {
    const denseIndex = this.sparse[index];
    return denseIndex !== undefined ? this.dense[denseIndex] : undefined;
  }

  has(index: number): boolean {
    return this.sparse[index] !== undefined;
  }

  getDense(): T[] {
    return this.dense;
  }

  forEach(callback: (value: T, index: number) => void): void {
    for (let i = 0; i < this.dense.length; i++) {
      callback(this.dense[i], this.indices[i]);
    }
  }
}

// Usage: Store components in packed arrays
class PackedComponentStorage {
  private storage = new Map<ComponentConstructor<any>, PackedArray<any>>();

  addComponent(entityId: number, component: any): void {
    const type = component.constructor;
    let array = this.storage.get(type);

    if (!array) {
      array = new PackedArray();
      this.storage.set(type, array);
    }

    array.add(entityId, component);
  }

  removeComponent(entityId: number, type: ComponentConstructor<any>): void {
    const array = this.storage.get(type);
    if (array) {
      array.remove(entityId);
    }
  }

  getComponent(entityId: number, type: ComponentConstructor<any>): any {
    const array = this.storage.get(type);
    return array ? array.get(entityId) : undefined;
  }
}
```

### 5. System Profiling

```typescript
// profiling/SystemProfiler.ts
export interface SystemStats {
  name: string;
  updateTime: number;
  avgUpdateTime: number;
  maxUpdateTime: number;
  callCount: number;
}

export class SystemProfiler {
  private stats = new Map<System, SystemStats>();
  private enabled = true;

  enable(): void {
    this.enabled = true;
  }

  disable(): void {
    this.enabled = false;
  }

  wrap(system: System): System {
    if (!this.enabled) return system;

    const stats: SystemStats = {
      name: system.constructor.name,
      updateTime: 0,
      avgUpdateTime: 0,
      maxUpdateTime: 0,
      callCount: 0,
    };

    this.stats.set(system, stats);

    const originalUpdate = system.update.bind(system);

    system.update = (world: World, deltaTime: number) => {
      const start = performance.now();
      originalUpdate(world, deltaTime);
      const time = performance.now() - start;

      stats.updateTime = time;
      stats.callCount++;
      stats.avgUpdateTime = stats.avgUpdateTime * 0.9 + time * 0.1;
      stats.maxUpdateTime = Math.max(stats.maxUpdateTime, time);
    };

    return system;
  }

  getStats(): SystemStats[] {
    return Array.from(this.stats.values()).sort(
      (a, b) => b.avgUpdateTime - a.avgUpdateTime
    );
  }

  printStats(): void {
    console.log('System Performance Stats:');
    console.log('-'.repeat(60));
    console.log('System'.padEnd(30) + 'Avg (ms)'.padEnd(15) + 'Max (ms)');
    console.log('-'.repeat(60));

    for (const stats of this.getStats()) {
      console.log(
        stats.name.padEnd(30) +
          stats.avgUpdateTime.toFixed(2).padEnd(15) +
          stats.maxUpdateTime.toFixed(2)
      );
    }
  }

  reset(): void {
    for (const stats of this.stats.values()) {
      stats.avgUpdateTime = 0;
      stats.maxUpdateTime = 0;
      stats.callCount = 0;
    }
  }
}

// Usage
const profiler = new SystemProfiler();

const movementSystem = profiler.wrap(new MovementSystem());
const physicsSystem = profiler.wrap(new PhysicsSystem());

// Later...
profiler.printStats();
```

### 6. Query Caching

```typescript
// core/QueryCache.ts
export class QueryCache {
  private cache = new Map<string, Entity[]>();
  private dirty = new Set<string>();

  get<T extends any[]>(
    key: string,
    componentTypes: ComponentConstructor<any>[],
    world: World
  ): Entity[] {
    if (this.dirty.has(key)) {
      // Refresh cache
      const entities = world.query(componentTypes).entities;
      this.cache.set(key, entities);
      this.dirty.delete(key);
    }

    return this.cache.get(key) ?? [];
  }

  invalidate(key: string): void {
    this.dirty.add(key);
  }

  invalidateAll(): void {
    for (const key of this.cache.keys()) {
      this.dirty.add(key);
    }
  }

  clear(): void {
    this.cache.clear();
    this.dirty.clear();
  }
}

// Usage in system
class OptimizedSystem extends UpdateSystem {
  private queryCache: QueryCache;
  private cacheKey = 'movement';

  constructor(queryCache: QueryCache) {
    super();
    this.queryCache = queryCache;
  }

  update(world: World, deltaTime: number): void {
    const entities = this.queryCache.get(
      this.cacheKey,
      [Transform, Velocity],
      world
    );

    for (const entity of entities) {
      const [transform, velocity] = entity.getComponents(Transform, Velocity);
      // Process...
    }
  }
}
```

### 7. Batch Processing

```typescript
// systems/BatchProcessor.ts
export class BatchProcessor {
  static batchSize = 1000;

  static processBatches<T>(
    items: T[],
    process: (item: T, index: number) => void,
    maxTimeMs: number = 16
  ): boolean {
    const startTime = performance.now();
    let processed = 0;

    for (let i = 0; i < items.length; i++) {
      process(items[i], i);
      processed++;

      // Check time budget every batch
      if (processed % this.batchSize === 0) {
        const elapsed = performance.now() - startTime;
        if (elapsed >= maxTimeMs) {
          return false; // Not finished, continue next frame
        }
      }
    }

    return true; // Finished
  }
}

// Usage: Spread expensive work across frames
class ExpensiveSystem extends UpdateSystem {
  private currentBatch = 0;
  private batchSize = 100;

  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, Mesh]>([Transform, Mesh]).entities;

    const start = this.currentBatch * this.batchSize;
    const end = Math.min(start + this.batchSize, entities.length);

    for (let i = start; i < end; i++) {
      const entity = entities[i];
      // Expensive operation...
      this.expensiveOperation(entity);
    }

    this.currentBatch++;
    if (this.currentBatch * this.batchSize >= entities.length) {
      this.currentBatch = 0; // Reset for next cycle
    }
  }

  private expensiveOperation(entity: Entity): void {
    // Heavy computation
  }
}
```

## Performance Measurement

### Memory Profiling

```typescript
// profiling/MemoryProfiler.ts
export class MemoryProfiler {
  private baseline: number = 0;

  start(): void {
    if (performance.memory) {
      this.baseline = performance.memory.usedJSHeapSize;
    }
  }

  measure(): number {
    if (performance.memory) {
      return performance.memory.usedJSHeapSize - this.baseline;
    }
    return 0;
  }

  printMemoryUsage(): void {
    if (performance.memory) {
      const used = performance.memory.usedJSHeapSize / 1048576;
      const total = performance.memory.totalJSHeapSize / 1048576;
      const limit = performance.memory.jsHeapSizeLimit / 1048576;

      console.log(`Memory Usage:`);
      console.log(`  Used: ${used.toFixed(2)} MB`);
      console.log(`  Total: ${total.toFixed(2)} MB`);
      console.log(`  Limit: ${limit.toFixed(2)} MB`);
    }
  }
}

// Usage
const memProfiler = new MemoryProfiler();
memProfiler.start();

// Create many entities
for (let i = 0; i < 10000; i++) {
  const entity = world.createEntity();
  entity.addComponent(new Transform());
  entity.addComponent(new Velocity());
}

console.log(`Created 10000 entities: ${memProfiler.measure() / 1048576} MB`);
```

### Entity Count Profiling

```typescript
// profiling/EntityProfiler.ts
export class EntityProfiler {
  static profile(world: World): void {
    const entities = world.getAllEntities();
    const componentCounts = new Map<string, number>();

    for (const entity of entities) {
      const components = entity.getAllComponents();
      for (const component of components) {
        const name = component.constructor.name;
        componentCounts.set(name, (componentCounts.get(name) ?? 0) + 1);
      }
    }

    console.log(`Total Entities: ${entities.length}`);
    console.log('Component Distribution:');

    const sorted = Array.from(componentCounts.entries()).sort((a, b) => b[1] - a[1]);

    for (const [name, count] of sorted) {
      console.log(`  ${name}: ${count}`);
    }
  }
}
```

## Usage Examples

```typescript
// Example 1: Using archetype storage
const archetypeStorage = new ArchetypeStorage();

// Add entities
for (let i = 0; i < 1000; i++) {
  const entity = world.createEntity();
  entity.addComponent(new Transform());
  entity.addComponent(new Velocity());
  archetypeStorage.addEntity(entity, [Transform, Velocity]);
}

// Iterate efficiently
archetypeStorage.iterateArchetype<[Transform, Velocity]>(
  [Transform, Velocity],
  ([transform, velocity], entity) => {
    transform.position.x += velocity.x;
  }
);

// Example 2: Component pooling
const transformPool = new ComponentPool<Transform>(
  () => new Transform(),
  (t) => {
    t.position.set(0, 0, 0);
    t.rotation.set(0, 0, 0, 1);
    t.scale.set(1, 1, 1);
  }
);

transformPool.prewarm(1000);

// Example 3: Profiling systems
const profiler = new SystemProfiler();

systemManager.add(profiler.wrap(new MovementSystem()));
systemManager.add(profiler.wrap(new PhysicsSystem()));
systemManager.add(profiler.wrap(new RenderSystem()));

// Print stats every 5 seconds
setInterval(() => {
  profiler.printStats();
}, 5000);

// Example 4: Batch processing
class VisibilitySystem extends UpdateSystem {
  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, Renderable]>([
      Transform,
      Renderable,
    ]).entities;

    BatchProcessor.processBatches(
      entities,
      (entity) => {
        const [transform, renderable] = entity.getComponents(Transform, Renderable);
        renderable.visible = camera.frustum.containsPoint(transform.position);
      },
      16 // Max 16ms per frame
    );
  }
}

// Example 5: Memory optimization
const memProfiler = new MemoryProfiler();

// Before optimization
memProfiler.start();
for (let i = 0; i < 10000; i++) {
  const entity = world.createEntity();
  entity.addComponent(new Transform());
}
console.log(`Before: ${memProfiler.measure() / 1048576} MB`);

// After optimization (with pooling)
memProfiler.start();
for (let i = 0; i < 10000; i++) {
  const entity = entityPool.acquire();
  const transform = transformPool.acquire();
  entity.addComponent(transform);
}
console.log(`After: ${memProfiler.measure() / 1048576} MB`);
```

## Checklist

- [ ] Profile systems to identify bottlenecks
- [ ] Use archetype storage for cache efficiency
- [ ] Pool frequently created/destroyed components
- [ ] Pool frequently created/destroyed entities
- [ ] Cache query results
- [ ] Batch expensive operations
- [ ] Use packed arrays for components
- [ ] Measure memory usage
- [ ] Optimize hot paths
- [ ] Test with target entity count

## Common Pitfalls

1. **Premature optimization**: Optimize based on profiling data
2. **Over-pooling**: Memory overhead from unused pool objects
3. **Cache thrashing**: Poor data locality
4. **Synchronous heavy operations**: Frame drops
5. **Memory leaks**: Not releasing pooled objects
6. **Allocations in update loop**: GC pressure
7. **Deep object hierarchies**: Cache misses

## Performance Tips

### Data-Oriented Design
- Store components in contiguous arrays
- Process similar data together
- Minimize pointer chasing
- Use struct-of-arrays instead of array-of-structs
- Align data to cache lines

### Memory Optimization
- Pool all frequently allocated objects
- Prewarm pools to avoid runtime allocations
- Use typed arrays for numeric data
- Clear unused references
- Monitor memory usage

### CPU Optimization
- Profile systems regularly
- Batch similar operations
- Use early exits in loops
- Minimize function calls in hot paths
- Cache expensive calculations

### Mobile Optimization
- Reduce entity count
- Use simpler queries
- Disable non-essential systems
- Lower update frequencies
- Profile on actual devices

## Related Skills

- `ecs-architecture` - Overall ECS structure
- `ecs-system-patterns` - System implementation
- `ecs-queries` - Query optimization
- `ecs-component-patterns` - Component design
- `threejs-performance-profiling` - Three.js profiling

## References

- Data-Oriented Design (Mike Acton)
- Unity DOTS performance
- Cache-friendly code patterns
- JavaScript performance optimization
