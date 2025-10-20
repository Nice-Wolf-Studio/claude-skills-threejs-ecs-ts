---
name: ecs-architecture
description: Design and implement Entity Component System architecture with TypeScript for game development
---

# ECS Architecture

## When to Use

Use this skill when:
- Building a game or simulation with many interactive objects
- Refactoring object-oriented game code for better performance
- Implementing data-oriented design patterns
- Creating scalable game systems with complex interactions

## Core Principles

1. **Composition over Inheritance**: Entities are composed of components, not class hierarchies
2. **Data Locality**: Components store data; systems process data in contiguous memory
3. **Separation of Concerns**: Components = data, Systems = logic, Entities = IDs
4. **Type Safety**: Leverage TypeScript for compile-time guarantees
5. **Performance**: Cache-friendly iteration over components

## Architecture Overview

```
Entity: Unique ID
    ↓
Components: Pure data (Transform, Velocity, Health, Renderable)
    ↓
Systems: Logic that processes entities with specific component combinations
```

## Implementation

### 1. Entity Manager

```typescript
export type EntityId = number & { readonly __brand: 'EntityId' };

export class EntityManager {
  private nextId = 0;
  private readonly entities = new Set<EntityId>();

  create(): EntityId {
    const id = this.nextId++ as EntityId;
    this.entities.add(id);
    return id;
  }

  destroy(id: EntityId): void {
    this.entities.delete(id);
  }

  exists(id: EntityId): boolean {
    return this.entities.has(id);
  }

  getAll(): ReadonlySet<EntityId> {
    return this.entities;
  }

  clear(): void {
    this.entities.clear();
    this.nextId = 0;
  }
}
```

### 2. Component System

```typescript
export interface Component {
  readonly __componentBrand?: never;
}

export interface ComponentClass<T extends Component> {
  new (...args: any[]): T;
}

export class ComponentManager {
  private readonly components = new Map<ComponentClass<any>, Map<EntityId, Component>>();

  register<T extends Component>(type: ComponentClass<T>): void {
    if (!this.components.has(type)) {
      this.components.set(type, new Map());
    }
  }

  add<T extends Component>(entity: EntityId, type: ComponentClass<T>, component: T): void {
    this.register(type);
    this.components.get(type)!.set(entity, component);
  }

  remove<T extends Component>(entity: EntityId, type: ComponentClass<T>): void {
    this.components.get(type)?.delete(entity);
  }

  get<T extends Component>(entity: EntityId, type: ComponentClass<T>): T | undefined {
    return this.components.get(type)?.get(entity) as T | undefined;
  }

  has<T extends Component>(entity: EntityId, type: ComponentClass<T>): boolean {
    return this.components.get(type)?.has(entity) ?? false;
  }

  getAll<T extends Component>(type: ComponentClass<T>): Map<EntityId, T> {
    return (this.components.get(type) as Map<EntityId, T>) ?? new Map();
  }

  removeAllForEntity(entity: EntityId): void {
    for (const componentMap of this.components.values()) {
      componentMap.delete(entity);
    }
  }

  clear(): void {
    this.components.clear();
  }
}
```

### 3. System Base Class

```typescript
export abstract class System {
  constructor(
    protected readonly entities: EntityManager,
    protected readonly components: ComponentManager
  ) {}

  abstract update(deltaTime: number): void;

  protected query<T extends Component>(
    ...componentTypes: ComponentClass<T>[]
  ): Array<{ entity: EntityId; components: Component[] }> {
    const results: Array<{ entity: EntityId; components: Component[] }> = [];

    for (const entity of this.entities.getAll()) {
      const entityComponents: Component[] = [];
      let hasAll = true;

      for (const type of componentTypes) {
        const component = this.components.get(entity, type);
        if (!component) {
          hasAll = false;
          break;
        }
        entityComponents.push(component);
      }

      if (hasAll) {
        results.push({ entity, components: entityComponents });
      }
    }

    return results;
  }
}
```

### 4. Example Components

```typescript
export class Transform implements Component {
  constructor(
    public x: number = 0,
    public y: number = 0,
    public z: number = 0,
    public rotation: number = 0,
    public scale: number = 1
  ) {}
}

export class Velocity implements Component {
  constructor(
    public vx: number = 0,
    public vy: number = 0,
    public vz: number = 0
  ) {}
}

export class Health implements Component {
  constructor(
    public current: number,
    public max: number
  ) {}

  get isDead(): boolean {
    return this.current <= 0;
  }
}

export class Renderable implements Component {
  constructor(
    public mesh: THREE.Mesh,
    public visible: boolean = true
  ) {}
}
```

### 5. Example System

```typescript
export class MovementSystem extends System {
  update(deltaTime: number): void {
    // Query entities with both Transform and Velocity components
    for (const result of this.query(Transform, Velocity)) {
      const transform = result.components[0] as Transform;
      const velocity = result.components[1] as Velocity;

      // Update position based on velocity
      transform.x += velocity.vx * deltaTime;
      transform.y += velocity.vy * deltaTime;
      transform.z += velocity.vz * deltaTime;
    }
  }
}

export class RenderSystem extends System {
  constructor(
    entities: EntityManager,
    components: ComponentManager,
    private readonly scene: THREE.Scene
  ) {
    super(entities, components);
  }

  update(_deltaTime: number): void {
    for (const result of this.query(Transform, Renderable)) {
      const transform = result.components[0] as Transform;
      const renderable = result.components[1] as Renderable;

      if (renderable.visible) {
        renderable.mesh.position.set(transform.x, transform.y, transform.z);
        renderable.mesh.rotation.y = transform.rotation;
        renderable.mesh.scale.setScalar(transform.scale);
      }

      renderable.mesh.visible = renderable.visible;
    }
  }

  addToScene(entity: EntityId): void {
    const renderable = this.components.get(entity, Renderable);
    if (renderable && !this.scene.children.includes(renderable.mesh)) {
      this.scene.add(renderable.mesh);
    }
  }

  removeFromScene(entity: EntityId): void {
    const renderable = this.components.get(entity, Renderable);
    if (renderable) {
      this.scene.remove(renderable.mesh);
    }
  }
}
```

### 6. World (ECS Container)

```typescript
export class World {
  private readonly entityManager = new EntityManager();
  private readonly componentManager = new ComponentManager();
  private readonly systems: System[] = [];

  createEntity(): EntityId {
    return this.entityManager.create();
  }

  destroyEntity(entity: EntityId): void {
    this.componentManager.removeAllForEntity(entity);
    this.entityManager.destroy(entity);
  }

  addComponent<T extends Component>(
    entity: EntityId,
    type: ComponentClass<T>,
    component: T
  ): void {
    this.componentManager.add(entity, type, component);
  }

  removeComponent<T extends Component>(entity: EntityId, type: ComponentClass<T>): void {
    this.componentManager.remove(entity, type);
  }

  getComponent<T extends Component>(entity: EntityId, type: ComponentClass<T>): T | undefined {
    return this.componentManager.get(entity, type);
  }

  hasComponent<T extends Component>(entity: EntityId, type: ComponentClass<T>): boolean {
    return this.componentManager.has(entity, type);
  }

  addSystem(system: System): void {
    this.systems.push(system);
  }

  update(deltaTime: number): void {
    for (const system of this.systems) {
      system.update(deltaTime);
    }
  }

  clear(): void {
    this.systems.length = 0;
    this.componentManager.clear();
    this.entityManager.clear();
  }

  getEntityManager(): EntityManager {
    return this.entityManager;
  }

  getComponentManager(): ComponentManager {
    return this.componentManager;
  }
}
```

### 7. Usage Example

```typescript
import * as THREE from 'three';

// Create world
const world = new World();

// Add systems
const movementSystem = new MovementSystem(
  world.getEntityManager(),
  world.getComponentManager()
);

const renderSystem = new RenderSystem(
  world.getEntityManager(),
  world.getComponentManager(),
  scene // Three.js scene
);

world.addSystem(movementSystem);
world.addSystem(renderSystem);

// Create entity
const player = world.createEntity();

// Add components
world.addComponent(player, Transform, new Transform(0, 0, 0));
world.addComponent(player, Velocity, new Velocity(1, 0, 0));
world.addComponent(player, Health, new Health(100, 100));

const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const mesh = new THREE.Mesh(geometry, material);
world.addComponent(player, Renderable, new Renderable(mesh));

renderSystem.addToScene(player);

// Game loop
let lastTime = performance.now();
function gameLoop(currentTime: number): void {
  const deltaTime = (currentTime - lastTime) / 1000; // Convert to seconds
  lastTime = currentTime;

  world.update(deltaTime);

  requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

## Checklist

- [ ] Implement EntityManager for entity lifecycle
- [ ] Create ComponentManager with type-safe component storage
- [ ] Define Component interface and component classes
- [ ] Create System base class with query functionality
- [ ] Implement World class to orchestrate ECS
- [ ] Add entity creation and destruction
- [ ] Implement component add/remove/get operations
- [ ] Create system update loop with delta time
- [ ] Test entity queries across multiple component types
- [ ] Verify component isolation and data locality

## Common Pitfalls

1. **Components with logic**: Keep components as pure data structures
2. **Systems with state**: Systems should be stateless; state belongs in components
3. **Entity references**: Use EntityId, not direct object references
4. **Inefficient queries**: Cache query results when possible
5. **Circular dependencies**: Systems should not depend on each other

## Performance Tips

- Use TypedArrays for numeric component data in hot paths
- Implement archetype-based storage for better cache locality
- Batch queries instead of querying per-entity
- Consider dirty flags to skip unchanged entities
- Use component pools to reduce allocations

## Related Skills

- `ecs-component-design` - Advanced component patterns
- `ecs-system-patterns` - System optimization techniques
- `typescript-ecs-types` - Type-safe ECS patterns
