---
name: ecs-queries
description: Efficient entity queries in ECS including archetype-based queries, filtered queries, cached queries, and query optimization
---

# ECS Queries

## When to Use

Use this skill when:
- Querying entities by component type
- Filtering entities based on criteria
- Optimizing system performance
- Implementing spatial queries
- Building complex entity searches
- Managing large numbers of entities

## Core Principles

1. **Cache Queries**: Create queries once, reuse many times
2. **Archetype-Based**: Group entities by component signature
3. **Lazy Evaluation**: Only iterate when needed
4. **Minimal Allocations**: Reuse arrays and iterators
5. **Type-Safe**: Leverage TypeScript for component access
6. **Performance-First**: O(1) or O(n) operations, never O(n²)

## Query System Implementation

### 1. Basic Query Interface

```typescript
// core/Query.ts
import { Entity } from './Entity';
import { ComponentConstructor } from './Component';

export interface QueryResult<T extends any[]> {
  entities: Entity[];
  iterate(callback: (entity: Entity, components: T) => void): void;
  filter(predicate: (entity: Entity, components: T) => boolean): QueryResult<T>;
  first(): Entity | undefined;
  count(): number;
}

export class Query<T extends any[]> implements QueryResult<T> {
  private componentTypes: ComponentConstructor<any>[];
  private cachedEntities: Entity[] = [];
  private dirty = true;

  constructor(
    componentTypes: ComponentConstructor<any>[],
    private world: World
  ) {
    this.componentTypes = componentTypes;
  }

  get entities(): Entity[] {
    if (this.dirty) {
      this.refresh();
    }
    return this.cachedEntities;
  }

  private refresh(): void {
    this.cachedEntities = this.world
      .getAllEntities()
      .filter((entity) => this.matches(entity));
    this.dirty = false;
  }

  private matches(entity: Entity): boolean {
    return this.componentTypes.every((type) => entity.hasComponent(type));
  }

  iterate(callback: (entity: Entity, components: T) => void): void {
    for (const entity of this.entities) {
      const components = entity.getComponents(...this.componentTypes) as T;
      callback(entity, components);
    }
  }

  filter(predicate: (entity: Entity, components: T) => boolean): QueryResult<T> {
    const filtered = new FilteredQuery<T>(this, predicate);
    return filtered;
  }

  first(): Entity | undefined {
    return this.entities[0];
  }

  count(): number {
    return this.entities.length;
  }

  markDirty(): void {
    this.dirty = true;
  }
}
```

### 2. Archetype-Based Query System

```typescript
// core/Archetype.ts
export type Archetype = string; // Sorted component type names joined by ':'

export class ArchetypeManager {
  private archetypes = new Map<Archetype, Set<Entity>>();
  private entityArchetypes = new Map<Entity, Archetype>();

  getArchetype(componentTypes: ComponentConstructor<any>[]): Archetype {
    const sorted = componentTypes
      .map((t) => t.name)
      .sort()
      .join(':');
    return sorted;
  }

  addEntity(entity: Entity, archetype: Archetype): void {
    let entities = this.archetypes.get(archetype);
    if (!entities) {
      entities = new Set();
      this.archetypes.set(archetype, entities);
    }
    entities.add(entity);
    this.entityArchetypes.set(entity, archetype);
  }

  removeEntity(entity: Entity): void {
    const archetype = this.entityArchetypes.get(entity);
    if (archetype) {
      const entities = this.archetypes.get(archetype);
      if (entities) {
        entities.delete(entity);
        if (entities.size === 0) {
          this.archetypes.delete(archetype);
        }
      }
      this.entityArchetypes.delete(entity);
    }
  }

  updateEntity(entity: Entity, newArchetype: Archetype): void {
    this.removeEntity(entity);
    this.addEntity(entity, newArchetype);
  }

  queryArchetypes(componentTypes: ComponentConstructor<any>[]): Set<Entity> {
    const required = new Set(componentTypes.map((t) => t.name));
    const results = new Set<Entity>();

    for (const [archetype, entities] of this.archetypes) {
      const componentNames = new Set(archetype.split(':'));

      // Check if archetype contains all required components
      const hasAll = Array.from(required).every((name) => componentNames.has(name));

      if (hasAll) {
        entities.forEach((entity) => results.add(entity));
      }
    }

    return results;
  }
}
```

### 3. Cached Query Manager

```typescript
// core/QueryManager.ts
export class QueryManager {
  private queries = new Map<string, Query<any>>();
  private archetypeManager: ArchetypeManager;

  constructor(archetypeManager: ArchetypeManager) {
    this.archetypeManager = archetypeManager;
  }

  query<T extends any[]>(
    componentTypes: ComponentConstructor<any>[]
  ): Query<T> {
    const key = this.getQueryKey(componentTypes);
    let query = this.queries.get(key);

    if (!query) {
      query = new ArchetypeQuery<T>(componentTypes, this.archetypeManager);
      this.queries.set(key, query);
    }

    return query as Query<T>;
  }

  private getQueryKey(componentTypes: ComponentConstructor<any>[]): string {
    return componentTypes
      .map((t) => t.name)
      .sort()
      .join(':');
  }

  onComponentAdded(entity: Entity): void {
    // Mark all queries as dirty
    this.queries.forEach((query) => query.markDirty());
  }

  onComponentRemoved(entity: Entity): void {
    // Mark all queries as dirty
    this.queries.forEach((query) => query.markDirty());
  }

  clear(): void {
    this.queries.clear();
  }
}

// Archetype-optimized query
class ArchetypeQuery<T extends any[]> extends Query<T> {
  constructor(
    componentTypes: ComponentConstructor<any>[],
    private archetypeManager: ArchetypeManager
  ) {
    super(componentTypes, null!); // World not needed
  }

  get entities(): Entity[] {
    if (this.dirty) {
      this.cachedEntities = Array.from(
        this.archetypeManager.queryArchetypes(this.componentTypes)
      );
      this.dirty = false;
    }
    return this.cachedEntities;
  }
}
```

### 4. Filtered Queries

```typescript
// core/FilteredQuery.ts
export class FilteredQuery<T extends any[]> implements QueryResult<T> {
  private baseQuery: QueryResult<T>;
  private predicate: (entity: Entity, components: T) => boolean;

  constructor(
    baseQuery: QueryResult<T>,
    predicate: (entity: Entity, components: T) => boolean
  ) {
    this.baseQuery = baseQuery;
    this.predicate = predicate;
  }

  get entities(): Entity[] {
    const results: Entity[] = [];
    this.baseQuery.iterate((entity, components) => {
      if (this.predicate(entity, components)) {
        results.push(entity);
      }
    });
    return results;
  }

  iterate(callback: (entity: Entity, components: T) => void): void {
    this.baseQuery.iterate((entity, components) => {
      if (this.predicate(entity, components)) {
        callback(entity, components);
      }
    });
  }

  filter(predicate: (entity: Entity, components: T) => boolean): QueryResult<T> {
    // Chain filters
    return new FilteredQuery<T>(this, predicate);
  }

  first(): Entity | undefined {
    for (const entity of this.baseQuery.entities) {
      const components = entity.getComponents(...this.componentTypes) as T;
      if (this.predicate(entity, components)) {
        return entity;
      }
    }
    return undefined;
  }

  count(): number {
    let count = 0;
    this.iterate(() => count++);
    return count;
  }
}
```

### 5. Spatial Queries

```typescript
// queries/SpatialQuery.ts
import { Transform } from '../components/Transform';
import { Vector3 } from 'three';

export class SpatialQuery {
  constructor(private world: World) {}

  withinRadius(
    center: Vector3,
    radius: number,
    componentTypes: ComponentConstructor<any>[] = [Transform]
  ): Entity[] {
    const radiusSq = radius * radius;
    const query = this.world.query<[Transform]>(componentTypes);

    return query
      .filter((entity, [transform]) => {
        const distSq = transform.position.distanceToSquared(center);
        return distSq <= radiusSq;
      })
      .entities;
  }

  withinBox(
    min: Vector3,
    max: Vector3,
    componentTypes: ComponentConstructor<any>[] = [Transform]
  ): Entity[] {
    const query = this.world.query<[Transform]>(componentTypes);

    return query
      .filter((entity, [transform]) => {
        const pos = transform.position;
        return (
          pos.x >= min.x &&
          pos.x <= max.x &&
          pos.y >= min.y &&
          pos.y <= max.y &&
          pos.z >= min.z &&
          pos.z <= max.z
        );
      })
      .entities;
  }

  nearest(
    position: Vector3,
    count: number = 1,
    componentTypes: ComponentConstructor<any>[] = [Transform]
  ): Entity[] {
    const query = this.world.query<[Transform]>(componentTypes);
    const withDistances: Array<{ entity: Entity; distance: number }> = [];

    query.iterate((entity, [transform]) => {
      const distance = transform.position.distanceTo(position);
      withDistances.push({ entity, distance });
    });

    // Sort by distance and return top N
    withDistances.sort((a, b) => a.distance - b.distance);
    return withDistances.slice(0, count).map((item) => item.entity);
  }

  raycast(
    origin: Vector3,
    direction: Vector3,
    maxDistance: number = Infinity
  ): Entity[] {
    // Simplified raycast (use proper physics for production)
    const query = this.world.query<[Transform]>([Transform]);
    const hits: Entity[] = [];

    query.iterate((entity, [transform]) => {
      const toEntity = new Vector3()
        .subVectors(transform.position, origin);
      const distance = toEntity.length();

      if (distance > maxDistance) return;

      const projection = toEntity.dot(direction);
      if (projection < 0) return; // Behind ray

      hits.push(entity);
    });

    return hits;
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic query
import { Transform } from './components/Transform';
import { Velocity } from './components/Velocity';

// Query all entities with Transform and Velocity
const movingEntities = world.query<[Transform, Velocity]>([Transform, Velocity]);

// Iterate over results
movingEntities.iterate((entity, [transform, velocity]) => {
  console.log(`Entity ${entity.id} at ${transform.position}`);
});

// Example 2: Filtered query
const fastEntities = movingEntities.filter((entity, [transform, velocity]) => {
  const speed = Math.sqrt(velocity.x ** 2 + velocity.y ** 2 + velocity.z ** 2);
  return speed > 10;
});

console.log(`Found ${fastEntities.count()} fast entities`);

// Example 3: Chained filters
const visibleFastEnemies = world
  .query<[Transform, Velocity, Enemy, Visible]>([
    Transform,
    Velocity,
    Enemy,
    Visible,
  ])
  .filter((entity, [transform, velocity]) => {
    const speed = Math.sqrt(velocity.x ** 2 + velocity.y ** 2 + velocity.z ** 2);
    return speed > 10;
  })
  .filter((entity, [transform]) => {
    return camera.frustum.containsPoint(transform.position);
  });

// Example 4: First match
const player = world
  .query<[Player, Transform]>([Player, Transform])
  .first();

if (player) {
  const [playerComp, transform] = player.getComponents(Player, Transform);
  console.log(`Player at ${transform.position}`);
}

// Example 5: Spatial query
const spatialQuery = new SpatialQuery(world);

// Find enemies near player
const nearbyEnemies = spatialQuery.withinRadius(
  playerPosition,
  20, // 20 units radius
  [Enemy, Transform]
);

// Find entities in a box
const entitiesInRoom = spatialQuery.withinBox(
  new Vector3(0, 0, 0),
  new Vector3(10, 10, 10),
  [Transform]
);

// Find nearest 3 health pickups
const nearestPickups = spatialQuery.nearest(
  playerPosition,
  3,
  [HealthPickup, Transform]
);

// Example 6: Complex system query
class CombatSystem extends UpdateSystem {
  update(world: World, deltaTime: number): void {
    // Find all attackers
    const attackers = world.query<[Transform, Attack, Faction]>([
      Transform,
      Attack,
      Faction,
    ]);

    // Find all damageable entities
    const targets = world.query<[Transform, Health, Faction]>([
      Transform,
      Health,
      Faction,
    ]);

    // Check for attacks
    attackers.iterate((attacker, [attackerTransform, attack, attackerFaction]) => {
      if (attack.cooldown > 0) return;

      // Find enemies in range
      const inRange = targets.filter((target, [targetTransform, health, targetFaction]) => {
        // Different faction
        if (targetFaction.id === attackerFaction.id) return false;

        // In range
        const distance = attackerTransform.position.distanceTo(targetTransform.position);
        return distance <= attack.range;
      });

      // Attack first target
      const target = inRange.first();
      if (target) {
        const [, health] = target.getComponents(Transform, Health, Faction);
        health.current -= attack.damage;
        attack.cooldown = attack.cooldownTime;
      }
    });
  }
}
```

## Query Optimization Patterns

### Pattern 1: Query Caching

```typescript
// ❌ Bad: Create query every frame
class BadSystem extends UpdateSystem {
  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, Velocity]>([Transform, Velocity]);
    // Process entities...
  }
}

// ✅ Good: Cache query
class GoodSystem extends UpdateSystem {
  private query: Query<[Transform, Velocity]>;

  constructor(world: World) {
    super();
    this.query = world.query<[Transform, Velocity]>([Transform, Velocity]);
  }

  update(world: World, deltaTime: number): void {
    this.query.iterate((entity, [transform, velocity]) => {
      // Process entities...
    });
  }
}
```

### Pattern 2: Early Exit

```typescript
// Stop iterating when condition met
const target = world
  .query<[Enemy, Transform]>([Enemy, Transform])
  .filter((entity, [enemy, transform]) => {
    return transform.position.distanceTo(playerPos) < 5;
  })
  .first(); // Returns immediately after finding first match
```

### Pattern 3: Batch Processing

```typescript
class BatchedSystem extends UpdateSystem {
  private batchSize = 100;

  update(world: World, deltaTime: number): void {
    const query = world.query<[Transform, Mesh]>([Transform, Mesh]);
    const entities = query.entities;

    // Process in batches to avoid frame drops
    const batchCount = Math.ceil(entities.length / this.batchSize);
    const batch = (frameCount % batchCount) * this.batchSize;

    for (let i = batch; i < Math.min(batch + this.batchSize, entities.length); i++) {
      const entity = entities[i];
      const [transform, mesh] = entity.getComponents(Transform, Mesh);
      // Expensive operation...
    }
  }
}
```

## Checklist

- [ ] Use cached queries in systems
- [ ] Leverage archetype system for fast queries
- [ ] Add filters for specific criteria
- [ ] Use first() for single entity searches
- [ ] Implement spatial queries for nearby entities
- [ ] Profile query performance
- [ ] Avoid creating queries every frame
- [ ] Consider batch processing for large result sets
- [ ] Use type-safe component access
- [ ] Document query purpose and usage

## Common Pitfalls

1. **Creating queries every frame**: Cache queries in system constructors
2. **O(n²) nested queries**: Use spatial partitioning or archetype optimization
3. **No early exit**: Use first() or break when found
4. **Allocating in loops**: Reuse arrays and objects
5. **Filtering after iteration**: Filter during query when possible
6. **Ignoring archetype changes**: Keep queries fresh
7. **Complex predicates**: Simplify or index data

## Performance Tips

### Query Performance
- Cache queries outside update loops
- Use archetype-based queries for O(n) performance
- Filter early in the pipeline
- Use spatial indexing for proximity queries
- Batch process large result sets

### Memory Optimization
- Reuse query result arrays
- Use iterators instead of allocating arrays
- Clear cached results when not needed
- Limit query result lifetime

### Profiling Queries
```typescript
export function profileQuery<T extends any[]>(
  name: string,
  query: QueryResult<T>,
  callback: (entity: Entity, components: T) => void
): void {
  const start = performance.now();
  let count = 0;

  query.iterate((entity, components) => {
    callback(entity, components);
    count++;
  });

  const time = performance.now() - start;
  console.log(`Query "${name}": ${count} entities in ${time.toFixed(2)}ms`);
}
```

## Related Skills

- `ecs-architecture` - Overall ECS structure
- `ecs-system-patterns` - System implementation
- `ecs-component-patterns` - Component design
- `ecs-performance` - ECS optimization
- `collision-system` - Spatial queries for physics

## References

- Archetype-based ECS design
- Unity DOTS queries
- Bevy ECS query system
- Data-Oriented Design (Mike Acton)
