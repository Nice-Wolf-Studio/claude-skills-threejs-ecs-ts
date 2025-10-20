---
name: r3f-state-management
description: State management in React Three Fiber using Zustand, React Context, and global stores for game state
---

# R3F State Management

## When to Use

Use this skill when:
- Managing global game state in R3F
- Sharing state between 3D and UI components
- Implementing player state, inventory, score
- Coordinating multiple 3D objects
- Optimizing state updates for performance

## Core Principles

1. **Zustand for Global State**: Fast, minimal re-renders
2. **React Context for Scoped State**: Component trees
3. **Local State for Component-Only**: useState
4. **Immutable Updates**: Never mutate state directly
5. **Selectors for Performance**: Subscribe to slices
6. **Avoid Props Drilling**: Use stores or context

## Implementation

### 1. Zustand Store Setup

```typescript
// stores/gameStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface GameState {
  // Player state
  playerPosition: [number, number, number];
  playerHealth: number;
  playerScore: number;

  // Game state
  isPaused: boolean;
  gameLevel: number;
  enemies: Array<{ id: string; position: [number, number, number] }>;

  // Actions
  setPlayerPosition: (position: [number, number, number]) => void;
  damagePlayer: (amount: number) => void;
  addScore: (points: number) => void;
  togglePause: () => void;
  nextLevel: () => void;
  spawnEnemy: (id: string, position: [number, number, number]) => void;
  removeEnemy: (id: string) => void;
  reset: () => void;
}

const initialState = {
  playerPosition: [0, 1, 0] as [number, number, number],
  playerHealth: 100,
  playerScore: 0,
  isPaused: false,
  gameLevel: 1,
  enemies: [],
};

export const useGameStore = create<GameState>()(
  devtools(
    persist(
      (set) => ({
        ...initialState,

        setPlayerPosition: (position) =>
          set({ playerPosition: position }),

        damagePlayer: (amount) =>
          set((state) => ({
            playerHealth: Math.max(0, state.playerHealth - amount),
          })),

        addScore: (points) =>
          set((state) => ({ playerScore: state.playerScore + points })),

        togglePause: () =>
          set((state) => ({ isPaused: !state.isPaused })),

        nextLevel: () =>
          set((state) => ({
            gameLevel: state.gameLevel + 1,
            enemies: [],
          })),

        spawnEnemy: (id, position) =>
          set((state) => ({
            enemies: [...state.enemies, { id, position }],
          })),

        removeEnemy: (id) =>
          set((state) => ({
            enemies: state.enemies.filter((e) => e.id !== id),
          })),

        reset: () => set(initialState),
      }),
      {
        name: 'game-storage',
        partialize: (state) => ({
          playerScore: state.playerScore,
          gameLevel: state.gameLevel,
        }),
      }
    )
  )
);
```

### 2. Using Store in Components

```typescript
// components/Player.tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGameStore } from '../stores/gameStore';
import { Mesh } from 'three';

export function Player() {
  const meshRef = useRef<Mesh>(null);

  // Subscribe to specific slice (prevents unnecessary re-renders)
  const position = useGameStore((state) => state.playerPosition);
  const health = useGameStore((state) => state.playerHealth);
  const setPosition = useGameStore((state) => state.setPlayerPosition);

  // Update store on position change
  useFrame(() => {
    if (meshRef.current) {
      const newPosition = [
        meshRef.current.position.x,
        meshRef.current.position.y,
        meshRef.current.position.z,
      ] as [number, number, number];

      // Only update if changed
      if (
        newPosition[0] !== position[0] ||
        newPosition[1] !== position[1] ||
        newPosition[2] !== position[2]
      ) {
        setPosition(newPosition);
      }
    }
  });

  // Color based on health
  const color = health > 66 ? 'green' : health > 33 ? 'yellow' : 'red';

  return (
    <mesh ref={meshRef} position={position}>
      <boxGeometry />
      <meshStandardMaterial color={color} />
    </mesh>
  );
}
```

### 3. UI Integration

```typescript
// components/HUD.tsx
import { useGameStore } from '../stores/gameStore';

export function HUD() {
  const health = useGameStore((state) => state.playerHealth);
  const score = useGameStore((state) => state.playerScore);
  const level = useGameStore((state) => state.gameLevel);
  const isPaused = useGameStore((state) => state.isPaused);
  const togglePause = useGameStore((state) => state.togglePause);

  return (
    <div className="hud">
      <div className="stat">Health: {health}%</div>
      <div className="stat">Score: {score}</div>
      <div className="stat">Level: {level}</div>

      <button onClick={togglePause}>
        {isPaused ? 'Resume' : 'Pause'}
      </button>
    </div>
  );
}
```

### 4. Derived State with Selectors

```typescript
// stores/selectors.ts
import { useGameStore } from './gameStore';
import { shallow } from 'zustand/shallow';

// Selector for player alive status
export const usePlayerAlive = () =>
  useGameStore((state) => state.playerHealth > 0);

// Selector for enemy count
export const useEnemyCount = () =>
  useGameStore((state) => state.enemies.length);

// Complex selector with shallow comparison
export const usePlayerStats = () =>
  useGameStore(
    (state) => ({
      health: state.playerHealth,
      score: state.playerScore,
      level: state.gameLevel,
    }),
    shallow
  );

// Computed value
export const useGameProgress = () =>
  useGameStore((state) => {
    const maxLevel = 10;
    return (state.gameLevel / maxLevel) * 100;
  });
```

### 5. Actions and Middleware

```typescript
// stores/actionsStore.ts
import { create } from 'zustand';

interface ActionsState {
  lastAction: string | null;
  actionHistory: string[];
  logAction: (action: string) => void;
}

export const useActionsStore = create<ActionsState>((set) => ({
  lastAction: null,
  actionHistory: [],

  logAction: (action) =>
    set((state) => ({
      lastAction: action,
      actionHistory: [...state.actionHistory.slice(-9), action],
    })),
}));

// Middleware to log all state changes
export const logger = (config: any) => (set: any, get: any, api: any) =>
  config(
    (...args: any[]) => {
      console.log('  applying', args);
      set(...args);
      console.log('  new state', get());
    },
    get,
    api
  );
```

### 6. React Context for Scoped State

```typescript
// contexts/LevelContext.tsx
import { createContext, useContext, useState, ReactNode } from 'react';

interface LevelContextValue {
  levelData: {
    enemies: number;
    timeLimit: number;
    difficulty: 'easy' | 'normal' | 'hard';
  };
  setLevelData: (data: Partial<LevelContextValue['levelData']>) => void;
}

const LevelContext = createContext<LevelContextValue | null>(null);

export function LevelProvider({ children }: { children: ReactNode }) {
  const [levelData, setLevelDataState] = useState({
    enemies: 5,
    timeLimit: 60,
    difficulty: 'normal' as const,
  });

  const setLevelData = (data: Partial<typeof levelData>) => {
    setLevelDataState((prev) => ({ ...prev, ...data }));
  };

  return (
    <LevelContext.Provider value={{ levelData, setLevelData }}>
      {children}
    </LevelContext.Provider>
  );
}

export function useLevel() {
  const context = useContext(LevelContext);
  if (!context) {
    throw new Error('useLevel must be used within LevelProvider');
  }
  return context;
}
```

### 7. Combining Stores

```typescript
// stores/index.ts
import { useGameStore } from './gameStore';
import { useActionsStore } from './actionsStore';

// Combined actions
export const useGameActions = () => {
  const gameActions = useGameStore((state) => ({
    setPlayerPosition: state.setPlayerPosition,
    damagePlayer: state.damagePlayer,
    addScore: state.addScore,
  }));

  const logAction = useActionsStore((state) => state.logAction);

  return {
    ...gameActions,
    damagePlayerWithLog: (amount: number) => {
      gameActions.damagePlayer(amount);
      logAction(`Player damaged: ${amount}`);
    },
    addScoreWithLog: (points: number) => {
      gameActions.addScore(points);
      logAction(`Score added: ${points}`);
    },
  };
};
```

### 8. Async Actions

```typescript
// stores/asyncStore.ts
import { create } from 'zustand';

interface AsyncState {
  isLoading: boolean;
  error: string | null;
  data: any | null;

  fetchData: (url: string) => Promise<void>;
  reset: () => void;
}

export const useAsyncStore = create<AsyncState>((set) => ({
  isLoading: false,
  error: null,
  data: null,

  fetchData: async (url) => {
    set({ isLoading: true, error: null });

    try {
      const response = await fetch(url);
      const data = await response.json();
      set({ data, isLoading: false });
    } catch (error) {
      set({
        error: (error as Error).message,
        isLoading: false,
      });
    }
  },

  reset: () => set({ isLoading: false, error: null, data: null }),
}));
```

### 9. Transient Updates (No Re-render)

```typescript
// stores/transientStore.ts
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

interface TransientState {
  mousePosition: { x: number; y: number };
  setMousePosition: (x: number, y: number) => void;
}

export const useTransientStore = create<TransientState>()(
  subscribeWithSelector((set) => ({
    mousePosition: { x: 0, y: 0 },
    setMousePosition: (x, y) => set({ mousePosition: { x, y } }),
  }))
);

// Subscribe without causing re-renders
export function subscribeToMouse(callback: (pos: { x: number; y: number }) => void) {
  return useTransientStore.subscribe(
    (state) => state.mousePosition,
    callback
  );
}

// Usage in component
function Component() {
  useEffect(() => {
    const unsubscribe = subscribeToMouse((pos) => {
      // Update Three.js object directly, no React re-render
      if (meshRef.current) {
        meshRef.current.position.x = pos.x;
        meshRef.current.position.y = pos.y;
      }
    });

    return unsubscribe;
  }, []);
}
```

## Usage Examples

```typescript
// App.tsx
import { Canvas } from '@react-three/fiber';
import { useGameStore } from './stores/gameStore';
import { Player } from './components/Player';
import { Enemy } from './components/Enemy';
import { HUD } from './components/HUD';

export default function Game() {
  const enemies = useGameStore((state) => state.enemies);
  const isPaused = useGameStore((state) => state.isPaused);

  return (
    <>
      <Canvas>
        <Player />

        {enemies.map((enemy) => (
          <Enemy key={enemy.id} id={enemy.id} position={enemy.position} />
        ))}

        {/* Pause overlay */}
        {isPaused && <PauseScreen />}
      </Canvas>

      <HUD />
    </>
  );
}
```

## Checklist

- [ ] Install Zustand (`npm install zustand`)
- [ ] Create global store for game state
- [ ] Use selectors to subscribe to slices
- [ ] Implement actions for state updates
- [ ] Connect UI components to store
- [ ] Connect 3D components to store
- [ ] Add persist middleware for save/load
- [ ] Use shallow comparison for object selectors
- [ ] Implement derived state with selectors
- [ ] Add devtools for debugging
- [ ] Test state updates don't cause unnecessary re-renders
- [ ] Profile with React DevTools

## Common Pitfalls

1. **Subscribing to entire store**: Causes many re-renders
2. **Not using selectors**: Performance issues
3. **Mutating state**: Use immutable updates
4. **Too many stores**: One or two is usually enough
5. **Storing Three.js objects**: Store data, not refs
6. **Not using shallow**: Object selectors always re-render

## Performance Tips

- Use selectors to subscribe to specific slices
- Use `shallow` for object selectors
- Batch updates in actions
- Use transient updates for high-frequency data (mouse, etc.)
- Don't store Three.js objects in state (use refs)
- Use `subscribeWithSelector` for fine-grained subscriptions
- Avoid nested objects (flatten state)
- Use immer middleware for complex updates

## Related Skills

- `r3f-component-patterns` - Component design
- `r3f-performance` - Performance optimization
- `r3f-ecs-integration` - ECS state management
- `ecs-component-patterns` - ECS components
