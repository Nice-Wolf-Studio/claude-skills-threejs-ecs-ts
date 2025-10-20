---
name: threejs-pbr-materials
description: Physically-Based Rendering materials in Three.js including metalness-roughness workflow, clear coat, transmission, and material optimization
---

# Three.js PBR Materials

## When to Use

Use this skill when:
- Creating realistic materials
- Implementing metalness-roughness workflow
- Adding clear coat, transmission, or sheen
- Setting up IBL (Image-Based Lighting)
- Optimizing material rendering
- Creating material libraries

## Core Principles

1. **Physical Accuracy**: Materials behave like real-world materials
2. **Energy Conservation**: Light bounces don't exceed input
3. **IBL Integration**: Use environment maps for realistic lighting
4. **Texture Maps**: Leverage full PBR texture set
5. **Performance Balance**: Quality vs render cost
6. **Material Instancing**: Share materials when possible

## Implementation

### 1. PBR Material Manager

```typescript
// materials/PBRMaterialManager.ts
import * as THREE from 'three';
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader';

export interface PBRMaterialConfig {
  color?: THREE.ColorRepresentation;
  metalness?: number;
  roughness?: number;
  map?: THREE.Texture;
  normalMap?: THREE.Texture;
  normalScale?: THREE.Vector2;
  roughnessMap?: THREE.Texture;
  metalnessMap?: THREE.Texture;
  aoMap?: THREE.Texture;
  aoMapIntensity?: number;
  emissive?: THREE.ColorRepresentation;
  emissiveMap?: THREE.Texture;
  emissiveIntensity?: number;
  envMap?: THREE.Texture;
  envMapIntensity?: number;
  clearcoat?: number;
  clearcoatRoughness?: number;
  transmission?: number;
  thickness?: number;
  ior?: number;
}

export class PBRMaterialManager {
  private materials = new Map<string, THREE.MeshStandardMaterial>();
  private envMap: THREE.Texture | null = null;
  private pmremGenerator: THREE.PMREMGenerator;

  constructor(private renderer: THREE.WebGLRenderer) {
    this.pmremGenerator = new THREE.PMREMGenerator(renderer);
  }

  async loadEnvironmentMap(url: string): Promise<THREE.Texture> {
    const loader = new RGBELoader();
    const hdrTexture = await loader.loadAsync(url);

    const envMap = this.pmremGenerator.fromEquirectangular(hdrTexture).texture;
    hdrTexture.dispose();

    this.envMap = envMap;
    return envMap;
  }

  createMaterial(id: string, config: PBRMaterialConfig = {}): THREE.MeshStandardMaterial {
    const material = new THREE.MeshStandardMaterial({
      color: config.color ?? 0xffffff,
      metalness: config.metalness ?? 0,
      roughness: config.roughness ?? 1,
      map: config.map,
      normalMap: config.normalMap,
      normalScale: config.normalScale ?? new THREE.Vector2(1, 1),
      roughnessMap: config.roughnessMap,
      metalnessMap: config.metalnessMap,
      aoMap: config.aoMap,
      aoMapIntensity: config.aoMapIntensity ?? 1,
      emissive: config.emissive ?? 0x000000,
      emissiveMap: config.emissiveMap,
      emissiveIntensity: config.emissiveIntensity ?? 1,
      envMap: config.envMap ?? this.envMap,
      envMapIntensity: config.envMapIntensity ?? 1,
    });

    this.materials.set(id, material);
    return material;
  }

  createPhysicalMaterial(
    id: string,
    config: PBRMaterialConfig = {}
  ): THREE.MeshPhysicalMaterial {
    const material = new THREE.MeshPhysicalMaterial({
      color: config.color ?? 0xffffff,
      metalness: config.metalness ?? 0,
      roughness: config.roughness ?? 1,
      map: config.map,
      normalMap: config.normalMap,
      normalScale: config.normalScale ?? new THREE.Vector2(1, 1),
      roughnessMap: config.roughnessMap,
      metalnessMap: config.metalnessMap,
      aoMap: config.aoMap,
      aoMapIntensity: config.aoMapIntensity ?? 1,
      emissive: config.emissive ?? 0x000000,
      emissiveMap: config.emissiveMap,
      emissiveIntensity: config.emissiveIntensity ?? 1,
      envMap: config.envMap ?? this.envMap,
      envMapIntensity: config.envMapIntensity ?? 1,
      clearcoat: config.clearcoat ?? 0,
      clearcoatRoughness: config.clearcoatRoughness ?? 0,
      transmission: config.transmission ?? 0,
      thickness: config.thickness ?? 0,
      ior: config.ior ?? 1.5,
    });

    return material;
  }

  getMaterial(id: string): THREE.MeshStandardMaterial | undefined {
    return this.materials.get(id);
  }

  setGlobalEnvMap(envMap: THREE.Texture): void {
    this.envMap = envMap;

    this.materials.forEach((material) => {
      material.envMap = envMap;
      material.needsUpdate = true;
    });
  }

  dispose(): void {
    this.materials.forEach((material) => material.dispose());
    this.materials.clear();

    if (this.envMap) {
      this.envMap.dispose();
    }

    this.pmremGenerator.dispose();
  }
}
```

### 2. Material Presets

```typescript
// materials/MaterialPresets.ts
import * as THREE from 'three';
import { PBRMaterialConfig } from './PBRMaterialManager';

export const MaterialPresets = {
  // Metals
  gold: {
    color: 0xffd700,
    metalness: 1,
    roughness: 0.3,
  } as PBRMaterialConfig,

  silver: {
    color: 0xc0c0c0,
    metalness: 1,
    roughness: 0.2,
  } as PBRMaterialConfig,

  copper: {
    color: 0xb87333,
    metalness: 1,
    roughness: 0.4,
  } as PBRMaterialConfig,

  aluminum: {
    color: 0xa8a9ad,
    metalness: 1,
    roughness: 0.5,
  } as PBRMaterialConfig,

  chrome: {
    color: 0xffffff,
    metalness: 1,
    roughness: 0.05,
  } as PBRMaterialConfig,

  // Non-metals
  plastic: {
    color: 0xff0000,
    metalness: 0,
    roughness: 0.5,
  } as PBRMaterialConfig,

  rubber: {
    color: 0x1a1a1a,
    metalness: 0,
    roughness: 0.9,
  } as PBRMaterialConfig,

  wood: {
    color: 0x8b4513,
    metalness: 0,
    roughness: 0.8,
  } as PBRMaterialConfig,

  ceramic: {
    color: 0xffffff,
    metalness: 0,
    roughness: 0.3,
  } as PBRMaterialConfig,

  glass: {
    color: 0xffffff,
    metalness: 0,
    roughness: 0,
    transmission: 1,
    thickness: 0.5,
    ior: 1.5,
  } as PBRMaterialConfig,

  water: {
    color: 0x0077be,
    metalness: 0,
    roughness: 0.1,
    transmission: 0.9,
    thickness: 0.1,
    ior: 1.333,
  } as PBRMaterialConfig,

  // Special
  carPaint: {
    color: 0xff0000,
    metalness: 0.9,
    roughness: 0.2,
    clearcoat: 1,
    clearcoatRoughness: 0.1,
  } as PBRMaterialConfig,

  skin: {
    color: 0xffdbac,
    metalness: 0,
    roughness: 0.6,
  } as PBRMaterialConfig,

  fabric: {
    color: 0x4169e1,
    metalness: 0,
    roughness: 0.9,
  } as PBRMaterialConfig,
};
```

### 3. Texture Loader for PBR

```typescript
// materials/PBRTextureLoader.ts
import * as THREE from 'three';

export interface PBRTextureSet {
  albedo?: THREE.Texture;
  normal?: THREE.Texture;
  roughness?: THREE.Texture;
  metalness?: THREE.Texture;
  ao?: THREE.Texture;
  emissive?: THREE.Texture;
  height?: THREE.Texture;
}

export class PBRTextureLoader {
  private textureLoader = new THREE.TextureLoader();
  private loadedSets = new Map<string, PBRTextureSet>();

  async loadTextureSet(
    basePath: string,
    fileNames: {
      albedo?: string;
      normal?: string;
      roughness?: string;
      metalness?: string;
      ao?: string;
      emissive?: string;
      height?: string;
    }
  ): Promise<PBRTextureSet> {
    const promises: Promise<THREE.Texture | undefined>[] = [];
    const keys: (keyof PBRTextureSet)[] = [];

    for (const [key, fileName] of Object.entries(fileNames)) {
      if (fileName) {
        keys.push(key as keyof PBRTextureSet);
        promises.push(
          this.textureLoader.loadAsync(`${basePath}/${fileName}`).catch(() => undefined)
        );
      }
    }

    const textures = await Promise.all(promises);

    const textureSet: PBRTextureSet = {};

    textures.forEach((texture, index) => {
      if (texture) {
        const key = keys[index];

        // Configure texture
        texture.wrapS = THREE.RepeatWrapping;
        texture.wrapT = THREE.RepeatWrapping;

        // Special handling for normal maps
        if (key === 'normal') {
          texture.encoding = THREE.LinearEncoding;
        } else if (key === 'albedo' || key === 'emissive') {
          texture.encoding = THREE.sRGBEncoding;
        }

        textureSet[key] = texture;
      }
    });

    this.loadedSets.set(basePath, textureSet);
    return textureSet;
  }

  getTextureSet(basePath: string): PBRTextureSet | undefined {
    return this.loadedSets.get(basePath);
  }

  dispose(): void {
    this.loadedSets.forEach((set) => {
      Object.values(set).forEach((texture) => texture?.dispose());
    });
    this.loadedSets.clear();
  }
}
```

### 4. Material Variant System

```typescript
// materials/MaterialVariantSystem.ts
import * as THREE from 'three';

export interface MaterialVariant {
  name: string;
  properties: {
    color?: THREE.ColorRepresentation;
    metalness?: number;
    roughness?: number;
    emissive?: THREE.ColorRepresentation;
    emissiveIntensity?: number;
    [key: string]: any;
  };
}

export class MaterialVariantSystem {
  private baseMaterials = new Map<string, THREE.Material>();
  private variants = new Map<string, MaterialVariant[]>();

  registerBaseMaterial(id: string, material: THREE.Material): void {
    this.baseMaterials.set(id, material);
  }

  addVariant(baseMaterialId: string, variant: MaterialVariant): void {
    if (!this.variants.has(baseMaterialId)) {
      this.variants.set(baseMaterialId, []);
    }

    this.variants.get(baseMaterialId)!.push(variant);
  }

  createVariant(baseMaterialId: string, variantName: string): THREE.Material | null {
    const baseMaterial = this.baseMaterials.get(baseMaterialId);
    if (!baseMaterial) return null;

    const variants = this.variants.get(baseMaterialId);
    if (!variants) return null;

    const variant = variants.find((v) => v.name === variantName);
    if (!variant) return null;

    // Clone base material
    const variantMaterial = baseMaterial.clone();

    // Apply variant properties
    Object.entries(variant.properties).forEach(([key, value]) => {
      if (key in variantMaterial) {
        (variantMaterial as any)[key] = value;
      }
    });

    variantMaterial.needsUpdate = true;

    return variantMaterial;
  }

  getVariantNames(baseMaterialId: string): string[] {
    const variants = this.variants.get(baseMaterialId);
    return variants ? variants.map((v) => v.name) : [];
  }
}
```

### 5. Material Property Animator

```typescript
// materials/MaterialAnimator.ts
import * as THREE from 'three';

export interface MaterialAnimation {
  property: string;
  from: any;
  to: any;
  duration: number;
  easing?: (t: number) => number;
}

export class MaterialAnimator {
  private animations = new Map<THREE.Material, MaterialAnimation[]>();
  private timers = new Map<THREE.Material, number>();

  addAnimation(material: THREE.Material, animation: MaterialAnimation): void {
    if (!this.animations.has(material)) {
      this.animations.set(material, []);
      this.timers.set(material, 0);
    }

    this.animations.get(material)!.push(animation);
  }

  update(deltaTime: number): void {
    this.animations.forEach((animations, material) => {
      let timer = this.timers.get(material)!;
      timer += deltaTime;

      animations.forEach((anim) => {
        const t = Math.min(timer / anim.duration, 1);
        const easedT = anim.easing ? anim.easing(t) : t;

        // Interpolate based on type
        if (typeof anim.from === 'number') {
          (material as any)[anim.property] = THREE.MathUtils.lerp(
            anim.from,
            anim.to,
            easedT
          );
        } else if (anim.from instanceof THREE.Color) {
          (material as any)[anim.property] = new THREE.Color().lerpColors(
            anim.from,
            anim.to,
            easedT
          );
        } else if (anim.from instanceof THREE.Vector3) {
          (material as any)[anim.property] = new THREE.Vector3().lerpVectors(
            anim.from,
            anim.to,
            easedT
          );
        }

        material.needsUpdate = true;
      });

      if (timer >= Math.max(...animations.map((a) => a.duration))) {
        this.animations.delete(material);
        this.timers.delete(material);
      } else {
        this.timers.set(material, timer);
      }
    });
  }

  clear(material: THREE.Material): void {
    this.animations.delete(material);
    this.timers.delete(material);
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic PBR material
import { PBRMaterialManager } from './materials/PBRMaterialManager';
import { MaterialPresets } from './materials/MaterialPresets';

const materialManager = new PBRMaterialManager(renderer);

// Load environment map
await materialManager.loadEnvironmentMap('/textures/env.hdr');

// Create gold material
const goldMaterial = materialManager.createMaterial('gold', MaterialPresets.gold);

// Apply to mesh
mesh.material = goldMaterial;

// Example 2: PBR with textures
import { PBRTextureLoader } from './materials/PBRTextureLoader';

const textureLoader = new PBRTextureLoader();

const textures = await textureLoader.loadTextureSet('/textures/wood', {
  albedo: 'albedo.jpg',
  normal: 'normal.jpg',
  roughness: 'roughness.jpg',
  ao: 'ao.jpg',
});

const woodMaterial = materialManager.createMaterial('wood', {
  map: textures.albedo,
  normalMap: textures.normal,
  roughnessMap: textures.roughness,
  aoMap: textures.ao,
  metalness: 0,
});

// Example 3: Glass with transmission
const glassMaterial = materialManager.createPhysicalMaterial('glass', {
  color: 0xffffff,
  metalness: 0,
  roughness: 0,
  transmission: 1,
  thickness: 0.5,
  ior: 1.5,
});

// Example 4: Car paint with clear coat
const carPaintMaterial = materialManager.createPhysicalMaterial('carPaint', {
  color: 0xff0000,
  metalness: 0.9,
  roughness: 0.2,
  clearcoat: 1,
  clearcoatRoughness: 0.1,
});

// Example 5: Material variants
import { MaterialVariantSystem } from './materials/MaterialVariantSystem';

const variantSystem = new MaterialVariantSystem();
variantSystem.registerBaseMaterial('plastic', plasticBaseMaterial);

variantSystem.addVariant('plastic', {
  name: 'red',
  properties: { color: 0xff0000 },
});

variantSystem.addVariant('plastic', {
  name: 'blue',
  properties: { color: 0x0000ff },
});

const redPlastic = variantSystem.createVariant('plastic', 'red');

// Example 6: Material animation
import { MaterialAnimator } from './materials/MaterialAnimator';

const animator = new MaterialAnimator();

animator.addAnimation(material, {
  property: 'emissiveIntensity',
  from: 0,
  to: 1,
  duration: 2,
  easing: (t) => t * t, // Ease in quad
});

function animate() {
  animator.update(deltaTime);
}
```

## Checklist

- [ ] Install HDR loader for environment maps
- [ ] Load environment map (.hdr or .exr)
- [ ] Set up PMREMGenerator for env map processing
- [ ] Create PBR materials with metalness/roughness
- [ ] Load full PBR texture set (albedo, normal, roughness, metalness, AO)
- [ ] Configure texture encoding (sRGB for color, linear for data)
- [ ] Set envMapIntensity appropriately
- [ ] Use MeshPhysicalMaterial for advanced effects (transmission, clear coat)
- [ ] Test materials under different lighting conditions
- [ ] Optimize material count (share when possible)
- [ ] Set up proper tone mapping on renderer

## Common Pitfalls

1. **Wrong texture encoding**: Colors look washed out or too dark
2. **No environment map**: Materials look flat
3. **Wrong metalness values**: Non-metals with metalness > 0
4. **Too many unique materials**: Performance issues
5. **Not setting IOR**: Glass/water looks wrong
6. **Forgetting aoMapIntensity**: AO has no effect
7. **sRGB on normal maps**: Incorrect normals

## Performance Tips

### Material Optimization
- Share materials between objects when possible
- Use texture atlases to reduce material count
- Limit unique material count (<50 on mobile, <200 on desktop)
- Use simpler materials for distant objects
- Disable features you don't need (clearcoat, transmission)

### Texture Optimization
- Compress textures (KTX2, Basis)
- Use appropriate resolution (512-2048px)
- Combine maps when possible (roughness + metalness in one texture)
- Use lower resolution for AO and roughness
- Mipmap important textures

### Environment Map
- Use compressed HDR formats
- Lower resolution for mobile (512-1024px)
- Prefilter environment map offline when possible
- Share one environment map across scene

### Material Types (performance order)
1. MeshBasicMaterial (fastest, no lighting)
2. MeshLambertMaterial (simple diffuse)
3. MeshPhongMaterial (specular highlights)
4. MeshStandardMaterial (PBR, good balance)
5. MeshPhysicalMaterial (full PBR, most features, slowest)

## Related Skills

- `threejs-texture-management` - Texture loading and compression
- `threejs-lighting` - Lighting setup for PBR
- `threejs-material-systems` - General material management
- `mobile-performance` - Mobile optimization

## References

- Three.js PBR Materials: https://threejs.org/docs/#api/en/materials/MeshStandardMaterial
- PBR Theory: https://learnopengl.com/PBR/Theory
- Metalness/Roughness Workflow: https://marmoset.co/posts/basic-theory-of-physically-based-rendering/
