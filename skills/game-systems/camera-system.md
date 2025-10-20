---
name: camera-system
description: Camera system for games including follow camera, orbit controls, shake effects, and cinematic cameras
---

# Camera System

## When to Use

Use this skill when:
- Implementing third-person follow camera
- Creating orbit/free-look camera
- Adding camera shake effects
- Building cinematics and cutscenes
- Managing multiple camera views
- Implementing camera transitions

## Core Principles

1. **Smooth Movement**: Use damping/lerping for smooth camera
2. **Configurable**: Expose camera settings
3. **Multiple Modes**: Support different camera behaviors
4. **Collision**: Prevent camera clipping through walls
5. **Transitions**: Smooth camera mode switching
6. **Shake Effects**: Screen shake for impact

## Camera System Implementation

### 1. Camera Components

```typescript
// components/CameraTarget.ts
export class CameraTarget {
  // Marks entity as camera target
  priority: number = 0;
}

// components/CameraController.ts
export enum CameraMode {
  Follow = 'follow',
  Orbit = 'orbit',
  FirstPerson = 'first-person',
  Fixed = 'fixed',
  Cinematic = 'cinematic',
}

export class CameraController {
  mode: CameraMode = CameraMode.Follow;

  // Follow mode settings
  followDistance: number = 10;
  followHeight: number = 5;
  followDamping: number = 5;
  lookAtOffset = new Vector3(0, 1, 0);

  // Orbit settings
  orbitDistance: number = 10;
  orbitMinDistance: number = 5;
  orbitMaxDistance: number = 20;
  orbitSpeed: number = 1;
  orbitDamping: number = 5;
  orbitAngleX: number = 0; // Pitch
  orbitAngleY: number = 0; // Yaw
  orbitMinAngleX: number = -Math.PI / 3;
  orbitMaxAngleX: number = Math.PI / 3;

  // First person settings
  firstPersonHeight: number = 1.6;
  firstPersonSensitivity: number = 0.002;

  // Shake settings
  shakeIntensity: number = 0;
  shakeDuration: number = 0;
  shakeTime: number = 0;

  // Collision
  collisionEnabled: boolean = true;
  collisionRadius: number = 0.5;

  constructor(mode: CameraMode = CameraMode.Follow) {
    this.mode = mode;
  }
}
```

### 2. Camera System

```typescript
// systems/CameraSystem.ts
export class CameraSystem extends UpdateSystem {
  priority = 65;
  private camera: THREE.Camera;
  private target: Entity | null = null;

  constructor(camera: THREE.Camera) {
    super();
    this.camera = camera;
  }

  update(world: World, deltaTime: number): void {
    // Find camera target
    this.updateTarget(world);

    // Update camera based on mode
    const controller = this.getController(world);
    if (!controller || !this.target) return;

    switch (controller.mode) {
      case CameraMode.Follow:
        this.updateFollowCamera(controller, deltaTime);
        break;

      case CameraMode.Orbit:
        this.updateOrbitCamera(controller, deltaTime);
        break;

      case CameraMode.FirstPerson:
        this.updateFirstPersonCamera(controller);
        break;

      case CameraMode.Fixed:
        // Fixed camera doesn't move
        break;

      case CameraMode.Cinematic:
        this.updateCinematicCamera(controller, deltaTime);
        break;
    }

    // Apply camera shake
    if (controller.shakeTime > 0) {
      this.applyCameraShake(controller, deltaTime);
    }

    // Handle collision
    if (controller.collisionEnabled) {
      this.handleCameraCollision(world, controller);
    }
  }

  private updateTarget(world: World): void {
    const targets = world.query<[Transform, CameraTarget]>([Transform, CameraTarget]);

    let highestPriority = -Infinity;
    let selectedTarget: Entity | null = null;

    targets.iterate((entity, [, target]) => {
      if (target.priority > highestPriority) {
        highestPriority = target.priority;
        selectedTarget = entity;
      }
    });

    this.target = selectedTarget;
  }

  private getController(world: World): CameraController | null {
    const controllers = world.query<[CameraController]>([CameraController]);
    const first = controllers.first();
    return first?.getComponent(CameraController) ?? null;
  }

  private updateFollowCamera(controller: CameraController, deltaTime: number): void {
    if (!this.target) return;

    const targetTransform = this.target.getComponent(Transform);
    if (!targetTransform) return;

    // Calculate desired position behind target
    const forward = new Vector3(0, 0, -1).applyQuaternion(targetTransform.rotation);
    const right = new Vector3(1, 0, 0).applyQuaternion(targetTransform.rotation);

    const desiredPosition = targetTransform.position.clone()
      .add(forward.multiplyScalar(-controller.followDistance))
      .add(new Vector3(0, controller.followHeight, 0));

    // Smoothly move camera to desired position
    const currentPosition = new Vector3(
      this.camera.position.x,
      this.camera.position.y,
      this.camera.position.z
    );

    const newPosition = new Vector3().lerpVectors(
      currentPosition,
      desiredPosition,
      1 - Math.exp(-controller.followDamping * deltaTime)
    );

    this.camera.position.set(newPosition.x, newPosition.y, newPosition.z);

    // Look at target with offset
    const lookAtPoint = targetTransform.position.clone().add(controller.lookAtOffset);
    this.camera.lookAt(lookAtPoint);
  }

  private updateOrbitCamera(controller: CameraController, deltaTime: number): void {
    if (!this.target) return;

    const targetTransform = this.target.getComponent(Transform);
    if (!targetTransform) return;

    // Calculate camera position from angles
    const x = Math.cos(controller.orbitAngleY) * Math.cos(controller.orbitAngleX);
    const y = Math.sin(controller.orbitAngleX);
    const z = Math.sin(controller.orbitAngleY) * Math.cos(controller.orbitAngleX);

    const offset = new Vector3(x, y, z).multiplyScalar(controller.orbitDistance);
    const desiredPosition = targetTransform.position.clone().add(offset);

    // Smooth movement
    const currentPosition = new Vector3(
      this.camera.position.x,
      this.camera.position.y,
      this.camera.position.z
    );

    const newPosition = new Vector3().lerpVectors(
      currentPosition,
      desiredPosition,
      1 - Math.exp(-controller.orbitDamping * deltaTime)
    );

    this.camera.position.set(newPosition.x, newPosition.y, newPosition.z);
    this.camera.lookAt(targetTransform.position);
  }

  private updateFirstPersonCamera(controller: CameraController): void {
    if (!this.target) return;

    const targetTransform = this.target.getComponent(Transform);
    if (!targetTransform) return;

    // Position camera at target position + height
    const cameraPosition = targetTransform.position.clone()
      .add(new Vector3(0, controller.firstPersonHeight, 0));

    this.camera.position.copy(cameraPosition);

    // Rotate based on target rotation
    this.camera.quaternion.copy(targetTransform.rotation);
  }

  private updateCinematicCamera(controller: CameraController, deltaTime: number): void {
    // Cinematic camera is controlled by external timeline/animation system
  }

  private applyCameraShake(controller: CameraController, deltaTime: number): void {
    controller.shakeTime -= deltaTime;

    if (controller.shakeTime <= 0) {
      controller.shakeTime = 0;
      controller.shakeIntensity = 0;
      return;
    }

    // Random shake offset
    const shake = new Vector3(
      (Math.random() - 0.5) * controller.shakeIntensity,
      (Math.random() - 0.5) * controller.shakeIntensity,
      (Math.random() - 0.5) * controller.shakeIntensity
    );

    this.camera.position.add(shake);
  }

  private handleCameraCollision(world: World, controller: CameraController): void {
    if (!this.target) return;

    const targetTransform = this.target.getComponent(Transform);
    if (!targetTransform) return;

    // Raycast from target to camera
    const direction = new Vector3()
      .subVectors(this.camera.position, targetTransform.position)
      .normalize();

    // TODO: Implement proper raycast against world geometry
    // If collision detected, move camera closer
  }

  shake(intensity: number, duration: number): void {
    const controller = this.getController(world);
    if (controller) {
      controller.shakeIntensity = intensity;
      controller.shakeDuration = duration;
      controller.shakeTime = duration;
    }
  }

  setMode(mode: CameraMode): void {
    const controller = this.getController(world);
    if (controller) {
      controller.mode = mode;
    }
  }

  getCamera(): THREE.Camera {
    return this.camera;
  }
}
```

### 3. Cinematic Camera System

```typescript
// camera/CinematicCamera.ts
export interface CameraKeyframe {
  time: number;
  position: Vector3;
  lookAt: Vector3;
  fov?: number;
}

export class CinematicCamera {
  private keyframes: CameraKeyframe[] = [];
  private currentTime = 0;
  private playing = false;
  private loop = false;

  addKeyframe(keyframe: CameraKeyframe): void {
    this.keyframes.push(keyframe);
    this.keyframes.sort((a, b) => a.time - b.time);
  }

  play(loop: boolean = false): void {
    this.playing = true;
    this.loop = loop;
    this.currentTime = 0;
  }

  stop(): void {
    this.playing = false;
    this.currentTime = 0;
  }

  pause(): void {
    this.playing = false;
  }

  resume(): void {
    this.playing = true;
  }

  update(camera: THREE.Camera, deltaTime: number): void {
    if (!this.playing || this.keyframes.length < 2) return;

    this.currentTime += deltaTime;

    const totalDuration = this.keyframes[this.keyframes.length - 1].time;

    if (this.currentTime >= totalDuration) {
      if (this.loop) {
        this.currentTime = 0;
      } else {
        this.stop();
        return;
      }
    }

    // Find keyframes to interpolate between
    let nextIndex = this.keyframes.findIndex((kf) => kf.time > this.currentTime);
    if (nextIndex === -1) nextIndex = this.keyframes.length - 1;

    const prevIndex = Math.max(0, nextIndex - 1);
    const prev = this.keyframes[prevIndex];
    const next = this.keyframes[nextIndex];

    if (prev === next) {
      // Use final keyframe
      camera.position.copy(prev.position);
      camera.lookAt(prev.lookAt);
      if (prev.fov !== undefined && camera instanceof THREE.PerspectiveCamera) {
        camera.fov = prev.fov;
        camera.updateProjectionMatrix();
      }
      return;
    }

    // Interpolate
    const t = (this.currentTime - prev.time) / (next.time - prev.time);

    // Position
    const position = new Vector3().lerpVectors(prev.position, next.position, t);
    camera.position.copy(position);

    // Look at
    const lookAt = new Vector3().lerpVectors(prev.lookAt, next.lookAt, t);
    camera.lookAt(lookAt);

    // FOV
    if (prev.fov !== undefined && next.fov !== undefined && camera instanceof THREE.PerspectiveCamera) {
      camera.fov = prev.fov + (next.fov - prev.fov) * t;
      camera.updateProjectionMatrix();
    }
  }

  isPlaying(): boolean {
    return this.playing;
  }
}
```

## Usage Examples

```typescript
// Example 1: Follow camera
const cameraSystem = new CameraSystem(camera);
systemManager.add(cameraSystem);

// Mark player as camera target
const player = world.createEntity();
player.addComponent(new Transform());
player.addComponent(new CameraTarget());

// Add camera controller
const cameraEntity = world.createEntity();
const controller = cameraEntity.addComponent(new CameraController(CameraMode.Follow));
controller.followDistance = 8;
controller.followHeight = 4;
controller.followDamping = 5;

// Example 2: Orbit camera with mouse input
const orbitController = new CameraController(CameraMode.Orbit);
orbitController.orbitDistance = 15;

document.addEventListener('mousemove', (e) => {
  if (e.buttons === 1) { // Left click
    orbitController.orbitAngleY -= e.movementX * orbitController.orbitSpeed * 0.01;
    orbitController.orbitAngleX -= e.movementY * orbitController.orbitSpeed * 0.01;

    // Clamp pitch
    orbitController.orbitAngleX = Math.max(
      orbitController.orbitMinAngleX,
      Math.min(orbitController.orbitMaxAngleX, orbitController.orbitAngleX)
    );
  }
});

document.addEventListener('wheel', (e) => {
  orbitController.orbitDistance += e.deltaY * 0.01;
  orbitController.orbitDistance = Math.max(
    orbitController.orbitMinDistance,
    Math.min(orbitController.orbitMaxDistance, orbitController.orbitDistance)
  );
});

// Example 3: Camera shake on explosion
eventBus.on(ExplosionEvent, (event) => {
  const distance = camera.position.distanceTo(event.position);
  const intensity = Math.max(0, 1 - distance / 20); // Fade with distance

  cameraSystem.shake(intensity * 0.5, 0.3);
});

// Example 4: Switch camera modes
function toggleCameraMode(): void {
  const currentMode = controller.mode;

  if (currentMode === CameraMode.Follow) {
    controller.mode = CameraMode.Orbit;
  } else {
    controller.mode = CameraMode.Follow;
  }
}

// Example 5: Cinematic sequence
const cinematic = new CinematicCamera();

cinematic.addKeyframe({
  time: 0,
  position: new Vector3(0, 5, 10),
  lookAt: new Vector3(0, 0, 0),
  fov: 75,
});

cinematic.addKeyframe({
  time: 3,
  position: new Vector3(10, 3, 5),
  lookAt: new Vector3(0, 1, 0),
  fov: 60,
});

cinematic.addKeyframe({
  time: 6,
  position: new Vector3(5, 8, -10),
  lookAt: new Vector3(0, 0, 0),
  fov: 75,
});

// Play cinematic
cinematic.play(false);

// Update in game loop
function gameLoop(deltaTime: number): void {
  if (cinematic.isPlaying()) {
    cinematic.update(camera, deltaTime);
  } else {
    // Normal camera system
    cameraSystem.update(world, deltaTime);
  }
}

// Example 6: Multiple camera targets with priorities
const boss = world.createEntity();
boss.addComponent(new Transform(new Vector3(20, 0, 0)));
const bossTarget = boss.addComponent(new CameraTarget());
bossTarget.priority = 10; // Higher priority during boss fight

const player = world.createEntity();
player.addComponent(new Transform());
const playerTarget = player.addComponent(new CameraTarget());
playerTarget.priority = 0; // Normal priority

// Camera will follow boss when active, player otherwise
```

## Checklist

- [ ] Create camera system
- [ ] Set camera target
- [ ] Configure camera mode
- [ ] Add camera damping/smoothing
- [ ] Implement camera shake
- [ ] Handle camera collision
- [ ] Add orbit controls
- [ ] Test camera transitions
- [ ] Create cinematic cameras
- [ ] Profile camera performance

## Common Pitfalls

1. **No damping**: Jerky camera movement
2. **Camera clipping**: Camera goes through walls
3. **Wrong look-at point**: Camera points at feet
4. **Too sensitive**: Hard to control orbit camera
5. **No min/max constraints**: Camera flips or zooms too far
6. **Forgetting FOV updates**: Projection matrix not updated
7. **Frame-rate dependent**: Camera speed varies with FPS

## Performance Tips

### Camera Optimization
- Update camera after physics
- Use exponential damping instead of lerp in update
- Limit raycast checks for collision
- Cache camera calculations
- Update FOV only when changed

### Memory Optimization
- Reuse Vector3 objects
- Avoid creating objects in update loop
- Pool camera shake calculations

### Mobile Considerations
- Simpler camera collision
- Fewer camera modes
- Touch-friendly orbit controls
- Lower damping values

## Related Skills

- `threejs-camera-controls` - Three.js camera patterns
- `input-system` - Camera input handling
- `ecs-system-patterns` - System implementation
- `collision-system` - Camera collision detection
- `mobile-performance` - Mobile optimization

## References

- Third-person camera design
- Orbit camera mathematics
- Camera shake techniques
- Cinematic camera systems
