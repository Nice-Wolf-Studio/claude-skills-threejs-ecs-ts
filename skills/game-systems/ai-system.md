---
name: ai-system
description: AI system for game entities including behavior trees, finite state machines, steering behaviors, and decision making
---

# AI System

## When to Use

Use this skill when:
- Creating enemy AI
- Implementing NPC behavior
- Building autonomous agents
- Creating steering and pathfinding
- Implementing decision-making systems
- Building tactical AI

## Core Principles

1. **Separation of Logic**: AI separate from rendering/physics
2. **Hierarchical Behavior**: Break complex behaviors into simple tasks
3. **State-Driven**: Use FSMs or behavior trees
4. **Performance-Aware**: Update frequency based on importance
5. **Debuggable**: Visualize AI state and decisions
6. **Data-Driven**: Configure AI in data files

## AI System Implementation

### 1. Finite State Machine (FSM)

```typescript
// ai/StateMachine.ts
export interface State {
  name: string;
  onEnter?: (entity: Entity) => void;
  onUpdate?: (entity: Entity, deltaTime: number) => void;
  onExit?: (entity: Entity) => void;
}

export interface Transition {
  from: string;
  to: string;
  condition: (entity: Entity) => boolean;
}

export class StateMachine {
  private states = new Map<string, State>();
  private transitions: Transition[] = [];
  private currentState: State | null = null;

  addState(state: State): void {
    this.states.set(state.name, state);
  }

  addTransition(transition: Transition): void {
    this.transitions.push(transition);
  }

  start(stateName: string, entity: Entity): void {
    const state = this.states.get(stateName);
    if (!state) {
      console.warn(`State not found: ${stateName}`);
      return;
    }

    if (this.currentState?.onExit) {
      this.currentState.onExit(entity);
    }

    this.currentState = state;

    if (this.currentState.onEnter) {
      this.currentState.onEnter(entity);
    }
  }

  update(entity: Entity, deltaTime: number): void {
    if (!this.currentState) return;

    // Update current state
    if (this.currentState.onUpdate) {
      this.currentState.onUpdate(entity, deltaTime);
    }

    // Check transitions
    for (const transition of this.transitions) {
      if (transition.from === this.currentState.name && transition.condition(entity)) {
        this.start(transition.to, entity);
        break;
      }
    }
  }

  getCurrentStateName(): string | null {
    return this.currentState?.name ?? null;
  }
}
```

### 2. Behavior Tree

```typescript
// ai/BehaviorTree.ts
export enum NodeStatus {
  Success = 'success',
  Failure = 'failure',
  Running = 'running',
}

export interface BehaviorNode {
  tick(entity: Entity, deltaTime: number): NodeStatus;
  reset?(): void;
}

// Leaf nodes
export class ActionNode implements BehaviorNode {
  constructor(private action: (entity: Entity, deltaTime: number) => NodeStatus) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    return this.action(entity, deltaTime);
  }
}

export class ConditionNode implements BehaviorNode {
  constructor(private condition: (entity: Entity) => boolean) {}

  tick(entity: Entity): NodeStatus {
    return this.condition(entity) ? NodeStatus.Success : NodeStatus.Failure;
  }
}

// Composite nodes
export class SequenceNode implements BehaviorNode {
  private currentIndex = 0;

  constructor(private children: BehaviorNode[]) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    while (this.currentIndex < this.children.length) {
      const status = this.children[this.currentIndex].tick(entity, deltaTime);

      if (status === NodeStatus.Failure) {
        this.reset();
        return NodeStatus.Failure;
      }

      if (status === NodeStatus.Running) {
        return NodeStatus.Running;
      }

      // Success - move to next child
      this.currentIndex++;
    }

    // All children succeeded
    this.reset();
    return NodeStatus.Success;
  }

  reset(): void {
    this.currentIndex = 0;
    this.children.forEach((child) => child.reset?.());
  }
}

export class SelectorNode implements BehaviorNode {
  private currentIndex = 0;

  constructor(private children: BehaviorNode[]) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    while (this.currentIndex < this.children.length) {
      const status = this.children[this.currentIndex].tick(entity, deltaTime);

      if (status === NodeStatus.Success) {
        this.reset();
        return NodeStatus.Success;
      }

      if (status === NodeStatus.Running) {
        return NodeStatus.Running;
      }

      // Failure - try next child
      this.currentIndex++;
    }

    // All children failed
    this.reset();
    return NodeStatus.Failure;
  }

  reset(): void {
    this.currentIndex = 0;
    this.children.forEach((child) => child.reset?.());
  }
}

export class ParallelNode implements BehaviorNode {
  constructor(
    private children: BehaviorNode[],
    private successThreshold: number = 1
  ) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    let successCount = 0;
    let runningCount = 0;

    for (const child of this.children) {
      const status = child.tick(entity, deltaTime);

      if (status === NodeStatus.Success) {
        successCount++;
      } else if (status === NodeStatus.Running) {
        runningCount++;
      }
    }

    if (successCount >= this.successThreshold) {
      return NodeStatus.Success;
    }

    if (runningCount > 0) {
      return NodeStatus.Running;
    }

    return NodeStatus.Failure;
  }

  reset(): void {
    this.children.forEach((child) => child.reset?.());
  }
}

// Decorator nodes
export class InverterNode implements BehaviorNode {
  constructor(private child: BehaviorNode) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    const status = this.child.tick(entity, deltaTime);

    if (status === NodeStatus.Success) return NodeStatus.Failure;
    if (status === NodeStatus.Failure) return NodeStatus.Success;
    return status;
  }

  reset(): void {
    this.child.reset?.();
  }
}

export class RepeaterNode implements BehaviorNode {
  private count = 0;

  constructor(private child: BehaviorNode, private maxCount: number = Infinity) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    if (this.count >= this.maxCount) {
      this.reset();
      return NodeStatus.Success;
    }

    const status = this.child.tick(entity, deltaTime);

    if (status === NodeStatus.Success || status === NodeStatus.Failure) {
      this.count++;
      this.child.reset?.();
    }

    return NodeStatus.Running;
  }

  reset(): void {
    this.count = 0;
    this.child.reset?.();
  }
}

export class BehaviorTree {
  constructor(private root: BehaviorNode) {}

  tick(entity: Entity, deltaTime: number): NodeStatus {
    return this.root.tick(entity, deltaTime);
  }

  reset(): void {
    this.root.reset?.();
  }
}
```

### 3. AI Components

```typescript
// components/AIController.ts
export class AIController {
  stateMachine: StateMachine | null = null;
  behaviorTree: BehaviorTree | null = null;
  updateFrequency: number = 10; // Hz
  timeSinceUpdate: number = 0;

  // State data
  target: Entity | null = null;
  lastKnownTargetPosition: Vector3 | null = null;
  alertLevel: number = 0; // 0 = idle, 1 = suspicious, 2 = alert
  patrolPoints: Vector3[] = [];
  currentPatrolIndex: number = 0;

  // Perception
  sightRange: number = 10;
  hearingRange: number = 5;
  fieldOfView: number = 120; // degrees

  constructor(updateFrequency: number = 10) {
    this.updateFrequency = updateFrequency;
  }
}

// components/Steering.ts
export class Steering {
  maxSpeed: number = 5;
  maxForce: number = 10;
  arrivalRadius: number = 0.5;
  slowingRadius: number = 2;

  // Current steering forces
  seek: Vector3 = new Vector3();
  flee: Vector3 = new Vector3();
  arrive: Vector3 = new Vector3();
  wander: Vector3 = new Vector3();

  // Wander state
  wanderAngle: number = 0;
  wanderRadius: number = 1;
  wanderDistance: number = 2;

  constructor(maxSpeed: number = 5, maxForce: number = 10) {
    this.maxSpeed = maxSpeed;
    this.maxForce = maxForce;
  }
}
```

### 4. AI Decision System

```typescript
// systems/AIDecisionSystem.ts
export class AIDecisionSystem extends IntervalSystem {
  priority = 30;

  constructor() {
    super(10); // 10 Hz update rate
  }

  protected tick(world: World): void {
    const entities = world.query<[Transform, AIController]>([Transform, AIController]);

    entities.iterate((entity, [transform, ai]) => {
      // Update state machine
      if (ai.stateMachine) {
        ai.stateMachine.update(entity, 1 / this.interval);
      }

      // Update behavior tree
      if (ai.behaviorTree) {
        ai.behaviorTree.tick(entity, 1 / this.interval);
      }

      // Perception updates
      this.updatePerception(entity, ai, transform, world);
    });
  }

  private updatePerception(
    entity: Entity,
    ai: AIController,
    transform: Transform,
    world: World
  ): void {
    // Find potential targets
    const targets = world.query<[Transform, Player]>([Transform, Player]);

    let closestTarget: Entity | null = null;
    let closestDistance = Infinity;

    targets.iterate((target, [targetTransform]) => {
      const distance = transform.position.distanceTo(targetTransform.position);

      // Check sight range
      if (distance > ai.sightRange) return;

      // Check field of view
      const toTarget = new Vector3()
        .subVectors(targetTransform.position, transform.position)
        .normalize();
      const forward = new Vector3(0, 0, -1).applyQuaternion(transform.rotation);
      const angle = Math.acos(forward.dot(toTarget)) * (180 / Math.PI);

      if (angle > ai.fieldOfView / 2) return;

      // TODO: Raycast for line of sight

      // Found target
      if (distance < closestDistance) {
        closestTarget = target;
        closestDistance = distance;
      }
    });

    if (closestTarget) {
      ai.target = closestTarget;
      const targetTransform = closestTarget.getComponent(Transform);
      ai.lastKnownTargetPosition = targetTransform.position.clone();
      ai.alertLevel = 2;
    } else if (ai.alertLevel > 0) {
      ai.alertLevel = Math.max(0, ai.alertLevel - 0.1);
    }
  }
}
```

### 5. Steering Behaviors System

```typescript
// systems/SteeringSystem.ts
export class SteeringSystem extends UpdateSystem {
  priority = 31;

  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, Velocity, Steering]>([
      Transform,
      Velocity,
      Steering,
    ]);

    entities.iterate((entity, [transform, velocity, steering]) => {
      const totalSteering = new Vector3();

      // Combine steering forces
      totalSteering.add(steering.seek);
      totalSteering.add(steering.flee);
      totalSteering.add(steering.arrive);
      totalSteering.add(steering.wander);

      // Limit steering force
      if (totalSteering.length() > steering.maxForce) {
        totalSteering.normalize().multiplyScalar(steering.maxForce);
      }

      // Apply steering to velocity
      velocity.add(totalSteering.multiplyScalar(deltaTime));

      // Limit velocity
      if (velocity.length() > steering.maxSpeed) {
        velocity.normalize().multiplyScalar(steering.maxSpeed);
      }

      // Clear steering forces
      steering.seek.set(0, 0, 0);
      steering.flee.set(0, 0, 0);
      steering.arrive.set(0, 0, 0);
      steering.wander.set(0, 0, 0);
    });
  }

  // Steering behaviors
  static seek(from: Vector3, to: Vector3, maxSpeed: number): Vector3 {
    return new Vector3().subVectors(to, from).normalize().multiplyScalar(maxSpeed);
  }

  static flee(from: Vector3, away: Vector3, maxSpeed: number): Vector3 {
    return new Vector3().subVectors(from, away).normalize().multiplyScalar(maxSpeed);
  }

  static arrive(
    from: Vector3,
    to: Vector3,
    maxSpeed: number,
    slowingRadius: number
  ): Vector3 {
    const desired = new Vector3().subVectors(to, from);
    const distance = desired.length();

    if (distance < slowingRadius) {
      desired.multiplyScalar((distance / slowingRadius) * maxSpeed);
    } else {
      desired.normalize().multiplyScalar(maxSpeed);
    }

    return desired;
  }

  static wander(
    steering: Steering,
    forward: Vector3,
    deltaTime: number
  ): Vector3 {
    // Update wander angle
    steering.wanderAngle += (Math.random() - 0.5) * Math.PI * deltaTime;

    // Calculate wander circle position
    const circleCenter = forward.clone().multiplyScalar(steering.wanderDistance);

    // Calculate displacement
    const displacement = new Vector3(
      Math.cos(steering.wanderAngle) * steering.wanderRadius,
      0,
      Math.sin(steering.wanderAngle) * steering.wanderRadius
    );

    return circleCenter.add(displacement);
  }
}
```

## Usage Examples

```typescript
// Example 1: Simple FSM for enemy AI
const enemyAI = new StateMachine();

// Define states
enemyAI.addState({
  name: 'patrol',
  onEnter: (entity) => {
    console.log('Starting patrol');
  },
  onUpdate: (entity, dt) => {
    const ai = entity.getComponent(AIController);
    const transform = entity.getComponent(Transform);
    const steering = entity.getComponent(Steering);

    // Move to next patrol point
    const target = ai.patrolPoints[ai.currentPatrolIndex];
    steering.arrive = SteeringSystem.arrive(
      transform.position,
      target,
      steering.maxSpeed,
      steering.slowingRadius
    );

    // Check if reached patrol point
    if (transform.position.distanceTo(target) < steering.arrivalRadius) {
      ai.currentPatrolIndex = (ai.currentPatrolIndex + 1) % ai.patrolPoints.length;
    }
  },
});

enemyAI.addState({
  name: 'chase',
  onUpdate: (entity, dt) => {
    const ai = entity.getComponent(AIController);
    const transform = entity.getComponent(Transform);
    const steering = entity.getComponent(Steering);

    if (ai.target) {
      const targetTransform = ai.target.getComponent(Transform);
      steering.seek = SteeringSystem.seek(
        transform.position,
        targetTransform.position,
        steering.maxSpeed
      );
    }
  },
});

enemyAI.addState({
  name: 'attack',
  onUpdate: (entity, dt) => {
    // Attack logic
  },
});

// Define transitions
enemyAI.addTransition({
  from: 'patrol',
  to: 'chase',
  condition: (entity) => {
    const ai = entity.getComponent(AIController);
    return ai.target !== null;
  },
});

enemyAI.addTransition({
  from: 'chase',
  to: 'attack',
  condition: (entity) => {
    const ai = entity.getComponent(AIController);
    const transform = entity.getComponent(Transform);
    if (!ai.target) return false;

    const targetTransform = ai.target.getComponent(Transform);
    return transform.position.distanceTo(targetTransform.position) < 2;
  },
});

enemyAI.addTransition({
  from: 'attack',
  to: 'patrol',
  condition: (entity) => {
    const ai = entity.getComponent(AIController);
    return ai.target === null;
  },
});

// Attach to entity
const enemy = world.createEntity();
const ai = enemy.addComponent(new AIController());
ai.stateMachine = enemyAI;
enemyAI.start('patrol', enemy);

// Example 2: Behavior tree for guard AI
const guardBehavior = new BehaviorTree(
  new SelectorNode([
    // Priority 1: If see player, chase
    new SequenceNode([
      new ConditionNode((entity) => {
        const ai = entity.getComponent(AIController);
        return ai.target !== null;
      }),
      new ActionNode((entity, dt) => {
        const ai = entity.getComponent(AIController);
        const transform = entity.getComponent(Transform);
        const steering = entity.getComponent(Steering);

        if (ai.target) {
          const targetPos = ai.target.getComponent(Transform).position;
          steering.seek = SteeringSystem.seek(
            transform.position,
            targetPos,
            steering.maxSpeed
          );
          return NodeStatus.Running;
        }
        return NodeStatus.Failure;
      }),
    ]),

    // Priority 2: Patrol
    new ActionNode((entity, dt) => {
      const ai = entity.getComponent(AIController);
      const transform = entity.getComponent(Transform);
      const steering = entity.getComponent(Steering);

      const target = ai.patrolPoints[ai.currentPatrolIndex];
      steering.arrive = SteeringSystem.arrive(
        transform.position,
        target,
        steering.maxSpeed,
        steering.slowingRadius
      );

      if (transform.position.distanceTo(target) < steering.arrivalRadius) {
        ai.currentPatrolIndex = (ai.currentPatrolIndex + 1) % ai.patrolPoints.length;
      }

      return NodeStatus.Running;
    }),
  ])
);

const guard = world.createEntity();
const guardAI = guard.addComponent(new AIController());
guardAI.behaviorTree = guardBehavior;

// Example 3: Flocking behavior
class FlockingSystem extends UpdateSystem {
  private separationRadius = 2;
  private alignmentRadius = 5;
  private cohesionRadius = 5;

  update(world: World, deltaTime: number): void {
    const boids = world.query<[Transform, Velocity, Steering, Boid]>([
      Transform,
      Velocity,
      Steering,
      Boid,
    ]);

    boids.iterate((entity, [transform, velocity, steering]) => {
      const separation = this.calculateSeparation(entity, transform, boids);
      const alignment = this.calculateAlignment(entity, velocity, boids);
      const cohesion = this.calculateCohesion(entity, transform, boids);

      steering.seek.add(separation.multiplyScalar(1.5));
      steering.seek.add(alignment.multiplyScalar(1.0));
      steering.seek.add(cohesion.multiplyScalar(1.0));
    });
  }

  private calculateSeparation(
    entity: Entity,
    transform: Transform,
    boids: QueryResult<any>
  ): Vector3 {
    const steer = new Vector3();
    let count = 0;

    boids.iterate((other, [otherTransform]) => {
      if (other === entity) return;

      const distance = transform.position.distanceTo(otherTransform.position);
      if (distance < this.separationRadius && distance > 0) {
        const diff = new Vector3()
          .subVectors(transform.position, otherTransform.position)
          .normalize()
          .divideScalar(distance);
        steer.add(diff);
        count++;
      }
    });

    if (count > 0) {
      steer.divideScalar(count);
    }

    return steer;
  }

  private calculateAlignment(
    entity: Entity,
    velocity: Velocity,
    boids: QueryResult<any>
  ): Vector3 {
    const avgVelocity = new Vector3();
    let count = 0;

    boids.iterate((other, [, otherVelocity]) => {
      if (other === entity) return;

      avgVelocity.add(otherVelocity);
      count++;
    });

    if (count > 0) {
      avgVelocity.divideScalar(count);
    }

    return avgVelocity;
  }

  private calculateCohesion(
    entity: Entity,
    transform: Transform,
    boids: QueryResult<any>
  ): Vector3 {
    const center = new Vector3();
    let count = 0;

    boids.iterate((other, [otherTransform]) => {
      if (other === entity) return;

      center.add(otherTransform.position);
      count++;
    });

    if (count > 0) {
      center.divideScalar(count);
      return SteeringSystem.seek(transform.position, center, 5);
    }

    return new Vector3();
  }
}
```

## Checklist

- [ ] Choose AI architecture (FSM, Behavior Tree, or hybrid)
- [ ] Define AI states or behavior nodes
- [ ] Implement state transitions or tree logic
- [ ] Add perception system (sight, hearing)
- [ ] Implement steering behaviors
- [ ] Configure AI update frequency
- [ ] Test AI decision making
- [ ] Add debug visualization
- [ ] Profile AI performance
- [ ] Handle edge cases

## Common Pitfalls

1. **Updating every frame**: Too expensive for many AI agents
2. **No perception limits**: AI knows everything
3. **Perfect aim**: Unrealistic and unfun
4. **Complex behavior trees**: Hard to debug
5. **State explosion**: Too many states in FSM
6. **No randomness**: Predictable AI
7. **Instant reactions**: No human-like delay

## Performance Tips

### AI Optimization
- Update AI at lower frequency (10-20 Hz)
- Use LOD for AI complexity
- Disable AI for distant/off-screen entities
- Cache expensive queries
- Use spatial partitioning for perception

### Memory Optimization
- Pool behavior tree nodes
- Share common behaviors
- Use flyweight pattern for AI data
- Limit perception queries

### Mobile Considerations
- Reduce AI update rate (5-10 Hz)
- Limit active AI agents (<20)
- Simpler behavior trees
- Disable perception for distant agents
- Use simplified steering

## Related Skills

- `ecs-system-patterns` - System implementation
- `ecs-queries` - Entity queries
- `ecs-events` - AI events
- `collision-system` - Line of sight
- `input-system` - Player control

## References

- Game AI Pro book series
- Behavior trees (game AI)
- Steering behaviors (Craig Reynolds)
- Goal-Oriented Action Planning (GOAP)
- Utility AI systems
