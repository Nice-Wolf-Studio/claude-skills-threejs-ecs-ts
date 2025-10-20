---
name: threejs-best-practices
description: Comprehensive best practices for Three.js development including optimization, architecture, debugging, and production deployment
---

# Three.js Best Practices

## When to Use

Use this skill when:
- Starting a new Three.js project
- Optimizing existing Three.js applications
- Preparing for production deployment
- Debugging performance issues
- Establishing coding standards
- Training team members

## Core Principles

1. **Measure First**: Profile before optimizing
2. **Start Simple**: Add complexity gradually
3. **Dispose Everything**: Prevent memory leaks
4. **Share Resources**: Reuse geometries, materials, textures
5. **Target Devices**: Test on actual hardware
6. **Progressive Enhancement**: Core experience + enhancements

## Best Practices

### 1. Project Setup

```typescript
// ✅ Good: Organized structure
/src
  /scenes
    MainScene.ts
    LevelScene.ts
  /entities
    Player.ts
    Enemy.ts
  /systems
    RenderSystem.ts
    PhysicsSystem.ts
  /components
    Transform.ts
    Renderable.ts
  /assets
    /textures
    /models
    /shaders
  /utils
    AssetLoader.ts
    PerformanceMonitor.ts
  main.ts

// ❌ Bad: Everything in one file
main.ts (5000+ lines)
```

### 2. Resource Management

```typescript
// ✅ Good: Proper disposal
class ResourceManager {
  private resources = new Map<string, THREE.Object3D>();

  load(id: string, resource: THREE.Object3D): void {
    this.resources.set(id, resource);
  }

  dispose(): void {
    this.resources.forEach((resource) => {
      resource.traverse((child) => {
        if (child instanceof THREE.Mesh) {
          child.geometry.dispose();

          if (Array.isArray(child.material)) {
            child.material.forEach((mat) => mat.dispose());
          } else {
            child.material.dispose();
          }
        }
      });
    });

    this.resources.clear();
  }
}

// ❌ Bad: No disposal
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
// Later...
scene.remove(mesh); // Memory leak! Geometry and material still in memory
```

### 3. Geometry Sharing

```typescript
// ✅ Good: Share geometry
const boxGeometry = new THREE.BoxGeometry(1, 1, 1);
const material1 = new THREE.MeshStandardMaterial({ color: 0xff0000 });
const material2 = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

const box1 = new THREE.Mesh(boxGeometry, material1);
const box2 = new THREE.Mesh(boxGeometry, material2); // Reuse geometry

// ❌ Bad: Create new geometry for each mesh
const box1 = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), material1);
const box2 = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), material2);
```

### 4. Update Loop

```typescript
// ✅ Good: Fixed timestep for physics, variable for rendering
class GameLoop {
  private lastTime = 0;
  private accumulator = 0;
  private readonly fixedDeltaTime = 1 / 60; // 60 FPS physics

  update(currentTime: number): void {
    const deltaTime = Math.min((currentTime - this.lastTime) / 1000, 0.1);
    this.lastTime = currentTime;

    this.accumulator += deltaTime;

    // Fixed timestep for physics
    while (this.accumulator >= this.fixedDeltaTime) {
      this.updatePhysics(this.fixedDeltaTime);
      this.accumulator -= this.fixedDeltaTime;
    }

    // Variable timestep for rendering
    this.render(deltaTime);
  }

  private updatePhysics(dt: number): void {
    // Physics updates with fixed timestep
  }

  private render(dt: number): void {
    // Render with variable timestep
  }
}

// ❌ Bad: Inconsistent timesteps
function animate() {
  const deltaTime = clock.getDelta(); // Varies wildly
  updatePhysics(deltaTime); // Physics becomes unstable
  renderer.render(scene, camera);
}
```

### 5. Draw Call Optimization

```typescript
// ✅ Good: Minimize draw calls
// Use InstancedMesh for repeated objects
const instancedMesh = new THREE.InstancedMesh(geometry, material, 1000);
scene.add(instancedMesh); // 1 draw call

// Merge static geometry
const mergedGeometry = BufferGeometryUtils.mergeBufferGeometries([geo1, geo2, geo3]);
const mergedMesh = new THREE.Mesh(mergedGeometry, material);
scene.add(mergedMesh); // 1 draw call instead of 3

// ❌ Bad: One mesh per object
for (let i = 0; i < 1000; i++) {
  const mesh = new THREE.Mesh(geometry, material.clone()); // 1000 draw calls!
  scene.add(mesh);
}
```

### 6. Material Management

```typescript
// ✅ Good: Share materials
const materialLibrary = {
  metal: new THREE.MeshStandardMaterial({ metalness: 1, roughness: 0.2 }),
  wood: new THREE.MeshStandardMaterial({ metalness: 0, roughness: 0.8 }),
  glass: new THREE.MeshPhysicalMaterial({ transmission: 1, thickness: 0.5 }),
};

// Reuse materials
mesh1.material = materialLibrary.metal;
mesh2.material = materialLibrary.metal;

// ❌ Bad: Create new material for each object
mesh1.material = new THREE.MeshStandardMaterial({ metalness: 1, roughness: 0.2 });
mesh2.material = new THREE.MeshStandardMaterial({ metalness: 1, roughness: 0.2 });
```

### 7. Texture Optimization

```typescript
// ✅ Good: Compressed textures, mipmaps, proper sizing
const textureLoader = new THREE.TextureLoader();

async function loadOptimizedTexture(url: string): Promise<THREE.Texture> {
  const texture = await textureLoader.loadAsync(url);

  // Enable mipmaps
  texture.generateMipmaps = true;
  texture.minFilter = THREE.LinearMipmapLinearFilter;
  texture.magFilter = THREE.LinearFilter;

  // Anisotropic filtering (improves quality at angles)
  texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

  return texture;
}

// ❌ Bad: No optimization
const texture = textureLoader.load('/huge-texture-8k.png'); // Way too large!
texture.minFilter = THREE.LinearFilter; // No mipmaps = shimmering
```

### 8. Camera Management

```typescript
// ✅ Good: Configure camera properly
const camera = new THREE.PerspectiveCamera(
  75, // FOV: 60-75 for games, 45-60 for architectural
  window.innerWidth / window.innerHeight,
  0.1, // Near: As far as possible without clipping
  1000  // Far: Only as far as needed
);

// Update on resize
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// ❌ Bad: Extreme values
const camera = new THREE.PerspectiveCamera(
  120, // Too wide, distortion
  window.innerWidth / window.innerHeight,
  0.001, // Too close, z-fighting
  100000 // Too far, precision issues
);
```

### 9. Lighting Strategy

```typescript
// ✅ Good: Limited, strategic lights
const lights = {
  ambient: new THREE.AmbientLight(0x404040, 0.5),
  sun: new THREE.DirectionalLight(0xffffff, 1),
  point1: new THREE.PointLight(0xff0000, 1, 10),
  point2: new THREE.PointLight(0x00ff00, 1, 10),
};

// Only 4 lights affecting meshes
scene.add(lights.ambient, lights.sun, lights.point1, lights.point2);

// ❌ Bad: Too many lights
for (let i = 0; i < 50; i++) {
  const light = new THREE.PointLight(0xffffff, 1, 10);
  scene.add(light); // Performance disaster!
}
```

### 10. Error Handling

```typescript
// ✅ Good: Graceful error handling
class AssetLoader {
  async loadModel(url: string): Promise<THREE.Object3D> {
    try {
      const loader = new GLTFLoader();
      const gltf = await loader.loadAsync(url);
      return gltf.scene;
    } catch (error) {
      console.error(`Failed to load model: ${url}`, error);

      // Return placeholder
      const placeholder = new THREE.Mesh(
        new THREE.BoxGeometry(1, 1, 1),
        new THREE.MeshBasicMaterial({ color: 0xff00ff, wireframe: true })
      );

      return placeholder;
    }
  }
}

// ❌ Bad: No error handling
async loadModel(url: string): Promise<THREE.Object3D> {
  const loader = new GLTFLoader();
  const gltf = await loader.loadAsync(url); // Crashes on error
  return gltf.scene;
}
```

## Performance Checklist

### Development
- [ ] Use renderer.info to monitor draw calls
- [ ] Profile with Chrome DevTools
- [ ] Test on target devices (not just dev machine)
- [ ] Set performance budgets (FPS, draw calls, memory)
- [ ] Use Stats.js or similar for real-time monitoring
- [ ] Enable source maps for debugging

### Geometry
- [ ] Share geometries when possible
- [ ] Use InstancedMesh for 100+ identical objects
- [ ] Merge static geometry
- [ ] Use appropriate detail levels (don't over-triangulate)
- [ ] Implement LOD for distant objects
- [ ] Dispose unused geometries

### Materials
- [ ] Limit unique materials (<50 on mobile, <200 on desktop)
- [ ] Share materials when possible
- [ ] Use simpler materials when appropriate (Lambert vs Standard)
- [ ] Disable features you don't need (e.g., vertex colors)
- [ ] Use material.needsUpdate sparingly

### Textures
- [ ] Compress textures (KTX2, Basis)
- [ ] Use appropriate resolutions (512-2048px most cases)
- [ ] Enable mipmaps for minification
- [ ] Use texture atlases
- [ ] Dispose unused textures
- [ ] Limit total texture memory (<100MB on mobile)

### Lighting
- [ ] Limit active lights (4-8 total)
- [ ] Use hemisphere + directional instead of many point lights
- [ ] Bake static lighting when possible
- [ ] Disable shadows on distant/small lights
- [ ] Use light probes for indirect lighting

### Shadows
- [ ] Limit shadow-casting lights (1-3)
- [ ] Use appropriate shadow map sizes (512-2048)
- [ ] Configure shadow camera frustum tightly
- [ ] Disable shadows on small/distant objects
- [ ] Use PCFSoftShadowMap or lower

### Post-Processing
- [ ] Limit passes (2-3 on mobile, 4-6 on desktop)
- [ ] Use lower resolution for expensive effects
- [ ] Provide quality settings
- [ ] Skip post-processing on very low-end devices

## Production Deployment

### Before Release
```typescript
// ✅ Enable production optimizations
renderer.shadowMap.autoUpdate = false; // Update manually when needed
renderer.info.autoReset = false; // Reset manually

// Use production builds
import * as THREE from 'three/build/three.module.js'; // Tree-shaken

// Compress assets
// - Use Draco for geometry
// - Use KTX2/Basis for textures
// - Minify and gzip JavaScript

// Add loading screens
class LoadingManager {
  private manager = new THREE.LoadingManager();

  constructor() {
    this.manager.onProgress = (url, loaded, total) => {
      const progress = (loaded / total) * 100;
      console.log(`Loading: ${progress}%`);
    };

    this.manager.onLoad = () => {
      console.log('Loading complete');
    };

    this.manager.onError = (url) => {
      console.error(`Error loading: ${url}`);
    };
  }

  getManager(): THREE.LoadingManager {
    return this.manager;
  }
}
```

### Monitoring
```typescript
// Add performance monitoring
class PerformanceTracker {
  track(event: string, data: any): void {
    // Send to analytics
    if (typeof window.analytics !== 'undefined') {
      window.analytics.track(event, {
        fps: Math.round(1 / data.deltaTime),
        drawCalls: renderer.info.render.calls,
        triangles: renderer.info.render.triangles,
        ...data,
      });
    }
  }
}
```

## Common Anti-Patterns

### ❌ Creating Objects in Render Loop
```typescript
// BAD
function animate() {
  const box = new THREE.Mesh(geometry, material); // Creates every frame!
  scene.add(box);
}

// GOOD
const box = new THREE.Mesh(geometry, material); // Create once
scene.add(box);

function animate() {
  // Just update position
  box.position.x += 0.1;
}
```

### ❌ Not Using Object Pooling
```typescript
// BAD
function spawnBullet() {
  const bullet = new THREE.Mesh(geometry, material);
  scene.add(bullet);
  // Later...
  scene.remove(bullet); // Creates garbage
}

// GOOD
class BulletPool {
  private pool: THREE.Mesh[] = [];

  acquire(): THREE.Mesh {
    return this.pool.pop() || new THREE.Mesh(geometry, material);
  }

  release(bullet: THREE.Mesh): void {
    bullet.visible = false;
    this.pool.push(bullet);
  }
}
```

### ❌ Excessive clone() Usage
```typescript
// BAD
const material1 = baseMaterial.clone(); // Heavy operation
const material2 = baseMaterial.clone();
const material3 = baseMaterial.clone();

// GOOD
const material = baseMaterial; // Share when possible
// OR use variants
const redMaterial = baseMaterial.clone();
redMaterial.color.set(0xff0000);
// Reuse redMaterial for all red objects
```

## Debugging Tips

### Visual Debugging
```typescript
// Helpers
scene.add(new THREE.AxesHelper(5));
scene.add(new THREE.GridHelper(100, 100));

const light = new THREE.DirectionalLight();
scene.add(new THREE.DirectionalLightHelper(light));

const camera = new THREE.PerspectiveCamera();
scene.add(new THREE.CameraHelper(camera));

// Wireframe mode
material.wireframe = true;

// Bounding boxes
const box = new THREE.Box3().setFromObject(mesh);
const helper = new THREE.Box3Helper(box, 0xffff00);
scene.add(helper);
```

### Performance Debugging
```typescript
// Check renderer info
console.log('Draw calls:', renderer.info.render.calls);
console.log('Triangles:', renderer.info.render.triangles);
console.log('Geometries:', renderer.info.memory.geometries);
console.log('Textures:', renderer.info.memory.textures);

// Profile specific operations
console.time('updatePhysics');
updatePhysics(deltaTime);
console.timeEnd('updatePhysics');
```

## Related Skills

- All other Three.js skills in this repository
- `mobile-performance` - Mobile-specific optimization
- `ecs-architecture` - Architecture patterns
- `typescript-game-types` - Type safety

## References

- Three.js Documentation: https://threejs.org/docs/
- Three.js Examples: https://threejs.org/examples/
- Three.js Fundamentals: https://threejsfundamentals.org/
- Performance Tips: https://discoverthreejs.com/tips-and-tricks/
