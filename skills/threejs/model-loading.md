---
name: threejs-model-loading
description: Load and optimize 3D models in GLTF, FBX, and OBJ formats with compression, progressive loading, and mobile optimization
---

# Three.js Model Loading

## When to Use

Use this skill when:
- Loading 3D models from external files
- Implementing asset loading systems
- Optimizing model size for mobile
- Setting up progressive loading
- Managing model memory

## Core Principles

1. **GLTF Preferred**: Use GLTF/GLB as primary format
2. **Compression**: Draco/MeshOpt for smaller files
3. **Progressive Loading**: Load low-res first, upgrade later
4. **Asset Management**: Cache and reuse loaded models
5. **Error Handling**: Graceful fallbacks for loading failures
6. **Mobile Optimization**: Reduce poly count and texture size

## Implementation

### 1. Model Loader Manager

```typescript
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader';
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader';

export interface LoadedModel {
  scene: THREE.Object3D;
  animations: THREE.AnimationClip[];
  materials: THREE.Material[];
  meshes: THREE.Mesh[];
}

export class ModelLoader {
  private gltfLoader: GLTFLoader;
  private fbxLoader: FBXLoader;
  private objLoader: OBJLoader;
  private loadingManager: THREE.LoadingManager;
  private cache = new Map<string, LoadedModel>();

  constructor(
    onProgress?: (url: string, loaded: number, total: number) => void
  ) {
    this.loadingManager = new THREE.LoadingManager();

    if (onProgress) {
      this.loadingManager.onProgress = onProgress;
    }

    // GLTF loader with Draco compression
    this.gltfLoader = new GLTFLoader(this.loadingManager);
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('/draco/');
    this.gltfLoader.setDRACOLoader(dracoLoader);

    this.fbxLoader = new FBXLoader(this.loadingManager);
    this.objLoader = new OBJLoader(this.loadingManager);
  }

  /**
   * Load GLTF model
   */
  async loadGLTF(url: string): Promise<LoadedModel> {
    // Check cache
    if (this.cache.has(url)) {
      return this.cloneModel(this.cache.get(url)!);
    }

    return new Promise((resolve, reject) => {
      this.gltfLoader.load(
        url,
        (gltf) => {
          const model: LoadedModel = {
            scene: gltf.scene,
            animations: gltf.animations,
            materials: this.extractMaterials(gltf.scene),
            meshes: this.extractMeshes(gltf.scene),
          };

          // Optimize model
          this.optimizeModel(model);

          // Cache
          this.cache.set(url, model);

          resolve(this.cloneModel(model));
        },
        undefined,
        reject
      );
    });
  }

  /**
   * Load FBX model
   */
  async loadFBX(url: string): Promise<LoadedModel> {
    if (this.cache.has(url)) {
      return this.cloneModel(this.cache.get(url)!);
    }

    return new Promise((resolve, reject) => {
      this.fbxLoader.load(
        url,
        (fbx) => {
          const model: LoadedModel = {
            scene: fbx,
            animations: fbx.animations || [],
            materials: this.extractMaterials(fbx),
            meshes: this.extractMeshes(fbx),
          };

          this.optimizeModel(model);
          this.cache.set(url, model);

          resolve(this.cloneModel(model));
        },
        undefined,
        reject
      );
    });
  }

  /**
   * Load OBJ model
   */
  async loadOBJ(url: string, mtlUrl?: string): Promise<LoadedModel> {
    if (this.cache.has(url)) {
      return this.cloneModel(this.cache.get(url)!);
    }

    return new Promise((resolve, reject) => {
      this.objLoader.load(
        url,
        (obj) => {
          const model: LoadedModel = {
            scene: obj,
            animations: [],
            materials: this.extractMaterials(obj),
            meshes: this.extractMeshes(obj),
          };

          this.optimizeModel(model);
          this.cache.set(url, model);

          resolve(this.cloneModel(model));
        },
        undefined,
        reject
      );
    });
  }

  /**
   * Load multiple models in parallel
   */
  async loadMultiple(urls: string[]): Promise<LoadedModel[]> {
    const promises = urls.map(url => {
      const ext = url.split('.').pop()?.toLowerCase();

      switch (ext) {
        case 'gltf':
        case 'glb':
          return this.loadGLTF(url);
        case 'fbx':
          return this.loadFBX(url);
        case 'obj':
          return this.loadOBJ(url);
        default:
          return Promise.reject(new Error(`Unsupported format: ${ext}`));
      }
    });

    return Promise.all(promises);
  }

  private extractMaterials(object: THREE.Object3D): THREE.Material[] {
    const materials: THREE.Material[] = [];

    object.traverse((child) => {
      if ((child as THREE.Mesh).isMesh) {
        const mesh = child as THREE.Mesh;
        if (Array.isArray(mesh.material)) {
          materials.push(...mesh.material);
        } else {
          materials.push(mesh.material);
        }
      }
    });

    return materials;
  }

  private extractMeshes(object: THREE.Object3D): THREE.Mesh[] {
    const meshes: THREE.Mesh[] = [];

    object.traverse((child) => {
      if ((child as THREE.Mesh).isMesh) {
        meshes.push(child as THREE.Mesh);
      }
    });

    return meshes;
  }

  private optimizeModel(model: LoadedModel): void {
    // Enable shadows
    model.scene.traverse((child) => {
      if ((child as THREE.Mesh).isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
      }
    });

    // Optimize materials
    model.materials.forEach((material) => {
      // Force double-sided to false if not needed
      if (material.side === THREE.DoubleSide) {
        material.side = THREE.FrontSide;
      }
    });
  }

  private cloneModel(model: LoadedModel): LoadedModel {
    return {
      scene: model.scene.clone(),
      animations: [...model.animations],
      materials: model.materials.map(m => m.clone()),
      meshes: [...model.meshes],
    };
  }

  /**
   * Get cached model
   */
  getCached(url: string): LoadedModel | undefined {
    return this.cache.get(url);
  }

  /**
   * Clear cache
   */
  clearCache(): void {
    this.cache.forEach(model => {
      model.materials.forEach(m => m.dispose());
      model.meshes.forEach(m => m.geometry.dispose());
    });
    this.cache.clear();
  }

  /**
   * Dispose specific model
   */
  dispose(url: string): void {
    const model = this.cache.get(url);
    if (model) {
      model.materials.forEach(m => m.dispose());
      model.meshes.forEach(m => m.geometry.dispose());
      this.cache.delete(url);
    }
  }
}
```

### 2. Progressive Loading

```typescript
export class ProgressiveModelLoader {
  private modelLoader: ModelLoader;
  private lowResCache = new Map<string, LoadedModel>();

  constructor() {
    this.modelLoader = new ModelLoader();
  }

  /**
   * Load low-res first, then high-res
   */
  async loadProgressive(
    lowResUrl: string,
    highResUrl: string,
    onLowResLoaded?: (model: LoadedModel) => void
  ): Promise<LoadedModel> {
    // Load low-res immediately
    const lowRes = await this.modelLoader.loadGLTF(lowResUrl);
    this.lowResCache.set(highResUrl, lowRes);

    onLowResLoaded?.(lowRes);

    // Load high-res in background
    const highRes = await this.modelLoader.loadGLTF(highResUrl);

    return highRes;
  }

  /**
   * Get low-res while high-res loads
   */
  getLowRes(highResUrl: string): LoadedModel | undefined {
    return this.lowResCache.get(highResUrl);
  }
}
```

### 3. Model Pool

```typescript
export class ModelPool {
  private pool = new Map<string, LoadedModel[]>();
  private modelLoader: ModelLoader;

  constructor() {
    this.modelLoader = new ModelLoader();
  }

  /**
   * Pre-load models into pool
   */
  async preload(url: string, count: number): Promise<void> {
    const models: LoadedModel[] = [];

    for (let i = 0; i < count; i++) {
      const model = await this.modelLoader.loadGLTF(url);
      models.push(model);
    }

    this.pool.set(url, models);
  }

  /**
   * Acquire model from pool
   */
  acquire(url: string): LoadedModel | null {
    const models = this.pool.get(url);
    if (!models || models.length === 0) return null;

    return models.pop()!;
  }

  /**
   * Return model to pool
   */
  release(url: string, model: LoadedModel): void {
    const models = this.pool.get(url) || [];
    models.push(model);
    this.pool.set(url, models);
  }

  /**
   * Get pool size
   */
  getPoolSize(url: string): number {
    return this.pool.get(url)?.length || 0;
  }

  dispose(): void {
    this.pool.clear();
  }
}
```

### 4. Mobile Model Optimizer

```typescript
export class MobileModelOptimizer {
  /**
   * Optimize model for mobile
   */
  static optimize(model: LoadedModel, tier: 'low' | 'medium' | 'high'): void {
    const settings = {
      low: {
        simplifyFactor: 0.5,
        maxTextureSize: 512,
        shadowQuality: false,
      },
      medium: {
        simplifyFactor: 0.75,
        maxTextureSize: 1024,
        shadowQuality: true,
      },
      high: {
        simplifyFactor: 1.0,
        maxTextureSize: 2048,
        shadowQuality: true,
      },
    };

    const config = settings[tier];

    // Optimize meshes
    model.meshes.forEach(mesh => {
      // Simplify geometry if needed
      // Note: Requires SimplifyModifier
      // mesh.geometry = simplifyGeometry(mesh.geometry, config.simplifyFactor);

      // Disable shadows on low-end
      if (!config.shadowQuality) {
        mesh.castShadow = false;
        mesh.receiveShadow = false;
      }
    });

    // Optimize materials
    model.materials.forEach(material => {
      if (material instanceof THREE.MeshStandardMaterial) {
        // Reduce texture sizes
        ['map', 'normalMap', 'roughnessMap', 'metalnessMap'].forEach(prop => {
          const texture = (material as any)[prop] as THREE.Texture;
          if (texture && texture.image) {
            this.resizeTexture(texture, config.maxTextureSize);
          }
        });

        // Disable expensive features on low-end
        if (tier === 'low') {
          material.envMap = null;
          material.normalMap = null;
        }
      }
    });
  }

  private static resizeTexture(texture: THREE.Texture, maxSize: number): void {
    const image = texture.image;
    if (!image || !image.width) return;

    if (image.width > maxSize || image.height > maxSize) {
      const canvas = document.createElement('canvas');
      const scale = maxSize / Math.max(image.width, image.height);

      canvas.width = Math.floor(image.width * scale);
      canvas.height = Math.floor(image.height * scale);

      const ctx = canvas.getContext('2d')!;
      ctx.drawImage(image, 0, 0, canvas.width, canvas.height);

      texture.image = canvas;
      texture.needsUpdate = true;
    }
  }
}
```

## Usage Examples

```typescript
// Basic loading
const loader = new ModelLoader((url, loaded, total) => {
  console.log(`Loading: ${(loaded / total * 100).toFixed(0)}%`);
});

const model = await loader.loadGLTF('/models/character.glb');
scene.add(model.scene);

// Progressive loading
const progressiveLoader = new ProgressiveModelLoader();

const model = await progressiveLoader.loadProgressive(
  '/models/character_low.glb',
  '/models/character_high.glb',
  (lowRes) => {
    // Show low-res immediately
    scene.add(lowRes.scene);
  }
);

// Replace with high-res when ready
scene.remove(lowResModel.scene);
scene.add(model.scene);

// Model pooling
const pool = new ModelPool();
await pool.preload('/models/enemy.glb', 10);

// Spawn enemies from pool
const enemy = pool.acquire('/models/enemy.glb');
if (enemy) {
  scene.add(enemy.scene);
}

// Return to pool when destroyed
pool.release('/models/enemy.glb', enemy);

// Mobile optimization
const model = await loader.loadGLTF('/models/game.glb');
const tier = detectDeviceTier();
MobileModelOptimizer.optimize(model, tier);
```

## Checklist

- [ ] Use GLTF/GLB as primary format
- [ ] Enable Draco compression for smaller files
- [ ] Implement loading progress indicators
- [ ] Cache loaded models for reuse
- [ ] Clone models instead of reloading
- [ ] Optimize models for target device tier
- [ ] Reduce texture sizes on mobile
- [ ] Implement model pooling for repeated models
- [ ] Handle loading errors gracefully
- [ ] Dispose models when no longer needed
- [ ] Use progressive loading for large models
- [ ] Test on actual mobile devices

## Common Pitfalls

1. **Not caching models**: Reloading same model multiple times
2. **Not cloning**: Modifying shared model instance
3. **Large textures on mobile**: OOM crashes
4. **Not disposing**: Memory leaks
5. **Missing Draco decoder**: Compressed models fail
6. **Blocking main thread**: Loading freezes UI

## Performance Tips

- Use Draco compression (70-90% smaller files)
- Combine multiple models into single GLTF
- Use texture atlases to reduce draw calls
- Implement model LOD (swap models by distance)
- Pool frequently spawned models
- Load models asynchronously (Web Workers)
- Use progressive loading for large scenes
- Keep poly count under 50K on mobile
- Limit texture size to 1024x1024 on mobile

## Related Skills

- `threejs-animation-systems` - Animating loaded models
- `threejs-texture-management` - Texture optimization
- `asset-bundling` - Asset loading strategies
- `mobile-performance` - Mobile optimization
