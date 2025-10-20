---
name: threejs-performance-profiling
description: Performance profiling and optimization for Three.js including FPS monitoring, draw call analysis, memory profiling, and bottleneck identification
---

# Three.js Performance Profiling

## When to Use

Use this skill when:
- Diagnosing performance issues
- Optimizing for target framerate (30/60 FPS)
- Identifying bottlenecks (CPU vs GPU)
- Monitoring memory usage
- Analyzing draw calls and vertex counts
- Profiling on mobile devices

## Core Principles

1. **Measure First**: Profile before optimizing
2. **Target Metrics**: Set clear performance goals
3. **Bottleneck Identification**: CPU vs GPU bound
4. **Continuous Monitoring**: Track performance over time
5. **Device Testing**: Profile on target devices
6. **Frame Budget**: 16ms for 60fps, 33ms for 30fps

## Implementation

### 1. Performance Monitor

```typescript
// profiling/PerformanceMonitor.ts
import * as THREE from 'three';

export interface PerformanceMetrics {
  fps: number;
  frameTime: number;
  drawCalls: number;
  triangles: number;
  points: number;
  lines: number;
  textures: number;
  geometries: number;
  programs: number;
  memoryUsed?: number;
}

export class PerformanceMonitor {
  private renderer: THREE.WebGLRenderer;
  private lastTime = performance.now();
  private frames = 0;
  private fpsUpdateInterval = 1000; // Update every 1 second

  public metrics: PerformanceMetrics = {
    fps: 60,
    frameTime: 16,
    drawCalls: 0,
    triangles: 0,
    points: 0,
    lines: 0,
    textures: 0,
    geometries: 0,
    programs: 0,
  };

  constructor(renderer: THREE.WebGLRenderer) {
    this.renderer = renderer;
  }

  update(): void {
    this.frames++;
    const currentTime = performance.now();
    const delta = currentTime - this.lastTime;

    if (delta >= this.fpsUpdateInterval) {
      this.metrics.fps = Math.round((this.frames * 1000) / delta);
      this.metrics.frameTime = delta / this.frames;

      this.frames = 0;
      this.lastTime = currentTime;
    }

    // Renderer info
    const info = this.renderer.info;

    this.metrics.drawCalls = info.render.calls;
    this.metrics.triangles = info.render.triangles;
    this.metrics.points = info.render.points;
    this.metrics.lines = info.render.lines;

    this.metrics.textures = info.memory.textures;
    this.metrics.geometries = info.memory.geometries;

    this.metrics.programs = info.programs?.length ?? 0;

    // Memory (if available)
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      this.metrics.memoryUsed = memory.usedJSHeapSize / 1048576; // MB
    }
  }

  getMetrics(): PerformanceMetrics {
    return { ...this.metrics };
  }

  reset(): void {
    this.renderer.info.reset();
  }
}
```

### 2. Performance HUD

```typescript
// profiling/PerformanceHUD.ts
import { PerformanceMonitor } from './PerformanceMonitor';

export class PerformanceHUD {
  private monitor: PerformanceMonitor;
  private container: HTMLDivElement;
  private updateInterval = 100; // Update UI every 100ms
  private lastUpdate = 0;

  constructor(monitor: PerformanceMonitor) {
    this.monitor = monitor;

    // Create HUD container
    this.container = document.createElement('div');
    this.container.style.cssText = `
      position: fixed;
      top: 10px;
      right: 10px;
      background: rgba(0, 0, 0, 0.8);
      color: #fff;
      font-family: monospace;
      font-size: 12px;
      padding: 10px;
      border-radius: 5px;
      z-index: 10000;
      pointer-events: none;
      min-width: 200px;
    `;

    document.body.appendChild(this.container);
  }

  update(): void {
    const now = performance.now();
    if (now - this.lastUpdate < this.updateInterval) return;

    this.lastUpdate = now;

    const metrics = this.monitor.getMetrics();

    const fpsColor = metrics.fps >= 55 ? '#0f0' : metrics.fps >= 30 ? '#ff0' : '#f00';

    this.container.innerHTML = `
      <div style="margin-bottom: 5px; font-weight: bold; color: ${fpsColor}">
        FPS: ${metrics.fps}
      </div>
      <div>Frame Time: ${metrics.frameTime.toFixed(2)}ms</div>
      <div>Draw Calls: ${metrics.drawCalls}</div>
      <div>Triangles: ${metrics.triangles.toLocaleString()}</div>
      ${metrics.points > 0 ? `<div>Points: ${metrics.points.toLocaleString()}</div>` : ''}
      ${metrics.lines > 0 ? `<div>Lines: ${metrics.lines.toLocaleString()}</div>` : ''}
      <div style="margin-top: 5px; border-top: 1px solid #444; padding-top: 5px">
        Textures: ${metrics.textures}
      </div>
      <div>Geometries: ${metrics.geometries}</div>
      <div>Programs: ${metrics.programs}</div>
      ${
        metrics.memoryUsed !== undefined
          ? `<div style="margin-top: 5px; border-top: 1px solid #444; padding-top: 5px">
        Memory: ${metrics.memoryUsed.toFixed(0)}MB
      </div>`
          : ''
      }
    `;
  }

  show(): void {
    this.container.style.display = 'block';
  }

  hide(): void {
    this.container.style.display = 'none';
  }

  destroy(): void {
    document.body.removeChild(this.container);
  }
}
```

### 3. Frame Time Graph

```typescript
// profiling/FrameTimeGraph.ts
export class FrameTimeGraph {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private frameTimes: number[] = [];
  private maxSamples = 100;
  private width = 300;
  private height = 100;

  constructor() {
    this.canvas = document.createElement('canvas');
    this.canvas.width = this.width;
    this.canvas.height = this.height;
    this.canvas.style.cssText = `
      position: fixed;
      bottom: 10px;
      right: 10px;
      background: rgba(0, 0, 0, 0.8);
      border: 1px solid #444;
      border-radius: 5px;
      z-index: 10000;
      pointer-events: none;
    `;

    this.ctx = this.canvas.getContext('2d')!;

    document.body.appendChild(this.canvas);
  }

  addSample(frameTime: number): void {
    this.frameTimes.push(frameTime);

    if (this.frameTimes.length > this.maxSamples) {
      this.frameTimes.shift();
    }

    this.draw();
  }

  private draw(): void {
    const ctx = this.ctx;

    // Clear
    ctx.clearRect(0, 0, this.width, this.height);

    if (this.frameTimes.length === 0) return;

    // Draw grid lines
    ctx.strokeStyle = '#333';
    ctx.lineWidth = 1;

    // 60 FPS line (16.67ms)
    const fps60Y = this.height - (16.67 / 50) * this.height;
    ctx.beginPath();
    ctx.moveTo(0, fps60Y);
    ctx.lineTo(this.width, fps60Y);
    ctx.stroke();

    // 30 FPS line (33.33ms)
    const fps30Y = this.height - (33.33 / 50) * this.height;
    ctx.beginPath();
    ctx.moveTo(0, fps30Y);
    ctx.lineTo(this.width, fps30Y);
    ctx.stroke();

    // Draw frame times
    const barWidth = this.width / this.maxSamples;

    this.frameTimes.forEach((time, i) => {
      const x = i * barWidth;
      const barHeight = Math.min((time / 50) * this.height, this.height);
      const y = this.height - barHeight;

      // Color based on performance
      if (time < 16.67) {
        ctx.fillStyle = '#0f0'; // Green: 60+ FPS
      } else if (time < 33.33) {
        ctx.fillStyle = '#ff0'; // Yellow: 30-60 FPS
      } else {
        ctx.fillStyle = '#f00'; // Red: <30 FPS
      }

      ctx.fillRect(x, y, barWidth - 1, barHeight);
    });

    // Draw labels
    ctx.fillStyle = '#fff';
    ctx.font = '10px monospace';
    ctx.fillText('60 FPS', 5, fps60Y - 2);
    ctx.fillText('30 FPS', 5, fps30Y - 2);
  }

  show(): void {
    this.canvas.style.display = 'block';
  }

  hide(): void {
    this.canvas.style.display = 'none';
  }

  destroy(): void {
    document.body.removeChild(this.canvas);
  }
}
```

### 4. Draw Call Analyzer

```typescript
// profiling/DrawCallAnalyzer.ts
import * as THREE from 'three';

export interface DrawCallReport {
  totalCalls: number;
  callsByMaterial: Map<string, number>;
  callsByGeometry: Map<string, number>;
  suggestions: string[];
}

export class DrawCallAnalyzer {
  private scene: THREE.Scene;

  constructor(scene: THREE.Scene) {
    this.scene = scene;
  }

  analyze(): DrawCallReport {
    const callsByMaterial = new Map<string, number>();
    const callsByGeometry = new Map<string, number>();
    const suggestions: string[] = [];

    let totalCalls = 0;
    let instancedMeshCount = 0;
    let regularMeshCount = 0;

    this.scene.traverse((object) => {
      if (object instanceof THREE.Mesh) {
        totalCalls++;

        // Material tracking
        const materialKey = object.material.uuid;
        callsByMaterial.set(materialKey, (callsByMaterial.get(materialKey) || 0) + 1);

        // Geometry tracking
        const geometryKey = object.geometry.uuid;
        callsByGeometry.set(geometryKey, (callsByGeometry.get(geometryKey) || 0) + 1);

        if (object instanceof THREE.InstancedMesh) {
          instancedMeshCount++;
        } else {
          regularMeshCount++;
        }
      }
    });

    // Generate suggestions
    if (totalCalls > 100) {
      suggestions.push(
        `High draw call count (${totalCalls}). Consider instancing or batching.`
      );
    }

    callsByGeometry.forEach((count, geometryKey) => {
      if (count > 10) {
        suggestions.push(
          `Geometry ${geometryKey.slice(0, 8)} used ${count} times. Consider using InstancedMesh.`
        );
      }
    });

    callsByMaterial.forEach((count, materialKey) => {
      if (count > 20) {
        suggestions.push(
          `Material ${materialKey.slice(0, 8)} used ${count} times. Meshes may be over-split.`
        );
      }
    });

    if (regularMeshCount > 50 && instancedMeshCount === 0) {
      suggestions.push(
        'No instanced meshes found. Consider using InstancedMesh for repeated objects.'
      );
    }

    return {
      totalCalls,
      callsByMaterial,
      callsByGeometry,
      suggestions,
    };
  }

  printReport(): void {
    const report = this.analyze();

    console.group('Draw Call Analysis');
    console.log(`Total Draw Calls: ${report.totalCalls}`);

    console.groupCollapsed('Calls by Material');
    report.callsByMaterial.forEach((count, materialKey) => {
      console.log(`${materialKey.slice(0, 8)}: ${count} calls`);
    });
    console.groupEnd();

    console.groupCollapsed('Calls by Geometry');
    report.callsByGeometry.forEach((count, geometryKey) => {
      console.log(`${geometryKey.slice(0, 8)}: ${count} meshes`);
    });
    console.groupEnd();

    if (report.suggestions.length > 0) {
      console.group('Suggestions');
      report.suggestions.forEach((suggestion) => console.warn(suggestion));
      console.groupEnd();
    }

    console.groupEnd();
  }
}
```

### 5. Memory Profiler

```typescript
// profiling/MemoryProfiler.ts
import * as THREE from 'three';

export interface MemoryReport {
  textures: {
    count: number;
    totalSize: number; // Bytes
    largest: { size: number; width: number; height: number } | null;
  };
  geometries: {
    count: number;
    totalVertices: number;
    totalIndices: number;
  };
  materials: {
    count: number;
  };
  jsHeap?: {
    used: number; // MB
    total: number; // MB
    limit: number; // MB
  };
}

export class MemoryProfiler {
  private scene: THREE.Scene;
  private renderer: THREE.WebGLRenderer;

  constructor(scene: THREE.Scene, renderer: THREE.WebGLRenderer) {
    this.scene = scene;
    this.renderer = renderer;
  }

  analyze(): MemoryReport {
    const textures: THREE.Texture[] = [];
    const geometries: THREE.BufferGeometry[] = [];
    const materials: THREE.Material[] = [];

    this.scene.traverse((object) => {
      if (object instanceof THREE.Mesh) {
        if (!geometries.includes(object.geometry)) {
          geometries.push(object.geometry);
        }

        const material = Array.isArray(object.material)
          ? object.material
          : [object.material];

        material.forEach((mat) => {
          if (!materials.includes(mat)) {
            materials.push(mat);

            // Collect textures
            Object.values(mat).forEach((value) => {
              if (value instanceof THREE.Texture && !textures.includes(value)) {
                textures.push(value);
              }
            });
          }
        });
      }
    });

    // Texture analysis
    let totalTextureSize = 0;
    let largestTexture: { size: number; width: number; height: number } | null = null;

    textures.forEach((texture) => {
      const image = texture.image;
      if (image && image.width && image.height) {
        const bytesPerPixel = 4; // RGBA
        const size = image.width * image.height * bytesPerPixel;
        totalTextureSize += size;

        if (!largestTexture || size > largestTexture.size) {
          largestTexture = { size, width: image.width, height: image.height };
        }
      }
    });

    // Geometry analysis
    let totalVertices = 0;
    let totalIndices = 0;

    geometries.forEach((geometry) => {
      const positions = geometry.attributes.position;
      if (positions) {
        totalVertices += positions.count;
      }

      const index = geometry.index;
      if (index) {
        totalIndices += index.count;
      }
    });

    // JS Heap (if available)
    let jsHeap: MemoryReport['jsHeap'];

    if ('memory' in performance) {
      const memory = (performance as any).memory;
      jsHeap = {
        used: memory.usedJSHeapSize / 1048576,
        total: memory.totalJSHeapSize / 1048576,
        limit: memory.jsHeapSizeLimit / 1048576,
      };
    }

    return {
      textures: {
        count: textures.length,
        totalSize: totalTextureSize,
        largest: largestTexture,
      },
      geometries: {
        count: geometries.length,
        totalVertices,
        totalIndices,
      },
      materials: {
        count: materials.length,
      },
      jsHeap,
    };
  }

  printReport(): void {
    const report = this.analyze();

    console.group('Memory Profiling');

    console.group('Textures');
    console.log(`Count: ${report.textures.count}`);
    console.log(
      `Total Size: ${(report.textures.totalSize / 1048576).toFixed(2)} MB`
    );
    if (report.textures.largest) {
      console.log(
        `Largest: ${report.textures.largest.width}x${report.textures.largest.height} (${(report.textures.largest.size / 1048576).toFixed(2)} MB)`
      );
    }
    console.groupEnd();

    console.group('Geometries');
    console.log(`Count: ${report.geometries.count}`);
    console.log(`Total Vertices: ${report.geometries.totalVertices.toLocaleString()}`);
    console.log(`Total Indices: ${report.geometries.totalIndices.toLocaleString()}`);
    console.groupEnd();

    console.group('Materials');
    console.log(`Count: ${report.materials.count}`);
    console.groupEnd();

    if (report.jsHeap) {
      console.group('JavaScript Heap');
      console.log(`Used: ${report.jsHeap.used.toFixed(0)} MB`);
      console.log(`Total: ${report.jsHeap.total.toFixed(0)} MB`);
      console.log(`Limit: ${report.jsHeap.limit.toFixed(0)} MB`);
      console.log(
        `Usage: ${((report.jsHeap.used / report.jsHeap.limit) * 100).toFixed(1)}%`
      );
      console.groupEnd();
    }

    console.groupEnd();
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic performance monitoring
import { PerformanceMonitor } from './profiling/PerformanceMonitor';
import { PerformanceHUD } from './profiling/PerformanceHUD';

const perfMonitor = new PerformanceMonitor(renderer);
const perfHUD = new PerformanceHUD(perfMonitor);

function animate() {
  perfMonitor.update();
  perfHUD.update();

  renderer.render(scene, camera);
}

// Example 2: Frame time graph
import { FrameTimeGraph } from './profiling/FrameTimeGraph';

const frameGraph = new FrameTimeGraph();

let lastFrameTime = performance.now();

function animate() {
  const currentTime = performance.now();
  const frameTime = currentTime - lastFrameTime;
  lastFrameTime = currentTime;

  frameGraph.addSample(frameTime);

  renderer.render(scene, camera);
}

// Example 3: Draw call analysis
import { DrawCallAnalyzer } from './profiling/DrawCallAnalyzer';

const analyzer = new DrawCallAnalyzer(scene);
analyzer.printReport();

// Example 4: Memory profiling
import { MemoryProfiler } from './profiling/MemoryProfiler';

const memoryProfiler = new MemoryProfiler(scene, renderer);
memoryProfiler.printReport();

// Example 5: Conditional quality adjustment
const perfMonitor = new PerformanceMonitor(renderer);

setInterval(() => {
  const metrics = perfMonitor.getMetrics();

  if (metrics.fps < 30) {
    // Reduce quality
    renderer.setPixelRatio(1);
    shadowManager.setQuality('low');
  } else if (metrics.fps > 55) {
    // Increase quality
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    shadowManager.setQuality('medium');
  }
}, 5000);
```

## Checklist

- [ ] Set up performance monitor
- [ ] Add HUD for real-time metrics
- [ ] Profile draw calls
- [ ] Analyze memory usage
- [ ] Track FPS over time
- [ ] Test on target devices
- [ ] Identify bottlenecks (CPU vs GPU)
- [ ] Set performance budget (30fps or 60fps)
- [ ] Implement adaptive quality
- [ ] Profile before and after optimizations

## Common Pitfalls

1. **Optimizing without profiling**: Wasted effort
2. **Only testing on high-end**: Misses performance issues
3. **Ignoring memory**: Crashes on mobile
4. **No performance budget**: Unclear goals
5. **Not testing on real devices**: Desktop != Mobile
6. **Profiling in dev mode**: Different from production
7. **Single frame profiling**: Need sustained performance

## Performance Tips

### Profiling Best Practices
- Profile on target devices (not just dev machine)
- Test in production build (dev builds are slower)
- Profile over time (not just single frames)
- Use Chrome DevTools Performance tab
- Monitor memory over long sessions
- Test worst-case scenarios (many objects, particles, etc.)

### Metrics to Track
- **FPS**: Target 60fps (desktop), 30fps (mobile)
- **Frame Time**: <16ms (60fps), <33ms (30fps)
- **Draw Calls**: <100 (mobile), <500 (desktop)
- **Triangles**: <100k (mobile), <1M (desktop)
- **Textures**: <100MB total
- **Memory**: <500MB (mobile), <2GB (desktop)

### Bottleneck Identification
- **CPU Bound**: High draw calls, complex logic
  - Reduce draw calls
  - Optimize update loops
  - Use instancing
- **GPU Bound**: High poly count, complex shaders
  - Reduce polygons
  - Simplify shaders
  - Lower resolution
- **Memory Bound**: Large textures, many geometries
  - Compress textures
  - Share geometries
  - Dispose unused resources

### Tools
- Chrome DevTools Performance tab
- Three.js Stats.js
- renderer.info object
- Spector.js (WebGL debugging)
- GPU profilers (browser specific)

## Related Skills

- `threejs-scene-setup` - Renderer setup
- `mobile-performance` - Mobile optimization
- `r3f-performance` - React optimization
- `threejs-geometry-management` - Geometry optimization
