---
name: threejs-sprites
description: Sprites and billboards in Three.js including sprite materials, texture atlases, sprite animation, and particle sprites
---

# Three.js Sprites

## When to Use

Use this skill when:
- Creating 2D elements in 3D space (labels, icons, UI)
- Implementing billboards (always face camera)
- Building sprite-based particle systems
- Creating foliage, grass, trees (impostors)
- Implementing health bars, damage numbers
- Making retro-style sprite graphics

## Core Principles

1. **Always Face Camera**: Sprites are always front-facing
2. **Depth Testing**: Sprites respect 3D depth
3. **Texture Atlases**: Batch sprites efficiently
4. **Performance**: Cheaper than 3D meshes
5. **Resolution Independence**: Scale without quality loss
6. **Alpha Blending**: Support transparency

## Implementation

### 1. Sprite Manager

```typescript
// sprites/SpriteManager.ts
import * as THREE from 'three';

export interface SpriteConfig {
  texture?: THREE.Texture;
  color?: THREE.ColorRepresentation;
  opacity?: number;
  size?: number;
  sizeAttenuation?: boolean;
  depthTest?: boolean;
  depthWrite?: boolean;
  blending?: THREE.Blending;
}

export class SpriteManager {
  private scene: THREE.Scene;
  private sprites = new Map<string, THREE.Sprite>();

  constructor(scene: THREE.Scene) {
    this.scene = scene;
  }

  createSprite(id: string, config: SpriteConfig = {}): THREE.Sprite {
    const material = new THREE.SpriteMaterial({
      map: config.texture,
      color: config.color ?? 0xffffff,
      opacity: config.opacity ?? 1,
      sizeAttenuation: config.sizeAttenuation ?? true,
      depthTest: config.depthTest ?? true,
      depthWrite: config.depthWrite ?? false,
      transparent: true,
      blending: config.blending ?? THREE.NormalBlending,
    });

    const sprite = new THREE.Sprite(material);

    if (config.size) {
      sprite.scale.set(config.size, config.size, 1);
    }

    this.sprites.set(id, sprite);
    this.scene.add(sprite);

    return sprite;
  }

  getSprite(id: string): THREE.Sprite | undefined {
    return this.sprites.get(id);
  }

  updateSpriteTexture(id: string, texture: THREE.Texture): void {
    const sprite = this.sprites.get(id);
    if (sprite && sprite.material instanceof THREE.SpriteMaterial) {
      sprite.material.map = texture;
      sprite.material.needsUpdate = true;
    }
  }

  updateSpriteColor(id: string, color: THREE.ColorRepresentation): void {
    const sprite = this.sprites.get(id);
    if (sprite && sprite.material instanceof THREE.SpriteMaterial) {
      sprite.material.color.set(color);
    }
  }

  updateSpriteOpacity(id: string, opacity: number): void {
    const sprite = this.sprites.get(id);
    if (sprite && sprite.material instanceof THREE.SpriteMaterial) {
      sprite.material.opacity = opacity;
    }
  }

  removeSprite(id: string): void {
    const sprite = this.sprites.get(id);
    if (sprite) {
      this.scene.remove(sprite);
      sprite.material.dispose();
      this.sprites.delete(id);
    }
  }

  dispose(): void {
    this.sprites.forEach((sprite) => {
      this.scene.remove(sprite);
      sprite.material.dispose();
    });
    this.sprites.clear();
  }
}
```

### 2. Sprite Animation System

```typescript
// sprites/SpriteAnimation.ts
import * as THREE from 'three';

export interface SpriteFrame {
  u: number; // UV coordinates
  v: number;
  width: number;
  height: number;
}

export class SpriteAnimation {
  private sprite: THREE.Sprite;
  private frames: SpriteFrame[];
  private currentFrame = 0;
  private frameTime: number;
  private elapsedTime = 0;
  private loop: boolean;
  private playing = false;

  constructor(
    sprite: THREE.Sprite,
    frames: SpriteFrame[],
    fps: number = 12,
    loop: boolean = true
  ) {
    this.sprite = sprite;
    this.frames = frames;
    this.frameTime = 1 / fps;
    this.loop = loop;
  }

  play(): void {
    this.playing = true;
    this.currentFrame = 0;
    this.elapsedTime = 0;
  }

  stop(): void {
    this.playing = false;
  }

  pause(): void {
    this.playing = false;
  }

  resume(): void {
    this.playing = true;
  }

  update(deltaTime: number): void {
    if (!this.playing) return;

    this.elapsedTime += deltaTime;

    if (this.elapsedTime >= this.frameTime) {
      this.elapsedTime = 0;
      this.currentFrame++;

      if (this.currentFrame >= this.frames.length) {
        if (this.loop) {
          this.currentFrame = 0;
        } else {
          this.currentFrame = this.frames.length - 1;
          this.playing = false;
        }
      }

      this.applyFrame();
    }
  }

  private applyFrame(): void {
    const frame = this.frames[this.currentFrame];
    const material = this.sprite.material as THREE.SpriteMaterial;

    if (material.map) {
      material.map.offset.set(frame.u, frame.v);
      material.map.repeat.set(frame.width, frame.height);
    }
  }

  getCurrentFrame(): number {
    return this.currentFrame;
  }

  isPlaying(): boolean {
    return this.playing;
  }
}
```

### 3. Texture Atlas for Sprites

```typescript
// sprites/SpriteAtlas.ts
import * as THREE from 'three';
import { SpriteFrame } from './SpriteAnimation';

export interface AtlasConfig {
  texture: THREE.Texture;
  frameWidth: number;
  frameHeight: number;
  columns: number;
  rows: number;
}

export class SpriteAtlas {
  private texture: THREE.Texture;
  private frameWidth: number;
  private frameHeight: number;
  private columns: number;
  private rows: number;

  constructor(config: AtlasConfig) {
    this.texture = config.texture;
    this.frameWidth = config.frameWidth;
    this.frameHeight = config.frameHeight;
    this.columns = config.columns;
    this.rows = config.rows;

    // Configure texture
    this.texture.wrapS = THREE.ClampToEdgeWrapping;
    this.texture.wrapT = THREE.ClampToEdgeWrapping;
    this.texture.minFilter = THREE.LinearFilter;
    this.texture.magFilter = THREE.NearestFilter; // Pixel-perfect
  }

  getFrame(index: number): SpriteFrame {
    const row = Math.floor(index / this.columns);
    const col = index % this.columns;

    const width = 1 / this.columns;
    const height = 1 / this.rows;

    return {
      u: col * width,
      v: 1 - (row + 1) * height, // Flip Y
      width,
      height,
    };
  }

  getFrameRange(start: number, end: number): SpriteFrame[] {
    const frames: SpriteFrame[] = [];

    for (let i = start; i <= end; i++) {
      frames.push(this.getFrame(i));
    }

    return frames;
  }

  getTexture(): THREE.Texture {
    return this.texture;
  }

  getTotalFrames(): number {
    return this.columns * this.rows;
  }
}
```

### 4. Billboard System (Always Face Camera)

```typescript
// sprites/BillboardSystem.ts
import * as THREE from 'three';

export type BillboardMode = 'spherical' | 'cylindrical' | 'none';

export class BillboardSystem {
  private objects = new Map<THREE.Object3D, BillboardMode>();
  private camera: THREE.Camera;

  constructor(camera: THREE.Camera) {
    this.camera = camera;
  }

  add(object: THREE.Object3D, mode: BillboardMode = 'spherical'): void {
    this.objects.set(object, mode);
  }

  remove(object: THREE.Object3D): void {
    this.objects.delete(object);
  }

  update(): void {
    const cameraPosition = this.camera.position;

    this.objects.forEach((mode, object) => {
      if (mode === 'spherical') {
        // Face camera completely
        object.lookAt(cameraPosition);
      } else if (mode === 'cylindrical') {
        // Face camera only on Y axis
        const targetPosition = new THREE.Vector3(
          cameraPosition.x,
          object.position.y,
          cameraPosition.z
        );
        object.lookAt(targetPosition);
      }
      // 'none' = no billboard effect
    });
  }

  clear(): void {
    this.objects.clear();
  }
}
```

### 5. Damage Number System

```typescript
// sprites/DamageNumberSystem.ts
import * as THREE from 'three';

export interface DamageNumberConfig {
  value: number;
  position: THREE.Vector3;
  color?: THREE.Color;
  duration?: number;
  scale?: number;
}

export class DamageNumberSystem {
  private scene: THREE.Scene;
  private canvas: HTMLCanvasElement;
  private context: CanvasRenderingContext2D;
  private activeNumbers: Array<{
    sprite: THREE.Sprite;
    startY: number;
    lifetime: number;
    maxLifetime: number;
  }> = [];

  constructor(scene: THREE.Scene) {
    this.scene = scene;

    // Create canvas for text rendering
    this.canvas = document.createElement('canvas');
    this.canvas.width = 128;
    this.canvas.height = 64;
    this.context = this.canvas.getContext('2d')!;
  }

  spawn(config: DamageNumberConfig): void {
    const texture = this.createTextTexture(
      config.value.toString(),
      config.color ?? new THREE.Color(0xffffff)
    );

    const material = new THREE.SpriteMaterial({
      map: texture,
      transparent: true,
      depthTest: false,
    });

    const sprite = new THREE.Sprite(material);
    sprite.position.copy(config.position);

    const scale = config.scale ?? 1;
    sprite.scale.set(scale, scale * 0.5, 1);

    this.scene.add(sprite);

    this.activeNumbers.push({
      sprite,
      startY: config.position.y,
      lifetime: 0,
      maxLifetime: config.duration ?? 1,
    });
  }

  private createTextTexture(text: string, color: THREE.Color): THREE.Texture {
    const ctx = this.context;

    // Clear canvas
    ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // Draw text
    ctx.font = 'bold 48px Arial';
    ctx.fillStyle = `rgb(${color.r * 255}, ${color.g * 255}, ${color.b * 255})`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(text, this.canvas.width / 2, this.canvas.height / 2);

    // Add outline
    ctx.strokeStyle = 'black';
    ctx.lineWidth = 4;
    ctx.strokeText(text, this.canvas.width / 2, this.canvas.height / 2);

    const texture = new THREE.CanvasTexture(this.canvas);
    texture.needsUpdate = true;

    return texture;
  }

  update(deltaTime: number): void {
    for (let i = this.activeNumbers.length - 1; i >= 0; i--) {
      const number = this.activeNumbers[i];
      number.lifetime += deltaTime;

      const t = number.lifetime / number.maxLifetime;

      // Float up
      number.sprite.position.y = number.startY + t * 2;

      // Fade out
      const material = number.sprite.material as THREE.SpriteMaterial;
      material.opacity = 1 - t;

      // Scale up then down
      const scale = 1 + Math.sin(t * Math.PI) * 0.3;
      number.sprite.scale.multiplyScalar(scale / number.sprite.scale.x);

      // Remove when done
      if (number.lifetime >= number.maxLifetime) {
        this.scene.remove(number.sprite);
        material.map?.dispose();
        material.dispose();
        this.activeNumbers.splice(i, 1);
      }
    }
  }

  clear(): void {
    this.activeNumbers.forEach(({ sprite }) => {
      this.scene.remove(sprite);
      sprite.material.dispose();
      if (sprite.material instanceof THREE.SpriteMaterial && sprite.material.map) {
        sprite.material.map.dispose();
      }
    });

    this.activeNumbers = [];
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic sprite
import { SpriteManager } from './sprites/SpriteManager';

const spriteManager = new SpriteManager(scene);

const textureLoader = new THREE.TextureLoader();
const texture = await textureLoader.loadAsync('/textures/icon.png');

const iconSprite = spriteManager.createSprite('icon', {
  texture,
  size: 1,
  sizeAttenuation: true,
});

iconSprite.position.set(0, 2, 0);

// Example 2: Animated sprite
import { SpriteAnimation } from './sprites/SpriteAnimation';
import { SpriteAtlas } from './sprites/SpriteAtlas';

const atlasTexture = await textureLoader.loadAsync('/textures/character.png');

const atlas = new SpriteAtlas({
  texture: atlasTexture,
  frameWidth: 64,
  frameHeight: 64,
  columns: 8,
  rows: 4,
});

const sprite = spriteManager.createSprite('character', {
  texture: atlas.getTexture(),
  size: 2,
});

// Walk animation (frames 0-7)
const walkFrames = atlas.getFrameRange(0, 7);
const walkAnim = new SpriteAnimation(sprite, walkFrames, 12, true);
walkAnim.play();

function animate() {
  walkAnim.update(deltaTime);
}

// Example 3: Billboard system
import { BillboardSystem } from './sprites/BillboardSystem';

const billboard = spriteManager.createSprite('billboard', {
  texture: treeTexture,
  size: 3,
});

billboard.position.set(10, 0, 10);

const billboardSystem = new BillboardSystem(camera);
billboardSystem.add(billboard, 'cylindrical'); // Face camera on Y axis only

function animate() {
  billboardSystem.update();
}

// Example 4: Damage numbers
import { DamageNumberSystem } from './sprites/DamageNumberSystem';

const damageNumbers = new DamageNumberSystem(scene);

// When enemy takes damage
damageNumbers.spawn({
  value: 42,
  position: enemy.position.clone().add(new THREE.Vector3(0, 2, 0)),
  color: new THREE.Color(0xff0000),
  duration: 1.5,
  scale: 1.5,
});

function animate() {
  damageNumbers.update(deltaTime);
}

// Example 5: Health bar sprite
const healthBarCanvas = document.createElement('canvas');
healthBarCanvas.width = 128;
healthBarCanvas.height = 16;
const ctx = healthBarCanvas.getContext('2d')!;

function updateHealthBar(health: number, maxHealth: number): THREE.Texture {
  ctx.clearRect(0, 0, healthBarCanvas.width, healthBarCanvas.height);

  // Background
  ctx.fillStyle = 'black';
  ctx.fillRect(0, 0, healthBarCanvas.width, healthBarCanvas.height);

  // Health
  const healthWidth = (health / maxHealth) * (healthBarCanvas.width - 4);
  ctx.fillStyle = health > 0.5 * maxHealth ? 'green' : 'red';
  ctx.fillRect(2, 2, healthWidth, healthBarCanvas.height - 4);

  const texture = new THREE.CanvasTexture(healthBarCanvas);
  texture.needsUpdate = true;
  return texture;
}

const healthBarSprite = spriteManager.createSprite('healthBar', {
  texture: updateHealthBar(100, 100),
  size: 1,
});

healthBarSprite.position.set(0, 3, 0);
```

## Checklist

- [ ] Load sprite textures
- [ ] Create sprite materials
- [ ] Set appropriate blending mode
- [ ] Configure size attenuation
- [ ] Use texture atlases for multiple sprites
- [ ] Implement sprite animation if needed
- [ ] Add billboard behavior for trees/grass
- [ ] Test transparency and depth sorting
- [ ] Optimize sprite count
- [ ] Handle sprite disposal

## Common Pitfalls

1. **No depth testing**: Sprites always on top
2. **Wrong blending mode**: Artifacts around edges
3. **Depth write enabled**: Sorting issues
4. **High resolution textures**: Wasted memory
5. **Individual textures**: Use atlases
6. **Not disposing**: Memory leaks
7. **sizeAttenuation false**: Sprites don't scale with distance

## Performance Tips

### Sprite Optimization
- Use texture atlases (1 texture for many sprites)
- Keep sprite count reasonable (<1000)
- Use lower resolution textures (64-256px)
- Share materials when possible
- Batch similar sprites

### Blending Modes
- **NormalBlending**: Standard transparency
- **AdditiveBlending**: Glowing effects
- **MultiplyBlending**: Shadows, dark effects
- **SubtractiveBlending**: Rare, special effects

### Mobile Considerations
- Limit sprite count (<500)
- Use smaller textures (32-128px)
- Prefer AdditiveBlending (faster on some GPUs)
- Avoid depthWrite entirely
- Use sprite atlases aggressively

### Texture Atlas Benefits
- Single draw call for multiple sprites
- Better GPU cache utilization
- Easier animation
- Smaller total file size

## Related Skills

- `threejs-texture-management` - Texture optimization
- `threejs-particles` - Particle sprites
- `mobile-performance` - Mobile optimization
- `threejs-material-systems` - Material configuration

## References

- Three.js Sprite: https://threejs.org/docs/#api/en/objects/Sprite
- Three.js SpriteMaterial: https://threejs.org/docs/#api/en/materials/SpriteMaterial
- Sprite Examples: https://threejs.org/examples/?q=sprite
