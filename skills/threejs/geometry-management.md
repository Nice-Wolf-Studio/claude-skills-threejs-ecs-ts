---
name: threejs-geometry-management
description: Efficient geometry creation, management, and optimization including BufferGeometry, instancing, and custom geometry for mobile performance
---

# Three.js Geometry Management

## When to Use

Use this skill when:
- Creating or modifying 3D geometry in Three.js
- Optimizing geometry for mobile performance
- Implementing instancing for repeated objects
- Building custom geometry procedurally
- Managing geometry memory and disposal

## Core Principles

1. **BufferGeometry Over Geometry**: Always use BufferGeometry for performance
2. **Attribute Efficiency**: Use typed arrays and minimize attribute count
3. **Instancing for Duplicates**: Use InstancedMesh for repeated geometry
4. **Indexed Geometry**: Share vertices with indices to reduce memory
5. **Dispose Properly**: Clean up geometry to prevent memory leaks
6. **LOD Strategy**: Use different detail levels based on distance

## Implementation

### 1. BufferGeometry Basics

```typescript
import * as THREE from 'three';

export class GeometryManager {
  private geometries = new Map<string, THREE.BufferGeometry>();

  /**
   * Create optimized box geometry with minimal vertices
   */
  createBox(width: number, height: number, depth: number, id?: string): THREE.BufferGeometry {
    const geometry = new THREE.BoxGeometry(width, height, depth);

    if (id) {
      this.geometries.set(id, geometry);
    }

    return geometry;
  }

  /**
   * Create custom geometry from vertex data
   */
  createCustomGeometry(
    vertices: Float32Array,
    indices?: Uint16Array | Uint32Array,
    normals?: Float32Array,
    uvs?: Float32Array
  ): THREE.BufferGeometry {
    const geometry = new THREE.BufferGeometry();

    // Position attribute (required)
    geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));

    // Indices for vertex sharing
    if (indices) {
      geometry.setIndex(new THREE.BufferAttribute(indices, 1));
    }

    // Normal attribute for lighting
    if (normals) {
      geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
    } else {
      geometry.computeVertexNormals(); // Auto-generate if not provided
    }

    // UV attribute for textures
    if (uvs) {
      geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
    }

    // Compute bounding sphere for frustum culling
    geometry.computeBoundingSphere();

    return geometry;
  }

  /**
   * Clone geometry efficiently
   */
  clone(geometry: THREE.BufferGeometry): THREE.BufferGeometry {
    return geometry.clone();
  }

  /**
   * Get or create cached geometry
   */
  getOrCreate(id: string, factory: () => THREE.BufferGeometry): THREE.BufferGeometry {
    if (this.geometries.has(id)) {
      return this.geometries.get(id)!;
    }

    const geometry = factory();
    this.geometries.set(id, geometry);
    return geometry;
  }

  /**
   * Dispose geometry and remove from cache
   */
  dispose(id: string): void {
    const geometry = this.geometries.get(id);
    if (geometry) {
      geometry.dispose();
      this.geometries.delete(id);
    }
  }

  /**
   * Dispose all geometries
   */
  disposeAll(): void {
    this.geometries.forEach(geometry => geometry.dispose());
    this.geometries.clear();
  }
}
```

### 2. Instanced Geometry

```typescript
export class InstancedGeometryManager {
  /**
   * Create instanced mesh for many identical objects
   */
  createInstanced(
    geometry: THREE.BufferGeometry,
    material: THREE.Material,
    count: number
  ): THREE.InstancedMesh {
    const mesh = new THREE.InstancedMesh(geometry, material, count);

    // Enable frustum culling per instance
    mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);

    return mesh;
  }

  /**
   * Update single instance transform
   */
  updateInstance(
    mesh: THREE.InstancedMesh,
    index: number,
    position: THREE.Vector3,
    rotation: THREE.Euler,
    scale: THREE.Vector3
  ): void {
    const matrix = new THREE.Matrix4();
    matrix.compose(
      position,
      new THREE.Quaternion().setFromEuler(rotation),
      scale
    );

    mesh.setMatrixAt(index, matrix);
    mesh.instanceMatrix.needsUpdate = true;
  }

  /**
   * Bulk update instances (more efficient)
   */
  updateInstancesBulk(
    mesh: THREE.InstancedMesh,
    transforms: Array<{
      position: THREE.Vector3;
      rotation: THREE.Euler;
      scale: THREE.Vector3;
    }>
  ): void {
    const matrix = new THREE.Matrix4();
    const quaternion = new THREE.Quaternion();

    transforms.forEach((transform, index) => {
      quaternion.setFromEuler(transform.rotation);
      matrix.compose(transform.position, quaternion, transform.scale);
      mesh.setMatrixAt(index, matrix);
    });

    mesh.instanceMatrix.needsUpdate = true;
  }

  /**
   * Set instance color (requires instanceColor attribute)
   */
  setInstanceColor(mesh: THREE.InstancedMesh, index: number, color: THREE.Color): void {
    if (!mesh.instanceColor) {
      mesh.instanceColor = new THREE.InstancedBufferAttribute(
        new Float32Array(mesh.count * 3),
        3
      );
    }

    mesh.setColorAt(index, color);
    mesh.instanceColor.needsUpdate = true;
  }
}
```

### 3. Procedural Geometry

```typescript
export class ProceduralGeometry {
  /**
   * Create plane with custom subdivisions
   */
  createPlane(
    width: number,
    height: number,
    widthSegments: number,
    heightSegments: number
  ): THREE.BufferGeometry {
    const geometry = new THREE.PlaneGeometry(width, height, widthSegments, heightSegments);

    // Access position attribute for vertex manipulation
    const positions = geometry.attributes.position.array as Float32Array;

    // Example: Add wave deformation
    for (let i = 0; i < positions.length; i += 3) {
      const x = positions[i];
      const y = positions[i + 1];
      positions[i + 2] = Math.sin(x * 0.5) * Math.cos(y * 0.5) * 0.5;
    }

    geometry.computeVertexNormals();
    return geometry;
  }

  /**
   * Create heightmap terrain
   */
  createTerrain(
    width: number,
    depth: number,
    segmentsX: number,
    segmentsZ: number,
    heightData: Float32Array
  ): THREE.BufferGeometry {
    const geometry = new THREE.PlaneGeometry(width, depth, segmentsX, segmentsZ);
    geometry.rotateX(-Math.PI / 2);

    const positions = geometry.attributes.position.array as Float32Array;

    for (let i = 0; i < positions.length; i += 3) {
      const heightIndex = Math.floor(i / 3);
      if (heightIndex < heightData.length) {
        positions[i + 1] = heightData[heightIndex]; // Y is up
      }
    }

    geometry.computeVertexNormals();
    return geometry;
  }

  /**
   * Create custom shape with vertices
   */
  createShape(points: THREE.Vector2[]): THREE.BufferGeometry {
    const shape = new THREE.Shape(points);
    const geometry = new THREE.ShapeGeometry(shape);
    return geometry;
  }

  /**
   * Extrude 2D shape into 3D
   */
  extrudeShape(
    points: THREE.Vector2[],
    depth: number,
    bevelEnabled: boolean = false
  ): THREE.BufferGeometry {
    const shape = new THREE.Shape(points);

    const extrudeSettings: THREE.ExtrudeGeometryOptions = {
      depth,
      bevelEnabled,
      bevelThickness: bevelEnabled ? 0.1 : 0,
      bevelSize: bevelEnabled ? 0.1 : 0,
      bevelSegments: bevelEnabled ? 3 : 0,
    };

    return new THREE.ExtrudeGeometry(shape, extrudeSettings);
  }
}
```

### 4. Geometry Modification

```typescript
export class GeometryModifier {
  /**
   * Merge multiple geometries into one (reduces draw calls)
   */
  mergeGeometries(geometries: THREE.BufferGeometry[]): THREE.BufferGeometry {
    return THREE.BufferGeometryUtils.mergeGeometries(geometries);
  }

  /**
   * Simplify geometry (reduce vertex count)
   */
  simplify(geometry: THREE.BufferGeometry, targetVertexCount: number): THREE.BufferGeometry {
    // Note: Requires SimplifyModifier from three/examples
    // This is a placeholder for the concept
    const simplified = geometry.clone();

    // In practice, use:
    // import { SimplifyModifier } from 'three/examples/jsm/modifiers/SimplifyModifier';
    // const modifier = new SimplifyModifier();
    // return modifier.modify(geometry, targetVertexCount);

    return simplified;
  }

  /**
   * Apply matrix transformation to geometry
   */
  applyTransform(
    geometry: THREE.BufferGeometry,
    position: THREE.Vector3,
    rotation: THREE.Euler,
    scale: THREE.Vector3
  ): void {
    const matrix = new THREE.Matrix4();
    matrix.compose(
      position,
      new THREE.Quaternion().setFromEuler(rotation),
      scale
    );

    geometry.applyMatrix4(matrix);
  }

  /**
   * Center geometry at origin
   */
  center(geometry: THREE.BufferGeometry): void {
    geometry.center();
  }

  /**
   * Scale geometry to fit within bounds
   */
  scaleToFit(geometry: THREE.BufferGeometry, maxSize: number): void {
    geometry.computeBoundingBox();
    const boundingBox = geometry.boundingBox!;

    const size = new THREE.Vector3();
    boundingBox.getSize(size);

    const maxDimension = Math.max(size.x, size.y, size.z);
    const scale = maxSize / maxDimension;

    geometry.scale(scale, scale, scale);
  }
}
```

### 5. LOD (Level of Detail)

```typescript
export class LODManager {
  /**
   * Create LOD group with multiple detail levels
   */
  createLOD(
    geometries: THREE.BufferGeometry[],
    material: THREE.Material,
    distances: number[]
  ): THREE.LOD {
    const lod = new THREE.LOD();

    geometries.forEach((geometry, index) => {
      const mesh = new THREE.Mesh(geometry, material);
      lod.addLevel(mesh, distances[index] || 0);
    });

    return lod;
  }

  /**
   * Generate LOD levels automatically
   */
  generateLODLevels(
    geometry: THREE.BufferGeometry,
    levels: number[]
  ): THREE.BufferGeometry[] {
    return levels.map(level => {
      if (level === 1) return geometry.clone();

      // Create simplified versions
      const simplified = geometry.clone();
      // Apply simplification based on level (0.5 = half vertices, etc.)
      return simplified;
    });
  }
}
```

### 6. Memory Management

```typescript
export class GeometryPool<T extends THREE.BufferGeometry> {
  private available: T[] = [];
  private inUse = new Set<T>();

  constructor(
    private factory: () => T,
    private initialSize: number = 10
  ) {
    // Pre-allocate geometries
    for (let i = 0; i < initialSize; i++) {
      this.available.push(this.factory());
    }
  }

  acquire(): T {
    let geometry = this.available.pop();

    if (!geometry) {
      geometry = this.factory();
    }

    this.inUse.add(geometry);
    return geometry;
  }

  release(geometry: T): void {
    if (this.inUse.has(geometry)) {
      this.inUse.delete(geometry);
      this.available.push(geometry);
    }
  }

  dispose(): void {
    this.available.forEach(g => g.dispose());
    this.inUse.forEach(g => g.dispose());
    this.available = [];
    this.inUse.clear();
  }

  get poolSize(): number {
    return this.available.length + this.inUse.size;
  }
}
```

## Usage Examples

```typescript
// Create geometry manager
const geometryManager = new GeometryManager();

// Create and cache geometry
const boxGeo = geometryManager.getOrCreate('box', () =>
  geometryManager.createBox(1, 1, 1)
);

// Instanced rendering (100 cubes)
const instancedManager = new InstancedGeometryManager();
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const instancedMesh = instancedManager.createInstanced(boxGeo, material, 100);

// Update instances
for (let i = 0; i < 100; i++) {
  instancedManager.updateInstance(
    instancedMesh,
    i,
    new THREE.Vector3(Math.random() * 10, Math.random() * 10, Math.random() * 10),
    new THREE.Euler(0, 0, 0),
    new THREE.Vector3(1, 1, 1)
  );
}

scene.add(instancedMesh);

// Object pooling
const geometryPool = new GeometryPool(() => new THREE.BoxGeometry(1, 1, 1), 20);
const geo1 = geometryPool.acquire();
// ... use geometry ...
geometryPool.release(geo1);

// Cleanup
geometryManager.disposeAll();
geometryPool.dispose();
```

## Checklist

- [ ] Use BufferGeometry instead of Geometry
- [ ] Use InstancedMesh for repeated objects (>50 instances)
- [ ] Implement indexed geometry to share vertices
- [ ] Compute bounding spheres for culling
- [ ] Cache frequently used geometries
- [ ] Implement geometry pooling for dynamic objects
- [ ] Use LOD for large/distant objects
- [ ] Dispose geometries when no longer needed
- [ ] Use typed arrays (Float32Array, Uint16Array)
- [ ] Minimize attribute count and size
- [ ] Merge static geometries to reduce draw calls
- [ ] Pre-allocate geometry for object pools

## Common Pitfalls

1. **Not disposing geometry**: Causes memory leaks
2. **Creating geometry in render loop**: Very expensive
3. **Too many draw calls**: Use instancing or merging
4. **High vertex count on mobile**: Use LOD or simplification
5. **Not computing normals**: Results in flat shading
6. **Forgetting to update instanceMatrix**: Instances won't move

## Performance Tips

- Use `InstancedMesh` for 50+ identical objects (10-100x faster)
- Merge static geometry with `BufferGeometryUtils.mergeGeometries()`
- Use `Uint16Array` for indices if vertex count < 65536
- Set `geometry.setUsage(THREE.StaticDrawUsage)` for unchanging geometry
- Pre-compute and cache procedural geometry
- Use geometry pooling for frequently created/destroyed objects
- Implement LOD to reduce vertex count at distance
- Keep vertex count under 10K per geometry on mobile

## Related Skills

- `threejs-mesh-operations` - Mesh creation and manipulation
- `threejs-optimization` - General performance optimization
- `ecs-component-design` - Geometry component patterns
- `mobile-performance` - Mobile-specific optimizations
