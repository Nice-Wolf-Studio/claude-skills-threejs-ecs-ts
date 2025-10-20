---
name: threejs-scene-setup
description: Set up a Three.js scene with best practices for performance, organization, and maintainability
---

# Three.js Scene Setup

## When to Use

Use this skill when:
- Starting a new Three.js project
- Setting up a new scene in an existing project
- Refactoring scene initialization code
- Ensuring proper resource management and cleanup

## Core Principles

1. **Proper Initialization Order**: Renderer → Scene → Camera → Lights → Objects
2. **Resource Management**: Track and dispose of geometries, materials, and textures
3. **Responsive Design**: Handle window resize events
4. **Performance**: Use efficient rendering settings
5. **Type Safety**: Leverage TypeScript for all Three.js objects

## Implementation Steps

### 1. Basic Scene Structure

```typescript
import * as THREE from 'three';

export class SceneManager {
  private renderer: THREE.WebGLRenderer;
  private scene: THREE.Scene;
  private camera: THREE.PerspectiveCamera;
  private animationFrameId: number | null = null;
  private disposables: Set<THREE.Object3D | THREE.Material | THREE.Texture> = new Set();

  constructor(private container: HTMLElement) {
    this.scene = new THREE.Scene();
    this.camera = this.createCamera();
    this.renderer = this.createRenderer();
    this.setupLights();
    this.setupResizeHandler();
  }

  private createCamera(): THREE.PerspectiveCamera {
    const camera = new THREE.PerspectiveCamera(
      75, // FOV
      this.container.clientWidth / this.container.clientHeight, // Aspect
      0.1, // Near plane
      1000 // Far plane
    );
    camera.position.set(0, 5, 10);
    camera.lookAt(0, 0, 0);
    return camera;
  }

  private createRenderer(): THREE.WebGLRenderer {
    const renderer = new THREE.WebGLRenderer({
      antialias: true,
      alpha: false,
      powerPreference: 'high-performance'
    });

    renderer.setSize(this.container.clientWidth, this.container.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // Cap at 2 for performance
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    renderer.outputColorSpace = THREE.SRGBColorSpace;

    this.container.appendChild(renderer.domElement);
    return renderer;
  }

  private setupLights(): void {
    // Ambient light for overall scene illumination
    const ambient = new THREE.AmbientLight(0xffffff, 0.4);
    this.scene.add(ambient);

    // Directional light for shadows
    const directional = new THREE.DirectionalLight(0xffffff, 0.8);
    directional.position.set(5, 10, 5);
    directional.castShadow = true;
    directional.shadow.mapSize.width = 2048;
    directional.shadow.mapSize.height = 2048;
    this.scene.add(directional);
  }

  private setupResizeHandler(): void {
    window.addEventListener('resize', this.handleResize);
  }

  private handleResize = (): void => {
    const width = this.container.clientWidth;
    const height = this.container.clientHeight;

    this.camera.aspect = width / height;
    this.camera.updateProjectionMatrix();
    this.renderer.setSize(width, height);
  };

  public start(): void {
    if (this.animationFrameId !== null) return;
    this.animate();
  }

  public stop(): void {
    if (this.animationFrameId !== null) {
      cancelAnimationFrame(this.animationFrameId);
      this.animationFrameId = null;
    }
  }

  private animate = (): void => {
    this.animationFrameId = requestAnimationFrame(this.animate);
    this.update();
    this.render();
  };

  protected update(): void {
    // Override in subclasses for game logic
  }

  private render(): void {
    this.renderer.render(this.scene, this.camera);
  }

  public addObject(object: THREE.Object3D): void {
    this.scene.add(object);
    this.trackDisposable(object);
  }

  public removeObject(object: THREE.Object3D): void {
    this.scene.remove(object);
    this.disposeObject(object);
  }

  private trackDisposable(object: THREE.Object3D): void {
    this.disposables.add(object);
    object.traverse((child) => {
      if ((child as THREE.Mesh).geometry) {
        this.disposables.add((child as THREE.Mesh).geometry as any);
      }
      if ((child as THREE.Mesh).material) {
        const material = (child as THREE.Mesh).material;
        if (Array.isArray(material)) {
          material.forEach(m => this.disposables.add(m));
        } else {
          this.disposables.add(material);
        }
      }
    });
  }

  private disposeObject(object: THREE.Object3D): void {
    object.traverse((child) => {
      if ((child as THREE.Mesh).geometry) {
        (child as THREE.Mesh).geometry.dispose();
      }
      if ((child as THREE.Mesh).material) {
        const material = (child as THREE.Mesh).material;
        if (Array.isArray(material)) {
          material.forEach(m => m.dispose());
        } else {
          material.dispose();
        }
      }
    });
  }

  public dispose(): void {
    this.stop();
    window.removeEventListener('resize', this.handleResize);

    // Dispose all tracked resources
    this.disposables.forEach((item) => {
      if ('dispose' in item && typeof item.dispose === 'function') {
        item.dispose();
      }
    });
    this.disposables.clear();

    // Clean up scene
    this.scene.clear();
    this.renderer.dispose();
    this.container.removeChild(this.renderer.domElement);
  }
}
```

### 2. Usage Example

```typescript
// Initialize
const container = document.getElementById('canvas-container')!;
const sceneManager = new SceneManager(container);

// Add objects
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
sceneManager.addObject(cube);

// Start rendering
sceneManager.start();

// Clean up on unmount/exit
window.addEventListener('beforeunload', () => {
  sceneManager.dispose();
});
```

## Checklist

- [ ] Create renderer with appropriate settings
- [ ] Initialize scene and camera with proper parameters
- [ ] Set up basic lighting (ambient + directional)
- [ ] Implement window resize handler
- [ ] Create animation loop with start/stop controls
- [ ] Implement resource tracking for disposal
- [ ] Add dispose method for cleanup
- [ ] Test on different screen sizes
- [ ] Verify no memory leaks on hot reload
- [ ] Cap pixel ratio at 2 for performance

## Common Pitfalls

1. **Not disposing resources**: Always dispose geometries, materials, and textures
2. **Unbounded pixel ratio**: Can cause performance issues on high-DPI displays
3. **Missing resize handler**: Scene won't adapt to window size changes
4. **No animation loop control**: Can't pause/resume rendering
5. **Improper camera setup**: Results in objects not visible or clipped

## Performance Tips

- Use `powerPreference: 'high-performance'` for dedicated GPU
- Cap pixel ratio at 2 to avoid excessive pixels on 4K+ displays
- Enable shadow maps only when needed
- Use `PCFSoftShadowMap` for quality/performance balance
- Track and dispose all Three.js objects to prevent memory leaks

## Related Skills

- `threejs-optimization` - Advanced performance optimization
- `threejs-lighting` - Advanced lighting setups
- `typescript-game-types` - Type-safe patterns for game objects
