---
name: threejs-math-utilities
description: Essential math utilities for Three.js game development including vector operations, quaternions, interpolation, curves, and collision helpers
---

# Three.js Math Utilities

## When to Use

Use this skill when:
- Implementing game mechanics (movement, rotation, physics)
- Working with vectors and quaternions
- Creating smooth interpolations and easing
- Generating curves and paths
- Calculating distances and angles
- Implementing collision detection helpers
- Building procedural generation

## Core Principles

1. **Use Built-in Math**: Three.js has extensive math utilities
2. **Performance**: Cache calculations, avoid unnecessary allocations
3. **Immutability**: Clone vectors before modifying when needed
4. **Readability**: Use descriptive variable names
5. **Precision**: Be aware of floating-point limitations
6. **Optimization**: Profile math-heavy operations

## Implementation

### 1. Vector Utilities

```typescript
// math/VectorUtils.ts
import * as THREE from 'three';

export class VectorUtils {
  // Distance utilities
  static distance2D(a: THREE.Vector3, b: THREE.Vector3): number {
    const dx = b.x - a.x;
    const dz = b.z - a.z;
    return Math.sqrt(dx * dx + dz * dz);
  }

  static distanceSquared2D(a: THREE.Vector3, b: THREE.Vector3): number {
    const dx = b.x - a.x;
    const dz = b.z - a.z;
    return dx * dx + dz * dz;
  }

  // Direction and angle
  static direction(from: THREE.Vector3, to: THREE.Vector3): THREE.Vector3 {
    return to.clone().sub(from).normalize();
  }

  static angleBetween(a: THREE.Vector3, b: THREE.Vector3): number {
    return Math.acos(THREE.MathUtils.clamp(a.dot(b), -1, 1));
  }

  static angle2D(from: THREE.Vector3, to: THREE.Vector3): number {
    return Math.atan2(to.z - from.z, to.x - from.x);
  }

  // Projection
  static projectOnPlane(vector: THREE.Vector3, planeNormal: THREE.Vector3): THREE.Vector3 {
    const projected = vector.clone();
    const distance = projected.dot(planeNormal);
    projected.sub(planeNormal.clone().multiplyScalar(distance));
    return projected;
  }

  static projectOnVector(vector: THREE.Vector3, onto: THREE.Vector3): THREE.Vector3 {
    const dot = vector.dot(onto);
    const lenSq = onto.lengthSq();
    return onto.clone().multiplyScalar(dot / lenSq);
  }

  // Random
  static randomInSphere(radius: number): THREE.Vector3 {
    const u = Math.random();
    const v = Math.random();
    const theta = 2 * Math.PI * u;
    const phi = Math.acos(2 * v - 1);
    const r = radius * Math.cbrt(Math.random());

    return new THREE.Vector3(
      r * Math.sin(phi) * Math.cos(theta),
      r * Math.sin(phi) * Math.sin(theta),
      r * Math.cos(phi)
    );
  }

  static randomInCircle(radius: number): THREE.Vector3 {
    const angle = Math.random() * Math.PI * 2;
    const r = Math.sqrt(Math.random()) * radius;
    return new THREE.Vector3(r * Math.cos(angle), 0, r * Math.sin(angle));
  }

  static randomOnSphere(radius: number): THREE.Vector3 {
    const u = Math.random();
    const v = Math.random();
    const theta = 2 * Math.PI * u;
    const phi = Math.acos(2 * v - 1);

    return new THREE.Vector3(
      radius * Math.sin(phi) * Math.cos(theta),
      radius * Math.sin(phi) * Math.sin(theta),
      radius * Math.cos(phi)
    );
  }

  // Clamping
  static clampLength(vector: THREE.Vector3, min: number, max: number): THREE.Vector3 {
    const length = vector.length();
    if (length === 0) return vector;

    const clamped = THREE.MathUtils.clamp(length, min, max);
    return vector.clone().multiplyScalar(clamped / length);
  }

  // Reflection
  static reflect(vector: THREE.Vector3, normal: THREE.Vector3): THREE.Vector3 {
    const dot = vector.dot(normal);
    return vector.clone().sub(normal.clone().multiplyScalar(2 * dot));
  }

  // Rotate around axis
  static rotateAroundAxis(
    vector: THREE.Vector3,
    axis: THREE.Vector3,
    angle: number
  ): THREE.Vector3 {
    const quaternion = new THREE.Quaternion();
    quaternion.setFromAxisAngle(axis, angle);
    return vector.clone().applyQuaternion(quaternion);
  }
}
```

### 2. Interpolation Utilities

```typescript
// math/InterpolationUtils.ts
import * as THREE from 'three';

export class InterpolationUtils {
  // Easing functions
  static easeInQuad(t: number): number {
    return t * t;
  }

  static easeOutQuad(t: number): number {
    return t * (2 - t);
  }

  static easeInOutQuad(t: number): number {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
  }

  static easeInCubic(t: number): number {
    return t * t * t;
  }

  static easeOutCubic(t: number): number {
    return --t * t * t + 1;
  }

  static easeInOutCubic(t: number): number {
    return t < 0.5 ? 4 * t * t * t : (t - 1) * (2 * t - 2) * (2 * t - 2) + 1;
  }

  static easeInElastic(t: number): number {
    return t === 0 ? 0 : t === 1 ? 1 : -Math.pow(2, 10 * (t - 1)) * Math.sin((t - 1.1) * 5 * Math.PI);
  }

  static easeOutElastic(t: number): number {
    return t === 0 ? 0 : t === 1 ? 1 : Math.pow(2, -10 * t) * Math.sin((t - 0.1) * 5 * Math.PI) + 1;
  }

  static smoothStep(min: number, max: number, value: number): number {
    const x = THREE.MathUtils.clamp((value - min) / (max - min), 0, 1);
    return x * x * (3 - 2 * x);
  }

  static smootherStep(min: number, max: number, value: number): number {
    const x = THREE.MathUtils.clamp((value - min) / (max - min), 0, 1);
    return x * x * x * (x * (x * 6 - 15) + 10);
  }

  // Vector interpolation
  static lerpVectors(
    a: THREE.Vector3,
    b: THREE.Vector3,
    t: number,
    easing?: (t: number) => number
  ): THREE.Vector3 {
    const easedT = easing ? easing(t) : t;
    return a.clone().lerp(b, easedT);
  }

  static slerpQuaternions(
    a: THREE.Quaternion,
    b: THREE.Quaternion,
    t: number,
    easing?: (t: number) => number
  ): THREE.Quaternion {
    const easedT = easing ? easing(t) : t;
    return a.clone().slerp(b, easedT);
  }

  // Spring interpolation
  static spring(
    current: number,
    target: number,
    velocity: { value: number },
    deltaTime: number,
    springConstant: number = 100,
    damping: number = 10
  ): number {
    const force = (target - current) * springConstant;
    const dampingForce = velocity.value * damping;
    const acceleration = force - dampingForce;

    velocity.value += acceleration * deltaTime;
    return current + velocity.value * deltaTime;
  }

  static springVector(
    current: THREE.Vector3,
    target: THREE.Vector3,
    velocity: THREE.Vector3,
    deltaTime: number,
    springConstant: number = 100,
    damping: number = 10
  ): THREE.Vector3 {
    const force = target.clone().sub(current).multiplyScalar(springConstant);
    const dampingForce = velocity.clone().multiplyScalar(damping);
    const acceleration = force.sub(dampingForce);

    velocity.add(acceleration.multiplyScalar(deltaTime));
    return current.clone().add(velocity.clone().multiplyScalar(deltaTime));
  }
}
```

### 3. Quaternion Utilities

```typescript
// math/QuaternionUtils.ts
import * as THREE from 'three';

export class QuaternionUtils {
  // Look rotation
  static lookRotation(forward: THREE.Vector3, up?: THREE.Vector3): THREE.Quaternion {
    const upVector = up || new THREE.Vector3(0, 1, 0);
    const matrix = new THREE.Matrix4();
    matrix.lookAt(new THREE.Vector3(), forward, upVector);
    return new THREE.Quaternion().setFromRotationMatrix(matrix);
  }

  // From-to rotation
  static fromToRotation(from: THREE.Vector3, to: THREE.Vector3): THREE.Quaternion {
    const axis = from.clone().cross(to).normalize();
    const angle = Math.acos(THREE.MathUtils.clamp(from.dot(to), -1, 1));
    return new THREE.Quaternion().setFromAxisAngle(axis, angle);
  }

  // Rotate towards
  static rotateTowards(
    from: THREE.Quaternion,
    to: THREE.Quaternion,
    maxRadians: number
  ): THREE.Quaternion {
    const angle = from.angleTo(to);
    if (angle === 0) return from.clone();

    const t = Math.min(1, maxRadians / angle);
    return from.clone().slerp(to, t);
  }

  // Angle between quaternions
  static angle(a: THREE.Quaternion, b: THREE.Quaternion): number {
    const dot = THREE.MathUtils.clamp(a.dot(b), -1, 1);
    return 2 * Math.acos(Math.abs(dot));
  }

  // Get forward/right/up vectors
  static getForward(quaternion: THREE.Quaternion): THREE.Vector3 {
    return new THREE.Vector3(0, 0, 1).applyQuaternion(quaternion);
  }

  static getRight(quaternion: THREE.Quaternion): THREE.Vector3 {
    return new THREE.Vector3(1, 0, 0).applyQuaternion(quaternion);
  }

  static getUp(quaternion: THREE.Quaternion): THREE.Vector3 {
    return new THREE.Vector3(0, 1, 0).applyQuaternion(quaternion);
  }

  // Euler angles
  static toEulerAngles(quaternion: THREE.Quaternion): { pitch: number; yaw: number; roll: number } {
    const euler = new THREE.Euler().setFromQuaternion(quaternion, 'YXZ');
    return {
      pitch: euler.x,
      yaw: euler.y,
      roll: euler.z,
    };
  }
}
```

### 4. Curve Utilities

```typescript
// math/CurveUtils.ts
import * as THREE from 'three';

export class CurveUtils {
  // Catmull-Rom spline
  static createCatmullRomPath(points: THREE.Vector3[]): THREE.CatmullRomCurve3 {
    return new THREE.CatmullRomCurve3(points);
  }

  // Bezier curve
  static createQuadraticBezier(
    start: THREE.Vector3,
    control: THREE.Vector3,
    end: THREE.Vector3
  ): THREE.QuadraticBezierCurve3 {
    return new THREE.QuadraticBezierCurve3(start, control, end);
  }

  static createCubicBezier(
    start: THREE.Vector3,
    control1: THREE.Vector3,
    control2: THREE.Vector3,
    end: THREE.Vector3
  ): THREE.CubicBezierCurve3 {
    return new THREE.CubicBezierCurve3(start, control1, control2, end);
  }

  // Arc
  static createArc(
    center: THREE.Vector3,
    radius: number,
    startAngle: number,
    endAngle: number,
    clockwise: boolean = false
  ): THREE.EllipseCurve {
    return new THREE.EllipseCurve(
      center.x,
      center.y,
      radius,
      radius,
      startAngle,
      endAngle,
      clockwise,
      0
    );
  }

  // Get points along curve
  static getPointsAlongCurve(curve: THREE.Curve<THREE.Vector3>, count: number): THREE.Vector3[] {
    return curve.getPoints(count);
  }

  // Get evenly spaced points
  static getEvenlySpacedPoints(
    curve: THREE.Curve<THREE.Vector3>,
    spacing: number
  ): THREE.Vector3[] {
    const length = curve.getLength();
    const count = Math.ceil(length / spacing);
    return curve.getSpacedPoints(count);
  }

  // Closest point on curve
  static closestPointOnCurve(
    curve: THREE.Curve<THREE.Vector3>,
    point: THREE.Vector3,
    divisions: number = 50
  ): { point: THREE.Vector3; t: number } {
    let minDistance = Infinity;
    let closestT = 0;
    let closestPoint = new THREE.Vector3();

    for (let i = 0; i <= divisions; i++) {
      const t = i / divisions;
      const curvePoint = curve.getPoint(t);
      const distance = point.distanceToSquared(curvePoint);

      if (distance < minDistance) {
        minDistance = distance;
        closestT = t;
        closestPoint = curvePoint;
      }
    }

    return { point: closestPoint, t: closestT };
  }
}
```

### 5. Collision Helpers

```typescript
// math/CollisionUtils.ts
import * as THREE from 'three';

export class CollisionUtils {
  // Sphere-sphere
  static sphereIntersectsSphere(
    center1: THREE.Vector3,
    radius1: number,
    center2: THREE.Vector3,
    radius2: number
  ): boolean {
    const distSq = center1.distanceToSquared(center2);
    const radiusSum = radius1 + radius2;
    return distSq <= radiusSum * radiusSum;
  }

  // Point-sphere
  static pointInSphere(point: THREE.Vector3, center: THREE.Vector3, radius: number): boolean {
    return point.distanceToSquared(center) <= radius * radius;
  }

  // Point-AABB
  static pointInAABB(point: THREE.Vector3, min: THREE.Vector3, max: THREE.Vector3): boolean {
    return (
      point.x >= min.x &&
      point.x <= max.x &&
      point.y >= min.y &&
      point.y <= max.y &&
      point.z >= min.z &&
      point.z <= max.z
    );
  }

  // AABB-AABB
  static aabbIntersectsAABB(
    min1: THREE.Vector3,
    max1: THREE.Vector3,
    min2: THREE.Vector3,
    max2: THREE.Vector3
  ): boolean {
    return (
      min1.x <= max2.x &&
      max1.x >= min2.x &&
      min1.y <= max2.y &&
      max1.y >= min2.y &&
      min1.z <= max2.z &&
      max1.z >= min2.z
    );
  }

  // Sphere-AABB
  static sphereIntersectsAABB(
    center: THREE.Vector3,
    radius: number,
    min: THREE.Vector3,
    max: THREE.Vector3
  ): boolean {
    const closestPoint = new THREE.Vector3(
      THREE.MathUtils.clamp(center.x, min.x, max.x),
      THREE.MathUtils.clamp(center.y, min.y, max.y),
      THREE.MathUtils.clamp(center.z, min.z, max.z)
    );

    return center.distanceToSquared(closestPoint) <= radius * radius;
  }

  // Ray-sphere
  static rayIntersectsSphere(
    rayOrigin: THREE.Vector3,
    rayDirection: THREE.Vector3,
    sphereCenter: THREE.Vector3,
    sphereRadius: number
  ): { hit: boolean; distance?: number; point?: THREE.Vector3 } {
    const oc = rayOrigin.clone().sub(sphereCenter);
    const a = rayDirection.dot(rayDirection);
    const b = 2 * oc.dot(rayDirection);
    const c = oc.dot(oc) - sphereRadius * sphereRadius;
    const discriminant = b * b - 4 * a * c;

    if (discriminant < 0) {
      return { hit: false };
    }

    const distance = (-b - Math.sqrt(discriminant)) / (2 * a);
    const point = rayOrigin.clone().add(rayDirection.clone().multiplyScalar(distance));

    return { hit: true, distance, point };
  }

  // Ray-plane
  static rayIntersectsPlane(
    rayOrigin: THREE.Vector3,
    rayDirection: THREE.Vector3,
    planePoint: THREE.Vector3,
    planeNormal: THREE.Vector3
  ): { hit: boolean; distance?: number; point?: THREE.Vector3 } {
    const denom = planeNormal.dot(rayDirection);

    if (Math.abs(denom) < 1e-6) {
      return { hit: false }; // Ray parallel to plane
    }

    const distance = planePoint.clone().sub(rayOrigin).dot(planeNormal) / denom;

    if (distance < 0) {
      return { hit: false }; // Plane behind ray
    }

    const point = rayOrigin.clone().add(rayDirection.clone().multiplyScalar(distance));

    return { hit: true, distance, point };
  }
}
```

## Usage Examples

```typescript
// Example 1: Vector operations
import { VectorUtils } from './math/VectorUtils';

const playerPos = new THREE.Vector3(0, 0, 0);
const enemyPos = new THREE.Vector3(10, 0, 5);

// Direction to enemy
const direction = VectorUtils.direction(playerPos, enemyPos);

// 2D distance (ignoring Y)
const distance = VectorUtils.distance2D(playerPos, enemyPos);

// Random position in circle around player
const randomPos = VectorUtils.randomInCircle(5);

// Example 2: Smooth interpolation
import { InterpolationUtils } from './math/InterpolationUtils';

// Smooth camera movement
const cameraVelocity = new THREE.Vector3();

function animate() {
  camera.position.copy(
    InterpolationUtils.springVector(
      camera.position,
      targetPosition,
      cameraVelocity,
      deltaTime,
      50, // Spring constant
      10  // Damping
    )
  );
}

// Example 3: Rotation utilities
import { QuaternionUtils } from './math/QuaternionUtils';

// Look at target
const lookRotation = QuaternionUtils.lookRotation(
  VectorUtils.direction(player.position, enemy.position)
);

// Smoothly rotate towards
player.quaternion.copy(
  QuaternionUtils.rotateTowards(
    player.quaternion,
    lookRotation,
    deltaTime * 2 // Max 2 radians per second
  )
);

// Example 4: Path following
import { CurveUtils } from './math/CurveUtils';

const waypoints = [
  new THREE.Vector3(0, 0, 0),
  new THREE.Vector3(10, 5, 0),
  new THREE.Vector3(20, 0, 10),
];

const path = CurveUtils.createCatmullRomPath(waypoints);

let pathProgress = 0;

function animate() {
  pathProgress += deltaTime * 0.1;
  if (pathProgress > 1) pathProgress = 0;

  const position = path.getPoint(pathProgress);
  enemy.position.copy(position);
}

// Example 5: Collision detection
import { CollisionUtils } from './math/CollisionUtils';

// Check if player hit enemy
if (
  CollisionUtils.sphereIntersectsSphere(
    player.position,
    1, // Player radius
    enemy.position,
    1.5 // Enemy radius
  )
) {
  console.log('Hit!');
}

// Check if projectile hit target
const hit = CollisionUtils.rayIntersectsSphere(
  projectile.position,
  projectile.velocity.clone().normalize(),
  target.position,
  target.radius
);

if (hit.hit) {
  console.log('Hit at distance:', hit.distance);
}
```

## Checklist

- [ ] Use Three.js built-in math utilities (Vector3, Quaternion, MathUtils)
- [ ] Cache frequently used calculations
- [ ] Use squared distances when possible (avoid sqrt)
- [ ] Clone vectors before modifying to avoid side effects
- [ ] Use appropriate data types (Vector2 for 2D, Vector3 for 3D)
- [ ] Profile math-heavy operations
- [ ] Use easing functions for smooth animations
- [ ] Implement object pooling for temporary vectors/quaternions
- [ ] Use quaternions for rotations (avoid gimbal lock)
- [ ] Test collision detection with edge cases

## Common Pitfalls

1. **Modifying input vectors**: Always clone before modifying
2. **Unnecessary allocations**: Reuse vectors in loops
3. **Sqrt in tight loops**: Use distanceSquared instead
4. **Gimbal lock with Euler angles**: Use quaternions
5. **Floating-point precision**: Use epsilon for comparisons
6. **Not normalizing directions**: Can cause scaling issues
7. **Dividing by zero**: Check for zero vectors/quaternions

## Performance Tips

### Vector Operations
- Use `distanceToSquared` instead of `distanceTo` when comparing distances
- Reuse vectors in update loops (don't create new ones)
- Use `copy()` instead of creating new vectors
- Cache frequently used calculations

### Quaternions
- Use `slerp` for smooth rotation, `lerp` for faster but less accurate
- Cache forward/right/up vectors if used multiple times per frame
- Use `setFromAxisAngle` instead of Euler angles

### Interpolation
- Use `lerp` for linear, `slerp` for quaternions
- Cache easing function results when possible
- Use cheaper easing functions (quad instead of elastic)

### Collision Detection
- Use broad-phase culling before detailed checks
- Check simpler shapes first (sphere before mesh)
- Use spatial partitioning for many objects
- Cache bounding volumes

## Related Skills

- `threejs-raycasting` - Advanced raycasting
- `collision-system` - Full collision system
- `ecs-component-patterns` - Using math in ECS
- `threejs-animation-systems` - Animation with math

## References

- Three.js Math: https://threejs.org/docs/#api/en/math/Math
- Vector3: https://threejs.org/docs/#api/en/math/Vector3
- Quaternion: https://threejs.org/docs/#api/en/math/Quaternion
- Curves: https://threejs.org/docs/#api/en/extras/curves/Curve
