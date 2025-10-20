---
name: r3f-component-patterns
description: Advanced React Three Fiber component patterns including reusable components, hooks, composition, and performance optimization
---

# React Three Fiber Component Patterns

## When to Use

Use this skill when:
- Building reusable 3D components with R3F
- Structuring React Three.js applications
- Implementing component composition
- Creating custom R3F hooks
- Optimizing React component performance

## Core Principles

1. **Declarative**: Define 3D scenes with JSX
2. **Reusable**: Create component libraries
3. **Composable**: Build complex scenes from simple parts
4. **Type-Safe**: Full TypeScript support
5. **Performance**: Use React.memo and useMemo
6. **Props-Driven**: Configure via props, not imperative code

## Implementation

### 1. Basic Component Patterns

```typescript
// components/Box.tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { Mesh } from 'three';

interface BoxProps {
  position?: [number, number, number];
  color?: string;
  scale?: number;
  rotationSpeed?: number;
  onClick?: () => void;
}

export function Box({
  position = [0, 0, 0],
  color = 'hotpink',
  scale = 1,
  rotationSpeed = 1,
  onClick,
}: BoxProps) {
  const meshRef = useRef<Mesh>(null);

  useFrame((state, delta) => {
    if (meshRef.current) {
      meshRef.current.rotation.y += delta * rotationSpeed;
    }
  });

  return (
    <mesh ref={meshRef} position={position} scale={scale} onClick={onClick}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color={color} />
    </mesh>
  );
}
```

### 2. Compound Components

```typescript
// components/Player.tsx
import { useRef } from 'react';
import { Group } from 'three';

interface PlayerProps {
  position?: [number, number, number];
  color?: string;
}

export function Player({ position = [0, 0, 0], color = 'blue' }: PlayerProps) {
  const groupRef = useRef<Group>(null);

  return (
    <group ref={groupRef} position={position}>
      {/* Body */}
      <mesh position={[0, 1, 0]} castShadow>
        <boxGeometry args={[1, 2, 1]} />
        <meshStandardMaterial color={color} />
      </mesh>

      {/* Head */}
      <mesh position={[0, 2.5, 0]} castShadow>
        <sphereGeometry args={[0.5, 16, 16]} />
        <meshStandardMaterial color={color} />
      </mesh>

      {/* Arms */}
      <mesh position={[-0.75, 1, 0]} castShadow>
        <boxGeometry args={[0.3, 1.5, 0.3]} />
        <meshStandardMaterial color={color} />
      </mesh>
      <mesh position={[0.75, 1, 0]} castShadow>
        <boxGeometry args={[0.3, 1.5, 0.3]} />
        <meshStandardMaterial color={color} />
      </mesh>
    </group>
  );
}
```

### 3. Container Components

```typescript
// components/Scene.tsx
import { Environment, OrbitControls } from '@react-three/drei';
import { ReactNode } from 'react';

interface SceneProps {
  children: ReactNode;
  enableControls?: boolean;
  environment?: 'sunset' | 'dawn' | 'night' | 'warehouse' | 'forest' | 'apartment';
  shadows?: boolean;
}

export function Scene({
  children,
  enableControls = true,
  environment = 'sunset',
  shadows = true,
}: SceneProps) {
  return (
    <>
      {/* Lighting */}
      <ambientLight intensity={0.4} />
      <directionalLight
        position={[5, 10, 5]}
        intensity={0.8}
        castShadow={shadows}
        shadow-mapSize={[1024, 1024]}
      />

      {/* Environment */}
      <Environment preset={environment} />

      {/* Controls */}
      {enableControls && (
        <OrbitControls
          enableDamping
          dampingFactor={0.05}
          enableZoom
          enablePan={false}
        />
      )}

      {/* Ground */}
      <mesh rotation={[-Math.PI / 2, 0, 0]} receiveShadow={shadows}>
        <planeGeometry args={[100, 100]} />
        <meshStandardMaterial color="#333" />
      </mesh>

      {/* User content */}
      {children}
    </>
  );
}
```

### 4. Render Props Pattern

```typescript
// components/WithHover.tsx
import { useState } from 'react';
import { ThreeEvent } from '@react-three/fiber';

interface WithHoverProps {
  children: (hovered: boolean, handlers: {
    onPointerOver: (e: ThreeEvent<PointerEvent>) => void;
    onPointerOut: (e: ThreeEvent<PointerEvent>) => void;
  }) => JSX.Element;
}

export function WithHover({ children }: WithHoverProps) {
  const [hovered, setHovered] = useState(false);

  const handlers = {
    onPointerOver: (e: ThreeEvent<PointerEvent>) => {
      e.stopPropagation();
      setHovered(true);
    },
    onPointerOut: (e: ThreeEvent<PointerEvent>) => {
      e.stopPropagation();
      setHovered(false);
    },
  };

  return children(hovered, handlers);
}

// Usage
function InteractiveBox() {
  return (
    <WithHover>
      {(hovered, handlers) => (
        <mesh {...handlers}>
          <boxGeometry />
          <meshStandardMaterial color={hovered ? 'orange' : 'hotpink'} />
        </mesh>
      )}
    </WithHover>
  );
}
```

### 5. Custom Hooks

```typescript
// hooks/useAnimation.ts
import { useRef, useEffect } from 'react';
import { useFrame } from '@react-three/fiber';

interface AnimationConfig {
  duration: number;
  loop?: boolean;
  easing?: (t: number) => number;
}

export function useAnimation(
  callback: (progress: number) => void,
  config: AnimationConfig
) {
  const startTimeRef = useRef<number | null>(null);
  const { duration, loop = false, easing = (t) => t } = config;

  useFrame((state) => {
    if (startTimeRef.current === null) {
      startTimeRef.current = state.clock.elapsedTime;
    }

    const elapsed = state.clock.elapsedTime - startTimeRef.current;
    let t = Math.min(elapsed / duration, 1);

    if (loop && t >= 1) {
      startTimeRef.current = state.clock.elapsedTime;
      t = 0;
    }

    const easedT = easing(t);
    callback(easedT);
  });
}

// Usage
function AnimatedBox() {
  const meshRef = useRef<Mesh>(null);

  useAnimation((t) => {
    if (meshRef.current) {
      meshRef.current.position.y = Math.sin(t * Math.PI * 2) * 2;
    }
  }, { duration: 2, loop: true });

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

```typescript
// hooks/useKeyboard.ts
import { useEffect, useState } from 'react';

export function useKeyboard() {
  const [keys, setKeys] = useState<Set<string>>(new Set());

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      setKeys((prev) => new Set(prev).add(e.key.toLowerCase()));
    };

    const handleKeyUp = (e: KeyboardEvent) => {
      setKeys((prev) => {
        const next = new Set(prev);
        next.delete(e.key.toLowerCase());
        return next;
      });
    };

    window.addEventListener('keydown', handleKeyDown);
    window.addEventListener('keyup', handleKeyUp);

    return () => {
      window.removeEventListener('keydown', handleKeyDown);
      window.removeEventListener('keyup', handleKeyUp);
    };
  }, []);

  return {
    keys,
    isPressed: (key: string) => keys.has(key.toLowerCase()),
  };
}
```

```typescript
// hooks/useFollowTarget.ts
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { Vector3, Object3D } from 'three';

export function useFollowTarget(
  target: Object3D | null,
  offset: Vector3 = new Vector3(0, 2, 5),
  smoothness: number = 0.1
) {
  const currentPos = useRef(new Vector3());

  useFrame((state) => {
    if (!target) return;

    const camera = state.camera;
    const targetPos = target.position.clone().add(offset);

    currentPos.current.lerp(targetPos, smoothness);
    camera.position.copy(currentPos.current);
    camera.lookAt(target.position);
  });
}
```

### 6. Higher-Order Components

```typescript
// components/withPhysics.tsx
import { ComponentType } from 'react';
import { useFrame } from '@react-three/fiber';

interface WithPhysicsProps {
  velocity?: [number, number, number];
  gravity?: number;
}

export function withPhysics<P extends object>(
  Component: ComponentType<P>
) {
  return function PhysicsComponent(props: P & WithPhysicsProps) {
    const { velocity = [0, 0, 0], gravity = -9.81, ...componentProps } = props;

    // Physics logic here
    useFrame((state, delta) => {
      // Apply gravity, velocity, etc.
    });

    return <Component {...(componentProps as P)} />;
  };
}

// Usage
const PhysicsBox = withPhysics(Box);

function Scene() {
  return <PhysicsBox velocity={[1, 5, 0]} gravity={-9.81} color="red" />;
}
```

### 7. Performance Optimization

```typescript
// components/OptimizedMesh.tsx
import { memo, useMemo } from 'react';
import * as THREE from 'three';

interface OptimizedMeshProps {
  position: [number, number, number];
  color: string;
  geometryArgs?: [number, number, number];
}

export const OptimizedMesh = memo(function OptimizedMesh({
  position,
  color,
  geometryArgs = [1, 1, 1],
}: OptimizedMeshProps) {
  // Memoize geometry
  const geometry = useMemo(
    () => new THREE.BoxGeometry(...geometryArgs),
    [geometryArgs]
  );

  // Memoize material
  const material = useMemo(
    () => new THREE.MeshStandardMaterial({ color }),
    [color]
  );

  return (
    <mesh position={position} geometry={geometry} material={material} />
  );
});
```

### 8. Instanced Components

```typescript
// components/InstancedBoxes.tsx
import { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import { InstancedMesh, Object3D } from 'three';

interface InstancedBoxesProps {
  count: number;
}

export function InstancedBoxes({ count }: InstancedBoxesProps) {
  const meshRef = useRef<InstancedMesh>(null);

  const dummy = useMemo(() => new Object3D(), []);

  // Initialize instance transforms
  useMemo(() => {
    if (!meshRef.current) return;

    for (let i = 0; i < count; i++) {
      dummy.position.set(
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20
      );
      dummy.rotation.set(
        Math.random() * Math.PI,
        Math.random() * Math.PI,
        Math.random() * Math.PI
      );
      dummy.updateMatrix();
      meshRef.current.setMatrixAt(i, dummy.matrix);
    }
    meshRef.current.instanceMatrix.needsUpdate = true;
  }, [count, dummy]);

  // Animate instances
  useFrame((state) => {
    if (!meshRef.current) return;

    for (let i = 0; i < count; i++) {
      meshRef.current.getMatrixAt(i, dummy.matrix);
      dummy.matrix.decompose(dummy.position, dummy.quaternion, dummy.scale);
      dummy.rotation.y += 0.01;
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

### 9. Conditional Rendering

```typescript
// components/ConditionalRender.tsx
interface ConditionalRenderProps {
  condition: boolean;
  fallback?: JSX.Element;
  children: JSX.Element;
}

export function ConditionalRender({
  condition,
  fallback = null,
  children,
}: ConditionalRenderProps) {
  return condition ? children : fallback;
}

// Usage
function Scene() {
  const [showPlayer, setShowPlayer] = useState(true);

  return (
    <>
      <ConditionalRender
        condition={showPlayer}
        fallback={<Placeholder />}
      >
        <Player />
      </ConditionalRender>
    </>
  );
}
```

## Usage Examples

```typescript
// App.tsx
import { Canvas } from '@react-three/fiber';
import { Scene } from './components/Scene';
import { Player } from './components/Player';
import { InstancedBoxes } from './components/InstancedBoxes';

export default function App() {
  return (
    <Canvas shadows camera={{ position: [0, 5, 10], fov: 75 }}>
      <Scene environment="sunset" enableControls>
        <Player position={[0, 1, 0]} color="blue" />
        <InstancedBoxes count={100} />
      </Scene>
    </Canvas>
  );
}
```

## Checklist

- [ ] Create reusable component library
- [ ] Use TypeScript for all props
- [ ] Implement React.memo for expensive components
- [ ] Use useMemo for geometries and materials
- [ ] Create custom hooks for common patterns
- [ ] Implement compound components for complex objects
- [ ] Use instancedMesh for repeated objects (100+)
- [ ] Add proper TypeScript types for all components
- [ ] Test components in isolation
- [ ] Document props and usage
- [ ] Implement error boundaries
- [ ] Use Suspense for loading states

## Common Pitfalls

1. **Creating geometries in render**: Use useMemo
2. **Not using memo**: Unnecessary re-renders
3. **Imperative Three.js code**: Keep it declarative
4. **Missing cleanup**: R3F handles this automatically
5. **Props drilling**: Use context or state management
6. **Not using refs properly**: Use useRef for Three.js objects

## Performance Tips

- Use `React.memo` for components that rarely change
- Memoize geometries and materials with `useMemo`
- Use `InstancedMesh` for 100+ identical objects
- Implement LOD with conditional rendering
- Use `useFrame` selectively (not in every component)
- Leverage Suspense for progressive loading
- Use `<Instances>` from drei for even better instancing
- Profile with React DevTools

## Related Skills

- `r3f-setup` - R3F project setup
- `r3f-performance` - Performance optimization
- `r3f-state-management` - State management patterns
- `r3f-ecs-integration` - ECS integration
