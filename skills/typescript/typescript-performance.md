---
name: typescript-performance
description: TypeScript performance optimization including compilation settings, type checking, and runtime performance patterns
---

# TypeScript Performance Optimization

## When to Use

Use this skill when:
- Optimizing TypeScript build times
- Improving type checking performance
- Reducing bundle size
- Optimizing runtime performance
- Debugging slow compilation
- Setting up production builds

## Core Principles

1. **Incremental Compilation**: Use incremental builds
2. **Strict Mode**: Enable strict checks for safety
3. **Tree Shaking**: Optimize dead code elimination
4. **Type Inference**: Let TypeScript infer types
5. **Project References**: Split large projects
6. **Bundle Optimization**: Minimize output size

## TypeScript Performance Implementation

### 1. Optimal tsconfig.json

```typescript
// tsconfig.json - Optimized for performance
{
  "compilerOptions": {
    // Module Resolution
    "module": "ESNext",
    "moduleResolution": "bundler", // Fastest for bundlers
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],

    // Performance
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",
    "skipLibCheck": true, // Skip checking .d.ts files
    "skipDefaultLibCheck": true,

    // Type Checking (strict but fast)
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,

    // Additional Checks
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,

    // Code Generation
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,
    "importHelpers": true, // Use tslib for smaller bundles
    "downlevelIteration": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,

    // Path Mapping (improves compilation speed)
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@systems/*": ["src/systems/*"],
      "@utils/*": ["src/utils/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.spec.ts",
    "**/*.test.ts"
  ]
}
```

### 2. Project References for Large Codebases

```typescript
// Root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/renderer" },
    { "path": "./packages/physics" },
    { "path": "./packages/ui" }
  ]
}

// packages/core/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}

// packages/renderer/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../core" }
  ],
  "include": ["src/**/*"]
}

// Build all projects
// tsc --build --verbose
```

### 3. Type-Only Imports/Exports

```typescript
// Bad - Imports value and type
import { Component, ComponentType } from './Component';

// Good - Type-only imports (better tree shaking)
import type { ComponentType } from './Component';
import { Component } from './Component';

// Type-only exports
export type { Entity, World } from './World';
export { createWorld } from './World';

// Inline type imports (TypeScript 4.5+)
import { createEntity, type Entity } from './Entity';

// Why: Type-only imports are erased at runtime
// Results in smaller bundles and faster compilation
```

### 4. Efficient Type Patterns

```typescript
// Avoid: Complex conditional types (slow type checking)
type SlowRecursive<T> = T extends Array<infer U>
  ? SlowRecursive<U>
  : T extends object
  ? { [K in keyof T]: SlowRecursive<T[K]> }
  : T;

// Better: Simpler types with inference
type FastFlat<T> = T extends Array<infer U> ? U : T;

// Avoid: Large union types
type ManyTypes = Type1 | Type2 | Type3 | ... | Type100; // Slow

// Better: Use discriminated unions
type Message =
  | { type: 'error'; error: Error }
  | { type: 'success'; data: string }
  | { type: 'loading' };

// Avoid: Deep object nesting
interface DeepNested {
  level1: {
    level2: {
      level3: {
        level4: {
          value: string;
        };
      };
    };
  };
}

// Better: Flatten structure
interface Flattened {
  level4Value: string;
}

// Use const assertions for literal types
const EVENTS = {
  CLICK: 'click',
  HOVER: 'hover',
  DRAG: 'drag',
} as const;

type EventType = typeof EVENTS[keyof typeof EVENTS];
// Result: 'click' | 'hover' | 'drag'

// Efficient generic constraints
export class Pool<T extends { reset(): void }> {
  private pool: T[] = [];

  // Fast type checking with constraint
  acquire(): T | undefined {
    return this.pool.pop();
  }
}
```

### 5. Runtime Performance Patterns

```typescript
// performance/TypeCache.ts
export class TypeCache<K, V> {
  private cache = new Map<K, V>();
  private hits = 0;
  private misses = 0;

  get(key: K, factory: () => V): V {
    let value = this.cache.get(key);
    if (value === undefined) {
      value = factory();
      this.cache.set(key, value);
      this.misses++;
    } else {
      this.hits++;
    }
    return value;
  }

  getStats(): { hits: number; misses: number; hitRate: number } {
    const total = this.hits + this.misses;
    return {
      hits: this.hits,
      misses: this.misses,
      hitRate: total > 0 ? this.hits / total : 0,
    };
  }

  clear(): void {
    this.cache.clear();
    this.hits = 0;
    this.misses = 0;
  }
}

// Efficient property access with symbols
const POSITION_KEY = Symbol('position');
const VELOCITY_KEY = Symbol('velocity');

export class Transform {
  private [POSITION_KEY]: Vector3;
  private [VELOCITY_KEY]: Vector3;

  get position(): Vector3 {
    return this[POSITION_KEY];
  }

  get velocity(): Vector3 {
    return this[VELOCITY_KEY];
  }
}

// Inline type guards (faster than function calls)
export interface WithHealth {
  health: number;
}

export function hasHealth(obj: unknown): obj is WithHealth {
  return typeof obj === 'object' && obj !== null && 'health' in obj;
}

// Use in performance-critical code
export function damageEntity(entity: unknown, amount: number): void {
  // Inline check (faster)
  if (typeof entity === 'object' && entity !== null && 'health' in entity) {
    (entity as WithHealth).health -= amount;
  }
}

// Avoid allocations in hot paths
export class VectorPool {
  private pool: Vector3[] = [];

  // Reuse vectors instead of creating new ones
  acquire(): Vector3 {
    return this.pool.pop() ?? new Vector3();
  }

  release(vector: Vector3): void {
    vector.set(0, 0, 0);
    this.pool.push(vector);
  }
}

// Use typed arrays for numeric data
export class PackedTransforms {
  // Single array instead of many objects
  private data: Float32Array;
  private count = 0;
  private readonly stride = 10; // x,y,z, qx,qy,qz,qw, sx,sy,sz

  constructor(capacity: number) {
    this.data = new Float32Array(capacity * this.stride);
  }

  setTransform(index: number, position: Vector3, rotation: Quaternion, scale: Vector3): void {
    const offset = index * this.stride;

    // Position
    this.data[offset + 0] = position.x;
    this.data[offset + 1] = position.y;
    this.data[offset + 2] = position.z;

    // Rotation
    this.data[offset + 3] = rotation.x;
    this.data[offset + 4] = rotation.y;
    this.data[offset + 5] = rotation.z;
    this.data[offset + 6] = rotation.w;

    // Scale
    this.data[offset + 7] = scale.x;
    this.data[offset + 8] = scale.y;
    this.data[offset + 9] = scale.z;
  }

  getPosition(index: number, out: Vector3): Vector3 {
    const offset = index * this.stride;
    out.x = this.data[offset + 0];
    out.y = this.data[offset + 1];
    out.z = this.data[offset + 2];
    return out;
  }
}
```

### 6. Build Optimization

```typescript
// vite.config.ts - Optimized Vite config
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],

  build: {
    target: 'es2022',
    minify: 'esbuild', // Faster than terser

    rollupOptions: {
      output: {
        // Manual chunk splitting
        manualChunks: {
          'three': ['three'],
          'vendor': ['react', 'react-dom'],
          'ecs': ['./src/ecs/World', './src/ecs/Entity', './src/ecs/System'],
        },
      },
    },

    // Source maps only for dev
    sourcemap: process.env.NODE_ENV === 'development',
  },

  // Optimize dependencies
  optimizeDeps: {
    include: ['three', 'react', 'react-dom'],
    exclude: ['@types/*'],
  },

  // Path aliases
  resolve: {
    alias: {
      '@': '/src',
      '@components': '/src/components',
      '@systems': '/src/systems',
      '@utils': '/src/utils',
    },
  },
});

// webpack.config.js - Optimized Webpack config
module.exports = {
  mode: 'production',

  optimization: {
    usedExports: true, // Tree shaking
    minimize: true,

    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10,
        },
        three: {
          test: /[\\/]node_modules[\\/]three[\\/]/,
          name: 'three',
          priority: 20,
        },
      },
    },
  },

  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },

  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: [
          {
            loader: 'ts-loader',
            options: {
              transpileOnly: true, // Faster, but skip type checking
              experimentalWatchApi: true,
            },
          },
        ],
        exclude: /node_modules/,
      },
    ],
  },
};
```

## Usage Examples

```typescript
// Example 1: Fast type inference
// Bad - Explicit return type (slower)
function slowMap<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}

// Good - Inferred return type (faster)
function fastMap<T, U>(arr: T[], fn: (item: T) => U) {
  return arr.map(fn);
}

// Example 2: Efficient const assertions
const CONFIG = {
  MAX_ENTITIES: 10000,
  TICK_RATE: 60,
  DEBUG: false,
} as const;

type Config = typeof CONFIG;
// Result: { readonly MAX_ENTITIES: 10000; ... }

// Example 3: Type-only dependencies
// package.json
{
  "dependencies": {
    "three": "^0.160.0"
  },
  "devDependencies": {
    "@types/three": "^0.160.0",
    "typescript": "^5.3.0"
  }
}

// Example 4: Incremental compilation
// Enable in tsconfig.json
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}

// Build times:
// First build: ~10s
// Subsequent builds: ~1s

// Example 5: Bundle size analysis
// Add to package.json scripts
{
  "scripts": {
    "build": "vite build",
    "analyze": "vite-bundle-visualizer"
  }
}

// Example 6: Efficient generic caching
class ComponentRegistry {
  private cache = new TypeCache<string, ComponentConstructor<any>>();

  register<T extends Component>(name: string, ctor: ComponentConstructor<T>): void {
    this.cache.get(name, () => ctor);
  }

  get(name: string): ComponentConstructor<any> | undefined {
    return this.cache.get(name, () => undefined as any);
  }
}

// Example 7: Build performance monitoring
// Add to build script
import { performance } from 'perf_hooks';

const start = performance.now();
// ... build process ...
const end = performance.now();

console.log(`Build completed in ${(end - start).toFixed(2)}ms`);
```

## Checklist

- [ ] Enable incremental compilation
- [ ] Configure strict type checking
- [ ] Use type-only imports/exports
- [ ] Optimize tsconfig.json
- [ ] Set up project references (large projects)
- [ ] Configure bundle optimization
- [ ] Enable tree shaking
- [ ] Profile compilation time
- [ ] Analyze bundle size
- [ ] Optimize runtime patterns

## Common Pitfalls

1. **Complex conditional types**: Slow type checking
2. **Large union types**: Performance degradation
3. **No incremental builds**: Slow rebuilds
4. **Type-value mixing**: Poor tree shaking
5. **Deep type nesting**: Slow inference
6. **No skipLibCheck**: Checking unnecessary files
7. **Inefficient generics**: Complex type resolution

## Performance Tips

### Compilation Optimization
- Use `skipLibCheck: true` to skip .d.ts files
- Enable incremental compilation
- Use project references for monorepos
- Limit use of complex conditional types
- Use type-only imports/exports

### Runtime Optimization
- Cache type instances
- Use typed arrays for numeric data
- Inline type guards in hot paths
- Avoid allocations in loops
- Use object pooling

### Bundle Optimization
- Configure tree shaking
- Use dynamic imports for code splitting
- Optimize chunk splitting strategy
- Enable minification
- Remove unused dependencies

### Mobile Considerations
- Smaller bundle targets
- Aggressive tree shaking
- Code splitting by route
- Lazy load heavy features
- Remove debug code in production

## Related Skills

- `typescript-game-types` - Game-specific type patterns
- `typescript-ecs-types` - ECS type system
- `ecs-performance` - ECS optimization
- `threejs-performance-profiling` - Rendering profiling
- `threejs-best-practices` - Three.js optimization

## References

- TypeScript performance docs
- Bundle optimization guides
- Type system performance
- Compilation optimization
