---
name: r3f-performance
description: React Three Fiber performance optimization with memoization, instancing, LOD, suspense, and mobile-specific optimizations
---

# React Three Fiber Performance

## When to Use

Use this skill when:
- Optimizing R3F applications for mobile
- Dealing with performance bottlenecks
- Rendering many objects (100+)
- Implementing adaptive quality
- Reducing frame drops and stuttering

## Core Principles

1. **Measure First**: Profile before optimizing
2. **React Optimization**: Prevent unnecessary re-renders
3. **Three.js Optimization**: Reduce draw calls, vertices
4. **Adaptive Quality**: Scale down on low-end devices
5. **Lazy Loading**: Load assets progressively
6. **Frame Budget**: Stay under 16ms (60fps) or 33ms (30fps)

## Implementation

### 1. React Memoization

```typescript
// components/OptimizedComponents.tsx
import { memo, useMemo, useCallback } from 'react';
import * as THREE from 'three';

// Memoize expensive components
export const MemoizedBox = memo(function Box({
  position,
  color,
}: {
  position: [number, number, number];
  color: string;
}) {
  // Memoize geometry (created once)
  const geometry = useMemo(() => new THREE.BoxGeometry(1, 1, 1), []);

  // Memoize material (recreated only when color changes)
  const material = useMemo(
    () => new THREE.MeshStandardMaterial({ color }),
    [color]
  );

  return <mesh position={position} geometry={geometry} material={material} />;
});

// Custom comparison function
export const SmartMemoBox = memo(
  function Box(props: { position: [number, number, number]; color: string }) {
    const geometry = useMemo(() => new THREE.BoxGeometry(1, 1, 1), []);
    const material = useMemo(
      () => new THREE.MeshStandardMaterial({ color: props.color }),
      [props.color]
    );

    return <mesh position={props.position} geometry={geometry} material={material} />;
  },
  (prev, next) => {
    // Only re-render if color changes (ignore position changes)
    return prev.color === next.color;
  }
);
```

### 2. Instancing for Many Objects

```typescript
// components/InstancedObjects.tsx
import { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import { InstancedMesh, Object3D, Matrix4 } from 'three';

interface InstancedObjectsProps {
  count: number;
  spread: number;
}

export function InstancedObjects({ count, spread }: InstancedObjectsProps) {
  const meshRef = useRef<InstancedMesh>(null);

  // Initialize instances
  const { positions, rotations, matrices } = useMemo(() => {
    const positions = new Float32Array(count * 3);
    const rotations = new Float32Array(count * 3);
    const matrices: Matrix4[] = [];

    const dummy = new Object3D();

    for (let i = 0; i < count; i++) {
      // Random positions
      positions[i * 3] = (Math.random() - 0.5) * spread;
      positions[i * 3 + 1] = (Math.random() - 0.5) * spread;
      positions[i * 3 + 2] = (Math.random() - 0.5) * spread;

      // Random rotations
      rotations[i * 3] = Math.random() * Math.PI;
      rotations[i * 3 + 1] = Math.random() * Math.PI;
      rotations[i * 3 + 2] = Math.random() * Math.PI;

      // Set matrix
      dummy.position.set(
        positions[i * 3],
        positions[i * 3 + 1],
        positions[i * 3 + 2]
      );
      dummy.rotation.set(
        rotations[i * 3],
        rotations[i * 3 + 1],
        rotations[i * 3 + 2]
      );
      dummy.updateMatrix();

      matrices.push(dummy.matrix.clone());
    }

    return { positions, rotations, matrices };
  }, [count, spread]);

  // Apply matrices on mount
  useMemo(() => {
    if (!meshRef.current) return;

    matrices.forEach((matrix, i) => {
      meshRef.current!.setMatrixAt(i, matrix);
    });
    meshRef.current.instanceMatrix.needsUpdate = true;
  }, [matrices]);

  // Animate (optional - remove if static)
  useFrame((state, delta) => {
    if (!meshRef.current) return;

    const dummy = new Object3D();

    for (let i = 0; i < count; i++) {
      meshRef.current.getMatrixAt(i, dummy.matrix);
      dummy.matrix.decompose(dummy.position, dummy.quaternion, dummy.scale);

      // Rotate
      rotations[i * 3 + 1] += delta * 0.5;
      dummy.rotation.y = rotations[i * 3 + 1];

      dummy.updateMatrix();
      meshRef.current.setMatrixAt(i, dummy.matrix);
    }

    meshRef.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh ref={meshRef} args={[undefined, undefined, count]}>
      <boxGeometry />
      <meshStandardMaterial />
    </instancedMesh>
  );
}
```

### 3. LOD (Level of Detail)

```typescript
// components/LODObject.tsx
import { useRef, useState } from 'react';
import { useFrame, useThree } from '@react-three/fiber';
import { Mesh, Vector3 } from 'three';

interface LODObjectProps {
  position: [number, number, number];
}

export function LODObject({ position }: LODObjectProps) {
  const { camera } = useThree();
  const meshRef = useRef<Mesh>(null);
  const [lod, setLOD] = useState<'high' | 'medium' | 'low'>('high');

  // Calculate LOD based on distance
  useFrame(() => {
    if (!meshRef.current) return;

    const distance = camera.position.distanceTo(
      new Vector3(...position)
    );

    if (distance < 10) {
      setLOD('high');
    } else if (distance < 30) {
      setLOD('medium');
    } else {
      setLOD('low');
    }
  });

  const geometry = {
    high: <sphereGeometry args={[1, 32, 32]} />,
    medium: <sphereGeometry args={[1, 16, 16]} />,
    low: <sphereGeometry args={[1, 8, 8]} />,
  }[lod];

  return (
    <mesh ref={meshRef} position={position}>
      {geometry}
      <meshStandardMaterial />
    </mesh>
  );
}
```

### 4. Adaptive Quality

```typescript
// components/AdaptiveCanvas.tsx
import { Canvas } from '@react-three/fiber';
import { AdaptiveDpr, AdaptiveEvents, PerformanceMonitor } from '@react-three/drei';
import { useState, ReactNode } from 'react';

interface AdaptiveCanvasProps {
  children: ReactNode;
}

export function AdaptiveCanvas({ children }: AdaptiveCanvasProps) {
  const [dpr, setDpr] = useState(1.5);

  return (
    <Canvas
      dpr={dpr}
      performance={{ min: 0.5 }}
      gl={{ powerPreference: 'high-performance' }}
    >
      {/* Automatically adjust DPR based on performance */}
      <AdaptiveDpr pixelated />

      {/* Reduce event sampling rate when performance drops */}
      <AdaptiveEvents />

      {/* Monitor performance and adjust quality */}
      <PerformanceMonitor
        onIncline={() => setDpr(2)}
        onDecline={() => setDpr(1)}
      >
        {children}
      </PerformanceMonitor>
    </Canvas>
  );
}
```

### 5. Suspense and Lazy Loading

```typescript
// components/LazyModel.tsx
import { Suspense } from 'react';
import { useGLTF, Preload } from '@react-three/drei';

function Model({ url }: { url: string }) {
  const { scene } = useGLTF(url);
  return <primitive object={scene} />;
}

export function LazyLoadedScene() {
  return (
    <Suspense fallback={<LoadingPlaceholder />}>
      <Model url="/models/heavy-model.glb" />

      {/* Preload other models in background */}
      <Preload all />
    </Suspense>
  );
}

function LoadingPlaceholder() {
  return (
    <mesh>
      <boxGeometry />
      <meshBasicMaterial color="gray" wireframe />
    </mesh>
  );
}
```

### 6. Conditional Frame Updates

```typescript
// hooks/useConditionalFrame.ts
import { useFrame } from '@react-three/fiber';
import { useRef } from 'react';

export function useConditionalFrame(
  callback: (delta: number) => void,
  condition: () => boolean,
  throttle: number = 0
) {
  const lastUpdate = useRef(0);

  useFrame((state, delta) => {
    // Skip if condition not met
    if (!condition()) return;

    // Throttle updates
    if (throttle > 0) {
      const now = state.clock.elapsedTime;
      if (now - lastUpdate.current < throttle) return;
      lastUpdate.current = now;
    }

    callback(delta);
  });
}

// Usage
function ThrottledAnimation() {
  const meshRef = useRef<Mesh>(null);

  useConditionalFrame(
    (delta) => {
      if (meshRef.current) {
        meshRef.current.rotation.y += delta;
      }
    },
    () => meshRef.current !== null,
    0.1 // Update every 100ms instead of every frame
  );

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

### 7. Frustum Culling

```typescript
// components/FrustumCulled.tsx
import { useFrame, useThree } from '@react-three/fiber';
import { useRef, useState } from 'react';
import { Mesh, Box3, Frustum, Matrix4 } from 'three';

interface FrustumCulledProps {
  children: JSX.Element;
  position: [number, number, number];
}

export function FrustumCulled({ children, position }: FrustumCulledProps) {
  const { camera } = useThree();
  const meshRef = useRef<Mesh>(null);
  const [visible, setVisible] = useState(true);

  useFrame(() => {
    if (!meshRef.current) return;

    const frustum = new Frustum();
    frustum.setFromProjectionMatrix(
      new Matrix4().multiplyMatrices(
        camera.projectionMatrix,
        camera.matrixWorldInverse
      )
    );

    meshRef.current.geometry.computeBoundingBox();
    const box = new Box3().setFromObject(meshRef.current);

    setVisible(frustum.intersectsBox(box));
  });

  return visible ? (
    <group ref={meshRef} position={position}>
      {children}
    </group>
  ) : null;
}
```

### 8. Performance Monitoring

```typescript
// components/PerformanceStats.tsx
import { useFrame } from '@react-three/fiber';
import { useState } from 'react';

export function PerformanceStats() {
  const [fps, setFps] = useState(60);
  const [frameTime, setFrameTime] = useState(16);

  let lastTime = performance.now();
  let frames = 0;

  useFrame(() => {
    frames++;
    const currentTime = performance.now();
    const delta = currentTime - lastTime;

    if (delta >= 1000) {
      setFps(Math.round((frames * 1000) / delta));
      setFrameTime(delta / frames);
      frames = 0;
      lastTime = currentTime;
    }
  });

  return (
    <Html position={[0, 5, 0]}>
      <div style={{ color: 'white', background: 'rgba(0,0,0,0.8)', padding: '10px' }}>
        <div>FPS: {fps}</div>
        <div>Frame Time: {frameTime.toFixed(2)}ms</div>
        <div>Status: {fps >= 55 ? '✅ Good' : fps >= 30 ? '⚠️ OK' : '❌ Poor'}</div>
      </div>
    </Html>
  );
}
```

## Usage Examples

```typescript
// Optimized game scene
import { AdaptiveCanvas } from './components/AdaptiveCanvas';
import { InstancedObjects } from './components/InstancedObjects';
import { LODObject } from './components/LODObject';

export default function Game() {
  return (
    <AdaptiveCanvas>
      <Suspense fallback={null}>
        {/* 1000 instanced objects (single draw call) */}
        <InstancedObjects count={1000} spread={100} />

        {/* LOD objects at various distances */}
        {Array.from({ length: 10 }).map((_, i) => (
          <LODObject key={i} position={[i * 5, 0, i * 5]} />
        ))}

        {/* Heavy model with lazy loading */}
        <Model url="/models/character.glb" />
      </Suspense>

      <PerformanceStats />
    </AdaptiveCanvas>
  );
}
```

## Checklist

- [ ] Use React.memo for components
- [ ] Memoize geometries and materials
- [ ] Use InstancedMesh for 100+ objects
- [ ] Implement LOD for distant objects
- [ ] Enable AdaptiveDpr
- [ ] Use Suspense for lazy loading
- [ ] Implement frustum culling
- [ ] Monitor FPS and frame time
- [ ] Throttle expensive frame updates
- [ ] Profile with React DevTools
- [ ] Test on target mobile devices
- [ ] Set performance budget (60fps or 30fps)

## Common Pitfalls

1. **Creating objects in render**: Use useMemo
2. **Not using instancing**: Rendering objects one by one
3. **No LOD**: High poly at all distances
4. **Updating everything every frame**: Use conditionals
5. **Not profiling**: Optimizing the wrong things
6. **Tight coupling**: Hard to memoize

## Performance Tips

### React Level
- Use `React.memo` aggressively
- Memoize props with `useMemo`/`useCallback`
- Avoid inline functions in JSX
- Use context sparingly (causes many re-renders)
- Implement shouldComponentUpdate equivalent

### Three.js Level
- Use `InstancedMesh` for 100+ identical objects
- Implement LOD (2-3 detail levels)
- Merge static geometries
- Use texture atlases
- Limit material count (<50)
- Use `BufferGeometry` always
- Enable frustum culling
- Use `renderer.info` to monitor draw calls

### Mobile Specific
- Cap DPR at 1-1.5
- Use simpler materials (Lambert over Standard)
- Disable shadows on low-end
- Reduce texture sizes (512-1024px max)
- Limit particle count (<100)
- Use AdaptiveEvents
- Implement aggressive LOD
- Monitor memory usage

### Profiling
- React DevTools Profiler
- Chrome DevTools Performance tab
- Three.js renderer.info object
- FPS counter in development
- Test on real devices

## Related Skills

- `r3f-component-patterns` - Component design
- `mobile-performance` - Mobile optimization
- `threejs-optimization` - Three.js specific optimization
- `r3f-state-management` - State optimization
