---
name: memory-management
description: Mobile memory management including resource disposal, memory pooling, texture management, and leak prevention
---

# Mobile Memory Management

## When to Use

Use this skill when:
- Optimizing for mobile memory constraints
- Preventing memory leaks
- Managing texture memory
- Implementing object pooling
- Handling low memory warnings
- Optimizing garbage collection

## Core Principles

1. **Proactive Disposal**: Clean up resources explicitly
2. **Object Pooling**: Reuse instead of allocate
3. **Texture Management**: Compress and limit VRAM
4. **Lazy Loading**: Load resources on demand
5. **Memory Monitoring**: Track and respond to pressure
6. **GC Optimization**: Minimize garbage collection pauses

## Memory Management Implementation

### 1. Resource Manager

```typescript
// memory/ResourceManager.ts
export interface Disposable {
  dispose(): void;
}

export class ResourceManager {
  private resources = new Map<string, Disposable>();
  private refCounts = new Map<string, number>();
  private memoryUsage = 0;

  register<T extends Disposable>(id: string, resource: T): T {
    if (this.resources.has(id)) {
      // Resource already exists, increment ref count
      this.refCounts.set(id, (this.refCounts.get(id) ?? 0) + 1);
      return this.resources.get(id) as T;
    }

    this.resources.set(id, resource);
    this.refCounts.set(id, 1);

    // Track memory usage
    if ('memory' in resource) {
      this.memoryUsage += (resource as any).memory;
    }

    return resource;
  }

  get<T extends Disposable>(id: string): T | undefined {
    return this.resources.get(id) as T | undefined;
  }

  release(id: string): void {
    const refCount = this.refCounts.get(id) ?? 0;

    if (refCount <= 1) {
      // Last reference, dispose resource
      const resource = this.resources.get(id);
      if (resource) {
        resource.dispose();

        if ('memory' in resource) {
          this.memoryUsage -= (resource as any).memory;
        }
      }

      this.resources.delete(id);
      this.refCounts.delete(id);
    } else {
      // Decrement ref count
      this.refCounts.set(id, refCount - 1);
    }
  }

  getMemoryUsage(): number {
    return this.memoryUsage;
  }

  clear(): void {
    for (const resource of this.resources.values()) {
      resource.dispose();
    }

    this.resources.clear();
    this.refCounts.clear();
    this.memoryUsage = 0;
  }

  // Get resources sorted by memory usage
  getResourcesByMemory(): Array<{ id: string; memory: number }> {
    const resources: Array<{ id: string; memory: number }> = [];

    for (const [id, resource] of this.resources) {
      if ('memory' in resource) {
        resources.push({ id, memory: (resource as any).memory });
      }
    }

    return resources.sort((a, b) => b.memory - a.memory);
  }
}
```

### 2. Texture Memory Manager

```typescript
// memory/TextureManager.ts
export class TextureManager {
  private textures = new Map<string, THREE.Texture>();
  private vramUsage = 0;
  private readonly MAX_VRAM_MB = 256; // Mobile limit

  constructor(private resourceManager: ResourceManager) {}

  async load(url: string, options: { compress?: boolean } = {}): Promise<THREE.Texture> {
    // Check if already loaded
    if (this.textures.has(url)) {
      return this.textures.get(url)!;
    }

    // Check VRAM limit
    if (this.vramUsage >= this.MAX_VRAM_MB * 1024 * 1024) {
      this.evictLeastRecentlyUsed();
    }

    const loader = new THREE.TextureLoader();
    const texture = await loader.loadAsync(url);

    // Apply mobile-friendly settings
    texture.generateMipmaps = true;
    texture.minFilter = THREE.LinearMipmapLinearFilter;
    texture.magFilter = THREE.LinearFilter;

    // Compress if requested
    if (options.compress) {
      this.applyCompression(texture);
    }

    // Calculate and track VRAM usage
    const memorySize = this.calculateTextureSize(texture);
    this.vramUsage += memorySize;

    // Register with resource manager
    this.resourceManager.register(url, {
      dispose: () => {
        texture.dispose();
        this.vramUsage -= memorySize;
        this.textures.delete(url);
      },
    });

    this.textures.set(url, texture);
    return texture;
  }

  private calculateTextureSize(texture: THREE.Texture): number {
    const image = texture.image;
    if (!image) return 0;

    const width = image.width || 512;
    const height = image.height || 512;

    // RGBA = 4 bytes per pixel, plus mipmaps (~1.33x)
    return width * height * 4 * 1.33;
  }

  private applyCompression(texture: THREE.Texture): void {
    // Note: Actual compression requires compressed texture formats
    // This is a placeholder for format selection logic

    // For mobile, prefer:
    // - ASTC on iOS
    // - ETC2 on Android
    // - Fallback to lower resolution

    // Example: Reduce resolution
    if (texture.image) {
      const maxSize = 1024;
      if (texture.image.width > maxSize || texture.image.height > maxSize) {
        texture.image.width = Math.min(texture.image.width, maxSize);
        texture.image.height = Math.min(texture.image.height, maxSize);
        texture.needsUpdate = true;
      }
    }
  }

  private evictLeastRecentlyUsed(): void {
    // Simple strategy: remove oldest texture
    const firstKey = this.textures.keys().next().value;
    if (firstKey) {
      this.resourceManager.release(firstKey);
    }
  }

  release(url: string): void {
    this.resourceManager.release(url);
  }

  getVRAMUsage(): number {
    return this.vramUsage;
  }

  getVRAMLimit(): number {
    return this.MAX_VRAM_MB * 1024 * 1024;
  }

  getVRAMUsagePercent(): number {
    return (this.vramUsage / this.getVRAMLimit()) * 100;
  }
}
```

### 3. Object Pooling

```typescript
// memory/ObjectPool.ts
export interface Poolable {
  reset(): void;
}

export class ObjectPool<T extends Poolable> {
  private pool: T[] = [];
  private active = new Set<T>();
  private factory: () => T;
  private initialSize: number;
  private maxSize: number;

  constructor(
    factory: () => T,
    initialSize: number = 10,
    maxSize: number = 100
  ) {
    this.factory = factory;
    this.initialSize = initialSize;
    this.maxSize = maxSize;

    // Pre-allocate initial pool
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(factory());
    }
  }

  acquire(): T {
    let obj: T;

    if (this.pool.length > 0) {
      obj = this.pool.pop()!;
    } else if (this.active.size < this.maxSize) {
      obj = this.factory();
    } else {
      // Pool exhausted, reuse oldest active object
      console.warn('Object pool exhausted, reusing active object');
      obj = this.active.values().next().value;
      this.active.delete(obj);
      obj.reset();
    }

    this.active.add(obj);
    return obj;
  }

  release(obj: T): void {
    if (!this.active.has(obj)) {
      console.warn('Releasing object not from this pool');
      return;
    }

    this.active.delete(obj);
    obj.reset();

    if (this.pool.length < this.maxSize) {
      this.pool.push(obj);
    }
    // If pool is full, let object be garbage collected
  }

  releaseAll(): void {
    for (const obj of this.active) {
      obj.reset();
      if (this.pool.length < this.maxSize) {
        this.pool.push(obj);
      }
    }
    this.active.clear();
  }

  getActiveCount(): number {
    return this.active.size;
  }

  getPoolSize(): number {
    return this.pool.length;
  }

  clear(): void {
    this.pool = [];
    this.active.clear();
  }
}

// Common poolable types
export class PooledVector3 implements Poolable {
  vector: THREE.Vector3;

  constructor() {
    this.vector = new THREE.Vector3();
  }

  reset(): void {
    this.vector.set(0, 0, 0);
  }
}

export class PooledEntity implements Poolable {
  entity: Entity;

  constructor(world: World) {
    this.entity = world.createEntity();
  }

  reset(): void {
    // Remove all components
    for (const component of this.entity.getComponents()) {
      this.entity.removeComponent(component.constructor as ComponentConstructor<any>);
    }
    this.entity.active = false;
  }
}
```

### 4. Memory Monitor

```typescript
// memory/MemoryMonitor.ts
export interface MemoryInfo {
  usedJSHeapSize: number; // Bytes
  totalJSHeapSize: number;
  jsHeapSizeLimit: number;
  usagePercent: number;
}

export class MemoryMonitor {
  private checkInterval: number;
  private intervalId: number | null = null;
  private listeners = new Set<(info: MemoryInfo) => void>();
  private warningThreshold = 0.85; // 85% usage
  private criticalThreshold = 0.95; // 95% usage

  constructor(checkInterval: number = 5000) {
    this.checkInterval = checkInterval;
  }

  start(): void {
    if (this.intervalId !== null) return;

    this.intervalId = window.setInterval(() => {
      this.checkMemory();
    }, this.checkInterval);

    // Initial check
    this.checkMemory();
  }

  stop(): void {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  private checkMemory(): void {
    const info = this.getMemoryInfo();

    // Notify listeners
    for (const listener of this.listeners) {
      listener(info);
    }

    // Check thresholds
    if (info.usagePercent >= this.criticalThreshold) {
      this.onCriticalMemory();
    } else if (info.usagePercent >= this.warningThreshold) {
      this.onWarningMemory();
    }
  }

  getMemoryInfo(): MemoryInfo {
    if ('memory' in performance) {
      const memory = (performance as any).memory;

      return {
        usedJSHeapSize: memory.usedJSHeapSize,
        totalJSHeapSize: memory.totalJSHeapSize,
        jsHeapSizeLimit: memory.jsHeapSizeLimit,
        usagePercent: (memory.usedJSHeapSize / memory.jsHeapSizeLimit) * 100,
      };
    }

    // Fallback for browsers without memory API
    return {
      usedJSHeapSize: 0,
      totalJSHeapSize: 0,
      jsHeapSizeLimit: 0,
      usagePercent: 0,
    };
  }

  private onWarningMemory(): void {
    console.warn('Memory usage above warning threshold');
    // Trigger gentle cleanup
  }

  private onCriticalMemory(): void {
    console.error('Memory usage critical!');
    // Trigger aggressive cleanup
    this.forceGarbageCollection();
  }

  private forceGarbageCollection(): void {
    // Trigger GC indirectly by nulling references
    // Note: Can't force GC in JavaScript, but can encourage it

    // Clear caches, pools, etc.
    window.dispatchEvent(new CustomEvent('memory:critical'));
  }

  onMemoryChange(listener: (info: MemoryInfo) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  setWarningThreshold(percent: number): void {
    this.warningThreshold = Math.max(0, Math.min(1, percent));
  }

  setCriticalThreshold(percent: number): void {
    this.criticalThreshold = Math.max(0, Math.min(1, percent));
  }
}
```

### 5. Geometry Pool Manager

```typescript
// memory/GeometryManager.ts
export class GeometryManager {
  private geometries = new Map<string, THREE.BufferGeometry>();
  private instances = new Map<THREE.BufferGeometry, number>();

  constructor(private resourceManager: ResourceManager) {}

  createBox(width: number, height: number, depth: number): THREE.BufferGeometry {
    const key = `box:${width}:${height}:${depth}`;
    return this.getOrCreate(key, () => new THREE.BoxGeometry(width, height, depth));
  }

  createSphere(radius: number, segments: number = 32): THREE.BufferGeometry {
    const key = `sphere:${radius}:${segments}`;
    return this.getOrCreate(key, () => new THREE.SphereGeometry(radius, segments, segments));
  }

  createPlane(width: number, height: number): THREE.BufferGeometry {
    const key = `plane:${width}:${height}`;
    return this.getOrCreate(key, () => new THREE.PlaneGeometry(width, height));
  }

  private getOrCreate(key: string, factory: () => THREE.BufferGeometry): THREE.BufferGeometry {
    let geometry = this.geometries.get(key);

    if (!geometry) {
      geometry = factory();
      this.geometries.set(key, geometry);
      this.instances.set(geometry, 0);

      // Register for cleanup
      this.resourceManager.register(key, {
        dispose: () => {
          geometry!.dispose();
          this.geometries.delete(key);
          this.instances.delete(geometry!);
        },
      });
    }

    // Increment instance count
    this.instances.set(geometry, (this.instances.get(geometry) ?? 0) + 1);

    return geometry;
  }

  release(geometry: THREE.BufferGeometry): void {
    const count = (this.instances.get(geometry) ?? 0) - 1;

    if (count <= 0) {
      // Find and release from resource manager
      for (const [key, geo] of this.geometries) {
        if (geo === geometry) {
          this.resourceManager.release(key);
          break;
        }
      }
    } else {
      this.instances.set(geometry, count);
    }
  }

  clear(): void {
    for (const geometry of this.geometries.values()) {
      geometry.dispose();
    }

    this.geometries.clear();
    this.instances.clear();
  }
}
```

### 6. Material Manager

```typescript
// memory/MaterialManager.ts
export class MaterialManager {
  private materials = new Map<string, THREE.Material>();
  private instances = new Map<THREE.Material, number>();

  constructor(private resourceManager: ResourceManager) {}

  createStandard(options: {
    color?: THREE.ColorRepresentation;
    map?: THREE.Texture;
    metalness?: number;
    roughness?: number;
  }): THREE.MeshStandardMaterial {
    const key = this.hashOptions('standard', options);
    return this.getOrCreate(key, () => new THREE.MeshStandardMaterial(options));
  }

  createBasic(options: {
    color?: THREE.ColorRepresentation;
    map?: THREE.Texture;
  }): THREE.MeshBasicMaterial {
    const key = this.hashOptions('basic', options);
    return this.getOrCreate(key, () => new THREE.MeshBasicMaterial(options));
  }

  private hashOptions(type: string, options: any): string {
    // Simple hash of material options
    return `${type}:${JSON.stringify(options)}`;
  }

  private getOrCreate<T extends THREE.Material>(key: string, factory: () => T): T {
    let material = this.materials.get(key) as T;

    if (!material) {
      material = factory();
      this.materials.set(key, material);
      this.instances.set(material, 0);

      // Register for cleanup
      this.resourceManager.register(key, {
        dispose: () => {
          material!.dispose();
          this.materials.delete(key);
          this.instances.delete(material!);
        },
      });
    }

    // Increment instance count
    this.instances.set(material, (this.instances.get(material) ?? 0) + 1);

    return material;
  }

  release(material: THREE.Material): void {
    const count = (this.instances.get(material) ?? 0) - 1;

    if (count <= 0) {
      // Find and release from resource manager
      for (const [key, mat] of this.materials) {
        if (mat === material) {
          this.resourceManager.release(key);
          break;
        }
      }
    } else {
      this.instances.set(material, count);
    }
  }

  clear(): void {
    for (const material of this.materials.values()) {
      material.dispose();
    }

    this.materials.clear();
    this.instances.clear();
  }
}
```

## Usage Examples

```typescript
// Example 1: Setup memory management
const resourceManager = new ResourceManager();
const textureManager = new TextureManager(resourceManager);
const geometryManager = new GeometryManager(resourceManager);
const materialManager = new MaterialManager(resourceManager);

// Example 2: Load and manage textures
const texture = await textureManager.load('/textures/diffuse.jpg', {
  compress: true,
});

// Use texture...

// Release when done
textureManager.release('/textures/diffuse.jpg');

// Example 3: Use geometry pooling
const boxGeometry = geometryManager.createBox(1, 1, 1);
const sphereGeometry = geometryManager.createSphere(0.5);

const mesh1 = new THREE.Mesh(boxGeometry, material);
const mesh2 = new THREE.Mesh(boxGeometry, material); // Reuses geometry

scene.add(mesh1, mesh2);

// Release when removing from scene
scene.remove(mesh1);
geometryManager.release(boxGeometry);

// Example 4: Object pooling for entities
const entityPool = new ObjectPool(() => new PooledEntity(world), 50, 200);

// Spawn entity
const pooled = entityPool.acquire();
const entity = pooled.entity;
entity.addComponent(new Transform(new Vector3(0, 0, 0)));
entity.active = true;

// Return to pool when destroyed
entity.active = false;
entityPool.release(pooled);

// Example 5: Memory monitoring
const memoryMonitor = new MemoryMonitor(5000);

memoryMonitor.onMemoryChange((info) => {
  console.log(`Memory usage: ${info.usagePercent.toFixed(1)}%`);

  if (info.usagePercent > 90) {
    // Aggressive cleanup
    entityPool.releaseAll();
    textureManager.getResourcesByMemory().slice(0, 10).forEach((res) => {
      resourceManager.release(res.id);
    });
  }
});

memoryMonitor.start();

// Example 6: Respond to low memory
window.addEventListener('memory:critical', () => {
  // Clear all pools
  entityPool.clear();

  // Release unused textures
  textureManager.evictLeastRecentlyUsed();

  // Clear geometry cache
  geometryManager.clear();

  // Clear material cache
  materialManager.clear();

  // Force scene cleanup
  scene.traverse((obj) => {
    if (obj instanceof THREE.Mesh) {
      obj.geometry.dispose();

      if (Array.isArray(obj.material)) {
        obj.material.forEach((mat) => mat.dispose());
      } else {
        obj.material.dispose();
      }
    }
  });
});

// Example 7: Disposable scene manager
class DisposableScene {
  private scene: THREE.Scene;
  private objects = new Set<THREE.Object3D>();

  constructor() {
    this.scene = new THREE.Scene();
  }

  add(object: THREE.Object3D): void {
    this.scene.add(object);
    this.objects.add(object);
  }

  remove(object: THREE.Object3D): void {
    this.scene.remove(object);
    this.objects.delete(object);
    this.disposeObject(object);
  }

  private disposeObject(object: THREE.Object3D): void {
    object.traverse((child) => {
      if (child instanceof THREE.Mesh) {
        geometryManager.release(child.geometry);
        if (Array.isArray(child.material)) {
          child.material.forEach((mat) => materialManager.release(mat));
        } else {
          materialManager.release(child.material);
        }
      }
    });
  }

  dispose(): void {
    for (const object of this.objects) {
      this.disposeObject(object);
    }

    this.objects.clear();
    this.scene.clear();
  }
}
```

## Checklist

- [ ] Implement ResourceManager
- [ ] Create TextureManager with VRAM tracking
- [ ] Set up object pooling
- [ ] Add memory monitoring
- [ ] Create geometry/material managers
- [ ] Handle low memory events
- [ ] Implement proper disposal
- [ ] Test for memory leaks
- [ ] Profile memory usage
- [ ] Add memory usage UI

## Common Pitfalls

1. **Missing dispose calls**: Memory leaks
2. **No ref counting**: Premature disposal
3. **Unbounded pools**: Memory growth
4. **Texture duplication**: Wasted VRAM
5. **No memory monitoring**: Crashes on low memory
6. **Event listener leaks**: Accumulating handlers
7. **Circular references**: GC can't collect

## Performance Tips

### Memory Efficiency
- Pool frequently created objects
- Share geometries and materials
- Compress textures for mobile
- Monitor VRAM usage
- Dispose resources explicitly

### Leak Prevention
- Use ref counting for shared resources
- Remove event listeners
- Clear intervals and timeouts
- Null references when done
- Profile with DevTools

### GC Optimization
- Minimize allocations in hot paths
- Reuse objects via pooling
- Avoid creating temporary objects
- Use typed arrays
- Batch operations

### Mobile Considerations
- Lower VRAM limits (256MB typical)
- More aggressive texture compression
- Smaller object pools
- More frequent cleanup
- Monitor device memory API

## Related Skills

- `mobile-battery-optimization` - Power management
- `mobile-performance` - General optimization
- `threejs-texture-management` - Texture optimization
- `threejs-geometry-management` - Geometry sharing
- `ecs-performance` - ECS memory patterns

## References

- JavaScript memory management
- WebGL memory limits
- Mobile VRAM constraints
- Object pooling patterns
