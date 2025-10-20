---
name: threejs-particles
description: Particle systems for Three.js including GPU particles, instanced particles, particle emitters, and effects
---

# Three.js Particles

## When to Use

Use this skill when:
- Creating particle effects (fire, smoke, rain, snow)
- Implementing explosion/impact effects
- Building magic/ability visual effects
- Creating ambient particles (dust, sparkles)
- Optimizing particle rendering for mobile
- Managing large particle counts (1000+)

## Core Principles

1. **GPU Acceleration**: Use Points or InstancedMesh for performance
2. **Particle Pooling**: Reuse particles instead of creating/destroying
3. **Batching**: Group similar particles in one draw call
4. **LOD**: Reduce particle count at distance
5. **Culling**: Don't update off-screen particles
6. **Mobile Limits**: Cap at 100-500 particles on mobile

## Implementation

### 1. Basic Particle System

```typescript
// particles/ParticleSystem.ts
import * as THREE from 'three';

export interface ParticleConfig {
  count?: number;
  texture?: THREE.Texture;
  color?: THREE.Color;
  size?: number;
  sizeAttenuation?: boolean;
  transparent?: boolean;
  blending?: THREE.Blending;
  depthWrite?: boolean;
}

export interface Particle {
  position: THREE.Vector3;
  velocity: THREE.Vector3;
  acceleration: THREE.Vector3;
  lifetime: number;
  age: number;
  size: number;
  color: THREE.Color;
  alpha: number;
}

export class ParticleSystem {
  private geometry: THREE.BufferGeometry;
  private material: THREE.PointsMaterial;
  private points: THREE.Points;
  private particles: Particle[] = [];
  private count: number;

  private positions: Float32Array;
  private colors: Float32Array;
  private sizes: Float32Array;

  constructor(config: ParticleConfig = {}) {
    this.count = config.count ?? 1000;

    // Create buffers
    this.positions = new Float32Array(this.count * 3);
    this.colors = new Float32Array(this.count * 3);
    this.sizes = new Float32Array(this.count);

    // Geometry
    this.geometry = new THREE.BufferGeometry();
    this.geometry.setAttribute(
      'position',
      new THREE.BufferAttribute(this.positions, 3)
    );
    this.geometry.setAttribute('color', new THREE.BufferAttribute(this.colors, 3));
    this.geometry.setAttribute('size', new THREE.BufferAttribute(this.sizes, 1));

    // Material
    this.material = new THREE.PointsMaterial({
      size: config.size ?? 0.1,
      sizeAttenuation: config.sizeAttenuation ?? true,
      transparent: config.transparent ?? true,
      vertexColors: true,
      blending: config.blending ?? THREE.AdditiveBlending,
      depthWrite: config.depthWrite ?? false,
      map: config.texture,
    });

    // Points
    this.points = new THREE.Points(this.geometry, this.material);

    // Initialize particle pool
    for (let i = 0; i < this.count; i++) {
      this.particles.push({
        position: new THREE.Vector3(),
        velocity: new THREE.Vector3(),
        acceleration: new THREE.Vector3(),
        lifetime: 0,
        age: Number.MAX_VALUE, // Inactive
        size: 0.1,
        color: new THREE.Color(0xffffff),
        alpha: 1,
      });
    }
  }

  emit(
    position: THREE.Vector3,
    velocity: THREE.Vector3,
    lifetime: number,
    options: {
      size?: number;
      color?: THREE.Color;
      alpha?: number;
      acceleration?: THREE.Vector3;
    } = {}
  ): void {
    // Find inactive particle
    const particle = this.particles.find((p) => p.age >= p.lifetime);
    if (!particle) return;

    // Reset particle
    particle.position.copy(position);
    particle.velocity.copy(velocity);
    particle.acceleration.copy(options.acceleration || new THREE.Vector3(0, 0, 0));
    particle.lifetime = lifetime;
    particle.age = 0;
    particle.size = options.size ?? 0.1;
    particle.color.copy(options.color || new THREE.Color(0xffffff));
    particle.alpha = options.alpha ?? 1;
  }

  update(deltaTime: number): void {
    let activeCount = 0;

    for (let i = 0; i < this.count; i++) {
      const particle = this.particles[i];

      if (particle.age >= particle.lifetime) {
        // Inactive - move far away
        this.positions[i * 3] = 1000000;
        this.positions[i * 3 + 1] = 1000000;
        this.positions[i * 3 + 2] = 1000000;
        continue;
      }

      // Update physics
      particle.velocity.add(particle.acceleration.clone().multiplyScalar(deltaTime));
      particle.position.add(particle.velocity.clone().multiplyScalar(deltaTime));
      particle.age += deltaTime;

      // Update buffers
      this.positions[i * 3] = particle.position.x;
      this.positions[i * 3 + 1] = particle.position.y;
      this.positions[i * 3 + 2] = particle.position.z;

      this.colors[i * 3] = particle.color.r;
      this.colors[i * 3 + 1] = particle.color.g;
      this.colors[i * 3 + 2] = particle.color.b;

      // Fade out over lifetime
      const lifetimeRatio = particle.age / particle.lifetime;
      this.material.opacity = particle.alpha * (1 - lifetimeRatio);

      this.sizes[i] = particle.size;

      activeCount++;
    }

    // Mark buffers for update
    this.geometry.attributes.position.needsUpdate = true;
    this.geometry.attributes.color.needsUpdate = true;
    this.geometry.attributes.size.needsUpdate = true;

    // Update draw range for performance
    this.geometry.setDrawRange(0, activeCount);
  }

  getMesh(): THREE.Points {
    return this.points;
  }

  dispose(): void {
    this.geometry.dispose();
    this.material.dispose();
  }
}
```

### 2. Particle Emitter

```typescript
// particles/ParticleEmitter.ts
import * as THREE from 'three';
import { ParticleSystem } from './ParticleSystem';

export interface EmitterConfig {
  rate?: number; // Particles per second
  lifetime?: number; // Particle lifetime in seconds
  position?: THREE.Vector3;
  positionSpread?: THREE.Vector3;
  velocity?: THREE.Vector3;
  velocitySpread?: THREE.Vector3;
  acceleration?: THREE.Vector3;
  size?: number;
  sizeSpread?: number;
  color?: THREE.Color;
  colorSpread?: THREE.Color;
  texture?: THREE.Texture;
  blending?: THREE.Blending;
}

export class ParticleEmitter {
  private particleSystem: ParticleSystem;
  private config: Required<EmitterConfig>;
  private emitTimer = 0;
  private enabled = true;

  constructor(particleSystem: ParticleSystem, config: EmitterConfig = {}) {
    this.particleSystem = particleSystem;

    this.config = {
      rate: config.rate ?? 10,
      lifetime: config.lifetime ?? 1,
      position: config.position ?? new THREE.Vector3(),
      positionSpread: config.positionSpread ?? new THREE.Vector3(0.1, 0.1, 0.1),
      velocity: config.velocity ?? new THREE.Vector3(0, 1, 0),
      velocitySpread: config.velocitySpread ?? new THREE.Vector3(0.1, 0.1, 0.1),
      acceleration: config.acceleration ?? new THREE.Vector3(0, -1, 0),
      size: config.size ?? 0.1,
      sizeSpread: config.sizeSpread ?? 0.05,
      color: config.color ?? new THREE.Color(0xffffff),
      colorSpread: config.colorSpread ?? new THREE.Color(0, 0, 0),
      texture: config.texture ?? null,
      blending: config.blending ?? THREE.AdditiveBlending,
    };
  }

  update(deltaTime: number): void {
    if (!this.enabled) return;

    this.emitTimer += deltaTime;

    const emitInterval = 1 / this.config.rate;

    while (this.emitTimer >= emitInterval) {
      this.emitParticle();
      this.emitTimer -= emitInterval;
    }
  }

  private emitParticle(): void {
    const position = this.config.position.clone().add(
      new THREE.Vector3(
        (Math.random() - 0.5) * this.config.positionSpread.x,
        (Math.random() - 0.5) * this.config.positionSpread.y,
        (Math.random() - 0.5) * this.config.positionSpread.z
      )
    );

    const velocity = this.config.velocity.clone().add(
      new THREE.Vector3(
        (Math.random() - 0.5) * this.config.velocitySpread.x,
        (Math.random() - 0.5) * this.config.velocitySpread.y,
        (Math.random() - 0.5) * this.config.velocitySpread.z
      )
    );

    const size = this.config.size + (Math.random() - 0.5) * this.config.sizeSpread;

    const color = this.config.color.clone().add(
      new THREE.Color(
        (Math.random() - 0.5) * this.config.colorSpread.r,
        (Math.random() - 0.5) * this.config.colorSpread.g,
        (Math.random() - 0.5) * this.config.colorSpread.b
      )
    );

    this.particleSystem.emit(position, velocity, this.config.lifetime, {
      size,
      color,
      acceleration: this.config.acceleration,
    });
  }

  setPosition(position: THREE.Vector3): void {
    this.config.position.copy(position);
  }

  setEnabled(enabled: boolean): void {
    this.enabled = enabled;
  }

  burst(count: number): void {
    for (let i = 0; i < count; i++) {
      this.emitParticle();
    }
  }
}
```

### 3. GPU Particle System (Custom Shader)

```typescript
// particles/GPUParticleSystem.ts
import * as THREE from 'three';

export class GPUParticleSystem {
  private geometry: THREE.BufferGeometry;
  private material: THREE.ShaderMaterial;
  private points: THREE.Points;
  private count: number;

  constructor(count: number) {
    this.count = count;

    // Buffers
    const positions = new Float32Array(count * 3);
    const velocities = new Float32Array(count * 3);
    const lifetimes = new Float32Array(count);
    const ages = new Float32Array(count);

    // Random initialization
    for (let i = 0; i < count; i++) {
      positions[i * 3] = (Math.random() - 0.5) * 10;
      positions[i * 3 + 1] = Math.random() * 10;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 10;

      velocities[i * 3] = (Math.random() - 0.5) * 0.5;
      velocities[i * 3 + 1] = Math.random() * 2;
      velocities[i * 3 + 2] = (Math.random() - 0.5) * 0.5;

      lifetimes[i] = 1 + Math.random() * 2;
      ages[i] = Math.random() * lifetimes[i];
    }

    // Geometry
    this.geometry = new THREE.BufferGeometry();
    this.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    this.geometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));
    this.geometry.setAttribute('lifetime', new THREE.BufferAttribute(lifetimes, 1));
    this.geometry.setAttribute('age', new THREE.BufferAttribute(ages, 1));

    // Shader material
    this.material = new THREE.ShaderMaterial({
      uniforms: {
        time: { value: 0 },
        deltaTime: { value: 0 },
        gravity: { value: new THREE.Vector3(0, -9.8, 0) },
      },
      vertexShader: `
        attribute vec3 velocity;
        attribute float lifetime;
        attribute float age;

        uniform float time;
        uniform float deltaTime;
        uniform vec3 gravity;

        varying float vLifetimeRatio;

        void main() {
          vLifetimeRatio = age / lifetime;

          vec3 pos = position;
          pos += velocity * age;
          pos += 0.5 * gravity * age * age;

          vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
          gl_Position = projectionMatrix * mvPosition;

          gl_PointSize = 50.0 * (1.0 - vLifetimeRatio) / -mvPosition.z;
        }
      `,
      fragmentShader: `
        varying float vLifetimeRatio;

        void main() {
          float alpha = 1.0 - vLifetimeRatio;
          float dist = length(gl_PointCoord - vec2(0.5));

          if (dist > 0.5) discard;

          gl_FragColor = vec4(1.0, 0.5, 0.0, alpha * (1.0 - dist * 2.0));
        }
      `,
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    });

    this.points = new THREE.Points(this.geometry, this.material);
  }

  update(deltaTime: number): void {
    this.material.uniforms.deltaTime.value = deltaTime;
    this.material.uniforms.time.value += deltaTime;

    const ages = this.geometry.attributes.age.array as Float32Array;
    const lifetimes = this.geometry.attributes.lifetime.array as Float32Array;

    for (let i = 0; i < this.count; i++) {
      ages[i] += deltaTime;

      if (ages[i] >= lifetimes[i]) {
        ages[i] = 0;
      }
    }

    this.geometry.attributes.age.needsUpdate = true;
  }

  getMesh(): THREE.Points {
    return this.points;
  }

  dispose(): void {
    this.geometry.dispose();
    this.material.dispose();
  }
}
```

### 4. Particle Effect Presets

```typescript
// particles/ParticlePresets.ts
import * as THREE from 'three';
import { EmitterConfig } from './ParticleEmitter';

export const ParticlePresets = {
  fire: {
    rate: 50,
    lifetime: 1.5,
    positionSpread: new THREE.Vector3(0.2, 0, 0.2),
    velocity: new THREE.Vector3(0, 2, 0),
    velocitySpread: new THREE.Vector3(0.5, 1, 0.5),
    acceleration: new THREE.Vector3(0, 0.5, 0),
    size: 0.3,
    sizeSpread: 0.1,
    color: new THREE.Color(0xff4400),
    colorSpread: new THREE.Color(0.2, 0.1, 0),
    blending: THREE.AdditiveBlending,
  } as EmitterConfig,

  smoke: {
    rate: 20,
    lifetime: 3,
    positionSpread: new THREE.Vector3(0.1, 0, 0.1),
    velocity: new THREE.Vector3(0, 1, 0),
    velocitySpread: new THREE.Vector3(0.2, 0.5, 0.2),
    acceleration: new THREE.Vector3(0, 0.1, 0),
    size: 0.5,
    sizeSpread: 0.2,
    color: new THREE.Color(0x666666),
    colorSpread: new THREE.Color(0.1, 0.1, 0.1),
    blending: THREE.NormalBlending,
  } as EmitterConfig,

  rain: {
    rate: 100,
    lifetime: 2,
    positionSpread: new THREE.Vector3(5, 0, 5),
    velocity: new THREE.Vector3(0, -10, 0),
    velocitySpread: new THREE.Vector3(0.5, 1, 0.5),
    acceleration: new THREE.Vector3(0, -5, 0),
    size: 0.05,
    sizeSpread: 0.01,
    color: new THREE.Color(0x4488ff),
    colorSpread: new THREE.Color(0, 0, 0),
    blending: THREE.AdditiveBlending,
  } as EmitterConfig,

  snow: {
    rate: 50,
    lifetime: 5,
    positionSpread: new THREE.Vector3(5, 0, 5),
    velocity: new THREE.Vector3(0, -1, 0),
    velocitySpread: new THREE.Vector3(0.5, 0.2, 0.5),
    acceleration: new THREE.Vector3(0, -0.1, 0),
    size: 0.1,
    sizeSpread: 0.05,
    color: new THREE.Color(0xffffff),
    colorSpread: new THREE.Color(0, 0, 0),
    blending: THREE.NormalBlending,
  } as EmitterConfig,

  explosion: {
    rate: 0, // Use burst() instead
    lifetime: 0.5,
    positionSpread: new THREE.Vector3(0, 0, 0),
    velocity: new THREE.Vector3(0, 0, 0),
    velocitySpread: new THREE.Vector3(5, 5, 5),
    acceleration: new THREE.Vector3(0, -5, 0),
    size: 0.2,
    sizeSpread: 0.1,
    color: new THREE.Color(0xff8800),
    colorSpread: new THREE.Color(0.2, 0, 0),
    blending: THREE.AdditiveBlending,
  } as EmitterConfig,

  sparkle: {
    rate: 30,
    lifetime: 0.5,
    positionSpread: new THREE.Vector3(0.5, 0.5, 0.5),
    velocity: new THREE.Vector3(0, 0, 0),
    velocitySpread: new THREE.Vector3(1, 1, 1),
    acceleration: new THREE.Vector3(0, 0, 0),
    size: 0.1,
    sizeSpread: 0.05,
    color: new THREE.Color(0xffff00),
    colorSpread: new THREE.Color(0.2, 0.2, 0),
    blending: THREE.AdditiveBlending,
  } as EmitterConfig,
};
```

## Usage Examples

```typescript
// Example 1: Basic particle system
import { ParticleSystem } from './particles/ParticleSystem';
import { ParticleEmitter } from './particles/ParticleEmitter';
import { ParticlePresets } from './particles/ParticlePresets';

const particleSystem = new ParticleSystem({ count: 1000 });
scene.add(particleSystem.getMesh());

const fireEmitter = new ParticleEmitter(particleSystem, ParticlePresets.fire);
fireEmitter.setPosition(new THREE.Vector3(0, 0, 0));

function animate() {
  fireEmitter.update(deltaTime);
  particleSystem.update(deltaTime);
}

// Example 2: Explosion burst
const explosionEmitter = new ParticleEmitter(
  particleSystem,
  ParticlePresets.explosion
);
explosionEmitter.setPosition(explosionPosition);
explosionEmitter.burst(100);

// Example 3: GPU particles
import { GPUParticleSystem } from './particles/GPUParticleSystem';

const gpuParticles = new GPUParticleSystem(10000);
scene.add(gpuParticles.getMesh());

function animate() {
  gpuParticles.update(deltaTime);
}

// Example 4: Dynamic rain
const rainEmitter = new ParticleEmitter(particleSystem, ParticlePresets.rain);

// Follow player
function animate() {
  rainEmitter.setPosition(player.position.clone().add(new THREE.Vector3(0, 10, 0)));
  rainEmitter.update(deltaTime);
  particleSystem.update(deltaTime);
}
```

## Checklist

- [ ] Choose particle count based on device (100-500 mobile, 1000-10000 desktop)
- [ ] Use particle pooling (don't create/destroy)
- [ ] Set up emitter with appropriate rate and lifetime
- [ ] Configure blending mode (Additive for glowing, Normal for solid)
- [ ] Add textures for better visual quality
- [ ] Implement particle culling (don't update off-screen)
- [ ] Test performance on target devices
- [ ] Use GPU particles for large counts (10000+)
- [ ] Add LOD (reduce count at distance)
- [ ] Profile particle system performance

## Common Pitfalls

1. **Too many particles**: Kills framerate
2. **Creating/destroying particles**: Use pooling
3. **No blending**: Particles look flat
4. **depthWrite = true**: Sorting issues with transparent particles
5. **Updating inactive particles**: Wasted CPU
6. **No culling**: Update particles behind camera
7. **Large textures**: Memory waste on particles

## Performance Tips

### Particle Counts
- Mobile low-end: 100-200
- Mobile high-end: 500-1000
- Desktop: 1000-5000
- High-end desktop: 10000+

### Optimization Strategies
- Use `Points` instead of individual meshes
- Pool particles (reuse instead of create/destroy)
- Batch similar particles in one system
- Use GPU particles for counts > 5000
- Implement frustum culling
- Use lower update rates (30Hz instead of 60Hz)
- Disable particles when far from camera
- Use texture atlases for multiple particle types
- Set `depthWrite: false` on material
- Use `AdditiveBlending` for glowing effects

### Mobile Specific
- Cap at 100-500 particles total
- Use simpler shaders (no custom shaders)
- Lower texture resolution (32x32 or 64x64)
- Reduce emission rate
- Shorter lifetime
- Disable particles entirely on very low-end

## Related Skills

- `threejs-texture-management` - Particle textures
- `mobile-performance` - Mobile optimization
- `threejs-material-systems` - Material setup
- `threejs-geometry-management` - BufferGeometry usage
