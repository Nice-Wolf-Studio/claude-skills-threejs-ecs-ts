---
name: mobile-performance
description: Comprehensive mobile optimization strategies for Three.js games including device detection, quality scaling, thermal throttling, and battery management
---

# Mobile Performance Optimization

## When to Use

Use this skill when:
- Building Three.js games for mobile devices
- Optimizing existing games for mobile
- Implementing adaptive quality settings
- Dealing with thermal throttling
- Reducing battery consumption

## Core Principles

1. **Device Detection**: Detect device capabilities and adjust accordingly
2. **Quality Scaling**: Implement multiple quality presets
3. **Frame Rate Management**: Adaptive FPS based on performance
4. **Thermal Awareness**: Reduce load when device heats up
5. **Battery Optimization**: Lower quality on battery power
6. **Progressive Enhancement**: Start low, scale up if possible

## Implementation

### 1. Device Detector

```typescript
export interface DeviceCapabilities {
  tier: 'low' | 'medium' | 'high';
  gpu: string;
  maxTexture Size: number;
  supportsWebGL2: boolean;
  isMobile: boolean;
  isIOS: boolean;
  isAndroid: boolean;
  memory: number; // GB
  cores: number;
  pixelRatio: number;
}

export class DeviceDetector {
  private capabilities: DeviceCapabilities;

  constructor() {
    this.capabilities = this.detect();
  }

  private detect(): DeviceCapabilities {
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl2') || canvas.getContext('webgl');

    if (!gl) {
      throw new Error('WebGL not supported');
    }

    const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
    const gpu = debugInfo
      ? gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL)
      : 'Unknown';

    const maxTextureSize = gl.getParameter(gl.MAX_TEXTURE_SIZE);
    const supportsWebGL2 = !!canvas.getContext('webgl2');

    // Device detection
    const ua = navigator.userAgent.toLowerCase();
    const isMobile = /mobile|android|iphone|ipad|ipod/.test(ua);
    const isIOS = /iphone|ipad|ipod/.test(ua);
    const isAndroid = /android/.test(ua);

    // Memory estimation (GB)
    const memory = (navigator as any).deviceMemory || this.estimateMemory(gpu);

    // CPU cores
    const cores = navigator.hardwareConcurrency || 4;

    // Pixel ratio (cap at 2 for performance)
    const pixelRatio = Math.min(window.devicePixelRatio, 2);

    // Determine device tier
    const tier = this.calculateTier(gpu, memory, cores, isMobile);

    return {
      tier,
      gpu,
      maxTextureSize,
      supportsWebGL2,
      isMobile,
      isIOS,
      isAndroid,
      memory,
      cores,
      pixelRatio,
    };
  }

  private estimateMemory(gpu: string): number {
    const gpuLower = gpu.toLowerCase();

    // High-end
    if (/adreno 6|mali-g7|apple a1[2-9]|m[1-9]/.test(gpuLower)) {
      return 6;
    }

    // Mid-range
    if (/adreno 5|mali-g5|apple a[9-11]/.test(gpuLower)) {
      return 4;
    }

    // Low-end
    return 2;
  }

  private calculateTier(
    gpu: string,
    memory: number,
    cores: number,
    isMobile: boolean
  ): 'low' | 'medium' | 'high' {
    const gpuLower = gpu.toLowerCase();

    // High-end devices
    if (
      memory >= 6 &&
      cores >= 6 &&
      (/adreno 6|mali-g7|apple a1[2-9]|m[1-9]|rtx|radeon rx/.test(gpuLower))
    ) {
      return 'high';
    }

    // Low-end devices
    if (
      memory <= 2 ||
      cores <= 4 ||
      /adreno [2-4]|mali-[4-5]|apple a[6-8]/.test(gpuLower)
    ) {
      return 'low';
    }

    // Medium by default
    return 'medium';
  }

  getCapabilities(): DeviceCapabilities {
    return this.capabilities;
  }

  getTier(): 'low' | 'medium' | 'high' {
    return this.capabilities.tier;
  }
}
```

### 2. Quality Settings Manager

```typescript
export interface QualitySettings {
  pixelRatio: number;
  shadowMapSize: number;
  enableShadows: boolean;
  enablePostProcessing: boolean;
  enableAntialiasing: boolean;
  textureMaxSize: number;
  maxLights: number;
  renderDistance: number;
  targetFPS: number;
  enableParticles: boolean;
  particleCount: number;
  enableReflections: boolean;
}

export class QualityManager {
  private currentSettings: QualitySettings;
  private readonly presets: Record<'low' | 'medium' | 'high', QualitySettings>;

  constructor() {
    this.presets = {
      low: {
        pixelRatio: 1,
        shadowMapSize: 512,
        enableShadows: false,
        enablePostProcessing: false,
        enableAntialiasing: false,
        textureMaxSize: 512,
        maxLights: 2,
        renderDistance: 50,
        targetFPS: 30,
        enableParticles: false,
        particleCount: 50,
        enableReflections: false,
      },
      medium: {
        pixelRatio: 1,
        shadowMapSize: 1024,
        enableShadows: true,
        enablePostProcessing: false,
        enableAntialiasing: false,
        textureMaxSize: 1024,
        maxLights: 4,
        renderDistance: 100,
        targetFPS: 60,
        enableParticles: true,
        particleCount: 100,
        enableReflections: false,
      },
      high: {
        pixelRatio: 2,
        shadowMapSize: 2048,
        enableShadows: true,
        enablePostProcessing: true,
        enableAntialiasing: true,
        textureMaxSize: 2048,
        maxLights: 8,
        renderDistance: 200,
        targetFPS: 60,
        enableParticles: true,
        particleCount: 500,
        enableReflections: true,
      },
    };

    // Auto-detect and set
    const detector = new DeviceDetector();
    this.currentSettings = this.presets[detector.getTier()];
  }

  setQuality(tier: 'low' | 'medium' | 'high'): void {
    this.currentSettings = { ...this.presets[tier] };
  }

  getSettings(): QualitySettings {
    return this.currentSettings;
  }

  updateSetting<K extends keyof QualitySettings>(
    key: K,
    value: QualitySettings[K]
  ): void {
    this.currentSettings[key] = value;
  }

  applyToRenderer(renderer: THREE.WebGLRenderer): void {
    const settings = this.currentSettings;

    renderer.setPixelRatio(settings.pixelRatio);
    renderer.shadowMap.enabled = settings.enableShadows;

    if (settings.enableShadows) {
      renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    }
  }
}
```

### 3. Performance Monitor

```typescript
export interface PerformanceMetrics {
  fps: number;
  frameTime: number;
  memory: number;
  drawCalls: number;
  triangles: number;
  geometries: number;
  textures: number;
}

export class PerformanceMonitor {
  private fps = 60;
  private frameTime = 16.67;
  private frames = 0;
  private lastTime = performance.now();
  private fpsHistory: number[] = [];
  private readonly historySize = 60;

  update(): void {
    this.frames++;
    const currentTime = performance.now();
    const delta = currentTime - this.lastTime;

    if (delta >= 1000) {
      this.fps = Math.round((this.frames * 1000) / delta);
      this.frameTime = delta / this.frames;

      this.fpsHistory.push(this.fps);
      if (this.fpsHistory.length > this.historySize) {
        this.fpsHistory.shift();
      }

      this.frames = 0;
      this.lastTime = currentTime;
    }
  }

  getMetrics(renderer: THREE.WebGLRenderer): PerformanceMetrics {
    const info = renderer.info;
    const memory = (performance as any).memory;

    return {
      fps: this.fps,
      frameTime: this.frameTime,
      memory: memory ? memory.usedJSHeapSize / 1048576 : 0, // MB
      drawCalls: info.render.calls,
      triangles: info.render.triangles,
      geometries: info.memory.geometries,
      textures: info.memory.textures,
    };
  }

  getAverageFPS(): number {
    if (this.fpsHistory.length === 0) return 60;

    const sum = this.fpsHistory.reduce((a, b) => a + b, 0);
    return Math.round(sum / this.fpsHistory.length);
  }

  isPerformanceLow(): boolean {
    return this.getAverageFPS() < 30;
  }

  isPerformanceGood(): boolean {
    return this.getAverageFPS() >= 55;
  }
}
```

### 4. Adaptive Quality System

```typescript
export class AdaptiveQualitySystem {
  private qualityManager: QualityManager;
  private performanceMonitor: PerformanceMonitor;
  private checkInterval = 5000; // Check every 5 seconds
  private lastCheck = 0;
  private currentTier: 'low' | 'medium' | 'high' = 'medium';

  constructor(
    qualityManager: QualityManager,
    performanceMonitor: PerformanceMonitor
  ) {
    this.qualityManager = qualityManager;
    this.performanceMonitor = performanceMonitor;

    // Start with detected tier
    const detector = new DeviceDetector();
    this.currentTier = detector.getTier();
    this.qualityManager.setQuality(this.currentTier);
  }

  update(time: number): void {
    if (time - this.lastCheck < this.checkInterval) {
      return;
    }

    this.lastCheck = time;
    const avgFPS = this.performanceMonitor.getAverageFPS();

    // Scale down if performance is low
    if (avgFPS < 30 && this.currentTier !== 'low') {
      if (this.currentTier === 'high') {
        this.currentTier = 'medium';
      } else {
        this.currentTier = 'low';
      }

      console.log(`[Adaptive Quality] Reducing to ${this.currentTier} (FPS: ${avgFPS})`);
      this.qualityManager.setQuality(this.currentTier);
    }

    // Scale up if performance is good
    else if (avgFPS >= 55 && this.currentTier !== 'high') {
      if (this.currentTier === 'low') {
        this.currentTier = 'medium';
      } else {
        this.currentTier = 'high';
      }

      console.log(`[Adaptive Quality] Increasing to ${this.currentTier} (FPS: ${avgFPS})`);
      this.qualityManager.setQuality(this.currentTier);
    }
  }

  getCurrentTier(): 'low' | 'medium' | 'high' {
    return this.currentTier;
  }
}
```

### 5. Battery Manager

```typescript
export class BatteryManager {
  private battery: any = null;
  private isCharging = true;
  private level = 1.0;

  async initialize(): Promise<void> {
    if ('getBattery' in navigator) {
      try {
        this.battery = await (navigator as any).getBattery();
        this.isCharging = this.battery.charging;
        this.level = this.battery.level;

        this.battery.addEventListener('chargingchange', () => {
          this.isCharging = this.battery.charging;
        });

        this.battery.addEventListener('levelchange', () => {
          this.level = this.battery.level;
        });
      } catch (error) {
        console.warn('Battery API not available:', error);
      }
    }
  }

  shouldReduceQuality(): boolean {
    // Reduce quality if battery is low and not charging
    return !this.isCharging && this.level < 0.2;
  }

  getLevel(): number {
    return this.level;
  }

  isDeviceCharging(): boolean {
    return this.isCharging;
  }
}
```

## Usage Examples

```typescript
// Device detection
const detector = new DeviceDetector();
const caps = detector.getCapabilities();
console.log(`Device tier: ${caps.tier}, GPU: ${caps.gpu}`);

// Quality management
const qualityManager = new QualityManager();
qualityManager.applyToRenderer(renderer);

// Performance monitoring
const perfMonitor = new PerformanceMonitor();

function animate() {
  perfMonitor.update();

  const metrics = perfMonitor.getMetrics(renderer);
  console.log(`FPS: ${metrics.fps}, Draw calls: ${metrics.drawCalls}`);

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

// Adaptive quality
const adaptiveQuality = new AdaptiveQualitySystem(qualityManager, perfMonitor);

function animateWithAdaptive(time: number) {
  adaptiveQuality.update(time);
  perfMonitor.update();

  renderer.render(scene, camera);
  requestAnimationFrame(animateWithAdaptive);
}

// Battery awareness
const batteryManager = new BatteryManager();
await batteryManager.initialize();

if (batteryManager.shouldReduceQuality()) {
  qualityManager.setQuality('low');
}
```

## Checklist

- [ ] Detect device capabilities on startup
- [ ] Set quality preset based on device tier
- [ ] Monitor FPS and frame time
- [ ] Implement adaptive quality scaling
- [ ] Check battery status and adjust quality
- [ ] Cap pixel ratio at 2.0 maximum
- [ ] Disable shadows on low-end devices
- [ ] Reduce texture sizes on mobile
- [ ] Limit particle count on mobile
- [ ] Implement render distance culling
- [ ] Use simpler materials on low tier
- [ ] Display FPS counter in development

## Common Pitfalls

1. **Not capping pixel ratio**: Excessive pixels on high-DPI displays
2. **Static quality settings**: Doesn't adapt to thermal throttling
3. **Too aggressive scaling**: Quality drops too quickly
4. **Ignoring battery status**: Drains battery quickly
5. **Not monitoring metrics**: Can't detect performance issues
6. **Testing only on high-end**: Misses low-end performance

## Performance Tips

- Start with low quality, scale up based on metrics
- Cap pixel ratio at 1.0-1.5 on mobile
- Disable shadows on devices < 4GB RAM
- Use texture compression (KTX2/Basis)
- Limit draw calls to <100 on mobile
- Implement aggressive LOD on mobile
- Reduce render distance on low-end devices
- Use object pooling extensively
- Implement spatial partitioning/octree
- Profile with Chrome DevTools on real devices

## Related Skills

- `threejs-scene-setup` - Renderer configuration
- `threejs-optimization` - General optimization techniques
- `touch-input-handling` - Mobile input
- `texture-management` - Texture optimization
- `battery-optimization` - Advanced battery management
