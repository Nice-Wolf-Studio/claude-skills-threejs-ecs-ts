---
name: threejs-fog
description: Fog and atmospheric effects in Three.js including linear fog, exponential fog, distance-based fading, and custom atmospheric shaders
---

# Three.js Fog and Atmospheric Effects

## When to Use

Use this skill when:
- Creating depth perception in outdoor scenes
- Implementing atmospheric effects (fog, mist, haze)
- Optimizing by hiding distant geometry
- Creating mood and ambiance
- Implementing weather effects
- Managing draw distance gracefully

## Core Principles

1. **Depth Cues**: Fog improves depth perception
2. **Performance**: Hides distant objects naturally
3. **Atmosphere**: Sets scene mood and tone
4. **Consistency**: Match fog color to background
5. **Type Selection**: Linear vs exponential based on needs
6. **Mobile Considerations**: Fog is cheap, use it!

## Implementation

### 1. Fog Manager

```typescript
// fog/FogManager.ts
import * as THREE from 'three';

export type FogType = 'none' | 'linear' | 'exponential' | 'exponential2';

export interface FogConfig {
  type: FogType;
  color: THREE.ColorRepresentation;
  near?: number;
  far?: number;
  density?: number;
}

export class FogManager {
  private scene: THREE.Scene;
  private currentType: FogType = 'none';

  constructor(scene: THREE.Scene) {
    this.scene = scene;
  }

  setFog(config: FogConfig): void {
    this.currentType = config.type;

    switch (config.type) {
      case 'none':
        this.scene.fog = null;
        break;

      case 'linear':
        this.scene.fog = new THREE.Fog(
          config.color,
          config.near ?? 1,
          config.far ?? 100
        );
        break;

      case 'exponential':
        this.scene.fog = new THREE.FogExp2(config.color, config.density ?? 0.01);
        break;

      case 'exponential2':
        this.scene.fog = new THREE.FogExp2(config.color, config.density ?? 0.005);
        break;
    }

    // Update background to match fog color
    this.scene.background = new THREE.Color(config.color);
  }

  updateFogColor(color: THREE.ColorRepresentation): void {
    if (this.scene.fog) {
      this.scene.fog.color.set(color);
      this.scene.background = new THREE.Color(color);
    }
  }

  updateLinearFog(near: number, far: number): void {
    if (this.scene.fog && this.scene.fog instanceof THREE.Fog) {
      this.scene.fog.near = near;
      this.scene.fog.far = far;
    }
  }

  updateExponentialFog(density: number): void {
    if (this.scene.fog && this.scene.fog instanceof THREE.FogExp2) {
      this.scene.fog.density = density;
    }
  }

  clear(): void {
    this.scene.fog = null;
    this.currentType = 'none';
  }

  getCurrentType(): FogType {
    return this.currentType;
  }
}
```

### 2. Fog Presets

```typescript
// fog/FogPresets.ts
import * as THREE from 'three';
import { FogConfig } from './FogManager';

export const FogPresets = {
  // Clear day
  clear: {
    type: 'linear' as const,
    color: 0x87ceeb,
    near: 100,
    far: 1000,
  } as FogConfig,

  // Light fog
  morning: {
    type: 'exponential' as const,
    color: 0xe6f2ff,
    density: 0.002,
  } as FogConfig,

  // Medium fog
  foggy: {
    type: 'exponential' as const,
    color: 0xcccccc,
    density: 0.01,
  } as FogConfig,

  // Dense fog
  heavyFog: {
    type: 'exponential2' as const,
    color: 0x999999,
    density: 0.015,
  } as FogConfig,

  // Dusk
  dusk: {
    type: 'linear' as const,
    color: 0xff6b35,
    near: 50,
    far: 300,
  } as FogConfig,

  // Night
  night: {
    type: 'exponential' as const,
    color: 0x1a1a2e,
    density: 0.008,
  } as FogConfig,

  // Underground/cave
  cave: {
    type: 'exponential2' as const,
    color: 0x0a0a0a,
    density: 0.05,
  } as FogConfig,

  // Desert haze
  desert: {
    type: 'linear' as const,
    color: 0xffd89b,
    near: 200,
    far: 800,
  } as FogConfig,

  // Underwater
  underwater: {
    type: 'exponential' as const,
    color: 0x006994,
    density: 0.02,
  } as FogConfig,

  // Toxic/pollution
  toxic: {
    type: 'exponential2' as const,
    color: 0x9acd32,
    density: 0.012,
  } as FogConfig,
};
```

### 3. Animated Fog

```typescript
// fog/AnimatedFog.ts
import * as THREE from 'three';

export class AnimatedFog {
  private scene: THREE.Scene;
  private baseColor: THREE.Color;
  private baseDensity: number;
  private time = 0;

  constructor(scene: THREE.Scene, color: THREE.ColorRepresentation, density: number) {
    this.scene = scene;
    this.baseColor = new THREE.Color(color);
    this.baseDensity = density;

    this.scene.fog = new THREE.FogExp2(color, density);
  }

  update(deltaTime: number): void {
    if (!this.scene.fog || !(this.scene.fog instanceof THREE.FogExp2)) return;

    this.time += deltaTime;

    // Pulse fog density
    const densityOffset = Math.sin(this.time * 0.5) * 0.003;
    this.scene.fog.density = this.baseDensity + densityOffset;

    // Subtle color shift
    const colorOffset = Math.sin(this.time * 0.2) * 0.1;
    this.scene.fog.color.setRGB(
      THREE.MathUtils.clamp(this.baseColor.r + colorOffset, 0, 1),
      THREE.MathUtils.clamp(this.baseColor.g + colorOffset, 0, 1),
      THREE.MathUtils.clamp(this.baseColor.b + colorOffset, 0, 1)
    );
  }

  setIntensity(intensity: number): void {
    this.baseDensity = intensity;
  }
}
```

### 4. Height Fog (Custom Shader)

```typescript
// fog/HeightFog.ts
import * as THREE from 'three';

export const HeightFogShader = {
  uniforms: {
    tDiffuse: { value: null },
    tDepth: { value: null },
    fogColor: { value: new THREE.Color(0xcccccc) },
    fogNear: { value: 1 },
    fogFar: { value: 100 },
    fogHeight: { value: 10 },
    fogFalloff: { value: 5 },
    cameraNear: { value: 0.1 },
    cameraFar: { value: 1000 },
  },

  vertexShader: `
    varying vec2 vUv;

    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,

  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform sampler2D tDepth;
    uniform vec3 fogColor;
    uniform float fogNear;
    uniform float fogFar;
    uniform float fogHeight;
    uniform float fogFalloff;
    uniform float cameraNear;
    uniform float cameraFar;

    varying vec2 vUv;

    float readDepth(sampler2D depthSampler, vec2 coord) {
      float fragCoordZ = texture2D(depthSampler, coord).x;
      float viewZ = perspectiveDepthToViewZ(fragCoordZ, cameraNear, cameraFar);
      return viewZToOrthographicDepth(viewZ, cameraNear, cameraFar);
    }

    float perspectiveDepthToViewZ(float invClipZ, float near, float far) {
      return (near * far) / ((far - near) * invClipZ - far);
    }

    float viewZToOrthographicDepth(float viewZ, float near, float far) {
      return (viewZ + near) / (near - far);
    }

    void main() {
      vec4 texel = texture2D(tDiffuse, vUv);
      float depth = readDepth(tDepth, vUv);

      // Calculate world Y position (approximate)
      float worldY = (1.0 - vUv.y) * fogHeight;

      // Height-based fog factor
      float heightFactor = 1.0 - clamp((worldY - fogHeight) / fogFalloff, 0.0, 1.0);

      // Distance-based fog factor
      float fogFactor = smoothstep(fogNear, fogFar, depth);

      // Combine factors
      float finalFogFactor = fogFactor * heightFactor;

      gl_FragColor = vec4(mix(texel.rgb, fogColor, finalFogFactor), texel.a);
    }
  `,
};
```

### 5. Weather System with Fog

```typescript
// fog/WeatherSystem.ts
import * as THREE from 'three';
import { FogManager, FogConfig } from './FogManager';

export type WeatherType = 'clear' | 'cloudy' | 'foggy' | 'rainy' | 'stormy';

export class WeatherSystem {
  private fogManager: FogManager;
  private currentWeather: WeatherType = 'clear';
  private transitionProgress = 0;
  private transitionDuration = 5; // Seconds

  private fromConfig: FogConfig | null = null;
  private toConfig: FogConfig | null = null;

  private weatherConfigs: Record<WeatherType, FogConfig> = {
    clear: {
      type: 'linear',
      color: 0x87ceeb,
      near: 100,
      far: 1000,
    },
    cloudy: {
      type: 'linear',
      color: 0xaaaaaa,
      near: 50,
      far: 500,
    },
    foggy: {
      type: 'exponential',
      color: 0xcccccc,
      density: 0.01,
    },
    rainy: {
      type: 'exponential',
      color: 0x666666,
      density: 0.005,
    },
    stormy: {
      type: 'exponential2',
      color: 0x333333,
      density: 0.015,
    },
  };

  constructor(fogManager: FogManager) {
    this.fogManager = fogManager;
  }

  setWeather(weather: WeatherType, immediate: boolean = false): void {
    const newConfig = this.weatherConfigs[weather];

    if (immediate) {
      this.fogManager.setFog(newConfig);
      this.currentWeather = weather;
    } else {
      // Start transition
      const currentFog = this.fogManager.getCurrentType();
      if (currentFog !== 'none') {
        this.fromConfig = { ...this.weatherConfigs[this.currentWeather] };
      }
      this.toConfig = newConfig;
      this.transitionProgress = 0;
    }
  }

  update(deltaTime: number): void {
    if (!this.toConfig) return;

    this.transitionProgress += deltaTime / this.transitionDuration;

    if (this.transitionProgress >= 1) {
      this.fogManager.setFog(this.toConfig);
      this.fromConfig = null;
      this.toConfig = null;
      return;
    }

    // Interpolate fog
    if (this.fromConfig && this.toConfig) {
      const t = this.transitionProgress;

      // Linear fog interpolation
      if (this.toConfig.type === 'linear') {
        const fromNear = this.fromConfig.near ?? 1;
        const fromFar = this.fromConfig.far ?? 100;
        const toNear = this.toConfig.near ?? 1;
        const toFar = this.toConfig.far ?? 100;

        const near = THREE.MathUtils.lerp(fromNear, toNear, t);
        const far = THREE.MathUtils.lerp(fromFar, toFar, t);

        this.fogManager.setFog({
          ...this.toConfig,
          near,
          far,
        });
      }
      // Exponential fog interpolation
      else if (this.toConfig.type === 'exponential' || this.toConfig.type === 'exponential2') {
        const fromDensity = this.fromConfig.density ?? 0.01;
        const toDensity = this.toConfig.density ?? 0.01;

        const density = THREE.MathUtils.lerp(fromDensity, toDensity, t);

        this.fogManager.setFog({
          ...this.toConfig,
          density,
        });
      }

      // Color interpolation
      const fromColor = new THREE.Color(this.fromConfig.color);
      const toColor = new THREE.Color(this.toConfig.color);
      const color = fromColor.lerp(toColor, t);

      this.fogManager.updateFogColor(color);
    }
  }

  getCurrentWeather(): WeatherType {
    return this.currentWeather;
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic fog setup
import { FogManager } from './fog/FogManager';

const fogManager = new FogManager(scene);

// Linear fog (good for outdoor scenes)
fogManager.setFog({
  type: 'linear',
  color: 0x87ceeb,
  near: 50,
  far: 200,
});

// Example 2: Using presets
import { FogPresets } from './fog/FogPresets';

fogManager.setFog(FogPresets.morning);

// Example 3: Animated fog
import { AnimatedFog } from './fog/AnimatedFog';

const animatedFog = new AnimatedFog(scene, 0xcccccc, 0.01);

function animate() {
  animatedFog.update(deltaTime);
}

// Example 4: Weather system
import { WeatherSystem } from './fog/WeatherSystem';

const weatherSystem = new WeatherSystem(fogManager);

// Start with clear weather
weatherSystem.setWeather('clear', true);

// Change to foggy (with transition)
weatherSystem.setWeather('foggy', false);

function animate() {
  weatherSystem.update(deltaTime);
}

// Example 5: Dynamic fog based on player position
const fogManager = new FogManager(scene);
fogManager.setFog({ type: 'linear', color: 0x87ceeb, near: 10, far: 100 });

function animate() {
  // Update fog based on player height
  const playerHeight = player.position.y;
  const near = 10 + playerHeight * 5;
  const far = 100 + playerHeight * 20;

  fogManager.updateLinearFog(near, far);
}
```

## Checklist

- [ ] Choose fog type (linear vs exponential)
- [ ] Set fog color to match background/skybox
- [ ] Configure fog near/far or density
- [ ] Test fog visibility at different distances
- [ ] Ensure fog affects all materials
- [ ] Match fog color with ambient light
- [ ] Test on target devices
- [ ] Consider performance impact (minimal)
- [ ] Implement weather transitions if needed
- [ ] Add fog to skybox/background

## Common Pitfalls

1. **Mismatched colors**: Fog color doesn't match background
2. **Too dense**: Can't see anything
3. **Too sparse**: No visible effect
4. **Wrong fog type**: Exponential when linear needed
5. **Materials don't respect fog**: Check material.fog property
6. **Fog on UI**: UI elements should ignore fog
7. **Sudden visibility changes**: Use smooth transitions

## Performance Tips

### Fog is Cheap!
- Fog has minimal performance cost
- Built into Three.js material shaders
- No additional draw calls
- Works on all devices

### When to Use Each Type
- **Linear Fog**: Outdoor scenes, predictable fade
  - Clear start/end points
  - Good for optimizing draw distance
- **Exponential Fog**: More realistic atmosphere
  - Density-based (more natural looking)
  - Better for caves, underwater
- **Exponential² Fog**: Denser fog
  - Even more realistic
  - Slightly more expensive

### Optimization Strategies
- Use fog to hide pop-in of LOD transitions
- Match fog far distance to camera far plane
- Use fog to reduce draw distance naturally
- Cull objects beyond fog visibility

### Mobile Considerations
- Fog works great on mobile!
- Use it to reduce draw distance
- Prefer linear fog (slightly faster)
- Lower fog density on low-end devices

## Related Skills

- `threejs-scene-setup` - Scene and background setup
- `threejs-lighting` - Lighting integration with fog
- `threejs-post-processing` - Advanced atmospheric effects
- `mobile-performance` - Mobile optimization

## References

- Three.js Fog: https://threejs.org/docs/#api/en/scenes/Fog
- Three.js FogExp2: https://threejs.org/docs/#api/en/scenes/FogExp2
- Atmospheric Scattering: https://developer.nvidia.com/gpugems/gpugems2/part-ii-shading-lighting-and-shadows/chapter-16-accurate-atmospheric-scattering
