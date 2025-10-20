---
name: react-three-fiber-setup
description: Complete React Three Fiber setup for mobile-optimized Three.js games with TypeScript, performance configuration, and best practices
---

# React Three Fiber Setup

## When to Use

Use this skill when:
- Building React Three.js applications
- Setting up a new R3F project
- Integrating Three.js with React components
- Creating declarative 3D scenes
- Building mobile-responsive 3D apps

## Core Principles

1. **Declarative**: Define scenes with JSX, not imperative code
2. **Component-Based**: Reusable 3D components
3. **React Integration**: Use hooks, context, state naturally
4. **Performance**: Automatic disposal, frame loop optimization
5. **TypeScript**: Full type safety for Three.js objects

## Implementation

### 1. Project Setup

```bash
# Create new React project with TypeScript
npm create vite@latest my-game -- --template react-ts
cd my-game

# Install R3F and dependencies
npm install three @react-three/fiber @react-three/drei
npm install --save-dev @types/three

# Optional: Additional R3F ecosystem
npm install @react-three/postprocessing zustand
```

### 2. Basic Canvas Setup

```typescript
// App.tsx
import { Canvas } from '@react-three/fiber';
import { Suspense } from 'react';
import './App.css';

function App() {
  return (
    <div className="app-container">
      <Canvas
        camera={{
          position: [0, 5, 10],
          fov: 75,
          near: 0.1,
          far: 1000,
        }}
        shadows
        gl={{
          antialias: true,
          alpha: false,
          powerPreference: 'high-performance',
        }}
        dpr={[1, 2]} // Pixel ratio: min 1, max 2
        performance={{
          min: 0.5, // Min FPS threshold (30fps)
        }}
      >
        <Suspense fallback={null}>
          <Scene />
        </Suspense>
      </Canvas>
    </div>
  );
}

export default App;
```

```css
/* App.css */
.app-container {
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  overflow: hidden;
  touch-action: none; /* Prevent default touch behaviors */
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

canvas {
  display: block;
  width: 100%;
  height: 100%;
}
```

### 3. Scene Component

```typescript
// Scene.tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { OrbitControls, Environment } from '@react-three/drei';
import * as THREE from 'three';

export function Scene() {
  const boxRef = useRef<THREE.Mesh>(null);

  // Animation loop
  useFrame((state, delta) => {
    if (boxRef.current) {
      boxRef.current.rotation.y += delta;
    }
  });

  return (
    <>
      {/* Lighting */}
      <ambientLight intensity={0.4} />
      <directionalLight
        position={[5, 10, 5]}
        intensity={0.8}
        castShadow
        shadow-mapSize={[1024, 1024]}
      />

      {/* Environment */}
      <Environment preset="sunset" />

      {/* Controls */}
      <OrbitControls
        enableDamping
        dampingFactor={0.05}
        enableZoom={true}
        enablePan={false}
      />

      {/* Objects */}
      <mesh ref={boxRef} castShadow position={[0, 1, 0]}>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="hotpink" />
      </mesh>

      {/* Ground */}
      <mesh receiveShadow rotation={[-Math.PI / 2, 0, 0]} position={[0, 0, 0]}>
        <planeGeometry args={[100, 100]} />
        <meshStandardMaterial color="#333" />
      </mesh>
    </>
  );
}
```

### 4. Mobile-Optimized Canvas

```typescript
// MobileCanvas.tsx
import { Canvas } from '@react-three/fiber';
import { Suspense, useEffect, useState } from 'react';
import { AdaptiveDpr, AdaptiveEvents } from '@react-three/drei';

interface DeviceInfo {
  isMobile: boolean;
  tier: 'low' | 'medium' | 'high';
}

function detectDevice(): DeviceInfo {
  const isMobile = /mobile|android|iphone|ipad/i.test(navigator.userAgent);

  const memory = (navigator as any).deviceMemory || 4;
  const cores = navigator.hardwareConcurrency || 4;

  let tier: 'low' | 'medium' | 'high' = 'medium';

  if (memory <= 2 || cores <= 4) {
    tier = 'low';
  } else if (memory >= 6 && cores >= 6) {
    tier = 'high';
  }

  return { isMobile, tier };
}

export function MobileCanvas({ children }: { children: React.ReactNode }) {
  const [deviceInfo, setDeviceInfo] = useState<DeviceInfo>(() => detectDevice());

  useEffect(() => {
    const info = detectDevice();
    setDeviceInfo(info);
    console.log('[Device]', info);
  }, []);

  // Quality settings based on device
  const qualitySettings = {
    low: {
      dpr: [1, 1],
      shadows: false,
      antialias: false,
      performance: { min: 0.3 },
    },
    medium: {
      dpr: [1, 1.5],
      shadows: true,
      antialias: false,
      performance: { min: 0.5 },
    },
    high: {
      dpr: [1, 2],
      shadows: true,
      antialias: true,
      performance: { min: 0.7 },
    },
  };

  const settings = qualitySettings[deviceInfo.tier];

  return (
    <Canvas
      camera={{
        position: [0, 5, 10],
        fov: 75,
      }}
      shadows={settings.shadows}
      gl={{
        antialias: settings.antialias,
        alpha: false,
        powerPreference: 'high-performance',
      }}
      dpr={settings.dpr as [number, number]}
      performance={settings.performance}
      style={{
        touchAction: 'none',
        width: '100%',
        height: '100%',
      }}
    >
      {/* Adaptive DPR based on performance */}
      <AdaptiveDpr pixelated />

      {/* Adaptive event sampling for performance */}
      <AdaptiveEvents />

      <Suspense fallback={null}>{children}</Suspense>
    </Canvas>
  );
}
```

### 5. Custom Hooks

```typescript
// hooks/useGameLoop.ts
import { useFrame } from '@react-three/fiber';
import { useRef } from 'react';

export function useGameLoop(
  callback: (deltaTime: number, elapsedTime: number) => void
) {
  const lastTimeRef = useRef(0);

  useFrame((state) => {
    const currentTime = state.clock.elapsedTime;
    const deltaTime = currentTime - lastTimeRef.current;
    lastTimeRef.current = currentTime;

    callback(deltaTime, currentTime);
  });
}
```

```typescript
// hooks/useTexture.ts
import { useLoader } from '@react-three/fiber';
import * as THREE from 'three';

export function useGameTexture(url: string) {
  const texture = useLoader(THREE.TextureLoader, url);

  // Mobile optimizations
  texture.minFilter = THREE.LinearFilter;
  texture.magFilter = THREE.LinearFilter;
  texture.generateMipmaps = false;

  return texture;
}
```

```typescript
// hooks/useKeyboard.ts
import { useEffect, useState } from 'react';

export function useKeyboard() {
  const [keys, setKeys] = useState<Set<string>>(new Set());

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      setKeys(prev => new Set(prev).add(e.key.toLowerCase()));
    };

    const handleKeyUp = (e: KeyboardEvent) => {
      setKeys(prev => {
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

  const isPressed = (key: string) => keys.has(key.toLowerCase());

  return { keys, isPressed };
}
```

### 6. Reusable Components

```typescript
// components/Player.tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

interface PlayerProps {
  position?: [number, number, number];
  color?: string;
}

export function Player({ position = [0, 0, 0], color = 'blue' }: PlayerProps) {
  const meshRef = useRef<THREE.Mesh>(null);

  useFrame((state, delta) => {
    if (meshRef.current) {
      // Hover animation
      meshRef.current.position.y = position[1] + Math.sin(state.clock.elapsedTime * 2) * 0.1;
    }
  });

  return (
    <mesh ref={meshRef} position={position} castShadow>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color={color} />
    </mesh>
  );
}
```

```typescript
// components/Ground.tsx
interface GroundProps {
  size?: number;
  color?: string;
}

export function Ground({ size = 100, color = '#333' }: GroundProps) {
  return (
    <mesh receiveShadow rotation={[-Math.PI / 2, 0, 0]} position={[0, 0, 0]}>
      <planeGeometry args={[size, size]} />
      <meshStandardMaterial color={color} />
    </mesh>
  );
}
```

### 7. TypeScript Types

```typescript
// types/r3f.d.ts
import { Object3DNode } from '@react-three/fiber';
import * as THREE from 'three';

declare module '@react-three/fiber' {
  interface ThreeElements {
    // Add custom elements if needed
  }
}

// Component prop types
export interface MeshComponentProps {
  position?: THREE.Vector3Tuple;
  rotation?: THREE.EulerTuple;
  scale?: THREE.Vector3Tuple | number;
  visible?: boolean;
  castShadow?: boolean;
  receiveShadow?: boolean;
}
```

## Usage Examples

```typescript
// Complete game setup
import { MobileCanvas } from './MobileCanvas';
import { Scene } from './Scene';
import { Player } from './components/Player';
import { Ground } from './components/Ground';

function Game() {
  return (
    <MobileCanvas>
      {/* Lighting */}
      <ambientLight intensity={0.4} />
      <directionalLight position={[5, 10, 5]} intensity={0.8} castShadow />

      {/* Game objects */}
      <Player position={[0, 1, 0]} color="hotpink" />
      <Ground size={50} />

      {/* Camera controls */}
      <OrbitControls />
    </MobileCanvas>
  );
}
```

## Checklist

- [ ] Install R3F and Three.js dependencies
- [ ] Set up TypeScript types
- [ ] Configure Canvas with mobile settings
- [ ] Implement adaptive DPR
- [ ] Add Suspense for loading states
- [ ] Create reusable component library
- [ ] Set up custom hooks for game logic
- [ ] Configure touch-action: none in CSS
- [ ] Test on actual mobile devices
- [ ] Implement quality presets
- [ ] Add error boundaries
- [ ] Set up proper disposal (automatic in R3F)

## Common Pitfalls

1. **Not using Suspense**: Loading causes errors
2. **High DPR on mobile**: Performance issues
3. **Missing touch-action: none**: Unwanted scrolling
4. **Not setting performance.min**: Doesn't adapt to low FPS
5. **Imperative Three.js code**: Breaks React patterns
6. **Not using useFrame**: Manual animation loops

## Performance Tips

- Use `<AdaptiveDpr />` for automatic quality scaling
- Set `dpr={[1, 2]}` to cap pixel ratio
- Use `performance={{ min: 0.5 }}` for adaptive mode
- Implement `<Preload />` from drei for asset loading
- Use `frameloop="demand"` for static scenes
- Leverage automatic disposal (no manual cleanup needed)
- Use `<Instances />` for repeated geometry

## Related Skills

- `r3f-component-patterns` - Advanced component patterns
- `r3f-ecs-integration` - Integrating ECS with R3F
- `mobile-performance` - Mobile optimization
- `touch-input-handling` - Touch controls for R3F
