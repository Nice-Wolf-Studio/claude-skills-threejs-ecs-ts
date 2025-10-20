---
name: threejs-camera-controls
description: Camera systems for Three.js including orbit controls, first-person, third-person, and cinematic cameras
---

# Three.js Camera Controls

## When to Use

Use this skill when:
- Implementing camera movement and controls
- Creating orbit/pan/zoom functionality
- Building first-person or third-person views
- Implementing cinematic camera paths
- Managing multiple camera perspectives

## Core Principles

1. **Smooth Movement**: Use lerp/slerp for natural motion
2. **Constraints**: Limit camera movement to valid ranges
3. **Input Handling**: Support mouse, touch, and gamepad
4. **Performance**: Update camera efficiently
5. **Predictable Behavior**: Consistent controls across platforms
6. **Decoupled**: Separate camera logic from scene logic

## Implementation

### 1. Orbit Camera

```typescript
// cameras/OrbitCameraController.ts
import * as THREE from 'three';

export interface OrbitControlsConfig {
  target?: THREE.Vector3;
  minDistance?: number;
  maxDistance?: number;
  minPolarAngle?: number;
  maxPolarAngle?: number;
  enableDamping?: boolean;
  dampingFactor?: number;
  enableZoom?: boolean;
  enablePan?: boolean;
  enableRotate?: boolean;
}

export class OrbitCameraController {
  private camera: THREE.PerspectiveCamera;
  private domElement: HTMLElement;
  private target: THREE.Vector3;

  // Spherical coordinates
  private spherical = new THREE.Spherical();
  private sphericalDelta = new THREE.Spherical();

  // Pan
  private panOffset = new THREE.Vector3();

  // State
  private rotateStart = new THREE.Vector2();
  private rotateEnd = new THREE.Vector2();
  private rotateDelta = new THREE.Vector2();

  private panStart = new THREE.Vector2();
  private panEnd = new THREE.Vector2();
  private panDelta = new THREE.Vector2();

  private scale = 1;
  private zoomSpeed = 1.0;

  private config: Required<OrbitControlsConfig>;

  constructor(
    camera: THREE.PerspectiveCamera,
    domElement: HTMLElement,
    config: OrbitControlsConfig = {}
  ) {
    this.camera = camera;
    this.domElement = domElement;
    this.target = config.target || new THREE.Vector3();

    this.config = {
      target: this.target,
      minDistance: config.minDistance ?? 1,
      maxDistance: config.maxDistance ?? Infinity,
      minPolarAngle: config.minPolarAngle ?? 0,
      maxPolarAngle: config.maxPolarAngle ?? Math.PI,
      enableDamping: config.enableDamping ?? true,
      dampingFactor: config.dampingFactor ?? 0.05,
      enableZoom: config.enableZoom ?? true,
      enablePan: config.enablePan ?? true,
      enableRotate: config.enableRotate ?? true,
    };

    this.setupEventListeners();
    this.update();
  }

  private setupEventListeners(): void {
    this.domElement.addEventListener('contextmenu', this.onContextMenu);
    this.domElement.addEventListener('pointerdown', this.onPointerDown);
    this.domElement.addEventListener('wheel', this.onMouseWheel);
    this.domElement.addEventListener('touchstart', this.onTouchStart);
  }

  private onContextMenu = (event: Event): void => {
    event.preventDefault();
  };

  private onPointerDown = (event: PointerEvent): void => {
    if (event.button === 0 && this.config.enableRotate) {
      this.rotateStart.set(event.clientX, event.clientY);
      this.domElement.addEventListener('pointermove', this.onPointerMove);
      this.domElement.addEventListener('pointerup', this.onPointerUp);
    } else if (event.button === 2 && this.config.enablePan) {
      this.panStart.set(event.clientX, event.clientY);
      this.domElement.addEventListener('pointermove', this.onPointerMovePan);
      this.domElement.addEventListener('pointerup', this.onPointerUp);
    }
  };

  private onPointerMove = (event: PointerEvent): void => {
    this.rotateEnd.set(event.clientX, event.clientY);
    this.rotateDelta.subVectors(this.rotateEnd, this.rotateStart).multiplyScalar(0.01);

    this.sphericalDelta.theta -= this.rotateDelta.x;
    this.sphericalDelta.phi -= this.rotateDelta.y;

    this.rotateStart.copy(this.rotateEnd);
  };

  private onPointerMovePan = (event: PointerEvent): void => {
    this.panEnd.set(event.clientX, event.clientY);
    this.panDelta.subVectors(this.panEnd, this.panStart).multiplyScalar(0.01);

    this.pan(this.panDelta.x, this.panDelta.y);

    this.panStart.copy(this.panEnd);
  };

  private onPointerUp = (): void => {
    this.domElement.removeEventListener('pointermove', this.onPointerMove);
    this.domElement.removeEventListener('pointermove', this.onPointerMovePan);
    this.domElement.removeEventListener('pointerup', this.onPointerUp);
  };

  private onMouseWheel = (event: WheelEvent): void => {
    if (!this.config.enableZoom) return;

    event.preventDefault();

    if (event.deltaY < 0) {
      this.scale /= 0.95;
    } else {
      this.scale *= 0.95;
    }
  };

  private onTouchStart = (event: TouchEvent): void => {
    if (event.touches.length === 1 && this.config.enableRotate) {
      this.rotateStart.set(event.touches[0].clientX, event.touches[0].clientY);
    }
  };

  private pan(deltaX: number, deltaY: number): void {
    const offset = new THREE.Vector3();
    const targetDistance = this.camera.position.distanceTo(this.target);

    // X axis
    offset.copy(this.camera.up).multiplyScalar(deltaY * targetDistance);
    this.panOffset.add(offset);

    // Y axis
    offset.setFromMatrixColumn(this.camera.matrix, 0);
    offset.multiplyScalar(-deltaX * targetDistance);
    this.panOffset.add(offset);
  }

  update(): void {
    const offset = new THREE.Vector3();
    const quat = new THREE.Quaternion().setFromUnitVectors(
      this.camera.up,
      new THREE.Vector3(0, 1, 0)
    );
    const quatInverse = quat.clone().invert();

    offset.copy(this.camera.position).sub(this.target);
    offset.applyQuaternion(quat);

    this.spherical.setFromVector3(offset);

    if (this.config.enableDamping) {
      this.spherical.theta += this.sphericalDelta.theta * this.config.dampingFactor;
      this.spherical.phi += this.sphericalDelta.phi * this.config.dampingFactor;
    } else {
      this.spherical.theta += this.sphericalDelta.theta;
      this.spherical.phi += this.sphericalDelta.phi;
    }

    // Clamp
    this.spherical.phi = Math.max(
      this.config.minPolarAngle,
      Math.min(this.config.maxPolarAngle, this.spherical.phi)
    );

    this.spherical.radius *= this.scale;
    this.spherical.radius = Math.max(
      this.config.minDistance,
      Math.min(this.config.maxDistance, this.spherical.radius)
    );

    // Apply pan
    this.target.add(this.panOffset);

    offset.setFromSpherical(this.spherical);
    offset.applyQuaternion(quatInverse);

    this.camera.position.copy(this.target).add(offset);
    this.camera.lookAt(this.target);

    if (this.config.enableDamping) {
      this.sphericalDelta.theta *= 1 - this.config.dampingFactor;
      this.sphericalDelta.phi *= 1 - this.config.dampingFactor;
      this.panOffset.multiplyScalar(1 - this.config.dampingFactor);
    } else {
      this.sphericalDelta.set(0, 0, 0);
      this.panOffset.set(0, 0, 0);
    }

    this.scale = 1;
  }

  dispose(): void {
    this.domElement.removeEventListener('contextmenu', this.onContextMenu);
    this.domElement.removeEventListener('pointerdown', this.onPointerDown);
    this.domElement.removeEventListener('wheel', this.onMouseWheel);
    this.domElement.removeEventListener('touchstart', this.onTouchStart);
  }
}
```

### 2. First-Person Camera

```typescript
// cameras/FirstPersonCamera.ts
import * as THREE from 'three';

export class FirstPersonCamera {
  private camera: THREE.PerspectiveCamera;
  private velocity = new THREE.Vector3();
  private direction = new THREE.Vector3();

  private moveForward = false;
  private moveBackward = false;
  private moveLeft = false;
  private moveRight = false;

  private yaw = 0;
  private pitch = 0;

  private speed = 10;
  private mouseSensitivity = 0.002;

  constructor(camera: THREE.PerspectiveCamera, domElement: HTMLElement) {
    this.camera = camera;

    this.setupControls(domElement);
  }

  private setupControls(domElement: HTMLElement): void {
    domElement.addEventListener('click', () => {
      domElement.requestPointerLock();
    });

    document.addEventListener('pointerlockchange', () => {
      if (document.pointerLockElement === domElement) {
        document.addEventListener('mousemove', this.onMouseMove);
      } else {
        document.removeEventListener('mousemove', this.onMouseMove);
      }
    });

    document.addEventListener('keydown', this.onKeyDown);
    document.addEventListener('keyup', this.onKeyUp);
  }

  private onMouseMove = (event: MouseEvent): void => {
    this.yaw -= event.movementX * this.mouseSensitivity;
    this.pitch -= event.movementY * this.mouseSensitivity;

    // Clamp pitch
    this.pitch = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, this.pitch));
  };

  private onKeyDown = (event: KeyboardEvent): void => {
    switch (event.code) {
      case 'KeyW':
      case 'ArrowUp':
        this.moveForward = true;
        break;
      case 'KeyS':
      case 'ArrowDown':
        this.moveBackward = true;
        break;
      case 'KeyA':
      case 'ArrowLeft':
        this.moveLeft = true;
        break;
      case 'KeyD':
      case 'ArrowRight':
        this.moveRight = true;
        break;
    }
  };

  private onKeyUp = (event: KeyboardEvent): void => {
    switch (event.code) {
      case 'KeyW':
      case 'ArrowUp':
        this.moveForward = false;
        break;
      case 'KeyS':
      case 'ArrowDown':
        this.moveBackward = false;
        break;
      case 'KeyA':
      case 'ArrowLeft':
        this.moveLeft = false;
        break;
      case 'KeyD':
      case 'ArrowRight':
        this.moveRight = false;
        break;
    }
  };

  update(deltaTime: number): void {
    // Update camera rotation
    this.camera.rotation.set(this.pitch, this.yaw, 0, 'YXZ');

    // Calculate movement direction
    this.direction.set(0, 0, 0);

    if (this.moveForward) this.direction.z -= 1;
    if (this.moveBackward) this.direction.z += 1;
    if (this.moveLeft) this.direction.x -= 1;
    if (this.moveRight) this.direction.x += 1;

    this.direction.normalize();

    // Apply rotation to direction
    this.direction.applyQuaternion(this.camera.quaternion);

    // Update velocity
    this.velocity.x = this.direction.x * this.speed;
    this.velocity.z = this.direction.z * this.speed;

    // Update position
    this.camera.position.add(this.velocity.clone().multiplyScalar(deltaTime));
  }

  setSpeed(speed: number): void {
    this.speed = speed;
  }

  setSensitivity(sensitivity: number): void {
    this.mouseSensitivity = sensitivity;
  }
}
```

### 3. Third-Person Camera

```typescript
// cameras/ThirdPersonCamera.ts
import * as THREE from 'three';

export interface ThirdPersonCameraConfig {
  offset?: THREE.Vector3;
  lookAtOffset?: THREE.Vector3;
  smoothness?: number;
  minDistance?: number;
  maxDistance?: number;
}

export class ThirdPersonCamera {
  private camera: THREE.PerspectiveCamera;
  private target: THREE.Object3D;
  private currentPosition = new THREE.Vector3();
  private currentLookAt = new THREE.Vector3();

  private config: Required<ThirdPersonCameraConfig>;

  constructor(
    camera: THREE.PerspectiveCamera,
    target: THREE.Object3D,
    config: ThirdPersonCameraConfig = {}
  ) {
    this.camera = camera;
    this.target = target;

    this.config = {
      offset: config.offset || new THREE.Vector3(0, 5, -10),
      lookAtOffset: config.lookAtOffset || new THREE.Vector3(0, 1, 0),
      smoothness: config.smoothness ?? 0.1,
      minDistance: config.minDistance ?? 5,
      maxDistance: config.maxDistance ?? 20,
    };

    // Initialize current positions
    this.currentPosition.copy(this.camera.position);
    this.currentLookAt.copy(this.target.position).add(this.config.lookAtOffset);
  }

  update(): void {
    // Calculate ideal position
    const idealOffset = this.config.offset.clone();
    idealOffset.applyQuaternion(this.target.quaternion);
    const idealPosition = this.target.position.clone().add(idealOffset);

    // Calculate ideal lookAt
    const idealLookAt = this.target.position.clone().add(this.config.lookAtOffset);

    // Smooth camera position
    this.currentPosition.lerp(idealPosition, this.config.smoothness);
    this.currentLookAt.lerp(idealLookAt, this.config.smoothness);

    // Apply to camera
    this.camera.position.copy(this.currentPosition);
    this.camera.lookAt(this.currentLookAt);
  }

  setOffset(offset: THREE.Vector3): void {
    this.config.offset.copy(offset);
  }

  setLookAtOffset(offset: THREE.Vector3): void {
    this.config.lookAtOffset.copy(offset);
  }

  setSmoothness(smoothness: number): void {
    this.config.smoothness = smoothness;
  }
}
```

### 4. Cinematic Camera Paths

```typescript
// cameras/CinematicCamera.ts
import * as THREE from 'three';

export interface CameraKeyframe {
  position: THREE.Vector3;
  lookAt: THREE.Vector3;
  time: number;
}

export class CinematicCamera {
  private camera: THREE.PerspectiveCamera;
  private keyframes: CameraKeyframe[] = [];
  private currentTime = 0;
  private isPlaying = false;
  private loop = false;

  private curve: THREE.CatmullRomCurve3 | null = null;
  private lookAtCurve: THREE.CatmullRomCurve3 | null = null;

  constructor(camera: THREE.PerspectiveCamera) {
    this.camera = camera;
  }

  addKeyframe(position: THREE.Vector3, lookAt: THREE.Vector3, time: number): void {
    this.keyframes.push({ position, lookAt, time });
    this.keyframes.sort((a, b) => a.time - b.time);

    // Rebuild curves
    this.buildCurves();
  }

  private buildCurves(): void {
    if (this.keyframes.length < 2) return;

    const positions = this.keyframes.map((kf) => kf.position);
    const lookAts = this.keyframes.map((kf) => kf.lookAt);

    this.curve = new THREE.CatmullRomCurve3(positions);
    this.lookAtCurve = new THREE.CatmullRomCurve3(lookAts);
  }

  play(loop = false): void {
    this.isPlaying = true;
    this.loop = loop;
    this.currentTime = 0;
  }

  stop(): void {
    this.isPlaying = false;
  }

  update(deltaTime: number): void {
    if (!this.isPlaying || !this.curve || !this.lookAtCurve) return;

    this.currentTime += deltaTime;

    const duration = this.keyframes[this.keyframes.length - 1].time;

    if (this.currentTime > duration) {
      if (this.loop) {
        this.currentTime = 0;
      } else {
        this.stop();
        return;
      }
    }

    const t = this.currentTime / duration;

    const position = this.curve.getPoint(t);
    const lookAt = this.lookAtCurve.getPoint(t);

    this.camera.position.copy(position);
    this.camera.lookAt(lookAt);
  }

  clear(): void {
    this.keyframes = [];
    this.curve = null;
    this.lookAtCurve = null;
  }
}
```

## Usage Examples

```typescript
// Example 1: Orbit camera
import { OrbitCameraController } from './cameras/OrbitCameraController';

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const orbitControls = new OrbitCameraController(camera, renderer.domElement, {
  target: new THREE.Vector3(0, 0, 0),
  minDistance: 5,
  maxDistance: 50,
  enableDamping: true,
});

function animate() {
  orbitControls.update();
  renderer.render(scene, camera);
}

// Example 2: First-person camera
import { FirstPersonCamera } from './cameras/FirstPersonCamera';

const fpCamera = new FirstPersonCamera(camera, renderer.domElement);

function animate() {
  fpCamera.update(deltaTime);
  renderer.render(scene, camera);
}

// Example 3: Third-person camera
import { ThirdPersonCamera } from './cameras/ThirdPersonCamera';

const player = scene.getObjectByName('player')!;
const tpCamera = new ThirdPersonCamera(camera, player, {
  offset: new THREE.Vector3(0, 5, -10),
  smoothness: 0.1,
});

function animate() {
  tpCamera.update();
  renderer.render(scene, camera);
}

// Example 4: Cinematic camera
import { CinematicCamera } from './cameras/CinematicCamera';

const cinematic = new CinematicCamera(camera);

cinematic.addKeyframe(new THREE.Vector3(0, 5, 10), new THREE.Vector3(0, 0, 0), 0);
cinematic.addKeyframe(new THREE.Vector3(10, 5, 0), new THREE.Vector3(0, 0, 0), 2);
cinematic.addKeyframe(new THREE.Vector3(0, 5, -10), new THREE.Vector3(0, 0, 0), 4);

cinematic.play(true);

function animate() {
  cinematic.update(deltaTime);
  renderer.render(scene, camera);
}
```

## Checklist

- [ ] Choose camera type (orbit, first-person, third-person, cinematic)
- [ ] Implement input handling (mouse, touch, keyboard)
- [ ] Add smooth camera movement (lerp/slerp)
- [ ] Implement constraints (min/max distance, angle limits)
- [ ] Add damping for natural feel
- [ ] Test on target devices
- [ ] Optimize update loop
- [ ] Handle edge cases (gimbal lock, collision)
- [ ] Add camera shake effects (optional)
- [ ] Implement camera transitions
- [ ] Test with different aspect ratios

## Common Pitfalls

1. **Gimbal lock**: Use quaternions instead of Euler angles
2. **Not clamping pitch**: Camera flips upside down
3. **No smoothing**: Jerky camera movement
4. **Forgetting to update**: Camera doesn't move
5. **Not handling resize**: Aspect ratio breaks
6. **No constraints**: Camera goes anywhere
7. **Memory leaks**: Not removing event listeners on dispose

## Performance Tips

- Update camera only when needed (not every frame if static)
- Use lerp for smooth movement (more efficient than springs)
- Debounce resize events
- Cache frequently accessed values
- Use quaternions for rotation (faster than matrices)
- Implement frustum culling with camera frustum
- Profile camera updates in performance-critical sections

## Related Skills

- `threejs-scene-setup` - Scene initialization
- `r3f-setup` - React Three Fiber camera setup
- `input-system` - Input handling
- `touch-input-handling` - Touch controls
