---
name: r3f-mobile-patterns
description: Mobile-specific React Three Fiber patterns including touch controls, device adaptation, battery optimization, and responsive layouts
---

# React Three Fiber Mobile Patterns

## When to Use

Use this skill when:
- Building mobile-first R3F games
- Implementing touch-based 3D controls
- Adapting quality for mobile devices
- Optimizing battery life
- Creating responsive 3D layouts

## Core Principles

1. **Mobile First**: Design for mobile, enhance for desktop
2. **Touch Optimized**: Gestures over mouse events
3. **Adaptive Quality**: Scale based on device capabilities
4. **Battery Aware**: Reduce work when battery is low
5. **Responsive**: Adapt to screen size and orientation
6. **Progressive Enhancement**: Core experience on all devices

## Implementation

### 1. Device Detection Hook

```typescript
// hooks/useDeviceCapabilities.ts
import { useState, useEffect } from 'react';

export interface DeviceCapabilities {
  isMobile: boolean;
  isTablet: boolean;
  isLowEnd: boolean;
  hasTouchScreen: boolean;
  pixelRatio: number;
  gpuTier: 'low' | 'medium' | 'high';
  maxTextureSize: number;
  screenWidth: number;
  screenHeight: number;
  orientation: 'portrait' | 'landscape';
  batteryLevel?: number;
  isCharging?: boolean;
}

export function useDeviceCapabilities(): DeviceCapabilities {
  const [capabilities, setCapabilities] = useState<DeviceCapabilities>(() => ({
    isMobile: /iPhone|iPad|iPod|Android/i.test(navigator.userAgent),
    isTablet: /iPad|Android/i.test(navigator.userAgent) && window.innerWidth >= 768,
    isLowEnd: navigator.hardwareConcurrency ? navigator.hardwareConcurrency <= 4 : true,
    hasTouchScreen: 'ontouchstart' in window || navigator.maxTouchPoints > 0,
    pixelRatio: Math.min(window.devicePixelRatio || 1, 2),
    gpuTier: 'medium',
    maxTextureSize: 2048,
    screenWidth: window.innerWidth,
    screenHeight: window.innerHeight,
    orientation: window.innerWidth > window.innerHeight ? 'landscape' : 'portrait',
  }));

  useEffect(() => {
    // Detect GPU tier (simplified)
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

    if (gl) {
      const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
      const renderer = debugInfo
        ? gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL)
        : '';

      const maxTexSize = gl.getParameter(gl.MAX_TEXTURE_SIZE);

      let gpuTier: 'low' | 'medium' | 'high' = 'medium';

      if (/Mali-4|Adreno \(TM\) 3|PowerVR SGX/i.test(renderer)) {
        gpuTier = 'low';
      } else if (/Apple A1[2-9]|Adreno \(TM\) [67]|Mali-G7/i.test(renderer)) {
        gpuTier = 'high';
      }

      setCapabilities((prev) => ({
        ...prev,
        gpuTier,
        maxTextureSize: maxTexSize,
      }));
    }

    // Battery API
    if ('getBattery' in navigator) {
      (navigator as any).getBattery().then((battery: any) => {
        const updateBattery = () => {
          setCapabilities((prev) => ({
            ...prev,
            batteryLevel: battery.level,
            isCharging: battery.charging,
          }));
        };

        updateBattery();
        battery.addEventListener('levelchange', updateBattery);
        battery.addEventListener('chargingchange', updateBattery);
      });
    }

    // Orientation changes
    const handleResize = () => {
      setCapabilities((prev) => ({
        ...prev,
        screenWidth: window.innerWidth,
        screenHeight: window.innerHeight,
        orientation: window.innerWidth > window.innerHeight ? 'landscape' : 'portrait',
      }));
    };

    window.addEventListener('resize', handleResize);
    window.addEventListener('orientationchange', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('orientationchange', handleResize);
    };
  }, []);

  return capabilities;
}
```

### 2. Adaptive Canvas

```typescript
// components/AdaptiveMobileCanvas.tsx
import { Canvas } from '@react-three/fiber';
import { ReactNode, useMemo } from 'react';
import { useDeviceCapabilities } from '../hooks/useDeviceCapabilities';

interface AdaptiveMobileCanvasProps {
  children: ReactNode;
}

export function AdaptiveMobileCanvas({ children }: AdaptiveMobileCanvasProps) {
  const device = useDeviceCapabilities();

  const canvasConfig = useMemo(() => {
    const baseConfig = {
      dpr: device.pixelRatio,
      performance: { min: 0.5 },
      gl: {
        powerPreference: 'high-performance' as const,
        antialias: false,
        stencil: false,
        depth: true,
      },
      shadows: false,
      camera: {
        fov: 75,
        near: 0.1,
        far: 1000,
      },
    };

    // Low-end devices
    if (device.gpuTier === 'low' || device.isLowEnd) {
      return {
        ...baseConfig,
        dpr: 1,
        performance: { min: 0.3 },
        gl: {
          ...baseConfig.gl,
          powerPreference: 'low-power' as const,
        },
      };
    }

    // High-end devices
    if (device.gpuTier === 'high') {
      return {
        ...baseConfig,
        dpr: Math.min(device.pixelRatio, 2),
        gl: {
          ...baseConfig.gl,
          antialias: true,
        },
        shadows: true,
      };
    }

    // Battery saving mode
    if (device.batteryLevel !== undefined && device.batteryLevel < 0.2 && !device.isCharging) {
      return {
        ...baseConfig,
        dpr: 1,
        performance: { min: 0.2 },
        gl: {
          ...baseConfig.gl,
          powerPreference: 'low-power' as const,
        },
      };
    }

    return baseConfig;
  }, [device]);

  return (
    <Canvas {...canvasConfig} style={{ touchAction: 'none' }}>
      {children}
    </Canvas>
  );
}
```

### 3. Touch Controls Component

```typescript
// components/TouchControls.tsx
import { useThree, useFrame } from '@react-three/fiber';
import { useEffect, useRef, useState } from 'react';
import { Vector2, Vector3 } from 'three';

interface TouchControlsProps {
  speed?: number;
  lookSensitivity?: number;
  onMove?: (direction: Vector3) => void;
}

export function TouchControls({
  speed = 5,
  lookSensitivity = 0.002,
  onMove,
}: TouchControlsProps) {
  const { camera, gl } = useThree();
  const [touches, setTouches] = useState<Map<number, Vector2>>(new Map());
  const moveDirection = useRef(new Vector3());

  useEffect(() => {
    const canvas = gl.domElement;

    const handleTouchStart = (e: TouchEvent) => {
      e.preventDefault();
      const newTouches = new Map(touches);

      for (let i = 0; i < e.changedTouches.length; i++) {
        const touch = e.changedTouches[i];
        newTouches.set(touch.identifier, new Vector2(touch.clientX, touch.clientY));
      }

      setTouches(newTouches);
    };

    const handleTouchMove = (e: TouchEvent) => {
      e.preventDefault();
      const newTouches = new Map(touches);

      for (let i = 0; i < e.changedTouches.length; i++) {
        const touch = e.changedTouches[i];
        const prevPos = newTouches.get(touch.identifier);

        if (prevPos) {
          const currentPos = new Vector2(touch.clientX, touch.clientY);
          const delta = currentPos.clone().sub(prevPos);

          // Left side = movement, right side = look
          if (touch.clientX < window.innerWidth / 2) {
            // Movement control
            const normalizedDelta = new Vector2(
              delta.x / window.innerWidth,
              delta.y / window.innerHeight
            );

            moveDirection.current.set(
              normalizedDelta.x * speed,
              0,
              -normalizedDelta.y * speed
            );
          } else {
            // Look control
            camera.rotation.y -= delta.x * lookSensitivity;
            camera.rotation.x -= delta.y * lookSensitivity;
            camera.rotation.x = Math.max(
              -Math.PI / 2,
              Math.min(Math.PI / 2, camera.rotation.x)
            );
          }

          newTouches.set(touch.identifier, currentPos);
        }
      }

      setTouches(newTouches);
    };

    const handleTouchEnd = (e: TouchEvent) => {
      e.preventDefault();
      const newTouches = new Map(touches);

      for (let i = 0; i < e.changedTouches.length; i++) {
        newTouches.delete(e.changedTouches[i].identifier);
      }

      setTouches(newTouches);
      moveDirection.current.set(0, 0, 0);
    };

    canvas.addEventListener('touchstart', handleTouchStart, { passive: false });
    canvas.addEventListener('touchmove', handleTouchMove, { passive: false });
    canvas.addEventListener('touchend', handleTouchEnd, { passive: false });
    canvas.addEventListener('touchcancel', handleTouchEnd, { passive: false });

    return () => {
      canvas.removeEventListener('touchstart', handleTouchStart);
      canvas.removeEventListener('touchmove', handleTouchMove);
      canvas.removeEventListener('touchend', handleTouchEnd);
      canvas.removeEventListener('touchcancel', handleTouchEnd);
    };
  }, [camera, gl, lookSensitivity, speed, touches]);

  useFrame(() => {
    if (onMove && moveDirection.current.length() > 0) {
      onMove(moveDirection.current.clone());
    }
  });

  return null;
}
```

### 4. Virtual Joystick

```typescript
// components/VirtualJoystick.tsx
import { useEffect, useRef, useState } from 'react';
import { Vector2 } from 'three';

interface VirtualJoystickProps {
  position?: 'left' | 'right';
  maxDistance?: number;
  onMove?: (direction: Vector2) => void;
}

export function VirtualJoystick({
  position = 'left',
  maxDistance = 50,
  onMove,
}: VirtualJoystickProps) {
  const [active, setActive] = useState(false);
  const [basePos, setBasePos] = useState<Vector2>(new Vector2());
  const [stickPos, setStickPos] = useState<Vector2>(new Vector2());
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const handleTouchStart = (e: TouchEvent) => {
      const touch = e.touches[0];
      const rect = containerRef.current!.getBoundingClientRect();

      const base = new Vector2(
        touch.clientX - rect.left,
        touch.clientY - rect.top
      );

      setBasePos(base);
      setStickPos(base);
      setActive(true);
    };

    const handleTouchMove = (e: TouchEvent) => {
      if (!active) return;

      const touch = e.touches[0];
      const rect = containerRef.current!.getBoundingClientRect();

      const current = new Vector2(
        touch.clientX - rect.left,
        touch.clientY - rect.top
      );

      const delta = current.clone().sub(basePos);
      const distance = delta.length();

      if (distance > maxDistance) {
        delta.normalize().multiplyScalar(maxDistance);
      }

      setStickPos(basePos.clone().add(delta));

      if (onMove) {
        const normalized = delta.clone().divideScalar(maxDistance);
        onMove(normalized);
      }
    };

    const handleTouchEnd = () => {
      setActive(false);
      setStickPos(basePos);
      if (onMove) {
        onMove(new Vector2(0, 0));
      }
    };

    const container = containerRef.current;
    container.addEventListener('touchstart', handleTouchStart);
    window.addEventListener('touchmove', handleTouchMove);
    window.addEventListener('touchend', handleTouchEnd);

    return () => {
      container.removeEventListener('touchstart', handleTouchStart);
      window.removeEventListener('touchmove', handleTouchMove);
      window.removeEventListener('touchend', handleTouchEnd);
    };
  }, [active, basePos, maxDistance, onMove]);

  return (
    <div
      ref={containerRef}
      style={{
        position: 'fixed',
        bottom: '20px',
        [position]: '20px',
        width: '150px',
        height: '150px',
        pointerEvents: 'auto',
      }}
    >
      {active && (
        <>
          {/* Base */}
          <div
            style={{
              position: 'absolute',
              left: `${basePos.x - 75}px`,
              top: `${basePos.y - 75}px`,
              width: '150px',
              height: '150px',
              borderRadius: '50%',
              backgroundColor: 'rgba(255, 255, 255, 0.1)',
              border: '2px solid rgba(255, 255, 255, 0.3)',
            }}
          />

          {/* Stick */}
          <div
            style={{
              position: 'absolute',
              left: `${stickPos.x - 25}px`,
              top: `${stickPos.y - 25}px`,
              width: '50px',
              height: '50px',
              borderRadius: '50%',
              backgroundColor: 'rgba(255, 255, 255, 0.5)',
              border: '2px solid rgba(255, 255, 255, 0.8)',
            }}
          />
        </>
      )}
    </div>
  );
}
```

### 5. Responsive Layout Hook

```typescript
// hooks/useResponsiveLayout.ts
import { useState, useEffect } from 'react';

export type Breakpoint = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

export interface ResponsiveConfig {
  breakpoint: Breakpoint;
  isMobile: boolean;
  isTablet: boolean;
  isDesktop: boolean;
  orientation: 'portrait' | 'landscape';
  width: number;
  height: number;
}

export function useResponsiveLayout(): ResponsiveConfig {
  const [config, setConfig] = useState<ResponsiveConfig>(() => {
    const width = window.innerWidth;
    const height = window.innerHeight;

    let breakpoint: Breakpoint = 'md';
    if (width < 640) breakpoint = 'xs';
    else if (width < 768) breakpoint = 'sm';
    else if (width < 1024) breakpoint = 'md';
    else if (width < 1280) breakpoint = 'lg';
    else breakpoint = 'xl';

    return {
      breakpoint,
      isMobile: width < 768,
      isTablet: width >= 768 && width < 1024,
      isDesktop: width >= 1024,
      orientation: width > height ? 'landscape' : 'portrait',
      width,
      height,
    };
  });

  useEffect(() => {
    const handleResize = () => {
      const width = window.innerWidth;
      const height = window.innerHeight;

      let breakpoint: Breakpoint = 'md';
      if (width < 640) breakpoint = 'xs';
      else if (width < 768) breakpoint = 'sm';
      else if (width < 1024) breakpoint = 'md';
      else if (width < 1280) breakpoint = 'lg';
      else breakpoint = 'xl';

      setConfig({
        breakpoint,
        isMobile: width < 768,
        isTablet: width >= 768 && width < 1024,
        isDesktop: width >= 1024,
        orientation: width > height ? 'landscape' : 'portrait',
        width,
        height,
      });
    };

    window.addEventListener('resize', handleResize);
    window.addEventListener('orientationchange', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('orientationchange', handleResize);
    };
  }, []);

  return config;
}
```

### 6. Battery-Aware Quality

```typescript
// hooks/useBatteryQuality.ts
import { useState, useEffect } from 'react';

export type QualityLevel = 'low' | 'medium' | 'high';

export function useBatteryQuality(): QualityLevel {
  const [quality, setQuality] = useState<QualityLevel>('high');

  useEffect(() => {
    if (!('getBattery' in navigator)) {
      return;
    }

    (navigator as any).getBattery().then((battery: any) => {
      const updateQuality = () => {
        const level = battery.level;
        const charging = battery.charging;

        if (charging) {
          setQuality('high');
        } else if (level < 0.15) {
          setQuality('low');
        } else if (level < 0.30) {
          setQuality('medium');
        } else {
          setQuality('high');
        }
      };

      updateQuality();
      battery.addEventListener('levelchange', updateQuality);
      battery.addEventListener('chargingchange', updateQuality);
    });
  }, []);

  return quality;
}
```

### 7. Complete Mobile Game Setup

```typescript
// App.tsx
import { Suspense } from 'react';
import { AdaptiveMobileCanvas } from './components/AdaptiveMobileCanvas';
import { TouchControls } from './components/TouchControls';
import { VirtualJoystick } from './components/VirtualJoystick';
import { useDeviceCapabilities } from './hooks/useDeviceCapabilities';
import { useResponsiveLayout } from './hooks/useResponsiveLayout';
import { Vector2, Vector3 } from 'three';

export default function MobileGame() {
  const device = useDeviceCapabilities();
  const layout = useResponsiveLayout();

  const handleJoystickMove = (direction: Vector2) => {
    console.log('Joystick:', direction);
    // Move player based on joystick input
  };

  const handleCameraMove = (direction: Vector3) => {
    console.log('Camera move:', direction);
    // Move camera/player
  };

  return (
    <div style={{ width: '100vw', height: '100vh', position: 'relative' }}>
      <AdaptiveMobileCanvas>
        <Suspense fallback={null}>
          {/* Scene */}
          <ambientLight intensity={0.5} />
          <directionalLight position={[10, 10, 5]} intensity={0.8} />

          {/* Player */}
          <mesh position={[0, 1, 0]}>
            <boxGeometry />
            <meshStandardMaterial color="hotpink" />
          </mesh>

          {/* Touch controls */}
          {device.hasTouchScreen && (
            <TouchControls speed={5} onMove={handleCameraMove} />
          )}
        </Suspense>
      </AdaptiveMobileCanvas>

      {/* UI Overlay */}
      {device.hasTouchScreen && layout.isMobile && (
        <>
          <VirtualJoystick position="left" onMove={handleJoystickMove} />

          <div
            style={{
              position: 'fixed',
              top: '10px',
              right: '10px',
              color: 'white',
              fontFamily: 'monospace',
              fontSize: '12px',
              background: 'rgba(0, 0, 0, 0.5)',
              padding: '10px',
              borderRadius: '5px',
            }}
          >
            <div>Device: {device.isMobile ? 'Mobile' : 'Desktop'}</div>
            <div>GPU: {device.gpuTier}</div>
            <div>Orientation: {layout.orientation}</div>
            {device.batteryLevel !== undefined && (
              <div>Battery: {Math.round(device.batteryLevel * 100)}%</div>
            )}
          </div>
        </>
      )}
    </div>
  );
}
```

## Usage Examples

```typescript
// Example 1: Adaptive rendering based on device
import { useDeviceCapabilities } from './hooks/useDeviceCapabilities';

function AdaptiveScene() {
  const device = useDeviceCapabilities();

  return (
    <>
      {/* Show more objects on high-end devices */}
      {device.gpuTier === 'high' ? (
        <HighQualityEnvironment />
      ) : (
        <LowQualityEnvironment />
      )}

      {/* Adjust particle count */}
      <Particles count={device.gpuTier === 'low' ? 100 : 1000} />
    </>
  );
}

// Example 2: Battery-aware rendering
import { useBatteryQuality } from './hooks/useBatteryQuality';

function BatteryAwareScene() {
  const quality = useBatteryQuality();

  return (
    <Canvas
      dpr={quality === 'low' ? 1 : 2}
      shadows={quality === 'high'}
    >
      {/* Scene content */}
    </Canvas>
  );
}

// Example 3: Orientation-aware layout
import { useResponsiveLayout } from './hooks/useResponsiveLayout';

function ResponsiveUI() {
  const layout = useResponsiveLayout();

  return (
    <div
      style={{
        flexDirection: layout.orientation === 'portrait' ? 'column' : 'row',
      }}
    >
      {/* UI elements */}
    </div>
  );
}
```

## Checklist

- [ ] Install dependencies (`@react-three/fiber`, `@react-three/drei`)
- [ ] Implement device detection hook
- [ ] Set up adaptive canvas with quality scaling
- [ ] Add touch controls for camera/movement
- [ ] Implement virtual joystick for movement
- [ ] Add battery monitoring and quality adjustment
- [ ] Handle orientation changes
- [ ] Test on actual mobile devices (iOS and Android)
- [ ] Optimize for low-end devices (reduce particles, shadows, post-processing)
- [ ] Add landscape/portrait layout switching
- [ ] Implement touch gesture recognition
- [ ] Add haptic feedback (if supported)
- [ ] Test battery drain on real devices
- [ ] Verify performance on target devices (maintain 30-60 FPS)

## Common Pitfalls

1. **Testing only in browser**: Real devices behave differently
2. **Not handling orientation changes**: Layout breaks on rotate
3. **Ignoring battery status**: Drains battery too fast
4. **Touch events not preventDefault**: Page scrolls during gameplay
5. **Fixed pixel ratio**: Should adapt to device capabilities
6. **No fallback for missing APIs**: Battery API not available everywhere
7. **Not testing on low-end devices**: Performance issues on cheap phones

## Performance Tips

### Device Detection
- Cache device capabilities (don't recalculate every frame)
- Use feature detection, not user agent sniffing
- Test actual GPU performance, not just model name

### Touch Controls
- Use `touchAction: 'none'` on canvas
- Debounce/throttle touch events if needed
- Use `passive: false` for preventDefault
- Keep touch handlers lightweight

### Battery Optimization
- Reduce quality when battery < 20%
- Lower frame rate on battery saver mode
- Disable shadows, post-processing on low battery
- Use simpler shaders on low battery

### Mobile Rendering
- Cap DPR at 2 (higher wastes performance)
- Use instancing for repeated objects
- Implement LOD aggressively
- Reduce texture sizes (max 1024px for mobile)
- Limit draw calls (<100 on mobile)
- Disable antialiasing on low-end
- Use simpler materials (Lambert over Standard)

### Responsive Design
- Support both portrait and landscape
- Adjust UI based on screen size
- Use viewport units (vw, vh) for sizing
- Test on various screen sizes (phones, tablets)

## Related Skills

- `r3f-setup` - Basic R3F setup
- `r3f-performance` - General performance optimization
- `mobile-performance` - Mobile optimization strategies
- `touch-input-handling` - Touch input implementation
- `r3f-state-management` - Managing game state
