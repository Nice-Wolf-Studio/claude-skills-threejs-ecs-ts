---
name: battery-optimization
description: Mobile battery optimization techniques including adaptive quality, power-efficient rendering, and background behavior
---

# Mobile Battery Optimization

## When to Use

Use this skill when:
- Optimizing for mobile battery life
- Implementing adaptive quality systems
- Managing background behavior
- Reducing power consumption
- Handling device thermal throttling
- Creating power-efficient games

## Core Principles

1. **Adaptive Quality**: Scale based on power state
2. **Intelligent Throttling**: Reduce updates when inactive
3. **Background Pause**: Stop rendering when hidden
4. **Thermal Management**: Detect and respond to heat
5. **Power Awareness**: Monitor battery state
6. **Efficient Rendering**: Minimize GPU/CPU usage

## Battery Optimization Implementation

### 1. Power State Management

```typescript
// power/PowerStateManager.ts
export enum PowerState {
  High = 'high', // Plugged in, full quality
  Normal = 'normal', // Good battery, normal quality
  Low = 'low', // Low battery, reduced quality
  Critical = 'critical', // Very low battery, minimal quality
}

export interface PowerMetrics {
  batteryLevel: number; // 0-1
  isCharging: boolean;
  isSaveModeEnabled: boolean;
  temperature?: number; // Device temperature if available
}

export class PowerStateManager {
  private state: PowerState = PowerState.Normal;
  private metrics: PowerMetrics = {
    batteryLevel: 1,
    isCharging: false,
    isSaveModeEnabled: false,
  };

  private listeners = new Set<(state: PowerState) => void>();

  constructor() {
    this.initBatteryAPI();
    this.initVisibilityAPI();
  }

  private async initBatteryAPI(): Promise<void> {
    if ('getBattery' in navigator) {
      try {
        const battery = await (navigator as any).getBattery();

        // Initial state
        this.updateMetrics({
          batteryLevel: battery.level,
          isCharging: battery.charging,
        });

        // Listen for changes
        battery.addEventListener('levelchange', () => {
          this.updateMetrics({ batteryLevel: battery.level });
        });

        battery.addEventListener('chargingchange', () => {
          this.updateMetrics({ isCharging: battery.charging });
        });
      } catch (error) {
        console.warn('Battery API not available:', error);
      }
    }
  }

  private initVisibilityAPI(): void {
    document.addEventListener('visibilitychange', () => {
      if (document.hidden) {
        this.onBackgrounded();
      } else {
        this.onForegrounded();
      }
    });
  }

  private updateMetrics(partial: Partial<PowerMetrics>): void {
    Object.assign(this.metrics, partial);
    this.updatePowerState();
  }

  private updatePowerState(): void {
    const newState = this.calculatePowerState();

    if (newState !== this.state) {
      this.state = newState;
      this.notifyListeners();
    }
  }

  private calculatePowerState(): PowerState {
    // Charging = high performance
    if (this.metrics.isCharging) {
      return PowerState.High;
    }

    // Battery-based states
    if (this.metrics.batteryLevel < 0.1) {
      return PowerState.Critical;
    } else if (this.metrics.batteryLevel < 0.2) {
      return PowerState.Low;
    } else {
      return PowerState.Normal;
    }
  }

  private notifyListeners(): void {
    for (const listener of this.listeners) {
      listener(this.state);
    }
  }

  private onBackgrounded(): void {
    // Notify listeners that app is backgrounded
    for (const listener of this.listeners) {
      listener(PowerState.Critical); // Treat as critical to pause everything
    }
  }

  private onForegrounded(): void {
    // Resume normal power state
    this.updatePowerState();
  }

  getPowerState(): PowerState {
    return this.state;
  }

  getMetrics(): PowerMetrics {
    return { ...this.metrics };
  }

  onStateChange(listener: (state: PowerState) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
}
```

### 2. Adaptive Quality System

```typescript
// power/AdaptiveQualityManager.ts
export interface QualitySettings {
  targetFPS: number;
  renderScale: number; // 0.5 - 1.0
  shadowQuality: 'none' | 'low' | 'medium' | 'high';
  particleLimit: number;
  drawDistance: number;
  postProcessing: boolean;
  antiAliasing: boolean;
  maxLights: number;
  textureQuality: number; // 0.25 - 1.0
}

export class AdaptiveQualityManager {
  private currentSettings: QualitySettings;

  private presets = new Map<PowerState, QualitySettings>([
    [
      PowerState.High,
      {
        targetFPS: 60,
        renderScale: 1.0,
        shadowQuality: 'high',
        particleLimit: 1000,
        drawDistance: 100,
        postProcessing: true,
        antiAliasing: true,
        maxLights: 8,
        textureQuality: 1.0,
      },
    ],
    [
      PowerState.Normal,
      {
        targetFPS: 60,
        renderScale: 0.9,
        shadowQuality: 'medium',
        particleLimit: 500,
        drawDistance: 75,
        postProcessing: true,
        antiAliasing: true,
        maxLights: 4,
        textureQuality: 0.75,
      },
    ],
    [
      PowerState.Low,
      {
        targetFPS: 30,
        renderScale: 0.75,
        shadowQuality: 'low',
        particleLimit: 200,
        drawDistance: 50,
        postProcessing: false,
        antiAliasing: false,
        maxLights: 2,
        textureQuality: 0.5,
      },
    ],
    [
      PowerState.Critical,
      {
        targetFPS: 20,
        renderScale: 0.5,
        shadowQuality: 'none',
        particleLimit: 50,
        drawDistance: 30,
        postProcessing: false,
        antiAliasing: false,
        maxLights: 1,
        textureQuality: 0.25,
      },
    ],
  ]);

  constructor(
    private renderer: THREE.WebGLRenderer,
    private scene: THREE.Scene,
    powerManager: PowerStateManager
  ) {
    this.currentSettings = this.presets.get(PowerState.Normal)!;

    // Listen for power state changes
    powerManager.onStateChange((state) => {
      this.applyPowerState(state);
    });
  }

  private applyPowerState(state: PowerState): void {
    const settings = this.presets.get(state);
    if (!settings) return;

    this.currentSettings = settings;
    this.applySettings();
  }

  private applySettings(): void {
    const s = this.currentSettings;

    // Render scale
    const width = window.innerWidth * s.renderScale;
    const height = window.innerHeight * s.renderScale;
    this.renderer.setSize(width, height, false);

    // Shadows
    this.renderer.shadowMap.enabled = s.shadowQuality !== 'none';
    if (s.shadowQuality !== 'none') {
      this.updateShadowQuality(s.shadowQuality);
    }

    // Post-processing
    // Toggle post-processing composer here

    // Anti-aliasing (requires renderer recreation)
    // Store setting for next renderer creation

    // Update all lights
    this.scene.traverse((obj) => {
      if (obj instanceof THREE.Light) {
        if ((obj as any).shadow) {
          obj.castShadow = s.shadowQuality !== 'none';
        }
      }
    });

    // Limit active lights
    this.limitLights(s.maxLights);

    // Update particle systems
    this.updateParticleLimits(s.particleLimit);

    // Update texture quality
    this.updateTextureQuality(s.textureQuality);
  }

  private updateShadowQuality(quality: 'low' | 'medium' | 'high'): void {
    const sizes = {
      low: 512,
      medium: 1024,
      high: 2048,
    };

    const size = sizes[quality];

    this.scene.traverse((obj) => {
      if (obj instanceof THREE.Light && (obj as any).shadow) {
        const light = obj as THREE.Light & { shadow: THREE.LightShadow };
        light.shadow.mapSize.width = size;
        light.shadow.mapSize.height = size;
        light.shadow.map?.dispose();
        light.shadow.map = null;
      }
    });
  }

  private limitLights(maxLights: number): void {
    let lightCount = 0;

    this.scene.traverse((obj) => {
      if (obj instanceof THREE.Light && !(obj instanceof THREE.AmbientLight)) {
        lightCount++;
        obj.visible = lightCount <= maxLights;
      }
    });
  }

  private updateParticleLimits(limit: number): void {
    // Update particle emitters to respect limit
    // Implementation depends on particle system
  }

  private updateTextureQuality(quality: number): void {
    this.scene.traverse((obj) => {
      if (obj instanceof THREE.Mesh) {
        const materials = Array.isArray(obj.material) ? obj.material : [obj.material];

        for (const material of materials) {
          if (material instanceof THREE.MeshStandardMaterial) {
            // Update texture anisotropy
            const textures = [
              material.map,
              material.normalMap,
              material.roughnessMap,
              material.metalnessMap,
            ];

            for (const texture of textures) {
              if (texture) {
                texture.anisotropy = Math.floor(16 * quality);
              }
            }
          }
        }
      }
    });
  }

  getSettings(): QualitySettings {
    return { ...this.currentSettings };
  }

  setCustomSettings(settings: Partial<QualitySettings>): void {
    Object.assign(this.currentSettings, settings);
    this.applySettings();
  }
}
```

### 3. Frame Rate Throttling

```typescript
// power/FrameRateThrottle.ts
export class FrameRateThrottle {
  private targetInterval: number;
  private lastFrameTime = 0;
  private accumulator = 0;

  constructor(private targetFPS: number) {
    this.targetInterval = 1000 / targetFPS;
  }

  setTargetFPS(fps: number): void {
    this.targetFPS = fps;
    this.targetInterval = 1000 / fps;
  }

  shouldRender(currentTime: number): boolean {
    const deltaTime = currentTime - this.lastFrameTime;
    this.accumulator += deltaTime;

    if (this.accumulator >= this.targetInterval) {
      this.lastFrameTime = currentTime;
      this.accumulator %= this.targetInterval;
      return true;
    }

    return false;
  }

  reset(): void {
    this.lastFrameTime = 0;
    this.accumulator = 0;
  }
}

// Usage in game loop
export class PowerAwareGameLoop {
  private throttle: FrameRateThrottle;
  private isPaused = false;
  private animationFrameId: number | null = null;

  constructor(
    private renderer: THREE.WebGLRenderer,
    private scene: THREE.Scene,
    private camera: THREE.Camera,
    powerManager: PowerStateManager,
    qualityManager: AdaptiveQualityManager
  ) {
    this.throttle = new FrameRateThrottle(60);

    // Adjust frame rate based on power state
    powerManager.onStateChange((state) => {
      const settings = qualityManager.getSettings();
      this.throttle.setTargetFPS(settings.targetFPS);

      // Pause on critical/background
      if (state === PowerState.Critical) {
        this.pause();
      } else {
        this.resume();
      }
    });

    // Pause when page is hidden
    document.addEventListener('visibilitychange', () => {
      if (document.hidden) {
        this.pause();
      } else {
        this.resume();
      }
    });
  }

  private tick = (currentTime: number): void => {
    if (this.isPaused) return;

    // Throttle based on target FPS
    if (this.throttle.shouldRender(currentTime)) {
      this.update(currentTime);
      this.render();
    }

    this.animationFrameId = requestAnimationFrame(this.tick);
  };

  private update(currentTime: number): void {
    // Update game logic
  }

  private render(): void {
    this.renderer.render(this.scene, this.camera);
  }

  start(): void {
    this.isPaused = false;
    this.throttle.reset();
    this.animationFrameId = requestAnimationFrame(this.tick);
  }

  pause(): void {
    this.isPaused = true;
    if (this.animationFrameId !== null) {
      cancelAnimationFrame(this.animationFrameId);
      this.animationFrameId = null;
    }
  }

  resume(): void {
    if (!this.isPaused) return;
    this.start();
  }

  stop(): void {
    this.pause();
  }
}
```

### 4. Thermal Management

```typescript
// power/ThermalManager.ts
export class ThermalManager {
  private temperature = 0;
  private isThrottling = false;
  private performanceObserver: PerformanceObserver | null = null;

  private readonly THROTTLE_TEMP = 45; // Celsius
  private readonly CRITICAL_TEMP = 50;

  constructor(private qualityManager: AdaptiveQualityManager) {
    this.initPerformanceMonitoring();
  }

  private initPerformanceMonitoring(): void {
    // Monitor frame drops as a proxy for thermal throttling
    if ('PerformanceObserver' in window) {
      try {
        this.performanceObserver = new PerformanceObserver((list) => {
          const entries = list.getEntries();

          // Long frames indicate thermal throttling
          for (const entry of entries) {
            if (entry.duration > 50) {
              // Frame took > 50ms (< 20 FPS)
              this.onPotentialThrottling();
            }
          }
        });

        this.performanceObserver.observe({ entryTypes: ['measure'] });
      } catch (error) {
        console.warn('PerformanceObserver not available:', error);
      }
    }
  }

  private onPotentialThrottling(): void {
    if (!this.isThrottling) {
      this.isThrottling = true;
      this.reduceLoad();
    }
  }

  private reduceLoad(): void {
    // Aggressively reduce quality
    const settings = this.qualityManager.getSettings();

    this.qualityManager.setCustomSettings({
      targetFPS: Math.max(20, settings.targetFPS - 10),
      renderScale: Math.max(0.5, settings.renderScale - 0.1),
      particleLimit: Math.floor(settings.particleLimit * 0.5),
      postProcessing: false,
    });

    // Reset throttling flag after cooldown
    setTimeout(() => {
      this.isThrottling = false;
    }, 5000);
  }

  dispose(): void {
    if (this.performanceObserver) {
      this.performanceObserver.disconnect();
    }
  }
}
```

### 5. Battery-Aware Updates

```typescript
// power/BatteryAwareSystem.ts
export abstract class BatteryAwareSystem extends UpdateSystem {
  protected updateInterval = 0; // 0 = every frame
  protected accumulator = 0;
  protected powerState: PowerState = PowerState.Normal;

  constructor(
    protected powerManager: PowerStateManager,
    baseInterval: number = 0
  ) {
    super();

    this.updateInterval = baseInterval;

    powerManager.onStateChange((state) => {
      this.powerState = state;
      this.onPowerStateChanged(state);
    });
  }

  update(world: World, deltaTime: number): void {
    // Skip updates based on power state
    if (this.shouldSkipUpdate()) {
      return;
    }

    // Interval-based updates
    if (this.updateInterval > 0) {
      this.accumulator += deltaTime;

      if (this.accumulator >= this.updateInterval) {
        this.doUpdate(world, this.accumulator);
        this.accumulator = 0;
      }
    } else {
      this.doUpdate(world, deltaTime);
    }
  }

  protected shouldSkipUpdate(): boolean {
    // Skip on critical power
    return this.powerState === PowerState.Critical;
  }

  protected onPowerStateChanged(state: PowerState): void {
    // Adjust update interval based on power state
    switch (state) {
      case PowerState.High:
        this.updateInterval = 0; // Every frame
        break;
      case PowerState.Normal:
        this.updateInterval = 0;
        break;
      case PowerState.Low:
        this.updateInterval = 0.1; // 10 times per second
        break;
      case PowerState.Critical:
        this.updateInterval = 0.5; // 2 times per second
        break;
    }
  }

  protected abstract doUpdate(world: World, deltaTime: number): void;
}

// Example usage
export class AISystem extends BatteryAwareSystem {
  constructor(powerManager: PowerStateManager) {
    super(powerManager, 0.1); // Base 10 updates/second
  }

  protected doUpdate(world: World, deltaTime: number): void {
    // AI logic here
    const aiEntities = world.query<[Transform, AIComponent]>([Transform, AIComponent]);

    aiEntities.iterate((entity, [transform, ai]) => {
      // Update AI
    });
  }

  protected onPowerStateChanged(state: PowerState): void {
    super.onPowerStateChanged(state);

    // Additional AI-specific adjustments
    if (state === PowerState.Low || state === PowerState.Critical) {
      // Reduce AI complexity
      // Limit pathfinding
      // Simplify decision trees
    }
  }
}
```

## Usage Examples

```typescript
// Example 1: Setup power management
const powerManager = new PowerStateManager();
const qualityManager = new AdaptiveQualityManager(renderer, scene, powerManager);
const thermalManager = new ThermalManager(qualityManager);

// Example 2: Power-aware game loop
const gameLoop = new PowerAwareGameLoop(
  renderer,
  scene,
  camera,
  powerManager,
  qualityManager
);

gameLoop.start();

// Example 3: Monitor battery state
powerManager.onStateChange((state) => {
  console.log(`Power state changed to: ${state}`);

  const metrics = powerManager.getMetrics();
  console.log(`Battery: ${(metrics.batteryLevel * 100).toFixed(0)}%`);
  console.log(`Charging: ${metrics.isCharging}`);
});

// Example 4: Manual quality adjustment
const settings = qualityManager.getSettings();

// User preference override
qualityManager.setCustomSettings({
  targetFPS: 30,
  renderScale: 0.8,
  shadowQuality: 'low',
});

// Example 5: Battery-aware AI system
class EnemyAISystem extends BatteryAwareSystem {
  constructor(powerManager: PowerStateManager) {
    super(powerManager, 0.2); // Update every 200ms
  }

  protected doUpdate(world: World, deltaTime: number): void {
    // Simple AI logic when on battery
    if (this.powerState === PowerState.Low || this.powerState === PowerState.Critical) {
      // Simplified AI
      return;
    }

    // Full AI when charging or normal battery
    // Complex pathfinding, decision making, etc.
  }
}

// Example 6: Adaptive particle system
class PowerAwareParticleSystem {
  private maxParticles: number;

  constructor(powerManager: PowerStateManager) {
    this.maxParticles = 1000;

    powerManager.onStateChange((state) => {
      switch (state) {
        case PowerState.High:
          this.maxParticles = 2000;
          break;
        case PowerState.Normal:
          this.maxParticles = 1000;
          break;
        case PowerState.Low:
          this.maxParticles = 500;
          break;
        case PowerState.Critical:
          this.maxParticles = 100;
          break;
      }

      this.cullExcessParticles();
    });
  }

  private cullExcessParticles(): void {
    // Remove particles beyond limit
  }
}

// Example 7: Background behavior
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // App is backgrounded
    gameLoop.pause();

    // Stop audio
    audioManager.pauseAll();

    // Reduce network activity
    networkManager.setBackgroundMode(true);
  } else {
    // App is foregrounded
    gameLoop.resume();
    audioManager.resumeAll();
    networkManager.setBackgroundMode(false);
  }
});
```

## Checklist

- [ ] Implement PowerStateManager
- [ ] Create adaptive quality system
- [ ] Add frame rate throttling
- [ ] Implement thermal management
- [ ] Create battery-aware systems
- [ ] Handle background/foreground
- [ ] Test on low battery devices
- [ ] Profile power consumption
- [ ] Add user quality overrides
- [ ] Document power states

## Common Pitfalls

1. **No background pause**: Drains battery when hidden
2. **Fixed frame rate**: Not adapting to power state
3. **Ignoring thermal**: Device overheats
4. **No user override**: Can't disable power saving
5. **Aggressive throttling**: Poor UX on battery
6. **Missing visibility API**: Runs when hidden
7. **No quality presets**: Poor default settings

## Performance Tips

### Power Efficiency
- Pause rendering when hidden
- Reduce frame rate on battery
- Disable expensive effects on low power
- Throttle non-critical systems
- Use visibility API

### Thermal Management
- Monitor frame time drops
- Reduce quality when throttling detected
- Provide cooldown periods
- Limit burst rendering
- Profile on device

### Battery Optimization
- Adaptive quality based on battery level
- Lower resolution on battery
- Disable post-processing
- Reduce particle counts
- Limit active lights

### Mobile Considerations
- Default to battery-friendly settings
- Make power saving optional
- Smooth transitions between quality levels
- Test on older devices
- Profile actual battery usage

## Related Skills

- `mobile-performance` - General mobile optimization
- `mobile-memory-management` - Memory efficiency
- `threejs-performance-profiling` - Performance monitoring
- `threejs-best-practices` - Rendering optimization
- `ecs-performance` - System optimization

## References

- Battery Status API
- Page Visibility API
- Thermal throttling detection
- Mobile power profiling
