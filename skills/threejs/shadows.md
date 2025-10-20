---
name: threejs-shadows
description: Shadow configuration and optimization for Three.js including shadow maps, CSM, PCFSS, and mobile shadow strategies
---

# Three.js Shadows

## When to Use

Use this skill when:
- Implementing realistic shadows
- Optimizing shadow performance
- Setting up shadow casting/receiving
- Fixing shadow artifacts
- Implementing dynamic shadows
- Mobile shadow optimization

## Core Principles

1. **Selective Shadows**: Not everything needs to cast shadows
2. **Appropriate Shadow Maps**: Size based on importance
3. **CSM for Terrain**: Cascaded Shadow Maps for large areas
4. **Soft Shadows**: Use PCF or PCFSS for realism
5. **Mobile Optimization**: Reduce quality on low-end devices
6. **Static Baking**: Bake shadows when possible

## Implementation

### 1. Shadow Manager

```typescript
// shadows/ShadowManager.ts
import * as THREE from 'three';

export interface ShadowConfig {
  enabled?: boolean;
  type?: THREE.ShadowMapType;
  autoUpdate?: boolean;
  needsUpdate?: boolean;
}

export interface LightShadowConfig {
  mapSize?: number;
  camera?: {
    near?: number;
    far?: number;
    left?: number;
    right?: number;
    top?: number;
    bottom?: number;
    fov?: number;
  };
  bias?: number;
  normalBias?: number;
  radius?: number;
}

export class ShadowManager {
  private renderer: THREE.WebGLRenderer;
  private scene: THREE.Scene;

  constructor(renderer: THREE.WebGLRenderer, scene: THREE.Scene) {
    this.renderer = renderer;
    this.scene = scene;
  }

  enable(config: ShadowConfig = {}): void {
    this.renderer.shadowMap.enabled = config.enabled ?? true;
    this.renderer.shadowMap.type = config.type ?? THREE.PCFSoftShadowMap;
    this.renderer.shadowMap.autoUpdate = config.autoUpdate ?? true;
    this.renderer.shadowMap.needsUpdate = config.needsUpdate ?? true;
  }

  disable(): void {
    this.renderer.shadowMap.enabled = false;
  }

  configureLightShadow(light: THREE.Light, config: LightShadowConfig = {}): void {
    if (!light.shadow) return;

    // Shadow map size
    const mapSize = config.mapSize ?? 2048;
    light.shadow.mapSize.width = mapSize;
    light.shadow.mapSize.height = mapSize;

    // Shadow camera
    if (light.shadow.camera) {
      const camera = light.shadow.camera;

      if (config.camera) {
        if ('near' in config.camera) camera.near = config.camera.near!;
        if ('far' in config.camera) camera.far = config.camera.far!;

        if (camera instanceof THREE.OrthographicCamera) {
          if ('left' in config.camera) camera.left = config.camera.left!;
          if ('right' in config.camera) camera.right = config.camera.right!;
          if ('top' in config.camera) camera.top = config.camera.top!;
          if ('bottom' in config.camera) camera.bottom = config.camera.bottom!;
        } else if (camera instanceof THREE.PerspectiveCamera) {
          if ('fov' in config.camera) camera.fov = config.camera.fov!;
        }

        camera.updateProjectionMatrix();
      }
    }

    // Bias
    if (config.bias !== undefined) {
      light.shadow.bias = config.bias;
    }

    if (config.normalBias !== undefined) {
      light.shadow.normalBias = config.normalBias;
    }

    if (config.radius !== undefined) {
      light.shadow.radius = config.radius;
    }

    light.shadow.needsUpdate = true;
  }

  configureDirectionalLight(
    light: THREE.DirectionalLight,
    config: LightShadowConfig = {}
  ): void {
    light.castShadow = true;

    this.configureLightShadow(light, {
      mapSize: config.mapSize ?? 2048,
      camera: {
        near: config.camera?.near ?? 0.5,
        far: config.camera?.far ?? 500,
        left: config.camera?.left ?? -50,
        right: config.camera?.right ?? 50,
        top: config.camera?.top ?? 50,
        bottom: config.camera?.bottom ?? -50,
      },
      bias: config.bias ?? -0.0001,
      normalBias: config.normalBias ?? 0.02,
      ...config,
    });
  }

  configureSpotLight(light: THREE.SpotLight, config: LightShadowConfig = {}): void {
    light.castShadow = true;

    this.configureLightShadow(light, {
      mapSize: config.mapSize ?? 1024,
      camera: {
        near: config.camera?.near ?? 0.5,
        far: config.camera?.far ?? 500,
        fov: config.camera?.fov ?? 50,
      },
      bias: config.bias ?? -0.0001,
      normalBias: config.normalBias ?? 0.02,
      ...config,
    });
  }

  configurePointLight(light: THREE.PointLight, config: LightShadowConfig = {}): void {
    light.castShadow = true;

    this.configureLightShadow(light, {
      mapSize: config.mapSize ?? 512,
      camera: {
        near: config.camera?.near ?? 0.5,
        far: config.camera?.far ?? 500,
      },
      bias: config.bias ?? -0.0001,
      normalBias: config.normalBias ?? 0.02,
      ...config,
    });
  }

  enableCastShadow(object: THREE.Object3D, recursive = true): void {
    object.castShadow = true;

    if (recursive) {
      object.traverse((child) => {
        if (child instanceof THREE.Mesh) {
          child.castShadow = true;
        }
      });
    }
  }

  enableReceiveShadow(object: THREE.Object3D, recursive = true): void {
    object.receiveShadow = true;

    if (recursive) {
      object.traverse((child) => {
        if (child instanceof THREE.Mesh) {
          child.receiveShadow = true;
        }
      });
    }
  }

  visualizeShadowCamera(light: THREE.Light): THREE.CameraHelper | null {
    if (!light.shadow || !light.shadow.camera) return null;

    const helper = new THREE.CameraHelper(light.shadow.camera);
    this.scene.add(helper);

    return helper;
  }

  optimizeForMobile(): void {
    // Reduce shadow map sizes
    this.scene.traverse((object) => {
      if (object instanceof THREE.Light && object.shadow) {
        object.shadow.mapSize.width = 512;
        object.shadow.mapSize.height = 512;
      }
    });

    // Use basic shadow map for better performance
    this.renderer.shadowMap.type = THREE.BasicShadowMap;
  }
}
```

### 2. Cascaded Shadow Maps (CSM)

```typescript
// shadows/CSMShadowManager.ts
import * as THREE from 'three';
import { CSM } from 'three/examples/jsm/csm/CSM';
import { CSMParameters } from 'three/examples/jsm/csm/CSMShader';

export class CSMShadowManager {
  private csm: CSM;
  private scene: THREE.Scene;
  private camera: THREE.Camera;

  constructor(
    scene: THREE.Scene,
    camera: THREE.Camera,
    renderer: THREE.WebGLRenderer,
    config: Partial<CSMParameters> = {}
  ) {
    this.scene = scene;
    this.camera = camera;

    this.csm = new CSM({
      maxFar: config.maxFar ?? 200,
      cascades: config.cascades ?? 4,
      mode: config.mode ?? 'practical',
      parent: scene,
      shadowMapSize: config.shadowMapSize ?? 2048,
      lightDirection: config.lightDirection ?? new THREE.Vector3(1, -1, 1).normalize(),
      camera: camera,
      ...config,
    });
  }

  update(): void {
    this.csm.update();
  }

  setLightDirection(direction: THREE.Vector3): void {
    this.csm.lightDirection.copy(direction);
  }

  setupMaterial(material: THREE.Material): void {
    this.csm.setupMaterial(material);
  }

  dispose(): void {
    this.csm.dispose();
  }

  get shadowMap(): THREE.DepthTexture[] {
    return this.csm.shadowMaps;
  }
}
```

### 3. Contact Shadows

```typescript
// shadows/ContactShadows.ts
import * as THREE from 'three';

export interface ContactShadowsConfig {
  opacity?: number;
  width?: number;
  height?: number;
  blur?: number;
  far?: number;
  resolution?: number;
}

export class ContactShadows {
  private scene: THREE.Scene;
  private camera: THREE.OrthographicCamera;
  private renderTarget: THREE.WebGLRenderTarget;
  private depthMaterial: THREE.MeshDepthMaterial;
  private blurPlane: THREE.Mesh;
  private shadowGroup: THREE.Group;

  constructor(scene: THREE.Scene, config: ContactShadowsConfig = {}) {
    this.scene = scene;

    const width = config.width ?? 10;
    const height = config.height ?? 10;
    const resolution = config.resolution ?? 512;

    // Shadow camera
    this.camera = new THREE.OrthographicCamera(
      -width / 2,
      width / 2,
      height / 2,
      -height / 2,
      0,
      config.far ?? 10
    );

    // Render target
    this.renderTarget = new THREE.WebGLRenderTarget(resolution, resolution);

    // Depth material
    this.depthMaterial = new THREE.MeshDepthMaterial();
    this.depthMaterial.depthPacking = THREE.RGBADepthPacking;
    this.depthMaterial.blending = THREE.NoBlending;

    // Shadow plane
    const shadowGeometry = new THREE.PlaneGeometry(width, height);
    const shadowMaterial = new THREE.MeshBasicMaterial({
      map: this.renderTarget.texture,
      opacity: config.opacity ?? 0.5,
      transparent: true,
      depthWrite: false,
    });

    this.blurPlane = new THREE.Mesh(shadowGeometry, shadowMaterial);
    this.blurPlane.rotation.x = -Math.PI / 2;

    this.shadowGroup = new THREE.Group();
    this.shadowGroup.add(this.blurPlane);
    this.scene.add(this.shadowGroup);
  }

  update(renderer: THREE.WebGLRenderer, objectsToCast: THREE.Object3D[]): void {
    // Store original materials
    const originalMaterials = new Map<THREE.Mesh, THREE.Material | THREE.Material[]>();

    objectsToCast.forEach((obj) => {
      obj.traverse((child) => {
        if (child instanceof THREE.Mesh) {
          originalMaterials.set(child, child.material);
          child.material = this.depthMaterial;
        }
      });
    });

    // Render depth
    const originalRenderTarget = renderer.getRenderTarget();
    renderer.setRenderTarget(this.renderTarget);
    renderer.render(this.scene, this.camera);
    renderer.setRenderTarget(originalRenderTarget);

    // Restore materials
    originalMaterials.forEach((material, mesh) => {
      mesh.material = material;
    });
  }

  setPosition(position: THREE.Vector3): void {
    this.shadowGroup.position.copy(position);
    this.camera.position.copy(position);
    this.camera.position.y += 5;
    this.camera.lookAt(position);
  }

  dispose(): void {
    this.renderTarget.dispose();
    this.depthMaterial.dispose();
    this.blurPlane.geometry.dispose();
    (this.blurPlane.material as THREE.Material).dispose();
    this.scene.remove(this.shadowGroup);
  }
}
```

### 4. Adaptive Shadow Quality

```typescript
// shadows/AdaptiveShadowQuality.ts
import * as THREE from 'three';

export type ShadowQuality = 'low' | 'medium' | 'high' | 'ultra';

export class AdaptiveShadowQuality {
  private renderer: THREE.WebGLRenderer;
  private lights: THREE.Light[] = [];

  private qualitySettings = {
    low: {
      shadowMapType: THREE.BasicShadowMap,
      mapSize: 512,
      enabled: true,
      bias: -0.001,
    },
    medium: {
      shadowMapType: THREE.PCFShadowMap,
      mapSize: 1024,
      enabled: true,
      bias: -0.0005,
    },
    high: {
      shadowMapType: THREE.PCFSoftShadowMap,
      mapSize: 2048,
      enabled: true,
      bias: -0.0001,
    },
    ultra: {
      shadowMapType: THREE.VSMShadowMap,
      mapSize: 4096,
      enabled: true,
      bias: -0.00005,
    },
  };

  constructor(renderer: THREE.WebGLRenderer) {
    this.renderer = renderer;
  }

  registerLight(light: THREE.Light): void {
    this.lights.push(light);
  }

  setQuality(quality: ShadowQuality): void {
    const settings = this.qualitySettings[quality];

    this.renderer.shadowMap.enabled = settings.enabled;
    this.renderer.shadowMap.type = settings.shadowMapType;

    this.lights.forEach((light) => {
      if (light.shadow) {
        light.shadow.mapSize.width = settings.mapSize;
        light.shadow.mapSize.height = settings.mapSize;
        light.shadow.bias = settings.bias;
        light.shadow.needsUpdate = true;
      }
    });
  }

  autoDetectQuality(): ShadowQuality {
    // Detect based on device capabilities
    const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
    const gpuTier = this.detectGPUTier();

    if (isMobile) {
      return gpuTier === 'high' ? 'medium' : 'low';
    }

    switch (gpuTier) {
      case 'low':
        return 'medium';
      case 'medium':
        return 'high';
      case 'high':
        return 'ultra';
      default:
        return 'medium';
    }
  }

  private detectGPUTier(): 'low' | 'medium' | 'high' {
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

    if (!gl) return 'low';

    const debugInfo = (gl as WebGLRenderingContext).getExtension('WEBGL_debug_renderer_info');
    if (!debugInfo) return 'medium';

    const renderer = (gl as WebGLRenderingContext).getParameter(
      debugInfo.UNMASKED_RENDERER_WEBGL
    );

    if (/Mali-4|Adreno \(TM\) 3|PowerVR SGX/i.test(renderer)) {
      return 'low';
    } else if (/Apple A1[2-9]|Adreno \(TM\) [67]|Mali-G7|RTX|GTX/i.test(renderer)) {
      return 'high';
    }

    return 'medium';
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic shadow setup
import { ShadowManager } from './shadows/ShadowManager';

const shadowManager = new ShadowManager(renderer, scene);

// Enable shadows
shadowManager.enable({
  type: THREE.PCFSoftShadowMap,
  autoUpdate: true,
});

// Configure directional light
const sunLight = new THREE.DirectionalLight(0xffffff, 1);
shadowManager.configureDirectionalLight(sunLight, {
  mapSize: 2048,
  camera: {
    left: -50,
    right: 50,
    top: 50,
    bottom: -50,
    near: 0.5,
    far: 500,
  },
  bias: -0.0001,
});

// Enable shadows on objects
const player = scene.getObjectByName('player');
shadowManager.enableCastShadow(player, true);

const ground = scene.getObjectByName('ground');
shadowManager.enableReceiveShadow(ground, false);

// Example 2: CSM for large terrain
import { CSMShadowManager } from './shadows/CSMShadowManager';

const csmManager = new CSMShadowManager(scene, camera, renderer, {
  cascades: 4,
  maxFar: 200,
  shadowMapSize: 2048,
  lightDirection: new THREE.Vector3(1, -1, 1).normalize(),
});

// Setup materials
terrain.traverse((child) => {
  if (child instanceof THREE.Mesh) {
    csmManager.setupMaterial(child.material);
  }
});

function animate() {
  csmManager.update();
  renderer.render(scene, camera);
}

// Example 3: Adaptive quality
import { AdaptiveShadowQuality } from './shadows/AdaptiveShadowQuality';

const adaptiveQuality = new AdaptiveShadowQuality(renderer);
adaptiveQuality.registerLight(sunLight);
adaptiveQuality.registerLight(spotLight);

// Auto-detect and apply
const quality = adaptiveQuality.autoDetectQuality();
adaptiveQuality.setQuality(quality);

// Or manual control
adaptiveQuality.setQuality('medium');

// Example 4: Contact shadows
import { ContactShadows } from './shadows/ContactShadows';

const contactShadows = new ContactShadows(scene, {
  opacity: 0.5,
  width: 20,
  height: 20,
  blur: 1,
  far: 10,
  resolution: 512,
});

function animate() {
  contactShadows.update(renderer, [player, enemy1, enemy2]);
  renderer.render(scene, camera);
}
```

## Checklist

- [ ] Enable shadow maps on renderer
- [ ] Configure shadow casting lights
- [ ] Set appropriate shadow map sizes (512-2048)
- [ ] Enable castShadow on dynamic objects
- [ ] Enable receiveShadow on ground/walls
- [ ] Adjust shadow camera frustum to fit scene
- [ ] Set shadow bias to prevent artifacts
- [ ] Test shadow quality on target devices
- [ ] Use CSM for large outdoor scenes
- [ ] Implement adaptive quality for mobile
- [ ] Profile shadow performance
- [ ] Optimize shadow map sizes based on distance

## Common Pitfalls

1. **Shadow acne**: Increase bias or normalBias
2. **Peter panning**: Decrease bias
3. **Low resolution shadows**: Increase mapSize
4. **Shadows cut off**: Adjust shadow camera frustum
5. **Too many shadow maps**: Use fewer lights with shadows
6. **Not updating shadow maps**: Set needsUpdate = true
7. **High map sizes on mobile**: Causes memory/performance issues

## Performance Tips

### Map Sizes
- Distant lights: 512x512
- Medium importance: 1024x1024
- Main light: 2048x2048
- Never exceed 4096x4096

### Shadow Types (performance order)
1. BasicShadowMap (fastest, lowest quality)
2. PCFShadowMap (good balance)
3. PCFSoftShadowMap (better quality, slower)
4. VSMShadowMap (best quality, slowest)

### Optimization Strategies
- Limit shadow-casting lights to 1-3
- Use directional lights (1 shadow map) over point lights (6 shadow maps)
- Disable shadows on small/distant objects
- Use contact shadows for subtle ground shadows
- Bake static shadows into lightmaps
- Use CSM only for large terrains
- Update shadow maps selectively (not every frame)
- Reduce shadow camera far plane
- Use lower resolution for point/spot lights

### Mobile Specific
- Use BasicShadowMap or PCFShadowMap
- Max 512-1024 shadow map size
- Limit to 1-2 shadow-casting lights
- Consider disabling shadows entirely on low-end devices

## Related Skills

- `threejs-lighting` - Light setup
- `threejs-scene-setup` - Renderer configuration
- `mobile-performance` - Mobile optimization
- `threejs-material-systems` - Material shadow response
