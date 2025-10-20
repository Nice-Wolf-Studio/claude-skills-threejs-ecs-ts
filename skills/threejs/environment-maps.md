---
name: threejs-environment-maps
description: Environment mapping and IBL (Image-Based Lighting) in Three.js including HDRI loading, skyboxes, reflection probes, and dynamic environment maps
---

# Three.js Environment Maps

## When to Use

Use this skill when:
- Creating realistic reflections on materials
- Implementing IBL (Image-Based Lighting)
- Adding skyboxes to scenes
- Creating dynamic reflections (mirrors, water)
- Setting up reflection probes for interiors
- Building photorealistic rendering

## Core Principles

1. **IBL for Realism**: Environment maps provide realistic lighting
2. **PMREM Processing**: Prefilter for PBR materials
3. **Resolution Balance**: Quality vs performance
4. **Cube vs Equirectangular**: Choose appropriate format
5. **Dynamic Updates**: Only when necessary
6. **Compression**: Use compressed formats for production

## Implementation

### 1. Environment Map Manager

```typescript
// environment/EnvironmentMapManager.ts
import * as THREE from 'three';
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader';
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader';

export interface EnvironmentConfig {
  url: string;
  type: 'hdr' | 'exr' | 'cube';
  background?: boolean;
  backgroundBlurriness?: number;
  backgroundIntensity?: number;
  environmentIntensity?: number;
}

export class EnvironmentMapManager {
  private renderer: THREE.WebGLRenderer;
  private pmremGenerator: THREE.PMREMGenerator;
  private scene: THREE.Scene;
  private currentEnvMap: THREE.Texture | null = null;

  constructor(renderer: THREE.WebGLRenderer, scene: THREE.Scene) {
    this.renderer = renderer;
    this.scene = scene;
    this.pmremGenerator = new THREE.PMREMGenerator(renderer);
    this.pmremGenerator.compileEquirectangularShader();
  }

  async loadHDRI(config: EnvironmentConfig): Promise<THREE.Texture> {
    let texture: THREE.Texture;

    if (config.type === 'hdr') {
      const loader = new RGBELoader();
      texture = await loader.loadAsync(config.url);
    } else if (config.type === 'exr') {
      const loader = new EXRLoader();
      texture = await loader.loadAsync(config.url);
    } else {
      throw new Error('Unsupported environment map type');
    }

    // Generate PMREM
    const envMap = this.pmremGenerator.fromEquirectangular(texture).texture;

    // Clean up original texture
    texture.dispose();

    // Store reference
    this.currentEnvMap = envMap;

    // Apply to scene
    this.scene.environment = envMap;

    if (config.background) {
      this.scene.background = envMap;

      if (config.backgroundBlurriness !== undefined) {
        this.scene.backgroundBlurriness = config.backgroundBlurriness;
      }

      if (config.backgroundIntensity !== undefined) {
        this.scene.backgroundIntensity = config.backgroundIntensity;
      }
    }

    if (config.environmentIntensity !== undefined) {
      this.scene.environmentIntensity = config.environmentIntensity;
    }

    return envMap;
  }

  async loadCubeMap(urls: string[]): Promise<THREE.CubeTexture> {
    if (urls.length !== 6) {
      throw new Error('Cube map requires 6 images');
    }

    const loader = new THREE.CubeTextureLoader();
    const cubeTexture = await loader.loadAsync(urls);

    // Generate PMREM for PBR
    const envMap = this.pmremGenerator.fromCubemap(cubeTexture).texture;

    this.currentEnvMap = envMap;
    this.scene.environment = envMap;
    this.scene.background = cubeTexture;

    return cubeTexture;
  }

  setEnvironmentIntensity(intensity: number): void {
    this.scene.environmentIntensity = intensity;
  }

  setBackgroundBlurriness(blurriness: number): void {
    this.scene.backgroundBlurriness = blurriness;
  }

  setBackgroundIntensity(intensity: number): void {
    this.scene.backgroundIntensity = intensity;
  }

  removeEnvironment(): void {
    this.scene.environment = null;
    this.scene.background = null;

    if (this.currentEnvMap) {
      this.currentEnvMap.dispose();
      this.currentEnvMap = null;
    }
  }

  dispose(): void {
    this.pmremGenerator.dispose();

    if (this.currentEnvMap) {
      this.currentEnvMap.dispose();
    }
  }
}
```

### 2. Dynamic Environment Map (Reflection Probes)

```typescript
// environment/ReflectionProbe.ts
import * as THREE from 'three';

export class ReflectionProbe {
  private cubeCamera: THREE.CubeCamera;
  private renderTarget: THREE.WebGLCubeRenderTarget;
  private position: THREE.Vector3;
  private scene: THREE.Scene;
  private excludeObjects: THREE.Object3D[] = [];

  constructor(
    scene: THREE.Scene,
    renderer: THREE.WebGLRenderer,
    position: THREE.Vector3 = new THREE.Vector3(),
    resolution: number = 256
  ) {
    this.scene = scene;
    this.position = position;

    // Create render target
    this.renderTarget = new THREE.WebGLCubeRenderTarget(resolution, {
      format: THREE.RGBAFormat,
      generateMipmaps: true,
      minFilter: THREE.LinearMipmapLinearFilter,
    });

    // Create cube camera
    this.cubeCamera = new THREE.CubeCamera(0.1, 1000, this.renderTarget);
    this.cubeCamera.position.copy(position);
  }

  update(renderer: THREE.WebGLRenderer): void {
    // Hide objects that shouldn't be reflected
    const originalVisibility = new Map<THREE.Object3D, boolean>();

    this.excludeObjects.forEach((obj) => {
      originalVisibility.set(obj, obj.visible);
      obj.visible = false;
    });

    // Update cube camera
    this.cubeCamera.update(renderer, this.scene);

    // Restore visibility
    originalVisibility.forEach((visible, obj) => {
      obj.visible = visible;
    });
  }

  getTexture(): THREE.Texture {
    return this.renderTarget.texture;
  }

  setPosition(position: THREE.Vector3): void {
    this.position.copy(position);
    this.cubeCamera.position.copy(position);
  }

  excludeObject(object: THREE.Object3D): void {
    this.excludeObjects.push(object);
  }

  dispose(): void {
    this.renderTarget.dispose();
  }
}
```

### 3. Skybox Builder

```typescript
// environment/SkyboxBuilder.ts
import * as THREE from 'three';

export type SkyboxType = 'gradient' | 'procedural' | 'textured';

export class SkyboxBuilder {
  private scene: THREE.Scene;
  private skybox: THREE.Mesh | null = null;

  constructor(scene: THREE.Scene) {
    this.scene = scene;
  }

  createGradientSky(topColor: THREE.Color, bottomColor: THREE.Color): THREE.Mesh {
    const geometry = new THREE.SphereGeometry(500, 32, 15);

    const shader = {
      uniforms: {
        topColor: { value: topColor },
        bottomColor: { value: bottomColor },
        offset: { value: 0 },
        exponent: { value: 0.6 },
      },
      vertexShader: `
        varying vec3 vWorldPosition;
        void main() {
          vec4 worldPosition = modelMatrix * vec4(position, 1.0);
          vWorldPosition = worldPosition.xyz;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform vec3 topColor;
        uniform vec3 bottomColor;
        uniform float offset;
        uniform float exponent;
        varying vec3 vWorldPosition;
        void main() {
          float h = normalize(vWorldPosition + offset).y;
          gl_FragColor = vec4(mix(bottomColor, topColor, max(pow(max(h, 0.0), exponent), 0.0)), 1.0);
        }
      `,
      side: THREE.BackSide,
    };

    const material = new THREE.ShaderMaterial(shader);
    this.skybox = new THREE.Mesh(geometry, material);

    this.scene.add(this.skybox);
    return this.skybox;
  }

  createProceduralSky(): THREE.Mesh {
    const geometry = new THREE.SphereGeometry(500, 32, 15);

    const shader = {
      uniforms: {
        sunPosition: { value: new THREE.Vector3(0, 1, 0) },
        turbidity: { value: 10 },
        rayleigh: { value: 2 },
        mieCoefficient: { value: 0.005 },
        mieDirectionalG: { value: 0.8 },
      },
      vertexShader: `
        varying vec3 vWorldPosition;
        void main() {
          vec4 worldPosition = modelMatrix * vec4(position, 1.0);
          vWorldPosition = worldPosition.xyz;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform vec3 sunPosition;
        uniform float turbidity;
        uniform float rayleigh;
        uniform float mieCoefficient;
        uniform float mieDirectionalG;

        varying vec3 vWorldPosition;

        const vec3 up = vec3(0.0, 1.0, 0.0);

        vec3 totalRayleigh(vec3 lambda) {
          return (8.0 * pow(3.14159, 3.0) * pow(pow(1.00029, 2.0) - 1.0, 2.0) * (6.0 + 3.0 * 0.035)) /
                 (3.0 * 2.545e25 * pow(lambda, vec3(4.0))) * (1.0 + 0.035);
        }

        void main() {
          vec3 direction = normalize(vWorldPosition);
          float zenithAngle = acos(max(0.0, dot(up, direction)));

          vec3 lambda = vec3(680e-9, 550e-9, 450e-9);
          vec3 rayleighCoefficient = totalRayleigh(lambda);

          vec3 skyColor = vec3(0.5, 0.7, 1.0) * rayleigh;
          gl_FragColor = vec4(skyColor, 1.0);
        }
      `,
      side: THREE.BackSide,
    };

    const material = new THREE.ShaderMaterial(shader);
    this.skybox = new THREE.Mesh(geometry, material);

    this.scene.add(this.skybox);
    return this.skybox;
  }

  async createTexturedSky(urls: string[]): Promise<THREE.Mesh> {
    const loader = new THREE.CubeTextureLoader();
    const texture = await loader.loadAsync(urls);

    const geometry = new THREE.BoxGeometry(500, 500, 500);
    const material = new THREE.MeshBasicMaterial({
      envMap: texture,
      side: THREE.BackSide,
    });

    this.skybox = new THREE.Mesh(geometry, material);
    this.scene.add(this.skybox);

    return this.skybox;
  }

  updateGradientColors(topColor: THREE.Color, bottomColor: THREE.Color): void {
    if (this.skybox && this.skybox.material instanceof THREE.ShaderMaterial) {
      this.skybox.material.uniforms.topColor.value = topColor;
      this.skybox.material.uniforms.bottomColor.value = bottomColor;
    }
  }

  remove(): void {
    if (this.skybox) {
      this.scene.remove(this.skybox);
      this.skybox.geometry.dispose();

      if (Array.isArray(this.skybox.material)) {
        this.skybox.material.forEach((mat) => mat.dispose());
      } else {
        this.skybox.material.dispose();
      }

      this.skybox = null;
    }
  }
}
```

### 4. Environment Preset Library

```typescript
// environment/EnvironmentPresets.ts
export const EnvironmentPresets = {
  studio: {
    url: '/textures/env/studio.hdr',
    type: 'hdr' as const,
    background: true,
    backgroundBlurriness: 0.5,
    environmentIntensity: 1.0,
  },

  sunset: {
    url: '/textures/env/sunset.hdr',
    type: 'hdr' as const,
    background: true,
    backgroundBlurriness: 0.0,
    environmentIntensity: 1.2,
  },

  warehouse: {
    url: '/textures/env/warehouse.hdr',
    type: 'hdr' as const,
    background: true,
    backgroundBlurriness: 0.3,
    environmentIntensity: 0.8,
  },

  forest: {
    url: '/textures/env/forest.hdr',
    type: 'hdr' as const,
    background: true,
    backgroundBlurriness: 0.0,
    environmentIntensity: 1.0,
  },

  night: {
    url: '/textures/env/night.hdr',
    type: 'hdr' as const,
    background: true,
    backgroundBlurriness: 0.0,
    environmentIntensity: 0.3,
  },

  apartment: {
    url: '/textures/env/apartment.exr',
    type: 'exr' as const,
    background: false,
    environmentIntensity: 1.0,
  },
};
```

### 5. Real-time Reflection System

```typescript
// environment/ReflectionSystem.ts
import * as THREE from 'three';
import { ReflectionProbe } from './ReflectionProbe';

export class ReflectionSystem {
  private probes = new Map<string, ReflectionProbe>();
  private updateQueue: string[] = [];
  private updateInterval: number;
  private updateTimer = 0;

  constructor(private scene: THREE.Scene, updateIntervalSeconds: number = 0.1) {
    this.updateInterval = updateIntervalSeconds;
  }

  addProbe(
    id: string,
    renderer: THREE.WebGLRenderer,
    position: THREE.Vector3,
    resolution: number = 256
  ): ReflectionProbe {
    const probe = new ReflectionProbe(this.scene, renderer, position, resolution);
    this.probes.set(id, probe);
    this.updateQueue.push(id);
    return probe;
  }

  getProbe(id: string): ReflectionProbe | undefined {
    return this.probes.get(id);
  }

  update(renderer: THREE.WebGLRenderer, deltaTime: number): void {
    this.updateTimer += deltaTime;

    if (this.updateTimer >= this.updateInterval && this.updateQueue.length > 0) {
      // Update one probe per interval
      const probeId = this.updateQueue.shift()!;
      const probe = this.probes.get(probeId);

      if (probe) {
        probe.update(renderer);
      }

      // Re-add to queue for next update
      this.updateQueue.push(probeId);

      this.updateTimer = 0;
    }
  }

  removeProbe(id: string): void {
    const probe = this.probes.get(id);
    if (probe) {
      probe.dispose();
      this.probes.delete(id);

      // Remove from queue
      const index = this.updateQueue.indexOf(id);
      if (index !== -1) {
        this.updateQueue.splice(index, 1);
      }
    }
  }

  dispose(): void {
    this.probes.forEach((probe) => probe.dispose());
    this.probes.clear();
    this.updateQueue = [];
  }
}
```

## Usage Examples

```typescript
// Example 1: Load HDRI environment
import { EnvironmentMapManager } from './environment/EnvironmentMapManager';

const envManager = new EnvironmentMapManager(renderer, scene);

await envManager.loadHDRI({
  url: '/textures/env/studio.hdr',
  type: 'hdr',
  background: true,
  backgroundBlurriness: 0.5,
  environmentIntensity: 1.0,
});

// Example 2: Load cube map
const cubeUrls = [
  '/textures/skybox/px.jpg', // positive x
  '/textures/skybox/nx.jpg', // negative x
  '/textures/skybox/py.jpg', // positive y
  '/textures/skybox/ny.jpg', // negative y
  '/textures/skybox/pz.jpg', // positive z
  '/textures/skybox/nz.jpg', // negative z
];

await envManager.loadCubeMap(cubeUrls);

// Example 3: Dynamic reflections (mirror, water)
import { ReflectionProbe } from './environment/ReflectionProbe';

const mirrorProbe = new ReflectionProbe(scene, renderer, new THREE.Vector3(0, 1, 0), 512);

// Exclude the mirror itself from reflection
mirrorProbe.excludeObject(mirrorMesh);

function animate() {
  // Update probe (expensive, do sparingly)
  if (needsUpdate) {
    mirrorProbe.update(renderer);
  }

  // Apply to mirror material
  mirrorMaterial.envMap = mirrorProbe.getTexture();
}

// Example 4: Gradient skybox
import { SkyboxBuilder } from './environment/SkyboxBuilder';

const skyboxBuilder = new SkyboxBuilder(scene);

skyboxBuilder.createGradientSky(
  new THREE.Color(0x87ceeb), // Top color
  new THREE.Color(0xffffff)  // Bottom color
);

// Example 5: Reflection system for multiple probes
import { ReflectionSystem } from './environment/ReflectionSystem';

const reflectionSystem = new ReflectionSystem(scene, 0.1);

// Add probes at strategic locations
reflectionSystem.addProbe('lobby', renderer, new THREE.Vector3(0, 2, 0), 256);
reflectionSystem.addProbe('hallway', renderer, new THREE.Vector3(20, 2, 0), 256);

function animate() {
  // Update one probe per frame (round-robin)
  reflectionSystem.update(renderer, deltaTime);
}

// Example 6: Using presets
import { EnvironmentPresets } from './environment/EnvironmentPresets';

await envManager.loadHDRI(EnvironmentPresets.sunset);

// Later, change environment
await envManager.loadHDRI(EnvironmentPresets.warehouse);
```

## Checklist

- [ ] Choose environment map format (HDR, EXR, or cube)
- [ ] Load environment map with appropriate loader
- [ ] Generate PMREM for PBR materials
- [ ] Apply to scene.environment for IBL
- [ ] Optionally set as scene.background
- [ ] Configure backgroundBlurriness for softer backgrounds
- [ ] Set appropriate environmentIntensity
- [ ] Test reflections on metallic materials
- [ ] Optimize resolution (256-1024 for most cases)
- [ ] Compress environment maps for production
- [ ] Implement dynamic reflections only where needed
- [ ] Profile environment map updates

## Common Pitfalls

1. **Not using PMREM**: Materials look incorrect
2. **Too high resolution**: Performance issues
3. **Updating every frame**: Dynamic reflections are expensive
4. **Wrong format**: Use HDR/EXR for IBL, not JPG/PNG
5. **Mismatched intensity**: Environment too bright/dark
6. **Not excluding objects**: Self-reflection artifacts
7. **No compression**: Large file sizes

## Performance Tips

### Environment Map Resolution
- Mobile: 256-512px
- Desktop: 512-1024px
- High-end: 1024-2048px
- Rarely need >2048px

### Format Selection
- **HDR (.hdr)**: Best for IBL, good compression
- **EXR (.exr)**: Higher quality, larger files
- **Cube maps (6 images)**: Good browser support, easier to author

### Dynamic Reflections
- Update only when needed (not every frame)
- Use lower resolution (256px often sufficient)
- Round-robin update multiple probes
- Skip updates when off-screen
- Cache results when possible

### Optimization Strategies
- Precompute PMREM offline when possible
- Use basis/KTX2 compression for environment maps
- Share environment maps across scenes
- Disable dynamic reflections on mobile
- Use screen-space reflections for planar surfaces

### Mobile Considerations
- Limit to 256-512px resolution
- Avoid dynamic reflections entirely
- Use simpler skyboxes (gradient instead of textured)
- Lower environmentIntensity
- Consider disabling reflections on low-end devices

## Related Skills

- `threejs-pbr-materials` - Materials that use environment maps
- `threejs-lighting` - Lighting integration
- `threejs-texture-management` - Texture optimization
- `mobile-performance` - Mobile optimization

## References

- Three.js PMREMGenerator: https://threejs.org/docs/#api/en/extras/PMREMGenerator
- Three.js Environment: https://threejs.org/docs/#api/en/scenes/Scene.environment
- IBL Theory: https://learnopengl.com/PBR/IBL/Diffuse-irradiance
- HDRI Haven (free HDRIs): https://hdri-haven.com/
