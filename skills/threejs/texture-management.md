---
name: threejs-texture-management
description: Texture loading, compression, atlasing, and memory optimization for mobile Three.js applications
---

# Three.js Texture Management

## When to Use

Use this skill when:
- Loading textures for materials
- Optimizing texture memory on mobile
- Creating texture atlases
- Implementing texture streaming
- Managing texture disposal

## Core Principles

1. **Compress Textures**: Use KTX2/Basis for GPU compression
2. **Power of Two**: Use POT dimensions for mipmaps
3. **Texture Atlases**: Combine multiple textures
4. **Lazy Loading**: Load textures on demand
5. **Proper Disposal**: Dispose textures to free memory
6. **Mipmap Management**: Generate/disable based on need

## Implementation

### 1. Texture Loader

```typescript
import * as THREE from 'three';

export class TextureManager {
  private textures = new Map<string, THREE.Texture>();
  private loader = new THREE.TextureLoader();
  private loading = new Map<string, Promise<THREE.Texture>>();

  /**
   * Load texture with caching
   */
  async load(url: string, options?: {
    flipY?: boolean;
    wrapS?: THREE.Wrapping;
    wrapT?: THREE.Wrapping;
    generateMipmaps?: boolean;
    minFilter?: THREE.MinificationTextureFilter;
    magFilter?: THREE.MagnificationTextureFilter;
  }): Promise<THREE.Texture> {
    // Return cached texture
    if (this.textures.has(url)) {
      return this.textures.get(url)!;
    }

    // Return in-progress load
    if (this.loading.has(url)) {
      return this.loading.get(url)!;
    }

    // Start new load
    const promise = new Promise<THREE.Texture>((resolve, reject) => {
      this.loader.load(
        url,
        (texture) => {
          // Apply options
          if (options?.flipY !== undefined) texture.flipY = options.flipY;
          if (options?.wrapS) texture.wrapS = options.wrapS;
          if (options?.wrapT) texture.wrapT = options.wrapT;
          if (options?.minFilter) texture.minFilter = options.minFilter;
          if (options?.magFilter) texture.magFilter = options.magFilter;

          if (options?.generateMipmaps !== undefined) {
            texture.generateMipmaps = options.generateMipmaps;
          }

          texture.needsUpdate = true;

          this.textures.set(url, texture);
          this.loading.delete(url);
          resolve(texture);
        },
        undefined,
        (error) => {
          this.loading.delete(url);
          reject(error);
        }
      );
    });

    this.loading.set(url, promise);
    return promise;
  }

  /**
   * Load multiple textures
   */
  async loadMultiple(urls: string[]): Promise<THREE.Texture[]> {
    return Promise.all(urls.map(url => this.load(url)));
  }

  /**
   * Create texture from canvas
   */
  fromCanvas(canvas: HTMLCanvasElement, id: string): THREE.CanvasTexture {
    const texture = new THREE.CanvasTexture(canvas);
    this.textures.set(id, texture);
    return texture;
  }

  /**
   * Create texture from data
   */
  fromData(
    data: Uint8Array | Float32Array,
    width: number,
    height: number,
    format: THREE.PixelFormat = THREE.RGBAFormat,
    type: THREE.TextureDataType = THREE.UnsignedByteType
  ): THREE.DataTexture {
    const texture = new THREE.DataTexture(data, width, height, format, type);
    texture.needsUpdate = true;
    return texture;
  }

  /**
   * Get cached texture
   */
  get(url: string): THREE.Texture | undefined {
    return this.textures.get(url);
  }

  /**
   * Dispose texture
   */
  dispose(url: string): void {
    const texture = this.textures.get(url);
    if (texture) {
      texture.dispose();
      this.textures.delete(url);
    }
  }

  /**
   * Dispose all textures
   */
  disposeAll(): void {
    this.textures.forEach(texture => texture.dispose());
    this.textures.clear();
  }
}
```

### 2. Compressed Texture Loading

```typescript
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader';
import { BasisTextureLoader } from 'three/examples/jsm/loaders/BasisTextureLoader';

export class CompressedTextureManager {
  private ktx2Loader: KTX2Loader;
  private basisLoader: BasisTextureLoader;
  private textures = new Map<string, THREE.CompressedTexture>();

  constructor(renderer: THREE.WebGLRenderer, transcoderPath: string) {
    this.ktx2Loader = new KTX2Loader();
    this.ktx2Loader.setTranscoderPath(transcoderPath);
    this.ktx2Loader.detectSupport(renderer);

    this.basisLoader = new BasisTextureLoader();
    this.basisLoader.setTranscoderPath(transcoderPath);
    this.basisLoader.detectSupport(renderer);
  }

  /**
   * Load KTX2 compressed texture
   */
  async loadKTX2(url: string): Promise<THREE.CompressedTexture> {
    if (this.textures.has(url)) {
      return this.textures.get(url)!;
    }

    return new Promise((resolve, reject) => {
      this.ktx2Loader.load(
        url,
        (texture) => {
          this.textures.set(url, texture);
          resolve(texture);
        },
        undefined,
        reject
      );
    });
  }

  /**
   * Load Basis compressed texture
   */
  async loadBasis(url: string): Promise<THREE.CompressedTexture> {
    if (this.textures.has(url)) {
      return this.textures.get(url)!;
    }

    return new Promise((resolve, reject) => {
      this.basisLoader.load(
        url,
        (texture) => {
          this.textures.set(url, texture);
          resolve(texture);
        },
        undefined,
        reject
      );
    });
  }

  dispose(): void {
    this.textures.forEach(texture => texture.dispose());
    this.textures.clear();
    this.ktx2Loader.dispose();
  }
}
```

### 3. Texture Atlas

```typescript
export interface AtlasSprite {
  name: string;
  x: number;
  y: number;
  width: number;
  height: number;
}

export class TextureAtlas {
  private texture: THREE.Texture;
  private sprites = new Map<string, AtlasSprite>();

  constructor(texture: THREE.Texture, sprites: AtlasSprite[]) {
    this.texture = texture;
    sprites.forEach(sprite => this.sprites.set(sprite.name, sprite));
  }

  /**
   * Get UV coordinates for sprite
   */
  getUV(spriteName: string): THREE.Vector4 | undefined {
    const sprite = this.sprites.get(spriteName);
    if (!sprite) return undefined;

    const u = sprite.x / this.texture.image.width;
    const v = sprite.y / this.texture.image.height;
    const uSize = sprite.width / this.texture.image.width;
    const vSize = sprite.height / this.texture.image.height;

    return new THREE.Vector4(u, v, uSize, vSize);
  }

  /**
   * Create sprite material
   */
  createSpriteMaterial(spriteName: string): THREE.SpriteMaterial | undefined {
    const uv = this.getUV(spriteName);
    if (!uv) return undefined;

    const material = new THREE.SpriteMaterial({
      map: this.texture,
    });

    // Adjust UV coordinates
    material.map!.offset.set(uv.x, uv.y);
    material.map!.repeat.set(uv.z, uv.w);

    return material;
  }

  getTexture(): THREE.Texture {
    return this.texture;
  }

  dispose(): void {
    this.texture.dispose();
  }
}
```

### 4. Texture Optimization

```typescript
export class TextureOptimizer {
  /**
   * Resize texture to POT dimensions
   */
  static resizeToPOT(
    texture: THREE.Texture,
    maxSize: number = 2048
  ): THREE.Texture {
    const image = texture.image;
    if (!image) return texture;

    const width = THREE.MathUtils.floorPowerOfTwo(
      Math.min(image.width, maxSize)
    );
    const height = THREE.MathUtils.floorPowerOfTwo(
      Math.min(image.height, maxSize)
    );

    if (width === image.width && height === image.height) {
      return texture;
    }

    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;

    const ctx = canvas.getContext('2d')!;
    ctx.drawImage(image, 0, 0, width, height);

    const resized = new THREE.CanvasTexture(canvas);
    resized.wrapS = texture.wrapS;
    resized.wrapT = texture.wrapT;
    resized.minFilter = texture.minFilter;
    resized.magFilter = texture.magFilter;

    return resized;
  }

  /**
   * Configure for mobile performance
   */
  static optimizeForMobile(texture: THREE.Texture): void {
    // Limit size
    if (texture.image && texture.image.width > 1024) {
      console.warn('Texture too large for mobile:', texture.image.width);
    }

    // Linear filtering is faster
    texture.minFilter = THREE.LinearFilter;
    texture.magFilter = THREE.LinearFilter;

    // Disable mipmaps if not needed
    texture.generateMipmaps = false;

    // Use repeat mode
    texture.wrapS = THREE.RepeatWrapping;
    texture.wrapT = THREE.RepeatWrapping;

    texture.needsUpdate = true;
  }

  /**
   * Compress texture quality
   */
  static compress(texture: THREE.Texture, quality: number = 0.8): THREE.Texture {
    const canvas = document.createElement('canvas');
    canvas.width = texture.image.width;
    canvas.height = texture.image.height;

    const ctx = canvas.getContext('2d')!;
    ctx.drawImage(texture.image, 0, 0);

    // Create compressed version
    const compressed = new THREE.CanvasTexture(canvas);

    return compressed;
  }
}
```

### 5. Texture Streaming

```typescript
export class TextureStreamer {
  private loadQueue: Array<{ url: string; priority: number }> = [];
  private loading = new Set<string>();
  private maxConcurrent = 3;

  /**
   * Add texture to load queue
   */
  enqueue(url: string, priority: number = 0): void {
    this.loadQueue.push({ url, priority });
    this.loadQueue.sort((a, b) => b.priority - a.priority);
    this.processQueue();
  }

  /**
   * Process load queue
   */
  private async processQueue(): Promise<void> {
    while (
      this.loadQueue.length > 0 &&
      this.loading.size < this.maxConcurrent
    ) {
      const item = this.loadQueue.shift()!;

      if (this.loading.has(item.url)) continue;

      this.loading.add(item.url);

      try {
        // Load texture
        await this.loadTexture(item.url);
      } finally {
        this.loading.delete(item.url);
        this.processQueue();
      }
    }
  }

  private async loadTexture(url: string): Promise<THREE.Texture> {
    return new Promise((resolve, reject) => {
      new THREE.TextureLoader().load(url, resolve, undefined, reject);
    });
  }

  clear(): void {
    this.loadQueue = [];
  }
}
```

## Usage Examples

```typescript
// Basic texture loading
const textureManager = new TextureManager();
const diffuseMap = await textureManager.load('/textures/diffuse.jpg', {
  wrapS: THREE.RepeatWrapping,
  wrapT: THREE.RepeatWrapping,
  generateMipmaps: true,
});

// Compressed textures (better for mobile)
const compressedManager = new CompressedTextureManager(
  renderer,
  '/libs/basis/'
);
const compressed = await compressedManager.loadKTX2('/textures/diffuse.ktx2');

// Texture atlas
const atlasTexture = await textureManager.load('/atlas.png');
const atlas = new TextureAtlas(atlasTexture, [
  { name: 'player', x: 0, y: 0, width: 64, height: 64 },
  { name: 'enemy', x: 64, y: 0, width: 64, height: 64 },
]);

const playerMaterial = atlas.createSpriteMaterial('player');

// Optimize for mobile
TextureOptimizer.optimizeForMobile(diffuseMap);

// Cleanup
textureManager.disposeAll();
```

## Checklist

- [ ] Use POT dimensions (256, 512, 1024, 2048)
- [ ] Compress textures (KTX2/Basis) for mobile
- [ ] Limit texture size (1024x1024 max on mobile)
- [ ] Use texture atlases to reduce draw calls
- [ ] Dispose textures when no longer needed
- [ ] Disable mipmaps for UI textures
- [ ] Use LinearFilter instead of nearest for performance
- [ ] Lazy load textures not immediately visible
- [ ] Cache loaded textures
- [ ] Use texture streaming for large worlds

## Common Pitfalls

1. **Non-POT textures with mipmaps**: WebGL warning, no mipmaps generated
2. **Not disposing textures**: Memory leaks
3. **Too many unique textures**: High memory usage
4. **Large texture sizes**: OOM on mobile devices
5. **Loading textures in render loop**: Performance hit
6. **Not using compression**: Larger downloads, more VRAM

## Performance Tips

- Use KTX2 compressed textures (5-10x smaller)
- Limit to 512x512 or 1024x1024 on mobile
- Use texture atlases (reduces draw calls)
- Set `generateMipmaps: false` for UI/sprites
- Use `THREE.LinearFilter` (faster than `THREE.LinearMipmapLinearFilter`)
- Lazy load textures outside viewport
- Pool and reuse textures where possible
- Use `texture.anisotropy = 0` on mobile

## Related Skills

- `threejs-material-systems` - Material setup and optimization
- `mobile-performance` - Mobile-specific optimization
- `threejs-model-loading` - Loading models with textures
- `asset-bundling` - Asset loading strategies
