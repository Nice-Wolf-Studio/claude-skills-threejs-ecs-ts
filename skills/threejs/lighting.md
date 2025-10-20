---
name: threejs-lighting
description: Comprehensive lighting systems for Three.js including ambient, directional, point, spot, and hemisphere lights with optimization strategies
---

# Three.js Lighting

## When to Use

Use this skill when:
- Setting up scene lighting
- Creating realistic lighting environments
- Implementing dynamic lighting
- Optimizing light performance
- Creating day/night cycles
- Implementing volumetric lighting

## Core Principles

1. **Layered Lighting**: Combine multiple light types
2. **Performance**: Limit active lights (<8 for mobile)
3. **Realism**: Use physically-based lighting values
4. **Optimization**: Bake static lights when possible
5. **Color Temperature**: Use realistic color temperatures
6. **Intensity**: Scale with inverse square law for realism

## Implementation

### 1. Light Manager

```typescript
// lights/LightManager.ts
import * as THREE from 'three';

export interface LightConfig {
  color?: number | string;
  intensity?: number;
  position?: THREE.Vector3;
  target?: THREE.Vector3;
  castShadow?: boolean;
}

export class LightManager {
  private scene: THREE.Scene;
  private lights = new Map<string, THREE.Light>();

  constructor(scene: THREE.Scene) {
    this.scene = scene;
  }

  addAmbientLight(id: string, config: LightConfig = {}): THREE.AmbientLight {
    const light = new THREE.AmbientLight(
      config.color ?? 0xffffff,
      config.intensity ?? 0.5
    );

    this.lights.set(id, light);
    this.scene.add(light);

    return light;
  }

  addDirectionalLight(id: string, config: LightConfig = {}): THREE.DirectionalLight {
    const light = new THREE.DirectionalLight(
      config.color ?? 0xffffff,
      config.intensity ?? 1
    );

    if (config.position) {
      light.position.copy(config.position);
    }

    if (config.target) {
      light.target.position.copy(config.target);
      this.scene.add(light.target);
    }

    if (config.castShadow) {
      light.castShadow = true;
      light.shadow.mapSize.width = 2048;
      light.shadow.mapSize.height = 2048;
      light.shadow.camera.near = 0.5;
      light.shadow.camera.far = 500;
      light.shadow.camera.left = -50;
      light.shadow.camera.right = 50;
      light.shadow.camera.top = 50;
      light.shadow.camera.bottom = -50;
    }

    this.lights.set(id, light);
    this.scene.add(light);

    return light;
  }

  addPointLight(id: string, config: LightConfig & { distance?: number; decay?: number } = {}): THREE.PointLight {
    const light = new THREE.PointLight(
      config.color ?? 0xffffff,
      config.intensity ?? 1,
      config.distance ?? 0,
      config.decay ?? 2
    );

    if (config.position) {
      light.position.copy(config.position);
    }

    if (config.castShadow) {
      light.castShadow = true;
      light.shadow.mapSize.width = 1024;
      light.shadow.mapSize.height = 1024;
      light.shadow.camera.near = 0.5;
      light.shadow.camera.far = 500;
    }

    this.lights.set(id, light);
    this.scene.add(light);

    return light;
  }

  addSpotLight(
    id: string,
    config: LightConfig & {
      distance?: number;
      angle?: number;
      penumbra?: number;
      decay?: number;
    } = {}
  ): THREE.SpotLight {
    const light = new THREE.SpotLight(
      config.color ?? 0xffffff,
      config.intensity ?? 1,
      config.distance ?? 0,
      config.angle ?? Math.PI / 3,
      config.penumbra ?? 0,
      config.decay ?? 2
    );

    if (config.position) {
      light.position.copy(config.position);
    }

    if (config.target) {
      light.target.position.copy(config.target);
      this.scene.add(light.target);
    }

    if (config.castShadow) {
      light.castShadow = true;
      light.shadow.mapSize.width = 1024;
      light.shadow.mapSize.height = 1024;
      light.shadow.camera.near = 0.5;
      light.shadow.camera.far = 500;
    }

    this.lights.set(id, light);
    this.scene.add(light);

    return light;
  }

  addHemisphereLight(
    id: string,
    skyColor: number | string = 0x87ceeb,
    groundColor: number | string = 0x654321,
    intensity: number = 1
  ): THREE.HemisphereLight {
    const light = new THREE.HemisphereLight(skyColor, groundColor, intensity);

    this.lights.set(id, light);
    this.scene.add(light);

    return light;
  }

  getLight(id: string): THREE.Light | undefined {
    return this.lights.get(id);
  }

  removeLight(id: string): void {
    const light = this.lights.get(id);
    if (light) {
      this.scene.remove(light);
      this.lights.delete(id);
      light.dispose();
    }
  }

  setLightIntensity(id: string, intensity: number): void {
    const light = this.lights.get(id);
    if (light) {
      light.intensity = intensity;
    }
  }

  setLightColor(id: string, color: number | string): void {
    const light = this.lights.get(id);
    if (light) {
      light.color.set(color);
    }
  }

  dispose(): void {
    this.lights.forEach((light) => {
      this.scene.remove(light);
      light.dispose();
    });
    this.lights.clear();
  }
}
```

### 2. Day/Night Cycle

```typescript
// lights/DayNightCycle.ts
import * as THREE from 'three';

export interface TimeOfDay {
  hour: number; // 0-24
  sunColor: THREE.Color;
  skyColor: THREE.Color;
  groundColor: THREE.Color;
  ambientIntensity: number;
  sunIntensity: number;
  sunPosition: THREE.Vector3;
}

export class DayNightCycle {
  private currentTime = 12; // Noon
  private timeSpeed = 1; // Hours per second
  private cycleLength = 24; // Hours

  private sunLight: THREE.DirectionalLight;
  private hemisphereLight: THREE.HemisphereLight;
  private ambientLight: THREE.AmbientLight;

  private readonly timeSteps: TimeOfDay[] = [
    // Midnight
    {
      hour: 0,
      sunColor: new THREE.Color(0x0c1445),
      skyColor: new THREE.Color(0x0a0a1e),
      groundColor: new THREE.Color(0x050508),
      ambientIntensity: 0.1,
      sunIntensity: 0,
      sunPosition: new THREE.Vector3(0, -1, 0),
    },
    // Dawn
    {
      hour: 6,
      sunColor: new THREE.Color(0xff6347),
      skyColor: new THREE.Color(0xff7f50),
      groundColor: new THREE.Color(0x2f1810),
      ambientIntensity: 0.3,
      sunIntensity: 0.5,
      sunPosition: new THREE.Vector3(1, 0, 0),
    },
    // Noon
    {
      hour: 12,
      sunColor: new THREE.Color(0xfffaf0),
      skyColor: new THREE.Color(0x87ceeb),
      groundColor: new THREE.Color(0x654321),
      ambientIntensity: 0.5,
      sunIntensity: 1,
      sunPosition: new THREE.Vector3(0, 1, 0),
    },
    // Dusk
    {
      hour: 18,
      sunColor: new THREE.Color(0xff4500),
      skyColor: new THREE.Color(0xff6347),
      groundColor: new THREE.Color(0x1a0f08),
      ambientIntensity: 0.3,
      sunIntensity: 0.5,
      sunPosition: new THREE.Vector3(-1, 0, 0),
    },
    // Midnight (wrap)
    {
      hour: 24,
      sunColor: new THREE.Color(0x0c1445),
      skyColor: new THREE.Color(0x0a0a1e),
      groundColor: new THREE.Color(0x050508),
      ambientIntensity: 0.1,
      sunIntensity: 0,
      sunPosition: new THREE.Vector3(0, -1, 0),
    },
  ];

  constructor(
    sunLight: THREE.DirectionalLight,
    hemisphereLight: THREE.HemisphereLight,
    ambientLight: THREE.AmbientLight
  ) {
    this.sunLight = sunLight;
    this.hemisphereLight = hemisphereLight;
    this.ambientLight = ambientLight;
  }

  update(deltaTime: number): void {
    this.currentTime += this.timeSpeed * deltaTime;

    if (this.currentTime >= this.cycleLength) {
      this.currentTime = 0;
    }

    this.applyTimeOfDay(this.currentTime);
  }

  private applyTimeOfDay(hour: number): void {
    // Find surrounding time steps
    let prevStep = this.timeSteps[0];
    let nextStep = this.timeSteps[1];

    for (let i = 0; i < this.timeSteps.length - 1; i++) {
      if (hour >= this.timeSteps[i].hour && hour < this.timeSteps[i + 1].hour) {
        prevStep = this.timeSteps[i];
        nextStep = this.timeSteps[i + 1];
        break;
      }
    }

    // Interpolate
    const duration = nextStep.hour - prevStep.hour;
    const elapsed = hour - prevStep.hour;
    const t = elapsed / duration;

    // Colors
    this.sunLight.color.lerpColors(prevStep.sunColor, nextStep.sunColor, t);
    this.hemisphereLight.color.lerpColors(prevStep.skyColor, nextStep.skyColor, t);
    this.hemisphereLight.groundColor.lerpColors(
      prevStep.groundColor,
      nextStep.groundColor,
      t
    );

    // Intensities
    this.ambientLight.intensity = THREE.MathUtils.lerp(
      prevStep.ambientIntensity,
      nextStep.ambientIntensity,
      t
    );
    this.sunLight.intensity = THREE.MathUtils.lerp(
      prevStep.sunIntensity,
      nextStep.sunIntensity,
      t
    );

    // Position
    this.sunLight.position.lerpVectors(prevStep.sunPosition, nextStep.sunPosition, t);
    this.sunLight.position.multiplyScalar(50);
  }

  setTime(hour: number): void {
    this.currentTime = hour % this.cycleLength;
  }

  setTimeSpeed(speed: number): void {
    this.timeSpeed = speed;
  }

  getCurrentTime(): number {
    return this.currentTime;
  }
}
```

### 3. Light Probe System

```typescript
// lights/LightProbeSystem.ts
import * as THREE from 'three';

export class LightProbeSystem {
  private scene: THREE.Scene;
  private lightProbe: THREE.LightProbe;

  constructor(scene: THREE.Scene) {
    this.scene = scene;
    this.lightProbe = new THREE.LightProbe();
    this.scene.add(this.lightProbe);
  }

  async fromCubeRenderTarget(cubeRenderTarget: THREE.WebGLCubeRenderTarget): Promise<void> {
    const generator = new THREE.PMREMGenerator(cubeRenderTarget.texture.source.data);
    const cubeUV = generator.fromCubemap(cubeRenderTarget.texture);

    this.lightProbe.copy(THREE.LightProbeGenerator.fromCubeRenderTarget(cubeRenderTarget));

    generator.dispose();
  }

  async fromEquirectangular(texture: THREE.Texture, renderer: THREE.WebGLRenderer): Promise<void> {
    const generator = new THREE.PMREMGenerator(renderer);
    const renderTarget = generator.fromEquirectangular(texture);

    this.lightProbe.copy(THREE.LightProbeGenerator.fromCubeRenderTarget(renderTarget));

    generator.dispose();
    renderTarget.dispose();
  }

  setIntensity(intensity: number): void {
    this.lightProbe.intensity = intensity;
  }

  dispose(): void {
    this.scene.remove(this.lightProbe);
  }
}
```

### 4. Dynamic Light Pool

```typescript
// lights/DynamicLightPool.ts
import * as THREE from 'three';

export class DynamicLightPool {
  private scene: THREE.Scene;
  private availableLights: THREE.PointLight[] = [];
  private activeLights = new Map<string, THREE.PointLight>();
  private maxLights: number;

  constructor(scene: THREE.Scene, maxLights: number = 8) {
    this.scene = scene;
    this.maxLights = maxLights;

    // Pre-create lights
    for (let i = 0; i < maxLights; i++) {
      const light = new THREE.PointLight(0xffffff, 0, 10, 2);
      light.visible = false;
      this.scene.add(light);
      this.availableLights.push(light);
    }
  }

  requestLight(
    id: string,
    position: THREE.Vector3,
    color: number | string,
    intensity: number,
    distance: number = 10
  ): THREE.PointLight | null {
    // If light already active, update it
    if (this.activeLights.has(id)) {
      const light = this.activeLights.get(id)!;
      light.position.copy(position);
      light.color.set(color);
      light.intensity = intensity;
      light.distance = distance;
      return light;
    }

    // Get available light
    const light = this.availableLights.pop();
    if (!light) {
      console.warn('No available lights in pool');
      return null;
    }

    // Configure light
    light.position.copy(position);
    light.color.set(color);
    light.intensity = intensity;
    light.distance = distance;
    light.visible = true;

    this.activeLights.set(id, light);

    return light;
  }

  releaseLight(id: string): void {
    const light = this.activeLights.get(id);
    if (light) {
      light.visible = false;
      light.intensity = 0;
      this.activeLights.delete(id);
      this.availableLights.push(light);
    }
  }

  updateLight(id: string, position: THREE.Vector3, intensity?: number): void {
    const light = this.activeLights.get(id);
    if (light) {
      light.position.copy(position);
      if (intensity !== undefined) {
        light.intensity = intensity;
      }
    }
  }

  dispose(): void {
    this.activeLights.forEach((light) => {
      this.scene.remove(light);
      light.dispose();
    });
    this.availableLights.forEach((light) => {
      this.scene.remove(light);
      light.dispose();
    });
    this.activeLights.clear();
    this.availableLights = [];
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic three-point lighting
import { LightManager } from './lights/LightManager';

const lightManager = new LightManager(scene);

// Key light (main light)
lightManager.addDirectionalLight('key', {
  color: 0xffffff,
  intensity: 1,
  position: new THREE.Vector3(5, 10, 5),
  castShadow: true,
});

// Fill light (soften shadows)
lightManager.addDirectionalLight('fill', {
  color: 0x8888ff,
  intensity: 0.4,
  position: new THREE.Vector3(-5, 5, 5),
});

// Back light (rim light)
lightManager.addDirectionalLight('back', {
  color: 0xffffff,
  intensity: 0.3,
  position: new THREE.Vector3(0, 5, -10),
});

// Ambient
lightManager.addAmbientLight('ambient', {
  color: 0x404040,
  intensity: 0.2,
});

// Example 2: Day/night cycle
import { DayNightCycle } from './lights/DayNightCycle';

const sunLight = lightManager.getLight('sun') as THREE.DirectionalLight;
const hemiLight = lightManager.getLight('hemi') as THREE.HemisphereLight;
const ambientLight = lightManager.getLight('ambient') as THREE.AmbientLight;

const dayNight = new DayNightCycle(sunLight, hemiLight, ambientLight);
dayNight.setTime(12); // Start at noon
dayNight.setTimeSpeed(0.1); // 0.1 hours per second

function animate() {
  dayNight.update(deltaTime);
  renderer.render(scene, camera);
}

// Example 3: Dynamic light pool
import { DynamicLightPool } from './lights/DynamicLightPool';

const lightPool = new DynamicLightPool(scene, 8);

// Spawn light
lightPool.requestLight(
  'torch_1',
  new THREE.Vector3(5, 2, 5),
  0xff6600,
  2,
  10
);

// Update light position
lightPool.updateLight('torch_1', playerPosition);

// Remove light
lightPool.releaseLight('torch_1');
```

## Checklist

- [ ] Set up ambient light for base illumination
- [ ] Add directional light for sun/moon
- [ ] Configure shadow casting for main lights
- [ ] Add point lights for localized lighting
- [ ] Implement hemisphere light for outdoor scenes
- [ ] Limit total active lights (< 8 for mobile)
- [ ] Use light pools for dynamic lights
- [ ] Bake static lighting when possible
- [ ] Test lighting on target devices
- [ ] Optimize shadow map sizes
- [ ] Use realistic color temperatures
- [ ] Profile light performance

## Common Pitfalls

1. **Too many lights**: Performance drops rapidly
2. **No ambient light**: Pure black shadows
3. **Wrong color temperatures**: Unrealistic look
4. **High shadow map sizes**: Memory/performance issues
5. **Not limiting light distance**: Affects unnecessary objects
6. **Forgetting to dispose**: Memory leaks
7. **Static lights in update loop**: Wasted CPU

## Performance Tips

- Limit total lights to 4-8 (mobile) or 8-16 (desktop)
- Use ambient/hemisphere for fill instead of multiple lights
- Bake static lighting into lightmaps
- Use light pools for dynamic lights
- Disable shadows on distant lights
- Use lower shadow map sizes for point lights (512-1024)
- Use directional lights for sun (one shadow map vs 6 for point)
- Turn off lights that are out of view
- Use hemisphere light instead of ambient + directional

## Related Skills

- `threejs-scene-setup` - Scene initialization
- `threejs-shadows` - Shadow configuration
- `threejs-material-systems` - Material lighting response
- `mobile-performance` - Mobile optimization
