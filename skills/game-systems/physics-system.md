---
name: physics-system
description: Physics simulation system for ECS including rigid body dynamics, forces, constraints, and integration with physics engines
---

# Physics System

## When to Use

Use this skill when:
- Implementing realistic physics simulation
- Adding gravity and forces
- Handling rigid body dynamics
- Integrating physics engines (Cannon.js, Rapier)
- Creating vehicle physics
- Building destruction systems

## Core Principles

1. **Fixed Timestep**: Physics updates at constant rate
2. **Separation of Concerns**: Physics separate from rendering
3. **Integration Ready**: Support external physics engines
4. **Deterministic**: Same input produces same output
5. **Performance-Aware**: Optimize for many bodies
6. **Sleeping Bodies**: Don't update static objects

## Physics System Implementation

### 1. Basic Physics Components

```typescript
// components/RigidBody.ts
import { Vector3 } from 'three';

export enum BodyType {
  Static = 'static',
  Dynamic = 'dynamic',
  Kinematic = 'kinematic',
}

export class RigidBody {
  type: BodyType = BodyType.Dynamic;
  mass: number = 1;
  velocity = new Vector3();
  angularVelocity = new Vector3();
  force = new Vector3();
  torque = new Vector3();

  // Material properties
  restitution: number = 0.3; // Bounciness (0-1)
  friction: number = 0.5; // Friction (0-1)
  linearDamping: number = 0.01; // Air resistance
  angularDamping: number = 0.01; // Rotational resistance

  // Constraints
  lockPositionX: boolean = false;
  lockPositionY: boolean = false;
  lockPositionZ: boolean = false;
  lockRotationX: boolean = false;
  lockRotationY: boolean = false;
  lockRotationZ: boolean = false;

  // Sleep state (optimization)
  isSleeping: boolean = false;
  sleepThreshold: number = 0.01;

  constructor(mass: number = 1, type: BodyType = BodyType.Dynamic) {
    this.mass = mass;
    this.type = type;
  }

  applyForce(force: Vector3, point?: Vector3): void {
    if (this.type !== BodyType.Dynamic) return;

    this.force.add(force);

    if (point) {
      // Apply torque if force applied at offset point
      // torque = point × force
      const torque = new Vector3().crossVectors(point, force);
      this.torque.add(torque);
    }
  }

  applyImpulse(impulse: Vector3): void {
    if (this.type !== BodyType.Dynamic) return;

    // Impulse = instant change in velocity
    this.velocity.add(impulse.clone().divideScalar(this.mass));
  }

  clearForces(): void {
    this.force.set(0, 0, 0);
    this.torque.set(0, 0, 0);
  }

  shouldSleep(): boolean {
    const speed = this.velocity.length() + this.angularVelocity.length();
    return speed < this.sleepThreshold;
  }
}

// components/Collider.ts
export enum ColliderShape {
  Box = 'box',
  Sphere = 'sphere',
  Capsule = 'capsule',
  Cylinder = 'cylinder',
  Mesh = 'mesh',
}

export class Collider {
  shape: ColliderShape = ColliderShape.Box;
  size = new Vector3(1, 1, 1); // Box dimensions or sphere radius
  offset = new Vector3(); // Offset from transform
  isTrigger: boolean = false; // Trigger vs solid collision
  layer: number = 0; // Collision layer
  mask: number = 0xffffffff; // Which layers to collide with

  constructor(shape: ColliderShape = ColliderShape.Box, size?: Vector3) {
    this.shape = shape;
    if (size) this.size.copy(size);
  }
}
```

### 2. Simple Physics System (No Engine)

```typescript
// systems/SimplePhysicsSystem.ts
import { Transform } from '../components/Transform';
import { RigidBody } from '../components/RigidBody';
import { UpdateSystem } from '../core/System';

export class SimplePhysicsSystem extends UpdateSystem {
  priority = 20;
  private gravity = new Vector3(0, -9.81, 0);
  private accumulator = 0;
  private fixedDeltaTime = 1 / 60; // 60 FPS physics

  update(world: World, deltaTime: number): void {
    this.accumulator += deltaTime;

    // Fixed timestep updates
    while (this.accumulator >= this.fixedDeltaTime) {
      this.fixedUpdate(world, this.fixedDeltaTime);
      this.accumulator -= this.fixedDeltaTime;
    }
  }

  private fixedUpdate(world: World, dt: number): void {
    const entities = world.query<[Transform, RigidBody]>([Transform, RigidBody]);

    entities.iterate((entity, [transform, body]) => {
      // Skip static and sleeping bodies
      if (body.type === BodyType.Static || body.isSleeping) return;

      // Apply gravity
      if (body.type === BodyType.Dynamic) {
        body.force.add(this.gravity.clone().multiplyScalar(body.mass));
      }

      // Calculate acceleration (F = ma → a = F/m)
      const acceleration = body.force.clone().divideScalar(body.mass);

      // Integrate velocity (v = v0 + a*dt)
      body.velocity.add(acceleration.multiplyScalar(dt));

      // Apply damping
      body.velocity.multiplyScalar(1 - body.linearDamping);
      body.angularVelocity.multiplyScalar(1 - body.angularDamping);

      // Apply constraints
      if (body.lockPositionX) body.velocity.x = 0;
      if (body.lockPositionY) body.velocity.y = 0;
      if (body.lockPositionZ) body.velocity.z = 0;

      // Integrate position (p = p0 + v*dt)
      if (body.type === BodyType.Dynamic) {
        transform.position.add(body.velocity.clone().multiplyScalar(dt));
      }

      // Integrate rotation
      if (!body.lockRotationX && !body.lockRotationY && !body.lockRotationZ) {
        const angularSpeed = body.angularVelocity.length();
        if (angularSpeed > 0) {
          const axis = body.angularVelocity.clone().normalize();
          const angle = angularSpeed * dt;
          const rotation = new Quaternion().setFromAxisAngle(axis, angle);
          transform.rotation.multiply(rotation);
        }
      }

      // Clear forces for next frame
      body.clearForces();

      // Check sleep state
      if (body.shouldSleep()) {
        body.isSleeping = true;
      }
    });
  }

  setGravity(gravity: Vector3): void {
    this.gravity.copy(gravity);
  }
}
```

### 3. Cannon.js Physics Integration

```typescript
// systems/CannonPhysicsSystem.ts
import * as CANNON from 'cannon-es';
import { Transform } from '../components/Transform';
import { RigidBody, BodyType } from '../components/RigidBody';
import { Collider, ColliderShape } from '../components/Collider';

export class CannonPhysicsSystem extends UpdateSystem {
  priority = 20;
  private world: CANNON.World;
  private bodies = new Map<Entity, CANNON.Body>();
  private fixedDeltaTime = 1 / 60;

  constructor() {
    super();

    // Create physics world
    this.world = new CANNON.World({
      gravity: new CANNON.Vec3(0, -9.81, 0),
    });

    // Collision detection optimization
    this.world.broadphase = new CANNON.SAPBroadphase(this.world);
    this.world.allowSleep = true;
  }

  update(gameWorld: World, deltaTime: number): void {
    // Add new physics bodies
    this.syncNewBodies(gameWorld);

    // Step physics simulation
    this.world.step(this.fixedDeltaTime, deltaTime, 3);

    // Sync physics back to transforms
    this.syncToTransforms(gameWorld);
  }

  private syncNewBodies(gameWorld: World): void {
    const entities = gameWorld.query<[Transform, RigidBody, Collider]>([
      Transform,
      RigidBody,
      Collider,
    ]);

    entities.iterate((entity, [transform, rigidBody, collider]) => {
      if (this.bodies.has(entity)) return; // Already created

      // Create Cannon body
      const shape = this.createShape(collider);
      const body = new CANNON.Body({
        mass: rigidBody.type === BodyType.Dynamic ? rigidBody.mass : 0,
        position: new CANNON.Vec3(
          transform.position.x,
          transform.position.y,
          transform.position.z
        ),
        quaternion: new CANNON.Quaternion(
          transform.rotation.x,
          transform.rotation.y,
          transform.rotation.z,
          transform.rotation.w
        ),
        type: this.getCannonBodyType(rigidBody.type),
        material: new CANNON.Material({
          friction: rigidBody.friction,
          restitution: rigidBody.restitution,
        }),
        linearDamping: rigidBody.linearDamping,
        angularDamping: rigidBody.angularDamping,
        sleepSpeedLimit: rigidBody.sleepThreshold,
      });

      body.addShape(shape, new CANNON.Vec3(
        collider.offset.x,
        collider.offset.y,
        collider.offset.z
      ));

      this.world.addBody(body);
      this.bodies.set(entity, body);
    });
  }

  private syncToTransforms(gameWorld: World): void {
    this.bodies.forEach((body, entity) => {
      const transform = entity.getComponent(Transform);
      const rigidBody = entity.getComponent(RigidBody);

      if (transform && rigidBody) {
        // Sync position
        transform.position.set(body.position.x, body.position.y, body.position.z);

        // Sync rotation
        transform.rotation.set(
          body.quaternion.x,
          body.quaternion.y,
          body.quaternion.z,
          body.quaternion.w
        );

        // Sync velocity
        rigidBody.velocity.set(body.velocity.x, body.velocity.y, body.velocity.z);
        rigidBody.angularVelocity.set(
          body.angularVelocity.x,
          body.angularVelocity.y,
          body.angularVelocity.z
        );

        // Sync sleep state
        rigidBody.isSleeping = body.sleepState === CANNON.Body.SLEEPING;
      }
    });
  }

  private createShape(collider: Collider): CANNON.Shape {
    switch (collider.shape) {
      case ColliderShape.Box:
        return new CANNON.Box(
          new CANNON.Vec3(collider.size.x / 2, collider.size.y / 2, collider.size.z / 2)
        );

      case ColliderShape.Sphere:
        return new CANNON.Sphere(collider.size.x);

      case ColliderShape.Cylinder:
        return new CANNON.Cylinder(
          collider.size.x,
          collider.size.x,
          collider.size.y,
          8
        );

      default:
        return new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5));
    }
  }

  private getCannonBodyType(type: BodyType): number {
    switch (type) {
      case BodyType.Static:
        return CANNON.Body.STATIC;
      case BodyType.Dynamic:
        return CANNON.Body.DYNAMIC;
      case BodyType.Kinematic:
        return CANNON.Body.KINEMATIC;
    }
  }

  removeBody(entity: Entity): void {
    const body = this.bodies.get(entity);
    if (body) {
      this.world.removeBody(body);
      this.bodies.delete(entity);
    }
  }

  raycast(from: Vector3, to: Vector3): CANNON.RaycastResult | null {
    const result = new CANNON.RaycastResult();
    this.world.raycastClosest(
      new CANNON.Vec3(from.x, from.y, from.z),
      new CANNON.Vec3(to.x, to.y, to.z),
      {},
      result
    );

    return result.hasHit ? result : null;
  }

  cleanup(): void {
    this.bodies.forEach((body) => this.world.removeBody(body));
    this.bodies.clear();
  }
}
```

### 4. Character Controller

```typescript
// systems/CharacterControllerSystem.ts
export class CharacterController {
  speed: number = 5;
  jumpForce: number = 5;
  isGrounded: boolean = false;
  groundCheckDistance: number = 0.1;

  constructor(speed: number = 5, jumpForce: number = 5) {
    this.speed = speed;
    this.jumpForce = jumpForce;
  }
}

export class CharacterControllerSystem extends UpdateSystem {
  priority = 25; // After physics

  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, RigidBody, CharacterController]>([
      Transform,
      RigidBody,
      CharacterController,
    ]);

    entities.iterate((entity, [transform, body, controller]) => {
      // Check if grounded
      controller.isGrounded = this.checkGrounded(world, transform, controller);

      // Apply movement input
      const input = this.getMovementInput();
      const moveDir = new Vector3(input.x, 0, input.z).normalize();

      if (moveDir.length() > 0) {
        const velocity = moveDir.multiplyScalar(controller.speed);
        body.velocity.x = velocity.x;
        body.velocity.z = velocity.z;
      }

      // Jump
      if (input.jump && controller.isGrounded) {
        body.velocity.y = controller.jumpForce;
      }
    });
  }

  private checkGrounded(
    world: World,
    transform: Transform,
    controller: CharacterController
  ): boolean {
    // Simple ground check (raycast downward)
    const from = transform.position.clone();
    const to = from.clone().add(new Vector3(0, -controller.groundCheckDistance, 0));

    // TODO: Implement raycast against physics world
    return transform.position.y <= 0.1;
  }

  private getMovementInput(): { x: number; z: number; jump: boolean } {
    // TODO: Get from input system
    return { x: 0, z: 0, jump: false };
  }
}
```

### 5. Vehicle Physics

```typescript
// components/Vehicle.ts
export class Vehicle {
  wheelBase: number = 2.5; // Distance between front and rear axles
  maxSteerAngle: number = 0.6; // Max steering angle in radians
  motorForce: number = 1000;
  brakeForce: number = 500;

  currentSteerAngle: number = 0;
  currentSpeed: number = 0;
  throttle: number = 0; // -1 to 1
  brake: number = 0; // 0 to 1
  steer: number = 0; // -1 to 1

  wheels: Vector3[] = [
    new Vector3(-1, 0, 1.5),   // Front left
    new Vector3(1, 0, 1.5),    // Front right
    new Vector3(-1, 0, -1.5),  // Rear left
    new Vector3(1, 0, -1.5),   // Rear right
  ];
}

// systems/VehiclePhysicsSystem.ts
export class VehiclePhysicsSystem extends UpdateSystem {
  priority = 22;

  update(world: World, deltaTime: number): void {
    const entities = world.query<[Transform, RigidBody, Vehicle]>([
      Transform,
      RigidBody,
      Vehicle,
    ]);

    entities.iterate((entity, [transform, body, vehicle]) => {
      // Steering
      vehicle.currentSteerAngle = vehicle.steer * vehicle.maxSteerAngle;

      // Calculate forces
      const forward = new Vector3(0, 0, -1).applyQuaternion(transform.rotation);
      const right = new Vector3(1, 0, 0).applyQuaternion(transform.rotation);

      // Motor force
      if (vehicle.throttle !== 0) {
        const motorForce = forward
          .clone()
          .multiplyScalar(vehicle.throttle * vehicle.motorForce);
        body.applyForce(motorForce);
      }

      // Brake force
      if (vehicle.brake > 0) {
        const brakeForce = body.velocity
          .clone()
          .multiplyScalar(-vehicle.brake * vehicle.brakeForce);
        body.applyForce(brakeForce);
      }

      // Steering (simplified)
      if (Math.abs(vehicle.currentSteerAngle) > 0.01) {
        const speed = body.velocity.length();
        if (speed > 0.1) {
          const turnRate = (vehicle.currentSteerAngle / vehicle.wheelBase) * speed;
          body.angularVelocity.y = turnRate;
        }
      }

      vehicle.currentSpeed = body.velocity.length();
    });
  }
}
```

## Usage Examples

```typescript
// Example 1: Simple physics
const physicsSystem = new SimplePhysicsSystem();
systemManager.add(physicsSystem);

// Create falling box
const box = world.createEntity();
box.addComponent(new Transform(new Vector3(0, 10, 0)));
box.addComponent(new RigidBody(1, BodyType.Dynamic));
box.addComponent(new Collider(ColliderShape.Box, new Vector3(1, 1, 1)));

// Create static ground
const ground = world.createEntity();
ground.addComponent(new Transform(new Vector3(0, 0, 0)));
ground.addComponent(new RigidBody(0, BodyType.Static));
ground.addComponent(new Collider(ColliderShape.Box, new Vector3(10, 0.1, 10)));

// Example 2: Cannon.js physics
const cannonPhysics = new CannonPhysicsSystem();
systemManager.add(cannonPhysics);

// Apply force to entity
const body = entity.getComponent(RigidBody);
body.applyForce(new Vector3(0, 100, 0));

// Apply impulse (instant velocity change)
body.applyImpulse(new Vector3(0, 5, 0));

// Example 3: Character controller
const player = world.createEntity();
player.addComponent(new Transform(new Vector3(0, 1, 0)));
player.addComponent(new RigidBody(1, BodyType.Dynamic));
player.addComponent(new Collider(ColliderShape.Capsule, new Vector3(0.5, 1, 0.5)));
player.addComponent(new CharacterController(5, 10));

// Lock rotation so character doesn't tip over
const playerBody = player.getComponent(RigidBody);
playerBody.lockRotationX = true;
playerBody.lockRotationZ = true;

// Example 4: Vehicle
const car = world.createEntity();
car.addComponent(new Transform(new Vector3(0, 1, 0)));
car.addComponent(new RigidBody(1000, BodyType.Dynamic));
car.addComponent(new Collider(ColliderShape.Box, new Vector3(2, 1, 4)));
car.addComponent(new Vehicle());

// Control vehicle
const vehicle = car.getComponent(Vehicle);
vehicle.throttle = 1; // Full throttle
vehicle.steer = -0.5; // Turn left
vehicle.brake = 0;

// Example 5: Ragdoll physics
function createRagdoll(world: World, position: Vector3): Entity {
  const ragdoll = world.createEntity();

  // Create body parts with joints
  const torso = createBodyPart(world, position, new Vector3(0.5, 1, 0.3));
  const head = createBodyPart(world, position.clone().add(new Vector3(0, 1.5, 0)),
    new Vector3(0.3, 0.3, 0.3));
  // ... more body parts

  return ragdoll;
}
```

## Checklist

- [ ] Choose physics engine (simple vs Cannon.js/Rapier)
- [ ] Add RigidBody components to moving objects
- [ ] Add Collider components for collision shapes
- [ ] Configure mass and material properties
- [ ] Set up collision layers and masks
- [ ] Implement fixed timestep for physics
- [ ] Sync physics to transforms
- [ ] Handle body creation and removal
- [ ] Test with different timesteps
- [ ] Profile physics performance

## Common Pitfalls

1. **Variable timestep**: Physics becomes unstable
2. **No collision layers**: Everything collides with everything
3. **Heavy bodies**: Performance issues with many dynamic bodies
4. **No sleeping**: Wasting CPU on static bodies
5. **Forgetting to sync**: Physics and rendering out of sync
6. **Large forces**: Bodies clip through geometry
7. **No constraints**: Characters tip over

## Performance Tips

### Physics Optimization
- Use fixed timestep (60 Hz typical)
- Enable body sleeping for static objects
- Use collision layers to reduce checks
- Simplify collision shapes (boxes > spheres > meshes)
- Reduce physics simulation frequency on mobile

### Memory Optimization
- Pool physics bodies
- Reuse shapes when possible
- Limit maximum active bodies
- Use compound shapes sparingly

### Mobile Considerations
- Lower physics update rate (30 Hz)
- Fewer dynamic bodies (<50)
- Simpler collision shapes
- Disable sleeping for performance
- Use spatial partitioning

## Related Skills

- `collision-system` - Collision detection
- `input-system` - Player control
- `ecs-performance` - ECS optimization
- `threejs-scene-setup` - Integration with Three.js
- `mobile-performance` - Mobile optimization

## References

- Cannon.js: https://github.com/pmndrs/cannon-es
- Rapier: https://rapier.rs/
- Game Physics Engine Development (Ian Millington)
- Real-Time Collision Detection (Christer Ericson)
