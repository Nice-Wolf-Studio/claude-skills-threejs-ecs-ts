---
name: ecs-events
description: Event-driven architecture in ECS including event buses, typed events, event queues, and event-based system communication
---

# ECS Events

## When to Use

Use this skill when:
- Systems need to communicate without tight coupling
- Implementing reactive game logic
- Broadcasting state changes
- Triggering animations or effects
- Handling user input events
- Coordinating between systems

## Core Principles

1. **Loose Coupling**: Systems don't directly reference each other
2. **Type Safety**: Events are strongly typed
3. **Queue-Based**: Events processed in order
4. **Frame-Delayed**: Events from frame N processed in frame N+1
5. **No Return Values**: Events are fire-and-forget
6. **Hierarchical**: Support event bubbling and capturing

## Event System Implementation

### 1. Event Base Classes

```typescript
// events/Event.ts
export interface Event {
  readonly type: string;
  readonly timestamp: number;
  cancelled?: boolean;
}

export abstract class BaseEvent implements Event {
  readonly timestamp: number;
  cancelled = false;

  constructor() {
    this.timestamp = performance.now();
  }

  get type(): string {
    return this.constructor.name;
  }

  cancel(): void {
    this.cancelled = true;
  }
}

// Example events
export class EntityDestroyedEvent extends BaseEvent {
  constructor(public readonly entity: Entity) {
    super();
  }
}

export class CollisionEvent extends BaseEvent {
  constructor(
    public readonly entityA: Entity,
    public readonly entityB: Entity,
    public readonly point: Vector3,
    public readonly normal: Vector3
  ) {
    super();
  }
}

export class DamageEvent extends BaseEvent {
  constructor(
    public readonly target: Entity,
    public readonly source: Entity | null,
    public readonly amount: number,
    public readonly damageType: string
  ) {
    super();
  }
}

export class InputEvent extends BaseEvent {
  constructor(
    public readonly action: string,
    public readonly pressed: boolean
  ) {
    super();
  }
}
```

### 2. Event Bus

```typescript
// events/EventBus.ts
type EventHandler<T extends Event> = (event: T) => void;
type EventType<T extends Event> = new (...args: any[]) => T;

export class EventBus {
  private listeners = new Map<string, Set<EventHandler<any>>>();
  private eventQueue: Event[] = [];
  private processingQueue: Event[] = [];
  private isProcessing = false;

  // Subscribe to events
  on<T extends Event>(
    eventType: EventType<T>,
    handler: EventHandler<T>
  ): () => void {
    const typeName = eventType.name;
    let handlers = this.listeners.get(typeName);

    if (!handlers) {
      handlers = new Set();
      this.listeners.set(typeName, handlers);
    }

    handlers.add(handler);

    // Return unsubscribe function
    return () => {
      const handlers = this.listeners.get(typeName);
      if (handlers) {
        handlers.delete(handler);
        if (handlers.size === 0) {
          this.listeners.delete(typeName);
        }
      }
    };
  }

  // Unsubscribe from events
  off<T extends Event>(
    eventType: EventType<T>,
    handler: EventHandler<T>
  ): void {
    const typeName = eventType.name;
    const handlers = this.listeners.get(typeName);
    if (handlers) {
      handlers.delete(handler);
      if (handlers.size === 0) {
        this.listeners.delete(typeName);
      }
    }
  }

  // Emit event (queued for next frame)
  emit<T extends Event>(event: T): void {
    if (this.isProcessing) {
      // If we're currently processing events, add to next frame's queue
      this.eventQueue.push(event);
    } else {
      this.eventQueue.push(event);
    }
  }

  // Emit event immediately (use sparingly)
  emitImmediate<T extends Event>(event: T): void {
    this.dispatchEvent(event);
  }

  // Process all queued events
  processEvents(): void {
    if (this.isProcessing) {
      console.warn('Already processing events');
      return;
    }

    // Swap queues
    this.processingQueue = this.eventQueue;
    this.eventQueue = [];
    this.isProcessing = true;

    // Process all events
    for (const event of this.processingQueue) {
      if (!event.cancelled) {
        this.dispatchEvent(event);
      }
    }

    this.processingQueue = [];
    this.isProcessing = false;
  }

  private dispatchEvent(event: Event): void {
    const handlers = this.listeners.get(event.type);
    if (handlers) {
      for (const handler of handlers) {
        handler(event);
        if (event.cancelled) break;
      }
    }
  }

  // Clear all listeners
  clear(): void {
    this.listeners.clear();
    this.eventQueue = [];
    this.processingQueue = [];
  }

  // Get statistics
  getStats(): { listeners: number; queuedEvents: number } {
    let totalListeners = 0;
    this.listeners.forEach((handlers) => {
      totalListeners += handlers.size;
    });

    return {
      listeners: totalListeners,
      queuedEvents: this.eventQueue.length,
    };
  }
}
```

### 3. Event-Driven Systems

```typescript
// systems/EventDrivenSystem.ts
export abstract class EventDrivenSystem extends UpdateSystem {
  protected eventBus: EventBus;
  private unsubscribers: Array<() => void> = [];

  constructor(eventBus: EventBus) {
    super();
    this.eventBus = eventBus;
  }

  // Helper to subscribe to events
  protected subscribe<T extends Event>(
    eventType: EventType<T>,
    handler: EventHandler<T>
  ): void {
    const unsubscribe = this.eventBus.on(eventType, handler);
    this.unsubscribers.push(unsubscribe);
  }

  cleanup(): void {
    // Unsubscribe from all events
    this.unsubscribers.forEach((unsub) => unsub());
    this.unsubscribers = [];
  }
}

// Example: Health system that listens to damage events
import { Health } from '../components/Health';
import { DamageEvent } from '../events/DamageEvent';

export class HealthSystem extends EventDrivenSystem {
  constructor(eventBus: EventBus) {
    super(eventBus);
    this.subscribe(DamageEvent, (event) => this.onDamage(event));
  }

  update(world: World, deltaTime: number): void {
    // Regular health regeneration
    const entities = world.query<[Health]>([Health]);

    entities.iterate((entity, [health]) => {
      if (health.current < health.max) {
        health.current = Math.min(
          health.current + health.regenRate * deltaTime,
          health.max
        );
      }
    });
  }

  private onDamage(event: DamageEvent): void {
    const health = event.target.getComponent(Health);
    if (!health) return;

    health.current -= event.amount;

    if (health.current <= 0) {
      health.current = 0;
      this.eventBus.emit(new EntityDestroyedEvent(event.target));
    }
  }
}
```

### 4. Priority Event Bus

```typescript
// events/PriorityEventBus.ts
interface PrioritizedHandler<T extends Event> {
  handler: EventHandler<T>;
  priority: number;
}

export class PriorityEventBus extends EventBus {
  private prioritizedListeners = new Map<string, PrioritizedHandler<any>[]>();

  onWithPriority<T extends Event>(
    eventType: EventType<T>,
    handler: EventHandler<T>,
    priority: number = 0
  ): () => void {
    const typeName = eventType.name;
    let handlers = this.prioritizedListeners.get(typeName);

    if (!handlers) {
      handlers = [];
      this.prioritizedListeners.set(typeName, handlers);
    }

    handlers.push({ handler, priority });
    // Sort by priority (higher numbers run first)
    handlers.sort((a, b) => b.priority - a.priority);

    return () => {
      const handlers = this.prioritizedListeners.get(typeName);
      if (handlers) {
        const index = handlers.findIndex((h) => h.handler === handler);
        if (index !== -1) {
          handlers.splice(index, 1);
        }
      }
    };
  }

  protected dispatchEvent(event: Event): void {
    const handlers = this.prioritizedListeners.get(event.type);
    if (handlers) {
      for (const { handler } of handlers) {
        handler(event);
        if (event.cancelled) break;
      }
    }
  }
}
```

### 5. Entity Events

```typescript
// events/EntityEventBus.ts
export class EntityEventBus extends EventBus {
  private entityListeners = new Map<Entity, Map<string, Set<EventHandler<any>>>>();

  // Subscribe to events from a specific entity
  onEntity<T extends Event>(
    entity: Entity,
    eventType: EventType<T>,
    handler: EventHandler<T>
  ): () => void {
    const typeName = eventType.name;
    let entityHandlers = this.entityListeners.get(entity);

    if (!entityHandlers) {
      entityHandlers = new Map();
      this.entityListeners.set(entity, entityHandlers);
    }

    let handlers = entityHandlers.get(typeName);
    if (!handlers) {
      handlers = new Set();
      entityHandlers.set(typeName, handlers);
    }

    handlers.add(handler);

    return () => {
      const entityHandlers = this.entityListeners.get(entity);
      if (entityHandlers) {
        const handlers = entityHandlers.get(typeName);
        if (handlers) {
          handlers.delete(handler);
        }
      }
    };
  }

  // Emit event from specific entity
  emitFrom<T extends Event>(entity: Entity, event: T): void {
    // Emit to entity-specific listeners
    const entityHandlers = this.entityListeners.get(entity);
    if (entityHandlers) {
      const handlers = entityHandlers.get(event.type);
      if (handlers) {
        for (const handler of handlers) {
          handler(event);
          if (event.cancelled) return;
        }
      }
    }

    // Also emit to global listeners
    this.emit(event);
  }

  // Clean up entity listeners
  clearEntity(entity: Entity): void {
    this.entityListeners.delete(entity);
  }
}
```

### 6. Event Recorder (Replay/Debug)

```typescript
// events/EventRecorder.ts
export class EventRecorder {
  private recordings: Event[] = [];
  private isRecording = false;
  private eventBus: EventBus;

  constructor(eventBus: EventBus) {
    this.eventBus = eventBus;
  }

  startRecording(): void {
    this.isRecording = true;
    this.recordings = [];

    // Intercept all events
    const originalEmit = this.eventBus.emit.bind(this.eventBus);
    this.eventBus.emit = (event: Event) => {
      if (this.isRecording) {
        this.recordings.push(this.cloneEvent(event));
      }
      originalEmit(event);
    };
  }

  stopRecording(): Event[] {
    this.isRecording = false;
    return this.recordings;
  }

  replay(events: Event[]): void {
    for (const event of events) {
      this.eventBus.emitImmediate(event);
    }
  }

  save(filename: string): void {
    const json = JSON.stringify(this.recordings, null, 2);
    // Save to file (implementation depends on environment)
  }

  load(json: string): void {
    this.recordings = JSON.parse(json);
  }

  private cloneEvent(event: Event): Event {
    return Object.assign(Object.create(Object.getPrototypeOf(event)), event);
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic event usage
const eventBus = new EventBus();

// Subscribe to events
eventBus.on(DamageEvent, (event) => {
  console.log(`${event.target.id} took ${event.amount} damage`);
});

// Emit events
eventBus.emit(new DamageEvent(player, enemy, 25, 'physical'));

// Process events (once per frame)
function gameLoop(deltaTime: number): void {
  eventBus.processEvents();
  systemManager.update(world, deltaTime);
}

// Example 2: Unsubscribing
const unsubscribe = eventBus.on(CollisionEvent, (event) => {
  console.log('Collision!');
});

// Later...
unsubscribe(); // Stop listening

// Example 3: Event-driven animation system
class AnimationEventSystem extends EventDrivenSystem {
  constructor(eventBus: EventBus) {
    super(eventBus);
    this.subscribe(DamageEvent, (event) => this.onDamage(event));
    this.subscribe(DeathEvent, (event) => this.onDeath(event));
    this.subscribe(JumpEvent, (event) => this.onJump(event));
  }

  update(world: World, deltaTime: number): void {
    // Normal animation updates
  }

  private onDamage(event: DamageEvent): void {
    const animator = event.target.getComponent(AnimationController);
    if (animator) {
      animator.play('hit', { oneShot: true });
    }
  }

  private onDeath(event: DeathEvent): void {
    const animator = event.entity.getComponent(AnimationController);
    if (animator) {
      animator.play('death', { loop: false });
    }
  }

  private onJump(event: JumpEvent): void {
    const animator = event.entity.getComponent(AnimationController);
    if (animator) {
      animator.play('jump', { oneShot: true });
    }
  }
}

// Example 4: Input event system
class InputEventSystem extends UpdateSystem {
  constructor(private eventBus: EventBus) {
    super();
    this.setupInputListeners();
  }

  private setupInputListeners(): void {
    window.addEventListener('keydown', (e) => {
      this.eventBus.emit(new InputEvent(e.key, true));
    });

    window.addEventListener('keyup', (e) => {
      this.eventBus.emit(new InputEvent(e.key, false));
    });
  }

  update(world: World, deltaTime: number): void {
    // Input system doesn't need update
  }
}

class PlayerControllerSystem extends EventDrivenSystem {
  constructor(eventBus: EventBus) {
    super(eventBus);
    this.subscribe(InputEvent, (event) => this.onInput(event));
  }

  update(world: World, deltaTime: number): void {
    // Update player movement based on input
  }

  private onInput(event: InputEvent): void {
    const player = world.query<[Player, Velocity]>([Player, Velocity]).first();
    if (!player) return;

    const [, velocity] = player.getComponents(Player, Velocity);

    if (event.action === 'w' && event.pressed) {
      velocity.z = -5;
    } else if (event.action === 's' && event.pressed) {
      velocity.z = 5;
    } else if (event.action === 'Space' && event.pressed) {
      this.eventBus.emit(new JumpEvent(player));
    }
  }
}

// Example 5: Combat system with events
class CombatSystem extends EventDrivenSystem {
  constructor(eventBus: EventBus) {
    super(eventBus);
    this.subscribe(AttackEvent, (event) => this.onAttack(event));
  }

  update(world: World, deltaTime: number): void {
    // Check for attacks
    const attackers = world.query<[Transform, Attack]>([Transform, Attack]);

    attackers.iterate((entity, [transform, attack]) => {
      attack.cooldown = Math.max(0, attack.cooldown - deltaTime);

      if (attack.wantsToAttack && attack.cooldown === 0) {
        this.eventBus.emit(new AttackEvent(entity, transform.position));
        attack.cooldown = attack.cooldownTime;
      }
    });
  }

  private onAttack(event: AttackEvent): void {
    // Find targets in range
    const spatialQuery = new SpatialQuery(world);
    const targets = spatialQuery.withinRadius(
      event.position,
      5, // Attack range
      [Health, Transform]
    );

    for (const target of targets) {
      if (target !== event.attacker) {
        this.eventBus.emit(new DamageEvent(target, event.attacker, 10, 'physical'));
      }
    }
  }
}

// Example 6: Sound system listening to events
class SoundSystem extends EventDrivenSystem {
  private audioContext: AudioContext;

  constructor(eventBus: EventBus) {
    super(eventBus);
    this.audioContext = new AudioContext();

    this.subscribe(DamageEvent, (event) => this.playSound('hit'));
    this.subscribe(DeathEvent, (event) => this.playSound('death'));
    this.subscribe(JumpEvent, (event) => this.playSound('jump'));
    this.subscribe(CollisionEvent, (event) => this.playSound('collision'));
  }

  update(world: World, deltaTime: number): void {
    // Sound system doesn't need update
  }

  private playSound(name: string): void {
    // Play sound using Web Audio API
  }
}

// Example 7: Event cancellation
eventBus.on(DamageEvent, (event) => {
  const armor = event.target.getComponent(Armor);
  if (armor && armor.isInvulnerable) {
    event.cancel(); // Prevent damage
  }
});
```

## Event Patterns

### Pattern 1: Event Chains

```typescript
// Events can trigger other events
class DeathSystem extends EventDrivenSystem {
  constructor(eventBus: EventBus) {
    super(eventBus);
    this.subscribe(DamageEvent, (event) => {
      const health = event.target.getComponent(Health);
      if (health && health.current <= 0) {
        // Damage event triggers death event
        this.eventBus.emit(new DeathEvent(event.target, event.source));
      }
    });

    this.subscribe(DeathEvent, (event) => {
      // Death event triggers loot drop event
      this.eventBus.emit(new LootDropEvent(event.entity));

      // And explosion effect event
      const transform = event.entity.getComponent(Transform);
      if (transform) {
        this.eventBus.emit(new ExplosionEvent(transform.position));
      }
    });
  }
}
```

### Pattern 2: Event Filtering

```typescript
// Only process events that meet criteria
eventBus.on(DamageEvent, (event) => {
  // Only log critical hits
  if (event.amount > 50) {
    console.log('CRITICAL HIT!');
  }
});

// Only process events for specific entity types
eventBus.on(CollisionEvent, (event) => {
  const isPlayer = event.entityA.hasComponent(Player) || event.entityB.hasComponent(Player);
  if (!isPlayer) return; // Ignore non-player collisions

  // Process player collision
});
```

### Pattern 3: Event Aggregation

```typescript
// Collect events over time for batch processing
class EventAggregator extends EventDrivenSystem {
  private damageEvents: DamageEvent[] = [];

  constructor(eventBus: EventBus) {
    super(eventBus);
    this.subscribe(DamageEvent, (event) => {
      this.damageEvents.push(event);
    });
  }

  update(world: World, deltaTime: number): void {
    if (this.damageEvents.length > 0) {
      // Process all damage at once
      this.processDamageBatch(this.damageEvents);
      this.damageEvents = [];
    }
  }

  private processDamageBatch(events: DamageEvent[]): void {
    // Batch processing logic
  }
}
```

## Checklist

- [ ] Define typed event classes
- [ ] Set up event bus in world
- [ ] Subscribe systems to relevant events
- [ ] Emit events at appropriate times
- [ ] Process events once per frame
- [ ] Handle event cancellation
- [ ] Unsubscribe when systems are destroyed
- [ ] Consider event priorities
- [ ] Profile event performance
- [ ] Document event flow

## Common Pitfalls

1. **Processing events immediately**: Queue events for next frame
2. **Memory leaks**: Unsubscribe when systems are destroyed
3. **Infinite loops**: Event triggers itself recursively
4. **Lost events**: Forgetting to call processEvents()
5. **Too many events**: Batch similar events
6. **Tight coupling**: Events with entity-specific logic
7. **Missing type safety**: Using string-based events

## Performance Tips

### Event System Optimization
- Process events in batches once per frame
- Use priority system sparingly
- Limit event payload size
- Unsubscribe unused listeners
- Profile event dispatch time

### Memory Optimization
- Pool event objects
- Limit event queue size
- Clear processed events immediately
- Use weak references where appropriate

### Debugging Events
```typescript
class EventDebugger {
  constructor(private eventBus: EventBus) {
    // Log all events
    const originalEmit = eventBus.emit.bind(eventBus);
    eventBus.emit = (event: Event) => {
      console.log(`Event: ${event.type}`, event);
      originalEmit(event);
    };
  }
}
```

## Related Skills

- `ecs-architecture` - Overall ECS structure
- `ecs-system-patterns` - System implementation
- `input-system` - Input event handling
- `collision-system` - Collision events
- `typescript-game-types` - Event type safety

## References

- Observer pattern (Gang of Four)
- Event sourcing architecture
- Unity Event System
- Reactive programming concepts
