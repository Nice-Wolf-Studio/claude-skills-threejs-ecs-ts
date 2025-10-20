---
name: typescript-ecs-types
description: TypeScript type patterns for Entity Component Systems including type-safe queries, component types, and system patterns
---

# TypeScript ECS Type Patterns

## When to Use

Use this skill when:
- Building type-safe ECS systems
- Creating component type hierarchies
- Implementing type-safe queries
- Defining system interfaces
- Building ECS APIs
- Creating component factories

## Core Principles

1. **Type Safety**: Compile-time component checking
2. **Inference**: Let TypeScript infer types
3. **Generics**: Reusable component patterns
4. **Discriminated Unions**: Type-safe events
5. **Mapped Types**: Transform component types
6. **Template Literals**: String type manipulation

## ECS Type System Implementation

### 1. Core Component Types

```typescript
// types/Component.ts
export interface Component {
  // Marker interface for all components
}

export type ComponentConstructor<T extends Component = Component> = new (...args: any[]) => T;

export type ComponentType<T extends Component = Component> = {
  new (...args: any[]): T;
  prototype: T;
};

// Extract component instance type from constructor
export type ComponentInstance<T> = T extends ComponentConstructor<infer C> ? C : never;

// Get constructor from instance
export type ComponentCtor<T extends Component> = ComponentConstructor<T>;

// Component with required properties
export interface ComponentWithId extends Component {
  id: string;
}

export interface ComponentWithName extends Component {
  name: string;
}

// Component type guards
export function isComponent(obj: unknown): obj is Component {
  return typeof obj === 'object' && obj !== null;
}

export function hasComponent<T extends Component>(
  entity: unknown,
  componentType: ComponentConstructor<T>
): entity is { getComponent(type: ComponentConstructor<T>): T } {
  return (
    typeof entity === 'object' &&
    entity !== null &&
    'getComponent' in entity &&
    typeof entity.getComponent === 'function'
  );
}
```

### 2. Type-Safe Entity

```typescript
// types/Entity.ts
export interface EntityComponents {
  // Map of component constructors to instances
  [key: string]: Component;
}

export class TypedEntity<TComponents extends EntityComponents = EntityComponents> {
  private components = new Map<ComponentConstructor<any>, Component>();

  addComponent<T extends Component>(
    component: T
  ): TypedEntity<TComponents & { [K in T['constructor']['name']]: T }> {
    const ctor = component.constructor as ComponentConstructor<T>;
    this.components.set(ctor, component);
    return this as any;
  }

  getComponent<T extends Component>(
    componentType: ComponentConstructor<T>
  ): T | undefined {
    return this.components.get(componentType) as T | undefined;
  }

  hasComponent<T extends Component>(componentType: ComponentConstructor<T>): boolean {
    return this.components.has(componentType);
  }

  removeComponent<T extends Component>(componentType: ComponentConstructor<T>): void {
    this.components.delete(componentType);
  }
}

// Usage with type inference
const entity = new TypedEntity()
  .addComponent(new Transform(new Vector3()))
  .addComponent(new Velocity(new Vector3()));

// TypeScript knows these exist
const transform = entity.getComponent(Transform); // Transform | undefined
const velocity = entity.getComponent(Velocity); // Velocity | undefined
```

### 3. Type-Safe Queries

```typescript
// types/Query.ts
export type QueryComponents<T extends readonly ComponentConstructor<any>[]> = {
  [K in keyof T]: T[K] extends ComponentConstructor<infer C> ? C : never;
};

export interface QueryResult<T extends readonly ComponentConstructor<any>[]> {
  entity: Entity;
  components: QueryComponents<T>;
}

export class TypedQuery<T extends readonly ComponentConstructor<any>[]> {
  constructor(private componentTypes: T) {}

  *iterate(world: World): Generator<QueryResult<T>> {
    const entities = world.queryArchetype(this.componentTypes);

    for (const entity of entities) {
      const components = this.componentTypes.map((type) =>
        entity.getComponent(type)
      ) as QueryComponents<T>;

      if (components.every((c) => c !== undefined)) {
        yield { entity, components };
      }
    }
  }

  // Helper to get strongly typed results
  forEach(world: World, callback: (result: QueryResult<T>) => void): void {
    for (const result of this.iterate(world)) {
      callback(result);
    }
  }
}

// Usage with type inference
const query = new TypedQuery([Transform, Velocity, Health] as const);

query.forEach(world, ({ entity, components }) => {
  const [transform, velocity, health] = components;
  // TypeScript knows: transform is Transform, velocity is Velocity, etc.

  transform.position.add(velocity.value);
  if (health.current <= 0) {
    entity.destroy();
  }
});

// Alternative: Tuple return type
export class TupleQuery<T extends readonly ComponentConstructor<any>[]> {
  constructor(private types: T) {}

  forEach(
    world: World,
    callback: (entity: Entity, components: QueryComponents<T>) => void
  ): void {
    const entities = world.queryArchetype(this.types);

    for (const entity of entities) {
      const components = this.types.map((type) =>
        entity.getComponent(type)
      ) as QueryComponents<T>;

      if (components.every((c) => c !== undefined)) {
        callback(entity, components);
      }
    }
  }
}

// Usage
const tupleQuery = new TupleQuery([Transform, Velocity] as const);

tupleQuery.forEach(world, (entity, [transform, velocity]) => {
  // Destructure directly
  transform.position.add(velocity.value);
});
```

### 4. System Type Patterns

```typescript
// types/System.ts
export interface System {
  priority?: number;
  enabled?: boolean;
  update(world: World, deltaTime: number): void;
}

export abstract class TypedSystem<TComponents extends readonly ComponentConstructor<any>[]>
  implements System
{
  priority = 0;
  enabled = true;

  constructor(protected componentTypes: TComponents) {}

  update(world: World, deltaTime: number): void {
    const query = new TypedQuery(this.componentTypes);

    query.forEach(world, ({ entity, components }) => {
      this.updateEntity(entity, components, deltaTime);
    });
  }

  protected abstract updateEntity(
    entity: Entity,
    components: QueryComponents<TComponents>,
    deltaTime: number
  ): void;
}

// Usage: Type-safe system implementation
export class MovementSystem extends TypedSystem<[typeof Transform, typeof Velocity]> {
  constructor() {
    super([Transform, Velocity] as const);
  }

  protected updateEntity(
    entity: Entity,
    [transform, velocity]: QueryComponents<[typeof Transform, typeof Velocity]>,
    deltaTime: number
  ): void {
    // TypeScript knows exact types
    transform.position.add(velocity.value.clone().multiplyScalar(deltaTime));
  }
}

// Generic system with constraints
export abstract class ComponentSystem<T extends Component> implements System {
  priority = 0;
  enabled = true;

  constructor(protected componentType: ComponentConstructor<T>) {}

  update(world: World, deltaTime: number): void {
    const entities = world.query([this.componentType]);

    entities.iterate((entity, [component]) => {
      this.updateComponent(entity, component as T, deltaTime);
    });
  }

  protected abstract updateComponent(entity: Entity, component: T, deltaTime: number): void;
}

// Usage
export class HealthRegenSystem extends ComponentSystem<Health> {
  constructor() {
    super(Health);
  }

  protected updateComponent(entity: Entity, health: Health, deltaTime: number): void {
    if (health.current < health.max) {
      health.current = Math.min(health.max, health.current + health.regenRate * deltaTime);
    }
  }
}
```

### 5. Component Registry with Types

```typescript
// types/Registry.ts
export class TypedRegistry {
  private components = new Map<string, ComponentConstructor<any>>();
  private names = new Map<ComponentConstructor<any>, string>();

  register<T extends Component>(name: string, ctor: ComponentConstructor<T>): void {
    this.components.set(name, ctor);
    this.names.set(ctor, name);
  }

  get<T extends Component>(name: string): ComponentConstructor<T> | undefined {
    return this.components.get(name) as ComponentConstructor<T> | undefined;
  }

  getName<T extends Component>(ctor: ComponentConstructor<T>): string | undefined {
    return this.names.get(ctor);
  }

  // Type-safe factory
  create<T extends Component>(
    name: string,
    ...args: ConstructorParameters<ComponentConstructor<T>>
  ): T | undefined {
    const ctor = this.components.get(name);
    if (!ctor) return undefined;

    return new ctor(...args) as T;
  }
}

// String literal types for component names
export type ComponentName =
  | 'Transform'
  | 'Velocity'
  | 'Health'
  | 'Sprite'
  | 'RigidBody'
  | 'Collider';

export class StrictRegistry {
  private components = new Map<ComponentName, ComponentConstructor<any>>();

  register<T extends Component>(name: ComponentName, ctor: ComponentConstructor<T>): void {
    this.components.set(name, ctor);
  }

  get<T extends Component>(name: ComponentName): ComponentConstructor<T> | undefined {
    return this.components.get(name) as ComponentConstructor<T> | undefined;
  }
}

// Usage
const registry = new StrictRegistry();
registry.register('Transform', Transform); // OK
registry.register('Invalid', Transform); // Error: Type '"Invalid"' is not assignable
```

### 6. Event Type Patterns

```typescript
// types/Events.ts
export interface BaseEvent {
  type: string;
  timestamp: number;
}

// Discriminated union for events
export type GameEvent =
  | { type: 'entity:created'; entity: Entity }
  | { type: 'entity:destroyed'; entity: Entity }
  | { type: 'component:added'; entity: Entity; component: Component }
  | { type: 'component:removed'; entity: Entity; componentType: string }
  | { type: 'damage'; target: Entity; source: Entity | null; amount: number }
  | { type: 'heal'; target: Entity; amount: number }
  | { type: 'death'; entity: Entity; cause: string };

// Type-safe event handler
export type EventHandler<T extends GameEvent> = (event: T) => void;

export class TypedEventBus {
  private handlers = new Map<GameEvent['type'], Set<EventHandler<any>>>();

  // Type-safe event subscription
  on<T extends GameEvent['type']>(
    type: T,
    handler: EventHandler<Extract<GameEvent, { type: T }>>
  ): () => void {
    let handlers = this.handlers.get(type);
    if (!handlers) {
      handlers = new Set();
      this.handlers.set(type, handlers);
    }

    handlers.add(handler);

    // Return unsubscribe function
    return () => {
      handlers?.delete(handler);
    };
  }

  // Type-safe event emission
  emit<T extends GameEvent>(event: T): void {
    const handlers = this.handlers.get(event.type);
    if (!handlers) return;

    for (const handler of handlers) {
      handler(event);
    }
  }
}

// Usage
const eventBus = new TypedEventBus();

// TypeScript enforces correct event shape
eventBus.on('damage', (event) => {
  // event is typed as: Extract<GameEvent, { type: 'damage' }>
  console.log(`${event.target.id} took ${event.amount} damage`);
});

eventBus.emit({
  type: 'damage',
  target: player,
  source: enemy,
  amount: 10,
}); // OK

eventBus.emit({
  type: 'damage',
  target: player,
  // Missing 'source' and 'amount' - TypeScript error!
});
```

### 7. Advanced Type Utilities

```typescript
// types/Utilities.ts

// Extract component types from entity
export type ExtractComponents<T> = T extends TypedEntity<infer C> ? C : never;

// Check if entity has specific component
export type HasComponent<T extends EntityComponents, C extends keyof EntityComponents> =
  C extends keyof T ? true : false;

// Merge component types
export type MergeComponents<T extends EntityComponents, U extends EntityComponents> = T & U;

// Optional components in query
export type OptionalComponent<T extends Component> = T | undefined;

export type QueryWithOptional<
  TRequired extends readonly ComponentConstructor<any>[],
  TOptional extends readonly ComponentConstructor<any>[]
> = {
  required: QueryComponents<TRequired>;
  optional: Partial<QueryComponents<TOptional>>;
};

// Mutable/Immutable component wrappers
export type Immutable<T> = {
  readonly [K in keyof T]: T[K] extends object ? Immutable<T[K]> : T[K];
};

export type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Component property extraction
export type ComponentProperty<T extends Component, K extends keyof T> = T[K];

export type ComponentProperties<T extends Component> = {
  [K in keyof T]: T[K];
};

// System query type helpers
export type SystemQuery<TSystem> = TSystem extends TypedSystem<infer C> ? C : never;

export type SystemComponents<TSystem> = TSystem extends TypedSystem<infer C>
  ? QueryComponents<C>
  : never;

// Archetype string type
export type Archetype = `${string}:${string}`;

export type ArchetypeComponents<T extends Archetype> = T extends `${infer C}:${infer Rest}`
  ? [C, ...ArchetypeComponents<Rest>]
  : [T];

// Component tuple from names
export type ComponentsFromNames<T extends readonly string[]> = {
  [K in keyof T]: T[K] extends keyof ComponentMap ? ComponentMap[T[K]] : never;
};

export interface ComponentMap {
  Transform: Transform;
  Velocity: Velocity;
  Health: Health;
  Sprite: Sprite;
}
```

## Usage Examples

```typescript
// Example 1: Type-safe entity builder
class EntityBuilder<T extends EntityComponents = {}> {
  private entity: TypedEntity<T>;

  constructor(world: World) {
    this.entity = world.createEntity() as TypedEntity<T>;
  }

  with<C extends Component>(
    component: C
  ): EntityBuilder<T & { [K in C['constructor']['name']]: C }> {
    this.entity.addComponent(component);
    return this as any;
  }

  build(): TypedEntity<T> {
    return this.entity;
  }
}

// Usage with full type inference
const player = new EntityBuilder(world)
  .with(new Transform(new Vector3()))
  .with(new Velocity(new Vector3()))
  .with(new Health(100))
  .build();

// TypeScript knows these components exist
player.getComponent(Transform); // Transform | undefined
player.getComponent(Health); // Health | undefined

// Example 2: Conditional component queries
type ConditionalQuery<T extends boolean> = T extends true
  ? [Transform, Velocity, Health]
  : [Transform, Velocity];

function createQuery<T extends boolean>(
  includeHealth: T
): TypedQuery<ConditionalQuery<T>> {
  const components = includeHealth
    ? ([Transform, Velocity, Health] as const)
    : ([Transform, Velocity] as const);

  return new TypedQuery(components as any);
}

// Example 3: Component serialization types
interface SerializedComponent {
  type: string;
  data: Record<string, any>;
}

type SerializableComponent = Component & {
  serialize(): SerializedComponent;
};

function isSerializable(component: Component): component is SerializableComponent {
  return 'serialize' in component && typeof component.serialize === 'function';
}

// Example 4: System dependencies
type SystemDependencies<T extends System[]> = T;

class OrderedSystemManager<T extends System[]> {
  constructor(private systems: SystemDependencies<T>) {}

  update(world: World, deltaTime: number): void {
    for (const system of this.systems) {
      if (system.enabled) {
        system.update(world, deltaTime);
      }
    }
  }
}

// Example 5: Component mixins
type ComponentMixin<T extends Component> = new (...args: any[]) => T;

function WithId<T extends ComponentConstructor<Component>>(Base: T) {
  return class extends Base implements ComponentWithId {
    id = Math.random().toString(36);
  };
}

function WithName<T extends ComponentConstructor<Component>>(Base: T) {
  return class extends Base implements ComponentWithName {
    name = 'Unnamed';
  };
}

// Usage
class Position extends Component {
  x = 0;
  y = 0;
}

const IdentifiedPosition = WithId(WithName(Position));
const pos = new IdentifiedPosition();
// pos has: x, y, id, name

// Example 6: Type-safe prefabs
type PrefabDefinition = {
  components: ComponentConstructor<any>[];
  properties: Record<string, any>;
};

class TypedPrefabManager {
  private prefabs = new Map<string, PrefabDefinition>();

  register(name: string, definition: PrefabDefinition): void {
    this.prefabs.set(name, definition);
  }

  instantiate<T extends readonly ComponentConstructor<any>[]>(
    name: string,
    world: World
  ): TypedEntity<QueryComponents<T>> | null {
    const definition = this.prefabs.get(name);
    if (!definition) return null;

    const entity = world.createEntity() as TypedEntity<any>;

    for (const ComponentType of definition.components) {
      const component = new ComponentType();
      entity.addComponent(component);
    }

    return entity;
  }
}

// Example 7: Generic component factory
class ComponentFactory {
  static create<T extends Component>(
    type: ComponentConstructor<T>,
    props?: Partial<T>
  ): T {
    const component = new type();

    if (props) {
      Object.assign(component, props);
    }

    return component;
  }
}

// Usage
const transform = ComponentFactory.create(Transform, {
  position: new Vector3(0, 5, 0),
});
```

## Checklist

- [ ] Define core component types
- [ ] Create type-safe entity class
- [ ] Implement typed queries
- [ ] Build system type patterns
- [ ] Create component registry
- [ ] Define event type system
- [ ] Add type utilities
- [ ] Test type inference
- [ ] Document type patterns
- [ ] Validate type safety

## Common Pitfalls

1. **Type assertions**: Overusing `as any`
2. **Weak typing**: Using `Component` instead of specific types
3. **No type guards**: Runtime type checks missing
4. **Complex generics**: Unreadable type signatures
5. **Missing constraints**: Generic types too broad
6. **Type/value mixing**: Confusion between types and values
7. **No inference**: Explicitly typing everything

## Performance Tips

### Type System Performance
- Use type inference over explicit types
- Avoid complex conditional types
- Limit generic constraint depth
- Use `as const` for literal types
- Cache component type lookups

### Runtime Performance
- Type guards inline in hot paths
- Cache component constructors
- Use Map for type lookups
- Avoid reflection where possible
- Pre-compute type relationships

### Development Experience
- Clear error messages with good types
- IntelliSense-friendly APIs
- Document complex types
- Use type aliases for readability
- Test type inference

### Mobile Considerations
- Simpler type hierarchies
- Fewer generic constraints
- Lighter type metadata
- Optimize bundle with tree shaking
- Remove type-only code

## Related Skills

- `typescript-game-types` - Game-specific types
- `typescript-performance` - TypeScript optimization
- `ecs-architecture` - Core ECS patterns
- `ecs-component-patterns` - Component design
- `ecs-system-patterns` - System implementation

## References

- TypeScript handbook
- Advanced types guide
- Generic constraints
- Type inference rules
