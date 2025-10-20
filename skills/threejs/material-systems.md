---
name: threejs-material-systems
description: Comprehensive material management including all Three.js material types, custom shaders, material pooling, and mobile optimization
---

# Three.js Material Systems

## When to Use

Use this skill when:
- Setting up materials for 3D objects
- Optimizing material performance on mobile
- Creating custom shader materials
- Managing material memory and reuse
- Implementing PBR workflows

## Core Principles

1. **Material Reuse**: Share materials across meshes to reduce draw calls
2. **Mobile-First Materials**: Use simpler materials (MeshLambert/MeshPhong) on mobile
3. **Texture Efficiency**: Minimize texture count and use atlases
4. **Shader Optimization**: Write efficient shaders, avoid branching
5. **Proper Disposal**: Dispose materials and textures to prevent leaks
6. **Parameter Updates**: Use `material.needsUpdate` sparingly

## Implementation

### 1. Material Manager

```typescript
import * as THREE from 'three';

export interface MaterialConfig {
  type: 'basic' | 'lambert' | 'phong' | 'standard' | 'physical' | 'toon';
  color?: number;
  map?: THREE.Texture;
  normalMap?: THREE.Texture;
  roughness?: number;
  metalness?: number;
  transparent?: boolean;
  opacity?: number;
}

export class MaterialManager {
  private materials = new Map<string, THREE.Material>();
  private sharedMaterials = new Map<string, THREE.Material>();

  /**
   * Create or get cached material
   */
  getMaterial(id: string, config: MaterialConfig): THREE.Material {
    if (this.materials.has(id)) {
      return this.materials.get(id)!;
    }

    const material = this.createMaterial(config);
    this.materials.set(id, material);
    return material;
  }

  /**
   * Create material based on config
   */
  private createMaterial(config: MaterialConfig): THREE.Material {
    const baseProps = {
      color: config.color ?? 0xffffff,
      map: config.map,
      transparent: config.transparent,
      opacity: config.opacity,
    };

    switch (config.type) {
      case 'basic':
        return new THREE.MeshBasicMaterial(baseProps);

      case 'lambert':
        return new THREE.MeshLambertMaterial(baseProps);

      case 'phong':
        return new THREE.MeshPhongMaterial({
          ...baseProps,
          normalMap: config.normalMap,
        });

      case 'standard':
        return new THREE.MeshStandardMaterial({
          ...baseProps,
          normalMap: config.normalMap,
          roughness: config.roughness ?? 0.5,
          metalness: config.metalness ?? 0.5,
        });

      case 'physical':
        return new THREE.MeshPhysicalMaterial({
          ...baseProps,
          normalMap: config.normalMap,
          roughness: config.roughness ?? 0.5,
          metalness: config.metalness ?? 0.5,
        });

      case 'toon':
        return new THREE.MeshToonMaterial(baseProps);

      default:
        return new THREE.MeshStandardMaterial(baseProps);
    }
  }

  /**
   * Clone material for unique instances
   */
  cloneMaterial(id: string): THREE.Material | undefined {
    const original = this.materials.get(id);
    if (!original) return undefined;

    return original.clone();
  }

  /**
   * Update material properties
   */
  updateMaterial(id: string, updates: Partial<MaterialConfig>): void {
    const material = this.materials.get(id);
    if (!material) return;

    Object.assign(material, updates);
    material.needsUpdate = true;
  }

  /**
   * Dispose material and remove from cache
   */
  dispose(id: string): void {
    const material = this.materials.get(id);
    if (material) {
      material.dispose();
      this.materials.delete(id);
    }
  }

  /**
   * Dispose all materials
   */
  disposeAll(): void {
    this.materials.forEach(material => material.dispose());
    this.materials.clear();
  }
}
```

### 2. Standard Materials

```typescript
export class StandardMaterials {
  /**
   * Create PBR material
   */
  static createPBR(options: {
    color?: number;
    roughness?: number;
    metalness?: number;
    map?: THREE.Texture;
    normalMap?: THREE.Texture;
    roughnessMap?: THREE.Texture;
    metalnessMap?: THREE.Texture;
    aoMap?: THREE.Texture;
    envMap?: THREE.Texture;
  }): THREE.MeshStandardMaterial {
    return new THREE.MeshStandardMaterial({
      color: options.color ?? 0xffffff,
      roughness: options.roughness ?? 0.5,
      metalness: options.metalness ?? 0.0,
      map: options.map,
      normalMap: options.normalMap,
      roughnessMap: options.roughnessMap,
      metalnessMap: options.metalnessMap,
      aoMap: options.aoMap,
      envMap: options.envMap,
    });
  }

  /**
   * Create mobile-optimized material (Lambert for performance)
   */
  static createMobile(color: number, map?: THREE.Texture): THREE.MeshLambertMaterial {
    return new THREE.MeshLambertMaterial({
      color,
      map,
    });
  }

  /**
   * Create transparent material
   */
  static createTransparent(
    color: number,
    opacity: number
  ): THREE.MeshStandardMaterial {
    return new THREE.MeshStandardMaterial({
      color,
      transparent: true,
      opacity,
      depthWrite: false, // Important for proper transparency sorting
    });
  }

  /**
   * Create toon shading material
   */
  static createToon(
    color: number,
    gradientMap?: THREE.Texture
  ): THREE.MeshToonMaterial {
    return new THREE.MeshToonMaterial({
      color,
      gradientMap,
    });
  }

  /**
   * Create emissive glowing material
   */
  static createEmissive(
    color: number,
    emissiveColor: number,
    emissiveIntensity: number
  ): THREE.MeshStandardMaterial {
    return new THREE.MeshStandardMaterial({
      color,
      emissive: emissiveColor,
      emissiveIntensity,
    });
  }
}
```

### 3. Custom Shader Materials

```typescript
export class CustomShaderMaterials {
  /**
   * Create basic custom shader material
   */
  static createCustom(
    vertexShader: string,
    fragmentShader: string,
    uniforms: { [key: string]: THREE.IUniform }
  ): THREE.ShaderMaterial {
    return new THREE.ShaderMaterial({
      vertexShader,
      fragmentShader,
      uniforms,
    });
  }

  /**
   * Create animated water shader
   */
  static createWater(): THREE.ShaderMaterial {
    const uniforms = {
      time: { value: 0.0 },
      color: { value: new THREE.Color(0x0077be) },
      opacity: { value: 0.8 },
    };

    const vertexShader = `
      uniform float time;
      varying vec2 vUv;
      varying vec3 vPosition;

      void main() {
        vUv = uv;
        vPosition = position;

        vec3 pos = position;
        pos.z += sin(pos.x * 2.0 + time) * 0.1;
        pos.z += cos(pos.y * 2.0 + time) * 0.1;

        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
      }
    `;

    const fragmentShader = `
      uniform vec3 color;
      uniform float opacity;
      varying vec2 vUv;

      void main() {
        gl_FragColor = vec4(color, opacity);
      }
    `;

    return new THREE.ShaderMaterial({
      uniforms,
      vertexShader,
      fragmentShader,
      transparent: true,
    });
  }

  /**
   * Create vertex color shader
   */
  static createVertexColor(): THREE.ShaderMaterial {
    const vertexShader = `
      attribute vec3 color;
      varying vec3 vColor;

      void main() {
        vColor = color;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `;

    const fragmentShader = `
      varying vec3 vColor;

      void main() {
        gl_FragColor = vec4(vColor, 1.0);
      }
    `;

    return new THREE.ShaderMaterial({
      vertexShader,
      fragmentShader,
      vertexColors: true,
    });
  }

  /**
   * Create dissolve effect shader
   */
  static createDissolve(): THREE.ShaderMaterial {
    const uniforms = {
      progress: { value: 0.0 },
      color: { value: new THREE.Color(0xff6600) },
      map: { value: null },
    };

    const vertexShader = `
      varying vec2 vUv;

      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `;

    const fragmentShader = `
      uniform float progress;
      uniform vec3 color;
      uniform sampler2D map;
      varying vec2 vUv;

      void main() {
        vec4 texColor = texture2D(map, vUv);
        float noise = fract(sin(dot(vUv, vec2(12.9898, 78.233))) * 43758.5453);

        if (noise > progress) {
          gl_FragColor = texColor;
        } else {
          // Dissolve edge glow
          float edge = smoothstep(progress - 0.1, progress, noise);
          gl_FragColor = vec4(color * edge, edge);
        }
      }
    `;

    return new THREE.ShaderMaterial({
      uniforms,
      vertexShader,
      fragmentShader,
      transparent: true,
    });
  }
}
```

### 4. Material Instances and Variants

```typescript
export class MaterialInstanceManager {
  private baseMaterial: THREE.Material;
  private instances = new Map<string, THREE.Material>();

  constructor(baseMaterial: THREE.Material) {
    this.baseMaterial = baseMaterial;
  }

  /**
   * Create material instance with overrides
   */
  createInstance(
    id: string,
    overrides: Partial<THREE.MaterialParameters>
  ): THREE.Material {
    const instance = this.baseMaterial.clone();
    Object.assign(instance, overrides);

    this.instances.set(id, instance);
    return instance;
  }

  /**
   * Get or create instance
   */
  getInstance(
    id: string,
    overrides?: Partial<THREE.MaterialParameters>
  ): THREE.Material {
    if (this.instances.has(id)) {
      return this.instances.get(id)!;
    }

    return overrides ? this.createInstance(id, overrides) : this.baseMaterial;
  }

  /**
   * Update all instances
   */
  updateAll(updates: Partial<THREE.MaterialParameters>): void {
    Object.assign(this.baseMaterial, updates);
    this.baseMaterial.needsUpdate = true;

    this.instances.forEach(instance => {
      Object.assign(instance, updates);
      instance.needsUpdate = true;
    });
  }

  /**
   * Dispose all instances
   */
  dispose(): void {
    this.instances.forEach(instance => instance.dispose());
    this.instances.clear();
    this.baseMaterial.dispose();
  }
}
```

### 5. Material Pooling

```typescript
export class MaterialPool<T extends THREE.Material> {
  private available: T[] = [];
  private inUse = new Set<T>();

  constructor(
    private factory: () => T,
    initialSize: number = 10
  ) {
    for (let i = 0; i < initialSize; i++) {
      this.available.push(this.factory());
    }
  }

  acquire(): T {
    let material = this.available.pop();

    if (!material) {
      material = this.factory();
    }

    this.inUse.add(material);
    return material;
  }

  release(material: T): void {
    if (this.inUse.has(material)) {
      this.inUse.delete(material);
      this.available.push(material);
    }
  }

  dispose(): void {
    this.available.forEach(m => m.dispose());
    this.inUse.forEach(m => m.dispose());
    this.available = [];
    this.inUse.clear();
  }
}
```

## Usage Examples

```typescript
// Material manager
const materialManager = new MaterialManager();

const playerMaterial = materialManager.getMaterial('player', {
  type: 'standard',
  color: 0x00ff00,
  roughness: 0.7,
  metalness: 0.3,
});

// Custom shader with animation
const waterMaterial = CustomShaderMaterials.createWater();

// Update in render loop
function animate(time: number) {
  waterMaterial.uniforms.time.value = time * 0.001;
}

// Material pooling
const materialPool = new MaterialPool(
  () => new THREE.MeshStandardMaterial({ color: 0xff0000 }),
  20
);

const mat1 = materialPool.acquire();
// ... use material ...
materialPool.release(mat1);

// Cleanup
materialManager.disposeAll();
materialPool.dispose();
```

## Checklist

- [ ] Reuse materials across multiple meshes
- [ ] Use simpler materials on mobile (Lambert/Phong)
- [ ] Dispose materials when no longer needed
- [ ] Pool materials for frequently created objects
- [ ] Use texture atlases to reduce material count
- [ ] Minimize use of transparent materials
- [ ] Set depthWrite=false for transparent objects
- [ ] Avoid material.needsUpdate in render loop
- [ ] Use InstancedMesh for color variations
- [ ] Cache material instances
- [ ] Use vertex colors instead of many materials
- [ ] Optimize shader code (avoid conditionals)

## Common Pitfalls

1. **Creating materials in render loop**: Extremely expensive
2. **Not disposing materials**: Memory leaks
3. **Too many unique materials**: Increases draw calls
4. **Excessive texture maps**: High memory usage
5. **Unnecessary needsUpdate calls**: Forces recompilation
6. **Transparent sorting issues**: Use renderOrder

## Performance Tips

- Share materials across meshes (reduces draw calls)
- Use `MeshLambertMaterial` on mobile instead of `MeshStandardMaterial`
- Limit material count to <50 unique materials
- Use material pooling for dynamic objects
- Disable features you don't need (shadows, fog, etc.)
- Use `side: THREE.FrontSide` when possible (default)
- Set `depthTest: true` and `depthWrite: true` for opaque objects
- Use `InstancedMesh` with `instanceColor` for color variations

## Related Skills

- `threejs-texture-management` - Texture loading and optimization
- `threejs-shader-development` - Advanced shader techniques
- `mobile-performance` - Mobile-specific optimizations
- `threejs-scene-setup` - Scene and rendering setup
