---
name: collision-system
description: ECS collision detection system with spatial partitioning, broadphase/narrowphase, AABB, and sphere collision for mobile games
---

# Collision System

## When to Use

Use this skill when:
- Implementing collision detection in ECS games
- Detecting entity interactions
- Implementing hit detection
- Building spatial partitioning
- Optimizing collision performance

## Core Principles

1. **Broadphase First**: Quick rejection using simple bounds
2. **Spatial Partitioning**: Grid or octree for large scenes
3. **Component-Based**: Collision data in components
4. **Event-Driven**: Emit collision events
5. **Performance**: Only check nearby entities
6. **Type Filtering**: Collision layers/masks

## Implementation

### 1. Collision Components

```typescript
// components/Collider.ts
import { Component } from '../core/Component';
import * as THREE from 'three';

export type ColliderShape = 'box' | 'sphere' | 'capsule';

export class Collider implements Component {
  shape: ColliderShape;
  size: THREE.Vector3; // For box: width, height, depth; For sphere: radius in x
  offset: THREE.Vector3;
  isTrigger: boolean;
  layer: number;
  mask: number; // Which layers to collide with

  constructor(
    shape: ColliderShape,
    size: THREE.Vector3,
    options?: {
      offset?: THREE.Vector3;
      isTrigger?: boolean;
      layer?: number;
      mask?: number;
    }
  ) {
    this.shape = shape;
    this.size = size;
    this.offset = options?.offset || new THREE.Vector3();
    this.isTrigger = options?.isTrigger || false;
    this.layer = options?.layer || 0;
    this.mask = options?.mask || 0xFFFFFFFF; // Collide with all by default
  }

  getBounds(position: THREE.Vector3): AABB | Sphere {
    const center = position.clone().add(this.offset);

    if (this.shape === 'sphere') {
      return new Sphere(center, this.size.x);
    } else {
      return new AABB(
        center.clone().sub(this.size.clone().multiplyScalar(0.5)),
        center.clone().add(this.size.clone().multiplyScalar(0.5))
      );
    }
  }
}

export class AABB {
  constructor(
    public min: THREE.Vector3,
    public max: THREE.Vector3
  ) {}

  intersects(other: AABB): boolean {
    return (
      this.min.x <= other.max.x && this.max.x >= other.min.x &&
      this.min.y <= other.max.y && this.max.y >= other.min.y &&
      this.min.z <= other.max.z && this.max.z >= other.min.z
    );
  }

  contains(point: THREE.Vector3): boolean {
    return (
      point.x >= this.min.x && point.x <= this.max.x &&
      point.y >= this.min.y && point.y <= this.max.y &&
      point.z >= this.min.z && point.z <= this.max.z
    );
  }

  getCenter(): THREE.Vector3 {
    return new THREE.Vector3().addVectors(this.min, this.max).multiplyScalar(0.5);
  }

  getSize(): THREE.Vector3 {
    return new THREE.Vector3().subVectors(this.max, this.min);
  }
}

export class Sphere {
  constructor(
    public center: THREE.Vector3,
    public radius: number
  ) {}

  intersects(other: Sphere): boolean {
    const distSq = this.center.distanceToSquared(other.center);
    const radSum = this.radius + other.radius;
    return distSq <= radSum * radSum;
  }

  intersectsAABB(aabb: AABB): boolean {
    const closestPoint = new THREE.Vector3(
      Math.max(aabb.min.x, Math.min(this.center.x, aabb.max.x)),
      Math.max(aabb.min.y, Math.min(this.center.y, aabb.max.y)),
      Math.max(aabb.min.z, Math.min(this.center.z, aabb.max.z))
    );

    const distSq = this.center.distanceToSquared(closestPoint);
    return distSq <= this.radius * this.radius;
  }

  contains(point: THREE.Vector3): boolean {
    return this.center.distanceToSquared(point) <= this.radius * this.radius;
  }
}
```

### 2. Spatial Grid

```typescript
// SpatialGrid.ts
import { EntityId } from '../ecs/types';

export class SpatialGrid {
  private cells = new Map<string, Set<EntityId>>();
  private entityCells = new Map<EntityId, Set<string>>();

  constructor(private cellSize: number) {}

  private getCellKey(x: number, y: number, z: number): string {
    return `${Math.floor(x / this.cellSize)},${Math.floor(y / this.cellSize)},${Math.floor(z / this.cellSize)}`;
  }

  /**
   * Insert entity at position
   */
  insert(entity: EntityId, bounds: AABB): void {
    // Remove from old cells
    this.remove(entity);

    const cells = new Set<string>();

    // Calculate which cells the bounds span
    const minCell = {
      x: Math.floor(bounds.min.x / this.cellSize),
      y: Math.floor(bounds.min.y / this.cellSize),
      z: Math.floor(bounds.min.z / this.cellSize),
    };

    const maxCell = {
      x: Math.floor(bounds.max.x / this.cellSize),
      y: Math.floor(bounds.max.y / this.cellSize),
      z: Math.floor(bounds.max.z / this.cellSize),
    };

    // Add to all overlapping cells
    for (let x = minCell.x; x <= maxCell.x; x++) {
      for (let y = minCell.y; y <= maxCell.y; y++) {
        for (let z = minCell.z; z <= maxCell.z; z++) {
          const key = `${x},${y},${z}`;

          if (!this.cells.has(key)) {
            this.cells.set(key, new Set());
          }

          this.cells.get(key)!.add(entity);
          cells.add(key);
        }
      }
    }

    this.entityCells.set(entity, cells);
  }

  /**
   * Remove entity from grid
   */
  remove(entity: EntityId): void {
    const cells = this.entityCells.get(entity);
    if (!cells) return;

    cells.forEach(key => {
      const cell = this.cells.get(key);
      if (cell) {
        cell.delete(entity);
        if (cell.size === 0) {
          this.cells.delete(key);
        }
      }
    });

    this.entityCells.delete(entity);
  }

  /**
   * Query nearby entities
   */
  query(bounds: AABB): Set<EntityId> {
    const result = new Set<EntityId>();

    const minCell = {
      x: Math.floor(bounds.min.x / this.cellSize),
      y: Math.floor(bounds.min.y / this.cellSize),
      z: Math.floor(bounds.min.z / this.cellSize),
    };

    const maxCell = {
      x: Math.floor(bounds.max.x / this.cellSize),
      y: Math.floor(bounds.max.y / this.cellSize),
      z: Math.floor(bounds.max.z / this.cellSize),
    };

    for (let x = minCell.x; x <= maxCell.x; x++) {
      for (let y = minCell.y; y <= maxCell.y; y++) {
        for (let z = minCell.z; z <= maxCell.z; z++) {
          const key = `${x},${y},${z}`;
          const cell = this.cells.get(key);

          if (cell) {
            cell.forEach(entity => result.add(entity));
          }
        }
      }
    }

    return result;
  }

  /**
   * Clear all cells
   */
  clear(): void {
    this.cells.clear();
    this.entityCells.clear();
  }

  /**
   * Get total cell count
   */
  getCellCount(): number {
    return this.cells.size;
  }
}
```

### 3. Collision System

```typescript
// systems/CollisionSystem.ts
import { System } from '../core/System';
import { Transform } from '../components/Transform';
import { Collider, AABB, Sphere } from '../components/Collider';
import { SpatialGrid } from './SpatialGrid';
import { EntityId } from '../ecs/types';

export interface CollisionEvent {
  entityA: EntityId;
  entityB: EntityId;
  point: THREE.Vector3;
  normal: THREE.Vector3;
}

export class CollisionSystem extends System {
  private spatialGrid: SpatialGrid;
  private collisionEvents: CollisionEvent[] = [];
  private callbacks = new Set<(event: CollisionEvent) => void>();

  constructor(
    entities: EntityManager,
    components: ComponentManager,
    cellSize: number = 10
  ) {
    super(entities, components);
    this.spatialGrid = new SpatialGrid(cellSize);
  }

  /**
   * Register collision callback
   */
  onCollision(callback: (event: CollisionEvent) => void): () => void {
    this.callbacks.add(callback);
    return () => this.callbacks.delete(callback);
  }

  update(deltaTime: number): void {
    this.collisionEvents = [];

    // Update spatial grid
    this.updateSpatialGrid();

    // Detect collisions
    this.detectCollisions();

    // Emit events
    this.collisionEvents.forEach(event => {
      this.callbacks.forEach(callback => callback(event));
    });
  }

  private updateSpatialGrid(): void {
    this.spatialGrid.clear();

    for (const result of this.query(Transform, Collider)) {
      const entity = result.entity;
      const transform = result.components[0] as Transform;
      const collider = result.components[1] as Collider;

      const position = new THREE.Vector3(transform.x, transform.y, transform.z);
      const bounds = collider.getBounds(position);

      if (bounds instanceof AABB) {
        this.spatialGrid.insert(entity, bounds);
      } else if (bounds instanceof Sphere) {
        // Convert sphere to AABB for grid insertion
        const aabb = new AABB(
          new THREE.Vector3(
            bounds.center.x - bounds.radius,
            bounds.center.y - bounds.radius,
            bounds.center.z - bounds.radius
          ),
          new THREE.Vector3(
            bounds.center.x + bounds.radius,
            bounds.center.y + bounds.radius,
            bounds.center.z + bounds.radius
          )
        );
        this.spatialGrid.insert(entity, aabb);
      }
    }
  }

  private detectCollisions(): void {
    const checked = new Set<string>();

    for (const result of this.query(Transform, Collider)) {
      const entityA = result.entity;
      const transformA = result.components[0] as Transform;
      const colliderA = result.components[1] as Collider;

      const positionA = new THREE.Vector3(transformA.x, transformA.y, transformA.z);
      const boundsA = colliderA.getBounds(positionA);

      // Query nearby entities
      let queryBounds: AABB;

      if (boundsA instanceof Sphere) {
        queryBounds = new AABB(
          new THREE.Vector3(
            boundsA.center.x - boundsA.radius,
            boundsA.center.y - boundsA.radius,
            boundsA.center.z - boundsA.radius
          ),
          new THREE.Vector3(
            boundsA.center.x + boundsA.radius,
            boundsA.center.y + boundsA.radius,
            boundsA.center.z + boundsA.radius
          )
        );
      } else {
        queryBounds = boundsA;
      }

      const nearby = this.spatialGrid.query(queryBounds);

      for (const entityB of nearby) {
        if (entityA === entityB) continue;

        // Avoid duplicate checks
        const pairKey = entityA < entityB
          ? `${entityA}-${entityB}`
          : `${entityB}-${entityA}`;

        if (checked.has(pairKey)) continue;
        checked.add(pairKey);

        const transformB = this.components.get(entityB, Transform);
        const colliderB = this.components.get(entityB, Collider);

        if (!transformB || !colliderB) continue;

        // Layer filtering
        if (!(colliderA.mask & (1 << colliderB.layer))) continue;
        if (!(colliderB.mask & (1 << colliderA.layer))) continue;

        const positionB = new THREE.Vector3(transformB.x, transformB.y, transformB.z);
        const boundsB = colliderB.getBounds(positionB);

        // Narrow phase collision detection
        const collision = this.checkCollision(boundsA, boundsB);

        if (collision) {
          this.collisionEvents.push({
            entityA,
            entityB,
            point: collision.point,
            normal: collision.normal,
          });
        }
      }
    }
  }

  private checkCollision(
    boundsA: AABB | Sphere,
    boundsB: AABB | Sphere
  ): { point: THREE.Vector3; normal: THREE.Vector3 } | null {
    // AABB vs AABB
    if (boundsA instanceof AABB && boundsB instanceof AABB) {
      if (!boundsA.intersects(boundsB)) return null;

      const centerA = boundsA.getCenter();
      const centerB = boundsB.getCenter();

      const normal = new THREE.Vector3().subVectors(centerB, centerA).normalize();
      const point = new THREE.Vector3().addVectors(centerA, centerB).multiplyScalar(0.5);

      return { point, normal };
    }

    // Sphere vs Sphere
    if (boundsA instanceof Sphere && boundsB instanceof Sphere) {
      if (!boundsA.intersects(boundsB)) return null;

      const normal = new THREE.Vector3().subVectors(boundsB.center, boundsA.center).normalize();
      const point = boundsA.center.clone().add(
        normal.clone().multiplyScalar(boundsA.radius)
      );

      return { point, normal };
    }

    // Sphere vs AABB
    if (boundsA instanceof Sphere && boundsB instanceof AABB) {
      if (!boundsA.intersectsAABB(boundsB)) return null;

      const closestPoint = new THREE.Vector3(
        Math.max(boundsB.min.x, Math.min(boundsA.center.x, boundsB.max.x)),
        Math.max(boundsB.min.y, Math.min(boundsA.center.y, boundsB.max.y)),
        Math.max(boundsB.min.z, Math.min(boundsA.center.z, boundsB.max.z))
      );

      const normal = new THREE.Vector3().subVectors(boundsA.center, closestPoint).normalize();
      const point = closestPoint;

      return { point, normal };
    }

    // AABB vs Sphere (swap and recurse)
    if (boundsA instanceof AABB && boundsB instanceof Sphere) {
      const result = this.checkCollision(boundsB, boundsA);
      if (result) {
        result.normal.negate();
      }
      return result;
    }

    return null;
  }

  /**
   * Raycast against all colliders
   */
  raycast(
    origin: THREE.Vector3,
    direction: THREE.Vector3,
    maxDistance: number = Infinity
  ): { entity: EntityId; point: THREE.Vector3; distance: number } | null {
    let closestHit: { entity: EntityId; point: THREE.Vector3; distance: number } | null = null;
    let closestDist = maxDistance;

    for (const result of this.query(Transform, Collider)) {
      const entity = result.entity;
      const transform = result.components[0] as Transform;
      const collider = result.components[1] as Collider;

      const position = new THREE.Vector3(transform.x, transform.y, transform.z);
      const bounds = collider.getBounds(position);

      let hit: THREE.Vector3 | null = null;

      if (bounds instanceof Sphere) {
        hit = this.raySphere(origin, direction, bounds);
      } else if (bounds instanceof AABB) {
        hit = this.rayAABB(origin, direction, bounds);
      }

      if (hit) {
        const dist = hit.distanceTo(origin);
        if (dist < closestDist) {
          closestDist = dist;
          closestHit = { entity, point: hit, distance: dist };
        }
      }
    }

    return closestHit;
  }

  private raySphere(
    origin: THREE.Vector3,
    direction: THREE.Vector3,
    sphere: Sphere
  ): THREE.Vector3 | null {
    const oc = new THREE.Vector3().subVectors(origin, sphere.center);
    const a = direction.dot(direction);
    const b = 2 * oc.dot(direction);
    const c = oc.dot(oc) - sphere.radius * sphere.radius;
    const discriminant = b * b - 4 * a * c;

    if (discriminant < 0) return null;

    const t = (-b - Math.sqrt(discriminant)) / (2 * a);
    if (t < 0) return null;

    return origin.clone().add(direction.clone().multiplyScalar(t));
  }

  private rayAABB(
    origin: THREE.Vector3,
    direction: THREE.Vector3,
    aabb: AABB
  ): THREE.Vector3 | null {
    const invDir = new THREE.Vector3(
      1 / direction.x,
      1 / direction.y,
      1 / direction.z
    );

    const t1 = (aabb.min.x - origin.x) * invDir.x;
    const t2 = (aabb.max.x - origin.x) * invDir.x;
    const t3 = (aabb.min.y - origin.y) * invDir.y;
    const t4 = (aabb.max.y - origin.y) * invDir.y;
    const t5 = (aabb.min.z - origin.z) * invDir.z;
    const t6 = (aabb.max.z - origin.z) * invDir.z;

    const tmin = Math.max(Math.max(Math.min(t1, t2), Math.min(t3, t4)), Math.min(t5, t6));
    const tmax = Math.min(Math.min(Math.max(t1, t2), Math.max(t3, t4)), Math.max(t5, t6));

    if (tmax < 0 || tmin > tmax) return null;

    const t = tmin < 0 ? tmax : tmin;
    return origin.clone().add(direction.clone().multiplyScalar(t));
  }
}
```

## Usage Examples

```typescript
// Add colliders to entities
const player = world.createEntity();
world.addComponent(player, Transform, new Transform(0, 1, 0));
world.addComponent(player, Collider, new Collider(
  'sphere',
  new THREE.Vector3(0.5, 0.5, 0.5), // radius
  { layer: 0, mask: 0xFFFFFFFF }
));

const enemy = world.createEntity();
world.addComponent(enemy, Transform, new Transform(5, 1, 5));
world.addComponent(enemy, Collider, new Collider(
  'box',
  new THREE.Vector3(1, 2, 1), // width, height, depth
  { layer: 1, mask: 1 << 0 } // Only collide with layer 0
));

// Set up collision system
const collisionSystem = new CollisionSystem(
  world.getEntityManager(),
  world.getComponentManager(),
  10 // cell size
);

world.addSystem(collisionSystem);

// Listen for collisions
collisionSystem.onCollision((event) => {
  console.log(`Collision between ${event.entityA} and ${event.entityB}`);

  // Handle collision
  const healthA = world.getComponent(event.entityA, Health);
  const healthB = world.getComponent(event.entityB, Health);

  if (healthA && healthB) {
    healthA.damage(10, performance.now());
  }
});

// Raycast
const hit = collisionSystem.raycast(
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(0, 0, 1).normalize(),
  100
);

if (hit) {
  console.log(`Hit entity ${hit.entity} at ${hit.point}`);
}
```

## Checklist

- [ ] Add Collider components to entities
- [ ] Set up spatial partitioning (grid/octree)
- [ ] Implement broadphase collision detection
- [ ] Implement narrowphase (precise) collision
- [ ] Use collision layers for filtering
- [ ] Emit collision events
- [ ] Handle trigger vs solid colliders
- [ ] Implement raycast support
- [ ] Optimize cell size for scene scale
- [ ] Test with 100+ colliding entities
- [ ] Profile collision performance
- [ ] Visualize colliders in debug mode

## Common Pitfalls

1. **No spatial partitioning**: O(n²) performance
2. **Wrong cell size**: Too small = overhead, too large = ineffective
3. **Not filtering layers**: Unnecessary checks
4. **Forgetting to update grid**: Stale positions
5. **Expensive narrowphase**: Use simple shapes
6. **Not handling triggers**: Everything is solid

## Performance Tips

- Use spatial grid (10-100x faster than brute force)
- Set appropriate cell size (2-3x object size)
- Use layers to filter collision pairs
- Use simple colliders (sphere/AABB) over complex mesh
- Update grid incrementally (dirty flags)
- Limit collision checks per frame
- Use collision islands for sleeping objects
- Profile with 1000+ entities to find bottlenecks

## Related Skills

- `physics-system` - Physics-based collision response
- `ecs-component-patterns` - Component design
- `threejs-raycasting` - Visual raycasting
- `ecs-events` - Event system integration
