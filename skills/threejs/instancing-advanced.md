---
name: threejs-instancing-advanced
description: Advanced instancing techniques in Three.js including InstancedMesh, dynamic updates, LOD instancing, frustum culling, and custom attributes
---

# Three.js Advanced Instancing

## When to Use

Use this skill when:
- Rendering many identical objects (100+)
- Optimizing draw calls
- Creating crowds, forests, particle systems
- Implementing procedural generation
- Building large-scale environments
- Need individual instance control

## Core Principles

1. **Single Draw Call**: All instances in one call
2. **GPU Transforms**: Matrices calculated on GPU
3. **Dynamic Updates**: Update only what changed
4. **Custom Attributes**: Per-instance data
5. **Frustum Culling**: Hide off-screen instances
6. **LOD Integration**: Multiple detail levels

## Implementation

### 1. Advanced Instanced Mesh Manager

```typescript
// instancing/InstancedMeshManager.ts
import * as THREE from 'three';

export interface InstanceData {
  position: THREE.Vector3;
  rotation: THREE.Euler;
  scale: THREE.Vector3;
  color?: THREE.Color;
  visible?: boolean;
}

export class InstancedMeshManager {
  private mesh: THREE.InstancedMesh;
  private dummy = new THREE.Object3D();
  private count: number;
  private visibilityArray: boolean[];

  // Custom attributes
  private colorAttribute?: THREE.InstancedBufferAttribute;
  private hasCustomColors: boolean;

  constructor(
    geometry: THREE.BufferGeometry,
    material: THREE.Material,
    count: number,
    useCustomColors: boolean = false
  ) {
    this.count = count;
    this.hasCustomColors = useCustomColors;
    this.visibilityArray = new Array(count).fill(true);

    this.mesh = new THREE.InstancedMesh(geometry, material, count);

    // Add custom color attribute if needed
    if (useCustomColors) {
      const colors = new Float32Array(count * 3);
      this.colorAttribute = new THREE.InstancedBufferAttribute(colors, 3);
      this.mesh.instanceColor = this.colorAttribute;
    }

    // Initialize all instances at origin (invisible by default)
    for (let i = 0; i < count; i++) {
      this.dummy.position.set(0, -10000, 0);
      this.dummy.updateMatrix();
      this.mesh.setMatrixAt(i, this.dummy.matrix);
    }

    this.mesh.instanceMatrix.needsUpdate = true;
  }

  setInstance(index: number, data: InstanceData): void {
    if (index >= this.count) return;

    // Set transform
    this.dummy.position.copy(data.position);
    this.dummy.rotation.copy(data.rotation);
    this.dummy.scale.copy(data.scale);
    this.dummy.updateMatrix();

    this.mesh.setMatrixAt(index, this.dummy.matrix);

    // Set color if provided
    if (data.color && this.colorAttribute) {
      this.colorAttribute.setXYZ(index, data.color.r, data.color.g, data.color.b);
    }

    // Track visibility
    if (data.visible !== undefined) {
      this.visibilityArray[index] = data.visible;

      if (!data.visible) {
        // Move far away to hide
        this.dummy.position.set(0, -10000, 0);
        this.dummy.updateMatrix();
        this.mesh.setMatrixAt(index, this.dummy.matrix);
      }
    }
  }

  updateInstanceTransform(
    index: number,
    position?: THREE.Vector3,
    rotation?: THREE.Euler,
    scale?: THREE.Vector3
  ): void {
    if (index >= this.count) return;

    // Get current matrix
    this.mesh.getMatrixAt(index, this.dummy.matrix);
    this.dummy.matrix.decompose(this.dummy.position, this.dummy.quaternion, this.dummy.scale);

    // Update provided values
    if (position) this.dummy.position.copy(position);
    if (rotation) this.dummy.rotation.copy(rotation);
    if (scale) this.dummy.scale.copy(scale);

    this.dummy.updateMatrix();
    this.mesh.setMatrixAt(index, this.dummy.matrix);
  }

  updateInstanceColor(index: number, color: THREE.Color): void {
    if (index >= this.count || !this.colorAttribute) return;

    this.colorAttribute.setXYZ(index, color.r, color.g, color.b);
    this.colorAttribute.needsUpdate = true;
  }

  setInstanceVisible(index: number, visible: boolean): void {
    if (index >= this.count) return;

    this.visibilityArray[index] = visible;

    if (!visible) {
      // Move far away
      this.dummy.position.set(0, -10000, 0);
      this.dummy.updateMatrix();
      this.mesh.setMatrixAt(index, this.dummy.matrix);
      this.mesh.instanceMatrix.needsUpdate = true;
    }
  }

  commitUpdates(): void {
    this.mesh.instanceMatrix.needsUpdate = true;

    if (this.colorAttribute) {
      this.colorAttribute.needsUpdate = true;
    }
  }

  getInstanceTransform(index: number): {
    position: THREE.Vector3;
    rotation: THREE.Euler;
    scale: THREE.Vector3;
  } {
    this.mesh.getMatrixAt(index, this.dummy.matrix);
    this.dummy.matrix.decompose(this.dummy.position, this.dummy.quaternion, this.dummy.scale);

    return {
      position: this.dummy.position.clone(),
      rotation: this.dummy.rotation.clone(),
      scale: this.dummy.scale.clone(),
    };
  }

  getMesh(): THREE.InstancedMesh {
    return this.mesh;
  }

  getCount(): number {
    return this.count;
  }

  dispose(): void {
    this.mesh.dispose();
  }
}
```

### 2. Dynamic Instance Pool

```typescript
// instancing/InstancePool.ts
import * as THREE from 'three';

export class InstancePool {
  private mesh: THREE.InstancedMesh;
  private availableIndices: number[] = [];
  private activeInstances = new Map<number, boolean>();
  private dummy = new THREE.Object3D();
  private maxCount: number;

  constructor(geometry: THREE.BufferGeometry, material: THREE.Material, maxCount: number) {
    this.maxCount = maxCount;
    this.mesh = new THREE.InstancedMesh(geometry, material, maxCount);

    // Initialize all as available
    for (let i = 0; i < maxCount; i++) {
      this.availableIndices.push(i);

      // Hide all instances initially
      this.dummy.position.set(0, -10000, 0);
      this.dummy.updateMatrix();
      this.mesh.setMatrixAt(i, this.dummy.matrix);
    }

    this.mesh.instanceMatrix.needsUpdate = true;
  }

  request(): number | null {
    if (this.availableIndices.length === 0) {
      console.warn('Instance pool exhausted');
      return null;
    }

    const index = this.availableIndices.pop()!;
    this.activeInstances.set(index, true);
    return index;
  }

  release(index: number): void {
    if (!this.activeInstances.has(index)) return;

    this.activeInstances.delete(index);
    this.availableIndices.push(index);

    // Hide instance
    this.dummy.position.set(0, -10000, 0);
    this.dummy.updateMatrix();
    this.mesh.setMatrixAt(index, this.dummy.matrix);
    this.mesh.instanceMatrix.needsUpdate = true;
  }

  updateInstance(
    index: number,
    position: THREE.Vector3,
    rotation: THREE.Euler,
    scale: THREE.Vector3
  ): void {
    if (!this.activeInstances.has(index)) return;

    this.dummy.position.copy(position);
    this.dummy.rotation.copy(rotation);
    this.dummy.scale.copy(scale);
    this.dummy.updateMatrix();

    this.mesh.setMatrixAt(index, this.dummy.matrix);
    this.mesh.instanceMatrix.needsUpdate = true;
  }

  getMesh(): THREE.InstancedMesh {
    return this.mesh;
  }

  getActiveCount(): number {
    return this.activeInstances.size;
  }

  getAvailableCount(): number {
    return this.availableIndices.length;
  }

  dispose(): void {
    this.mesh.dispose();
    this.activeInstances.clear();
    this.availableIndices = [];
  }
}
```

### 3. Instanced LOD System

```typescript
// instancing/InstancedLOD.ts
import * as THREE from 'three';

export interface LODLevel {
  geometry: THREE.BufferGeometry;
  distance: number;
}

export class InstancedLOD {
  private levels: Map<number, THREE.InstancedMesh> = new Map();
  private material: THREE.Material;
  private distances: number[];
  private count: number;
  private camera: THREE.Camera;
  private scene: THREE.Scene;

  private instancePositions: THREE.Vector3[];
  private dummy = new THREE.Object3D();

  constructor(
    scene: THREE.Scene,
    camera: THREE.Camera,
    material: THREE.Material,
    lodLevels: LODLevel[],
    count: number
  ) {
    this.scene = scene;
    this.camera = camera;
    this.material = material;
    this.count = count;
    this.distances = lodLevels.map((level) => level.distance).sort((a, b) => a - b);
    this.instancePositions = [];

    // Create instanced mesh for each LOD level
    lodLevels.forEach((level, index) => {
      const mesh = new THREE.InstancedMesh(level.geometry, material, count);
      mesh.frustumCulled = false; // We handle culling manually

      // Hide all instances initially
      for (let i = 0; i < count; i++) {
        this.dummy.position.set(0, -10000, 0);
        this.dummy.updateMatrix();
        mesh.setMatrixAt(i, this.dummy.matrix);
      }

      mesh.instanceMatrix.needsUpdate = true;

      this.levels.set(index, mesh);
      this.scene.add(mesh);
    });
  }

  setInstance(index: number, position: THREE.Vector3, rotation: THREE.Euler, scale: THREE.Vector3): void {
    if (index >= this.count) return;

    // Store position for distance calculation
    if (!this.instancePositions[index]) {
      this.instancePositions[index] = position.clone();
    } else {
      this.instancePositions[index].copy(position);
    }

    // Set transform on all LOD levels
    this.dummy.position.copy(position);
    this.dummy.rotation.copy(rotation);
    this.dummy.scale.copy(scale);
    this.dummy.updateMatrix();

    this.levels.forEach((mesh) => {
      mesh.setMatrixAt(index, this.dummy.matrix);
    });
  }

  update(): void {
    const cameraPosition = this.camera.position;

    // Calculate distances and determine LOD level for each instance
    const lodAssignments = new Array(this.count).fill(-1);

    for (let i = 0; i < this.count; i++) {
      if (!this.instancePositions[i]) continue;

      const distance = cameraPosition.distanceTo(this.instancePositions[i]);

      // Determine LOD level
      let lodLevel = this.levels.size - 1; // Start with lowest detail

      for (let j = 0; j < this.distances.length; j++) {
        if (distance < this.distances[j]) {
          lodLevel = j;
          break;
        }
      }

      lodAssignments[i] = lodLevel;
    }

    // Update visibility for each LOD level
    this.levels.forEach((mesh, levelIndex) => {
      for (let i = 0; i < this.count; i++) {
        if (lodAssignments[i] === levelIndex) {
          // Visible at this LOD
          if (this.instancePositions[i]) {
            mesh.getMatrixAt(i, this.dummy.matrix);
            // Matrix already set, just ensure it's not hidden
          }
        } else {
          // Hidden at this LOD
          this.dummy.position.set(0, -10000, 0);
          this.dummy.updateMatrix();
          mesh.setMatrixAt(i, this.dummy.matrix);
        }
      }

      mesh.instanceMatrix.needsUpdate = true;
    });
  }

  dispose(): void {
    this.levels.forEach((mesh) => {
      this.scene.remove(mesh);
      mesh.dispose();
    });

    this.levels.clear();
  }
}
```

### 4. Frustum Culling for Instances

```typescript
// instancing/InstanceFrustumCulling.ts
import * as THREE from 'three';

export class InstanceFrustumCulling {
  private mesh: THREE.InstancedMesh;
  private camera: THREE.Camera;
  private frustum = new THREE.Frustum();
  private projScreenMatrix = new THREE.Matrix4();
  private dummy = new THREE.Object3D();

  private instanceBounds: THREE.Box3[] = [];
  private originalMatrices: THREE.Matrix4[] = [];

  constructor(mesh: THREE.InstancedMesh, camera: THREE.Camera) {
    this.mesh = mesh;
    this.camera = camera;

    // Calculate bounding box for each instance
    const count = mesh.count;
    const boundingBox = new THREE.Box3().setFromBufferAttribute(
      mesh.geometry.attributes.position as THREE.BufferAttribute
    );

    for (let i = 0; i < count; i++) {
      mesh.getMatrixAt(i, this.dummy.matrix);
      const instanceBox = boundingBox.clone().applyMatrix4(this.dummy.matrix);
      this.instanceBounds.push(instanceBox);
      this.originalMatrices.push(this.dummy.matrix.clone());
    }
  }

  update(): void {
    // Update frustum
    this.projScreenMatrix.multiplyMatrices(
      this.camera.projectionMatrix,
      this.camera.matrixWorldInverse
    );
    this.frustum.setFromProjectionMatrix(this.projScreenMatrix);

    // Check each instance
    for (let i = 0; i < this.mesh.count; i++) {
      const box = this.instanceBounds[i];

      if (this.frustum.intersectsBox(box)) {
        // Visible - restore original matrix
        this.mesh.setMatrixAt(i, this.originalMatrices[i]);
      } else {
        // Not visible - hide
        this.dummy.position.set(0, -10000, 0);
        this.dummy.updateMatrix();
        this.mesh.setMatrixAt(i, this.dummy.matrix);
      }
    }

    this.mesh.instanceMatrix.needsUpdate = true;
  }

  updateInstanceBounds(index: number): void {
    if (index >= this.mesh.count) return;

    const boundingBox = new THREE.Box3().setFromBufferAttribute(
      this.mesh.geometry.attributes.position as THREE.BufferAttribute
    );

    this.mesh.getMatrixAt(index, this.dummy.matrix);
    this.originalMatrices[index].copy(this.dummy.matrix);

    const instanceBox = boundingBox.clone().applyMatrix4(this.dummy.matrix);
    this.instanceBounds[index].copy(instanceBox);
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic instanced mesh manager
import { InstancedMeshManager } from './instancing/InstancedMeshManager';

const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });

const manager = new InstancedMeshManager(geometry, material, 1000, true);

// Set instances
for (let i = 0; i < 100; i++) {
  manager.setInstance(i, {
    position: new THREE.Vector3(i * 2, 0, 0),
    rotation: new THREE.Euler(0, 0, 0),
    scale: new THREE.Vector3(1, 1, 1),
    color: new THREE.Color(Math.random(), Math.random(), Math.random()),
    visible: true,
  });
}

manager.commitUpdates();
scene.add(manager.getMesh());

// Example 2: Dynamic instance pool
import { InstancePool } from './instancing/InstancePool';

const pool = new InstancePool(geometry, material, 500);
scene.add(pool.getMesh());

// Request instance
const instanceId = pool.request();

if (instanceId !== null) {
  pool.updateInstance(
    instanceId,
    new THREE.Vector3(10, 0, 0),
    new THREE.Euler(0, 0, 0),
    new THREE.Vector3(1, 1, 1)
  );
}

// Release when done
pool.release(instanceId);

// Example 3: Instanced LOD
import { InstancedLOD } from './instancing/InstancedLOD';

const lodLevels = [
  { geometry: highDetailGeometry, distance: 10 },
  { geometry: mediumDetailGeometry, distance: 50 },
  { geometry: lowDetailGeometry, distance: 100 },
];

const instancedLOD = new InstancedLOD(scene, camera, material, lodLevels, 500);

// Set instances
for (let i = 0; i < 500; i++) {
  instancedLOD.setInstance(
    i,
    new THREE.Vector3(Math.random() * 200 - 100, 0, Math.random() * 200 - 100),
    new THREE.Euler(0, 0, 0),
    new THREE.Vector3(1, 1, 1)
  );
}

function animate() {
  instancedLOD.update(); // Switch LOD based on distance
  renderer.render(scene, camera);
}

// Example 4: Frustum culling
import { InstanceFrustumCulling } from './instancing/InstanceFrustumCulling';

const culling = new InstanceFrustumCulling(instancedMesh, camera);

function animate() {
  culling.update(); // Hide instances outside frustum
  renderer.render(scene, camera);
}
```

## Checklist

- [ ] Use InstancedMesh for 100+ identical objects
- [ ] Initialize all instances (even if hidden)
- [ ] Set instanceMatrix.needsUpdate after changes
- [ ] Use custom attributes for per-instance data
- [ ] Implement instance pooling for dynamic objects
- [ ] Add LOD for distant instances
- [ ] Implement frustum culling for large counts
- [ ] Profile draw calls (should be 1 per InstancedMesh)
- [ ] Test on target devices
- [ ] Monitor instance count limits

## Common Pitfalls

1. **Not setting needsUpdate**: Changes don't appear
2. **Updating every frame**: Expensive matrix updates
3. **Too many instances**: GPU memory limit
4. **No visibility culling**: Rendering off-screen instances
5. **Creating new matrices**: Cache and reuse
6. **Geometry not shared**: Defeats purpose of instancing
7. **Material not shared**: Multiple draw calls

## Performance Tips

### Instance Counts
- Mobile: 100-1000 instances
- Desktop: 1000-10000 instances
- High-end: 10000-100000+ instances

### Update Strategies
- **Static**: Set once, never update
- **Batch**: Update many at once, then commit
- **Sparse**: Only update changed instances
- **Pooled**: Reuse instances dynamically

### Memory Optimization
- Share geometry across instance groups
- Share materials when possible
- Use lower precision for matrices (Float32 vs Float64)
- Compress custom attributes

### Culling Strategies
- Frustum culling: Hide off-screen
- Distance culling: Hide very distant
- LOD: Lower detail at distance
- Occlusion culling: Hide blocked instances

### Mobile Considerations
- Limit to 500-1000 instances
- Use simpler geometry
- Implement aggressive culling
- Use lower LOD distances
- Monitor memory usage

## Related Skills

- `threejs-geometry-management` - Geometry optimization
- `threejs-performance-profiling` - Performance monitoring
- `mobile-performance` - Mobile optimization
- `threejs-particles` - Particle instancing

## References

- Three.js InstancedMesh: https://threejs.org/docs/#api/en/objects/InstancedMesh
- Instancing Examples: https://threejs.org/examples/?q=instance
- GPU Instancing: https://webgl2fundamentals.org/webgl/lessons/webgl-instanced-drawing.html
