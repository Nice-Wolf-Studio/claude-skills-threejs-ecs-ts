---
name: r3f-ecs-integration
description: Integrate Entity Component System architecture with React Three Fiber for high-performance declarative 3D games
---

# R3F ECS Integration

## When to Use

Use this skill when:
- Building complex games with R3F and ECS
- Managing many game entities efficiently
- Separating data (ECS) from presentation (React)
- Optimizing React Three Fiber performance
- Creating data-driven game architectures

## Core Principles

1. **ECS for Logic**: Use ECS for game state and systems
2. **React for Rendering**: Use R3F components for 3D presentation
3. **Unidirectional Flow**: ECS → React, not React → ECS
4. **Minimal Re-renders**: Update ECS data without triggering React
5. **Sync Strategy**: Batch updates from ECS to React

## Implementation

### 1. ECS React Context

```typescript
// contexts/ECSContext.tsx
import React, { createContext, useContext, useEffect, useRef } from 'react';
import { World } from '../ecs/World';
import { EntityManager } from '../ecs/EntityManager';
import { ComponentManager } from '../ecs/ComponentManager';

interface ECSContextValue {
  world: World;
  entityManager: EntityManager;
  componentManager: ComponentManager;
}

const ECSContext = createContext<ECSContextValue | null>(null);

export function ECSProvider({ children }: { children: React.ReactNode }) {
  const worldRef = useRef<World>();

  if (!worldRef.current) {
    worldRef.current = new World();
  }

  useEffect(() => {
    return () => {
      worldRef.current?.clear();
    };
  }, []);

  const value: ECSContextValue = {
    world: worldRef.current,
    entityManager: worldRef.current.getEntityManager(),
    componentManager: worldRef.current.getComponentManager(),
  };

  return <ECSContext.Provider value={value}>{children}</ECSContext.Provider>;
}

export function useECS(): ECSContextValue {
  const context = useContext(ECSContext);
  if (!context) {
    throw new Error('useECS must be used within ECSProvider');
  }
  return context;
}

export function useWorld(): World {
  return useECS().world;
}
```

### 2. ECS-Driven Rendering

```typescript
// components/EntityRenderer.tsx
import { useEffect, useState } from 'react';
import { useFrame } from '@react-three/fiber';
import { useECS } from '../contexts/ECSContext';
import { Transform, Renderable } from '../ecs/components';
import { EntityId } from '../ecs/types';

interface EntityRendererProps {
  entity: EntityId;
}

export function EntityRenderer({ entity }: EntityRendererProps) {
  const { componentManager } = useECS();
  const [, forceUpdate] = useState({});

  // Get components
  const transform = componentManager.get(entity, Transform);
  const renderable = componentManager.get(entity, Renderable);

  if (!transform || !renderable || !renderable.visible) {
    return null;
  }

  // Update mesh position from transform component
  useFrame(() => {
    if (renderable.mesh) {
      renderable.mesh.position.set(transform.x, transform.y, transform.z);
      renderable.mesh.rotation.y = transform.rotation;
      renderable.mesh.scale.setScalar(transform.scale);
    }
  });

  return <primitive object={renderable.mesh} />;
}
```

### 3. Entity Collection Renderer

```typescript
// components/EntityCollection.tsx
import { useEffect, useState } from 'react';
import { useECS } from '../contexts/ECSContext';
import { EntityRenderer } from './EntityRenderer';
import { Renderable, Transform } from '../ecs/components';
import { EntityId } from '../ecs/types';

export function EntityCollection() {
  const { componentManager, entityManager } = useECS();
  const [entities, setEntities] = useState<EntityId[]>([]);

  useEffect(() => {
    // Find all entities with Transform and Renderable components
    const updateEntities = () => {
      const renderables: EntityId[] = [];

      for (const entity of entityManager.getAll()) {
        if (
          componentManager.has(entity, Transform) &&
          componentManager.has(entity, Renderable)
        ) {
          renderables.push(entity);
        }
      }

      setEntities(renderables);
    };

    // Initial update
    updateEntities();

    // Poll for changes (or use event system)
    const interval = setInterval(updateEntities, 100);

    return () => clearInterval(interval);
  }, [componentManager, entityManager]);

  return (
    <>
      {entities.map(entity => (
        <EntityRenderer key={entity} entity={entity} />
      ))}
    </>
  );
}
```

### 4. ECS Game Loop Integration

```typescript
// components/ECSGameLoop.tsx
import { useFrame } from '@react-three/fiber';
import { useWorld } from '../contexts/ECSContext';

export function ECSGameLoop() {
  const world = useWorld();

  useFrame((_, delta) => {
    // Update all ECS systems
    world.update(delta);
  });

  return null; // This component doesn't render anything
}
```

### 5. Hooks for ECS Interaction

```typescript
// hooks/useEntity.ts
import { useEffect, useState } from 'react';
import { useECS } from '../contexts/ECSContext';
import { EntityId } from '../ecs/types';
import { ComponentClass, Component } from '../ecs/Component';

export function useEntity<T extends Component>(
  entity: EntityId,
  componentType: ComponentClass<T>
): T | undefined {
  const { componentManager } = useECS();
  const [component, setComponent] = useState<T | undefined>(() =>
    componentManager.get(entity, componentType)
  );

  useEffect(() => {
    // Poll for updates (or use event-based approach)
    const interval = setInterval(() => {
      const updated = componentManager.get(entity, componentType);
      if (updated !== component) {
        setComponent(updated);
      }
    }, 16); // ~60fps polling

    return () => clearInterval(interval);
  }, [entity, componentType, componentManager, component]);

  return component;
}
```

```typescript
// hooks/useCreateEntity.ts
import { useCallback } from 'react';
import { useECS } from '../contexts/ECSContext';
import { EntityId } from '../ecs/types';
import * as THREE from 'three';
import { Transform, Renderable, Velocity } from '../ecs/components';

export function useCreateEntity() {
  const { world } = useECS();

  const createPlayer = useCallback((position: THREE.Vector3) => {
    const entity = world.createEntity();

    // Add components
    world.addComponent(entity, Transform, new Transform(position.x, position.y, position.z));
    world.addComponent(entity, Velocity, new Velocity(0, 0, 0));

    // Create Three.js mesh
    const geometry = new THREE.BoxGeometry(1, 1, 1);
    const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
    const mesh = new THREE.Mesh(geometry, material);

    world.addComponent(entity, Renderable, new Renderable(mesh, true));

    return entity;
  }, [world]);

  const createEnemy = useCallback((position: THREE.Vector3) => {
    const entity = world.createEntity();

    world.addComponent(entity, Transform, new Transform(position.x, position.y, position.z));
    world.addComponent(entity, Velocity, new Velocity(0, 0, 0));

    const geometry = new THREE.BoxGeometry(1, 1, 1);
    const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const mesh = new THREE.Mesh(geometry, material);

    world.addComponent(entity, Renderable, new Renderable(mesh, true));

    return entity;
  }, [world]);

  return { createPlayer, createEnemy };
}
```

### 6. Event System Integration

```typescript
// ecs/EventBus.ts
export type GameEvent =
  | { type: 'entityCreated'; entity: EntityId }
  | { type: 'entityDestroyed'; entity: EntityId }
  | { type: 'componentAdded'; entity: EntityId; component: string }
  | { type: 'componentRemoved'; entity: EntityId; component: string };

export class EventBus {
  private listeners = new Map<GameEvent['type'], Set<(event: GameEvent) => void>>();

  on<T extends GameEvent['type']>(
    type: T,
    callback: (event: Extract<GameEvent, { type: T }>) => void
  ): () => void {
    if (!this.listeners.has(type)) {
      this.listeners.set(type, new Set());
    }

    this.listeners.get(type)!.add(callback as any);

    return () => {
      this.listeners.get(type)?.delete(callback as any);
    };
  }

  emit<T extends GameEvent>(event: T): void {
    const listeners = this.listeners.get(event.type);
    if (listeners) {
      listeners.forEach(callback => callback(event));
    }
  }
}
```

```typescript
// hooks/useECSEvent.ts
import { useEffect } from 'react';
import { useWorld } from '../contexts/ECSContext';
import { GameEvent } from '../ecs/EventBus';

export function useECSEvent<T extends GameEvent['type']>(
  type: T,
  callback: (event: Extract<GameEvent, { type: T }>) => void
) {
  const world = useWorld();

  useEffect(() => {
    const eventBus = world.getEventBus();
    const unsubscribe = eventBus.on(type, callback);
    return unsubscribe;
  }, [world, type, callback]);
}
```

### 7. Complete Game Setup

```typescript
// Game.tsx
import { Canvas } from '@react-three/fiber';
import { ECSProvider } from './contexts/ECSContext';
import { ECSGameLoop } from './components/ECSGameLoop';
import { EntityCollection } from './components/EntityCollection';
import { GameSetup } from './components/GameSetup';

export function Game() {
  return (
    <ECSProvider>
      <Canvas
        camera={{ position: [0, 10, 20], fov: 75 }}
        shadows
        gl={{ powerPreference: 'high-performance' }}
      >
        {/* ECS Game Loop */}
        <ECSGameLoop />

        {/* Render all entities */}
        <EntityCollection />

        {/* Setup game (spawns entities, etc.) */}
        <GameSetup />

        {/* Lighting */}
        <ambientLight intensity={0.5} />
        <directionalLight position={[5, 10, 5]} intensity={0.8} castShadow />

        {/* Ground */}
        <mesh rotation={[-Math.PI / 2, 0, 0]} receiveShadow>
          <planeGeometry args={[100, 100]} />
          <meshStandardMaterial color="#333" />
        </mesh>
      </Canvas>
    </ECSProvider>
  );
}
```

```typescript
// components/GameSetup.tsx
import { useEffect } from 'react';
import { useCreateEntity } from '../hooks/useCreateEntity';
import { useWorld } from '../contexts/ECSContext';
import { MovementSystem } from '../ecs/systems/MovementSystem';
import * as THREE from 'three';

export function GameSetup() {
  const { createPlayer, createEnemy } = useCreateEntity();
  const world = useWorld();

  useEffect(() => {
    // Initialize systems
    const movementSystem = new MovementSystem(
      world.getEntityManager(),
      world.getComponentManager()
    );
    world.addSystem(movementSystem);

    // Spawn entities
    createPlayer(new THREE.Vector3(0, 1, 0));
    createEnemy(new THREE.Vector3(5, 1, 5));
    createEnemy(new THREE.Vector3(-5, 1, -5));

    return () => {
      // Cleanup handled by ECSProvider
    };
  }, [createPlayer, createEnemy, world]);

  return null;
}
```

## Usage Examples

```typescript
// Using ECS within React components
function PlayerController() {
  const world = useWorld();
  const { createPlayer } = useCreateEntity();

  const handleSpawn = () => {
    const player = createPlayer(new THREE.Vector3(0, 1, 0));
    console.log('Spawned player:', player);
  };

  return <button onClick={handleSpawn}>Spawn Player</button>;
}

// Listening to ECS events
function EntityCounter() {
  const [count, setCount] = useState(0);

  useECSEvent('entityCreated', () => {
    setCount(c => c + 1);
  });

  useECSEvent('entityDestroyed', () => {
    setCount(c => c - 1);
  });

  return <div>Entities: {count}</div>;
}
```

## Checklist

- [ ] Set up ECS context provider
- [ ] Create ECS game loop component
- [ ] Implement entity collection renderer
- [ ] Add hooks for ECS interaction
- [ ] Set up event system for React updates
- [ ] Create entity factory hooks
- [ ] Implement component sync strategy
- [ ] Test performance with many entities (1000+)
- [ ] Add error boundaries
- [ ] Document ECS-React data flow
- [ ] Profile render performance
- [ ] Optimize re-render frequency

## Common Pitfalls

1. **Too many re-renders**: Poll less frequently or use events
2. **Direct DOM manipulation**: Use ECS data, let React render
3. **React state in ECS**: Keep ECS pure, React for UI only
4. **Syncing on every frame**: Batch updates, use dirty flags
5. **Not cleaning up entities**: Dispose Three.js objects
6. **Complex component hierarchies**: Flatten where possible

## Performance Tips

- Update ECS every frame, sync to React less frequently (100ms)
- Use `useMemo` and `useCallback` for stable references
- Implement dirty flags to track changed entities
- Use `React.memo` for EntityRenderer component
- Batch entity creation/destruction
- Use InstancedMesh for many similar entities
- Profile with React DevTools Profiler
- Keep component count reasonable (<1000 for 60fps)

## Related Skills

- `ecs-architecture` - Core ECS implementation
- `ecs-system-patterns` - System optimization
- `r3f-component-patterns` - React patterns
- `ecs-events` - Event-driven architecture
