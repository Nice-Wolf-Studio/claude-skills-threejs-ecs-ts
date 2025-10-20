---
name: threejs-post-processing
description: Post-processing effects for Three.js including bloom, SSAO, DOF, tone mapping, and custom shader passes
---

# Three.js Post-Processing

## When to Use

Use this skill when:
- Adding visual effects (bloom, blur, etc.)
- Implementing depth of field or motion blur
- Creating cinematic look with tone mapping
- Adding screen-space effects (SSAO, SSR)
- Building custom post-processing passes
- Optimizing post-processing for mobile

## Core Principles

1. **Selective Use**: Only add effects that enhance gameplay
2. **Performance Budget**: Each pass costs performance
3. **Mobile Considerations**: Limit passes on mobile
4. **Quality Settings**: Provide low/medium/high options
5. **Shader Optimization**: Minimize texture reads
6. **Resolution Scaling**: Use lower resolution for expensive effects

## Implementation

### 1. Post-Processing Manager

```typescript
// post-processing/PostProcessingManager.ts
import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass';
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass';
import { SSAOPass } from 'three/examples/jsm/postprocessing/SSAOPass';
import { SAOPass } from 'three/examples/jsm/postprocessing/SAOPass';
import { FXAAShader } from 'three/examples/jsm/shaders/FXAAShader';
import { GammaCorrectionShader } from 'three/examples/jsm/shaders/GammaCorrectionShader';

export interface PostProcessingConfig {
  bloom?: {
    enabled: boolean;
    strength?: number;
    radius?: number;
    threshold?: number;
  };
  ssao?: {
    enabled: boolean;
    kernelRadius?: number;
    minDistance?: number;
    maxDistance?: number;
  };
  fxaa?: {
    enabled: boolean;
  };
  gammaCorrection?: {
    enabled: boolean;
  };
  toneMapping?: {
    enabled: boolean;
    exposure?: number;
  };
}

export class PostProcessingManager {
  private renderer: THREE.WebGLRenderer;
  private scene: THREE.Scene;
  private camera: THREE.Camera;
  private composer: EffectComposer;
  private passes = new Map<string, any>();

  constructor(
    renderer: THREE.WebGLRenderer,
    scene: THREE.Scene,
    camera: THREE.Camera
  ) {
    this.renderer = renderer;
    this.scene = scene;
    this.camera = camera;

    this.composer = new EffectComposer(renderer);

    // Add render pass (always first)
    const renderPass = new RenderPass(scene, camera);
    this.composer.addPass(renderPass);
    this.passes.set('render', renderPass);
  }

  configure(config: PostProcessingConfig): void {
    // Remove all passes except render
    const renderPass = this.passes.get('render');
    this.composer.passes = [renderPass];

    // Bloom
    if (config.bloom?.enabled) {
      const bloomPass = new UnrealBloomPass(
        new THREE.Vector2(window.innerWidth, window.innerHeight),
        config.bloom.strength ?? 1.5,
        config.bloom.radius ?? 0.4,
        config.bloom.threshold ?? 0.85
      );
      this.composer.addPass(bloomPass);
      this.passes.set('bloom', bloomPass);
    }

    // SSAO
    if (config.ssao?.enabled) {
      const ssaoPass = new SSAOPass(
        this.scene,
        this.camera as THREE.PerspectiveCamera,
        window.innerWidth,
        window.innerHeight
      );

      ssaoPass.kernelRadius = config.ssao.kernelRadius ?? 16;
      ssaoPass.minDistance = config.ssao.minDistance ?? 0.005;
      ssaoPass.maxDistance = config.ssao.maxDistance ?? 0.1;

      this.composer.addPass(ssaoPass);
      this.passes.set('ssao', ssaoPass);
    }

    // FXAA
    if (config.fxaa?.enabled) {
      const fxaaPass = new ShaderPass(FXAAShader);
      const pixelRatio = this.renderer.getPixelRatio();

      fxaaPass.material.uniforms['resolution'].value.x =
        1 / (window.innerWidth * pixelRatio);
      fxaaPass.material.uniforms['resolution'].value.y =
        1 / (window.innerHeight * pixelRatio);

      this.composer.addPass(fxaaPass);
      this.passes.set('fxaa', fxaaPass);
    }

    // Gamma correction
    if (config.gammaCorrection?.enabled) {
      const gammaCorrectionPass = new ShaderPass(GammaCorrectionShader);
      this.composer.addPass(gammaCorrectionPass);
      this.passes.set('gammaCorrection', gammaCorrectionPass);
    }

    // Tone mapping
    if (config.toneMapping?.enabled) {
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
      this.renderer.toneMappingExposure = config.toneMapping.exposure ?? 1.0;
    }
  }

  getPass(name: string): any {
    return this.passes.get(name);
  }

  render(deltaTime?: number): void {
    this.composer.render(deltaTime);
  }

  setSize(width: number, height: number): void {
    this.composer.setSize(width, height);

    // Update FXAA resolution
    const fxaaPass = this.passes.get('fxaa');
    if (fxaaPass) {
      const pixelRatio = this.renderer.getPixelRatio();
      fxaaPass.material.uniforms['resolution'].value.x =
        1 / (width * pixelRatio);
      fxaaPass.material.uniforms['resolution'].value.y =
        1 / (height * pixelRatio);
    }
  }

  dispose(): void {
    this.composer.passes.forEach((pass) => {
      if ('dispose' in pass) {
        (pass as any).dispose();
      }
    });
    this.passes.clear();
  }
}
```

### 2. Custom Shader Pass

```typescript
// post-processing/CustomShaderPass.ts
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass';
import * as THREE from 'three';

// Vignette shader
export const VignetteShader = {
  uniforms: {
    tDiffuse: { value: null },
    offset: { value: 1.0 },
    darkness: { value: 1.0 },
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
    uniform float offset;
    uniform float darkness;

    varying vec2 vUv;

    void main() {
      vec4 texel = texture2D(tDiffuse, vUv);
      vec2 uv = (vUv - vec2(0.5)) * vec2(offset);
      float vignette = 1.0 - dot(uv, uv);
      vignette = clamp(pow(vignette, darkness), 0.0, 1.0);

      gl_FragColor = vec4(texel.rgb * vignette, texel.a);
    }
  `,
};

// Film grain shader
export const FilmGrainShader = {
  uniforms: {
    tDiffuse: { value: null },
    time: { value: 0.0 },
    intensity: { value: 0.5 },
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
    uniform float time;
    uniform float intensity;

    varying vec2 vUv;

    float random(vec2 st) {
      return fract(sin(dot(st.xy, vec2(12.9898, 78.233))) * 43758.5453123);
    }

    void main() {
      vec4 texel = texture2D(tDiffuse, vUv);
      float noise = random(vUv * time);

      gl_FragColor = vec4(texel.rgb + noise * intensity, texel.a);
    }
  `,
};

// Color grading shader
export const ColorGradingShader = {
  uniforms: {
    tDiffuse: { value: null },
    brightness: { value: 0.0 },
    contrast: { value: 1.0 },
    saturation: { value: 1.0 },
    hue: { value: 0.0 },
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
    uniform float brightness;
    uniform float contrast;
    uniform float saturation;
    uniform float hue;

    varying vec2 vUv;

    vec3 rgb2hsv(vec3 c) {
      vec4 K = vec4(0.0, -1.0 / 3.0, 2.0 / 3.0, -1.0);
      vec4 p = mix(vec4(c.bg, K.wz), vec4(c.gb, K.xy), step(c.b, c.g));
      vec4 q = mix(vec4(p.xyw, c.r), vec4(c.r, p.yzx), step(p.x, c.r));

      float d = q.x - min(q.w, q.y);
      float e = 1.0e-10;
      return vec3(abs(q.z + (q.w - q.y) / (6.0 * d + e)), d / (q.x + e), q.x);
    }

    vec3 hsv2rgb(vec3 c) {
      vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
      vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
      return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
    }

    void main() {
      vec4 texel = texture2D(tDiffuse, vUv);
      vec3 color = texel.rgb;

      // Brightness
      color += brightness;

      // Contrast
      color = (color - 0.5) * contrast + 0.5;

      // Saturation & Hue
      vec3 hsv = rgb2hsv(color);
      hsv.y *= saturation;
      hsv.x += hue;
      color = hsv2rgb(hsv);

      gl_FragColor = vec4(color, texel.a);
    }
  `,
};
```

### 3. Depth of Field

```typescript
// post-processing/DOFPass.ts
import * as THREE from 'three';
import { Pass } from 'three/examples/jsm/postprocessing/Pass';

export class DOFPass extends Pass {
  private scene: THREE.Scene;
  private camera: THREE.PerspectiveCamera;
  private renderTarget: THREE.WebGLRenderTarget;
  private depthMaterial: THREE.MeshDepthMaterial;
  private compositeMaterial: THREE.ShaderMaterial;

  public focusDistance = 10;
  public focalLength = 50;
  public fstop = 2.8;

  constructor(scene: THREE.Scene, camera: THREE.PerspectiveCamera) {
    super();

    this.scene = scene;
    this.camera = camera;

    // Render target for depth
    this.renderTarget = new THREE.WebGLRenderTarget(
      window.innerWidth,
      window.innerHeight,
      {
        minFilter: THREE.LinearFilter,
        magFilter: THREE.LinearFilter,
        format: THREE.RGBAFormat,
      }
    );

    // Depth material
    this.depthMaterial = new THREE.MeshDepthMaterial();
    this.depthMaterial.depthPacking = THREE.RGBADepthPacking;
    this.depthMaterial.blending = THREE.NoBlending;

    // Composite shader
    this.compositeMaterial = new THREE.ShaderMaterial({
      uniforms: {
        tDiffuse: { value: null },
        tDepth: { value: this.renderTarget.texture },
        focus: { value: this.focusDistance },
        aperture: { value: this.fstop },
        maxblur: { value: 0.01 },
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
        uniform float focus;
        uniform float aperture;
        uniform float maxblur;

        varying vec2 vUv;

        void main() {
          float depth = texture2D(tDepth, vUv).r;
          float blur = clamp(abs(depth - focus) / aperture, 0.0, maxblur);

          vec4 color = vec4(0.0);
          float total = 0.0;

          for (float x = -4.0; x <= 4.0; x += 1.0) {
            for (float y = -4.0; y <= 4.0; y += 1.0) {
              vec2 offset = vec2(x, y) * blur;
              color += texture2D(tDiffuse, vUv + offset);
              total += 1.0;
            }
          }

          gl_FragColor = color / total;
        }
      `,
    });
  }

  render(
    renderer: THREE.WebGLRenderer,
    writeBuffer: THREE.WebGLRenderTarget,
    readBuffer: THREE.WebGLRenderTarget
  ): void {
    // Render depth
    const originalRenderTarget = renderer.getRenderTarget();
    renderer.setRenderTarget(this.renderTarget);

    const originalOverrideMaterial = this.scene.overrideMaterial;
    this.scene.overrideMaterial = this.depthMaterial;
    renderer.render(this.scene, this.camera);
    this.scene.overrideMaterial = originalOverrideMaterial;

    renderer.setRenderTarget(originalRenderTarget);

    // Composite
    this.compositeMaterial.uniforms.tDiffuse.value = readBuffer.texture;
    this.compositeMaterial.uniforms.focus.value = this.focusDistance;
    this.compositeMaterial.uniforms.aperture.value = this.fstop;

    // Render to screen or writeBuffer
    if (this.renderToScreen) {
      renderer.setRenderTarget(null);
    } else {
      renderer.setRenderTarget(writeBuffer);
      if (this.clear) renderer.clear();
    }

    // Full screen quad
    const quad = new THREE.Mesh(
      new THREE.PlaneGeometry(2, 2),
      this.compositeMaterial
    );
    const quadScene = new THREE.Scene();
    quadScene.add(quad);
    const quadCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);

    renderer.render(quadScene, quadCamera);
  }

  setSize(width: number, height: number): void {
    this.renderTarget.setSize(width, height);
  }

  dispose(): void {
    this.renderTarget.dispose();
    this.depthMaterial.dispose();
    this.compositeMaterial.dispose();
  }
}
```

### 4. Quality Presets

```typescript
// post-processing/QualityPresets.ts
import { PostProcessingConfig } from './PostProcessingManager';

export const PostProcessingPresets = {
  mobile_low: {
    bloom: { enabled: false },
    ssao: { enabled: false },
    fxaa: { enabled: true },
    gammaCorrection: { enabled: true },
    toneMapping: { enabled: true, exposure: 1.0 },
  } as PostProcessingConfig,

  mobile_medium: {
    bloom: {
      enabled: true,
      strength: 0.5,
      radius: 0.2,
      threshold: 0.9,
    },
    ssao: { enabled: false },
    fxaa: { enabled: true },
    gammaCorrection: { enabled: true },
    toneMapping: { enabled: true, exposure: 1.0 },
  } as PostProcessingConfig,

  desktop_medium: {
    bloom: {
      enabled: true,
      strength: 1.0,
      radius: 0.4,
      threshold: 0.85,
    },
    ssao: {
      enabled: true,
      kernelRadius: 8,
      minDistance: 0.005,
      maxDistance: 0.05,
    },
    fxaa: { enabled: true },
    gammaCorrection: { enabled: true },
    toneMapping: { enabled: true, exposure: 1.0 },
  } as PostProcessingConfig,

  desktop_high: {
    bloom: {
      enabled: true,
      strength: 1.5,
      radius: 0.4,
      threshold: 0.85,
    },
    ssao: {
      enabled: true,
      kernelRadius: 16,
      minDistance: 0.005,
      maxDistance: 0.1,
    },
    fxaa: { enabled: true },
    gammaCorrection: { enabled: true },
    toneMapping: { enabled: true, exposure: 1.2 },
  } as PostProcessingConfig,

  cinematic: {
    bloom: {
      enabled: true,
      strength: 2.0,
      radius: 0.5,
      threshold: 0.8,
    },
    ssao: {
      enabled: true,
      kernelRadius: 32,
      minDistance: 0.001,
      maxDistance: 0.2,
    },
    fxaa: { enabled: true },
    gammaCorrection: { enabled: true },
    toneMapping: { enabled: true, exposure: 1.5 },
  } as PostProcessingConfig,
};
```

## Usage Examples

```typescript
// Example 1: Basic post-processing setup
import { PostProcessingManager } from './post-processing/PostProcessingManager';

const ppManager = new PostProcessingManager(renderer, scene, camera);

ppManager.configure({
  bloom: {
    enabled: true,
    strength: 1.5,
    radius: 0.4,
    threshold: 0.85,
  },
  fxaa: { enabled: true },
  gammaCorrection: { enabled: true },
});

function animate() {
  ppManager.render();
}

// Example 2: Quality presets
import { PostProcessingPresets } from './post-processing/QualityPresets';

const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
const preset = isMobile
  ? PostProcessingPresets.mobile_medium
  : PostProcessingPresets.desktop_high;

ppManager.configure(preset);

// Example 3: Custom shader pass
import { VignetteShader } from './post-processing/CustomShaderPass';
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass';

const vignettePass = new ShaderPass(VignetteShader);
vignettePass.uniforms['offset'].value = 1.0;
vignettePass.uniforms['darkness'].value = 1.5;

// Add to composer manually
ppManager.composer.addPass(vignettePass);

// Example 4: DOF
import { DOFPass } from './post-processing/DOFPass';

const dofPass = new DOFPass(scene, camera);
dofPass.focusDistance = 10;
dofPass.focalLength = 50;
dofPass.fstop = 2.8;

ppManager.composer.addPass(dofPass);
```

## Checklist

- [ ] Install `three` (post-processing is in examples/jsm)
- [ ] Set up EffectComposer
- [ ] Add RenderPass as first pass
- [ ] Add desired effects (bloom, SSAO, etc.)
- [ ] Configure effect parameters
- [ ] Add FXAA or SMAA for anti-aliasing
- [ ] Set up quality presets (low/medium/high)
- [ ] Test on target devices
- [ ] Profile performance impact
- [ ] Optimize render target sizes
- [ ] Handle window resize
- [ ] Implement quality switching based on FPS

## Common Pitfalls

1. **Too many passes**: Performance drops quickly
2. **No render target**: Black screen
3. **Wrong pass order**: Effects don't apply correctly
4. **Not updating FXAA resolution**: Blurry on resize
5. **Full resolution on mobile**: Too expensive
6. **No fallback for low-end devices**: Unplayable FPS
7. **Not disposing passes**: Memory leaks

## Performance Tips

### Pass Count
- Mobile: 2-3 passes max (render + FXAA + one effect)
- Desktop: 4-6 passes (render + effects + FXAA)
- High-end: 8+ passes

### Resolution Scaling
- Bloom: Half resolution (0.5x)
- SSAO: Half or quarter resolution
- DOF: Full resolution (needs sharp edges)
- Glow: Quarter resolution

### Mobile Optimization
- Use BasicBloomPass instead of UnrealBloomPass
- Skip SSAO entirely
- Use FXAA instead of SMAA
- Reduce bloom samples
- Lower render target precision (HalfFloatType)

### Effect Priority (cheapest to most expensive)
1. Tone mapping (renderer setting, free)
2. Gamma correction (single pass, very fast)
3. FXAA (single pass, fast)
4. Vignette/Color grading (single pass, fast)
5. Bloom (multiple passes, medium)
6. DOF (multiple passes, expensive)
7. SSAO (very expensive)
8. SSR (very expensive)

## Related Skills

- `threejs-scene-setup` - Renderer setup
- `mobile-performance` - Mobile optimization
- `r3f-performance` - React optimization
- `threejs-shadows` - Shadow effects
