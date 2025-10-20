---
name: spawn-system
description: Spawn system for games including entity spawning, spawn pools, waves, and spawn points
---

# Spawn System

## When to Use

Use this skill when:
- Spawning enemies or pickups
- Implementing wave-based gameplay
- Creating spawn points
- Managing entity pools
- Building procedural spawners
- Creating respawn systems

## Core Principles

1. **Pooling**: Reuse entities instead of creating/destroying
2. **Configurable**: Data-driven spawn configuration
3. **Wave Management**: Support wave-based spawning
4. **Spawn Points**: Multiple spawn locations
5. **Cooldowns**: Limit spawn rates
6. **Budget**: Limit active entities

## Spawn System Implementation

### 1. Spawn Components

```typescript
// components/Spawner.ts
export interface SpawnConfig {
  prefabName: string;
  count?: number;
  interval?: number;
  maxActive?: number;
  spawnOnStart?: boolean;
}

export class Spawner {
  configs: SpawnConfig[] = [];
  timeSinceLastSpawn = 0;
  currentConfigIndex = 0;
  activeEntities: Entity[] = [];
  enabled = true;

  constructor(configs: SpawnConfig[]) {
    this.configs = configs;
  }

  addConfig(config: SpawnConfig): void {
    this.configs.push(config);
  }

  getCurrentConfig(): SpawnConfig | null {
    return this.configs[this.currentConfigIndex] ?? null;
  }

  nextConfig(): void {
    this.currentConfigIndex = (this.currentConfigIndex + 1) % this.configs.length;
    this.timeSinceLastSpawn = 0;
  }

  reset(): void {
    this.currentConfigIndex = 0;
    this.timeSinceLastSpawn = 0;
    this.activeEntities = [];
  }
}

// components/SpawnPoint.ts
export class SpawnPoint {
  radius: number = 0; // 0 = exact position, >0 = random within radius
  active: boolean = true;

  constructor(radius: number = 0) {
    this.radius = radius;
  }

  getRandomOffset(): Vector3 {
    if (this.radius === 0) {
      return new Vector3();
    }

    const angle = Math.random() * Math.PI * 2;
    const distance = Math.random() * this.radius;

    return new Vector3(
      Math.cos(angle) * distance,
      0,
      Math.sin(angle) * distance
    );
  }
}
```

### 2. Spawn System

```typescript
// systems/SpawnSystem.ts
export class SpawnSystem extends UpdateSystem {
  priority = 15;
  private prefabManager: EntityPrefabManager;
  private world: World;

  constructor(prefabManager: EntityPrefabManager) {
    super();
    this.prefabManager = prefabManager;
  }

  update(world: World, deltaTime: number): void {
    this.world = world;

    const spawners = world.query<[Transform, Spawner, SpawnPoint]>([
      Transform,
      Spawner,
      SpawnPoint,
    ]);

    spawners.iterate((entity, [transform, spawner, spawnPoint]) => {
      if (!spawner.enabled || !spawnPoint.active) return;

      const config = spawner.getCurrentConfig();
      if (!config) return;

      // Update spawn timer
      spawner.timeSinceLastSpawn += deltaTime;

      // Check if should spawn
      const interval = config.interval ?? 1;
      if (spawner.timeSinceLastSpawn < interval) return;

      // Check max active limit
      const maxActive = config.maxActive ?? Infinity;
      const activeCount = spawner.activeEntities.filter((e) => e.active).length;

      if (activeCount >= maxActive) return;

      // Spawn entity
      this.spawnEntity(spawner, transform, spawnPoint, config);

      spawner.timeSinceLastSpawn = 0;

      // Check if config is complete
      const totalSpawned = spawner.activeEntities.length;
      if (config.count && totalSpawned >= config.count) {
        spawner.nextConfig();
      }
    });
  }

  private spawnEntity(
    spawner: Spawner,
    transform: Transform,
    spawnPoint: SpawnPoint,
    config: SpawnConfig
  ): void {
    // Instantiate from prefab
    const entity = this.prefabManager.instantiate(config.prefabName, this.world);
    if (!entity) {
      console.warn(`Failed to spawn prefab: ${config.prefabName}`);
      return;
    }

    // Set spawn position
    const entityTransform = entity.getComponent(Transform);
    if (entityTransform) {
      const spawnOffset = spawnPoint.getRandomOffset();
      entityTransform.position.copy(transform.position).add(spawnOffset);
    }

    // Track spawned entity
    spawner.activeEntities.push(entity);
  }

  spawnAt(prefabName: string, position: Vector3): Entity | null {
    const entity = this.prefabManager.instantiate(prefabName, this.world);
    if (entity) {
      const transform = entity.getComponent(Transform);
      if (transform) {
        transform.position.copy(position);
      }
    }
    return entity;
  }
}
```

### 3. Wave System

```typescript
// systems/WaveSystem.ts
export interface WaveConfig {
  waveNumber: number;
  spawnConfigs: SpawnConfig[];
  delay?: number; // Delay before starting wave
  completionDelay?: number; // Delay after wave completes
}

export class WaveSystem extends UpdateSystem {
  priority = 14;
  private waves: WaveConfig[] = [];
  private currentWaveIndex = 0;
  private waveState: 'waiting' | 'active' | 'complete' = 'waiting';
  private timeSinceWaveEnd = 0;
  private spawnSystem: SpawnSystem;
  private spawners: Entity[] = [];

  constructor(spawnSystem: SpawnSystem, waves: WaveConfig[] = []) {
    super();
    this.spawnSystem = spawnSystem;
    this.waves = waves;
  }

  update(world: World, deltaTime: number): void {
    if (this.waves.length === 0) return;

    switch (this.waveState) {
      case 'waiting':
        this.updateWaiting(world, deltaTime);
        break;

      case 'active':
        this.updateActive(world);
        break;

      case 'complete':
        this.updateComplete(deltaTime);
        break;
    }
  }

  private updateWaiting(world: World, deltaTime: number): void {
    const wave = this.waves[this.currentWaveIndex];
    if (!wave) return;

    this.timeSinceWaveEnd += deltaTime;

    if (this.timeSinceWaveEnd >= (wave.delay ?? 0)) {
      this.startWave(world, wave);
    }
  }

  private startWave(world: World, wave: WaveConfig): void {
    console.log(`Starting Wave ${wave.waveNumber}`);

    // Find or create spawners
    this.spawners = [];
    const spawnPoints = world.query<[Transform, SpawnPoint]>([Transform, SpawnPoint]);

    let configIndex = 0;

    spawnPoints.iterate((entity, [transform, spawnPoint]) => {
      if (!spawnPoint.active) return;

      // Assign spawn config to this spawner
      const config = wave.spawnConfigs[configIndex % wave.spawnConfigs.length];

      let spawner = entity.getComponent(Spawner);
      if (!spawner) {
        spawner = entity.addComponent(new Spawner([config]));
      } else {
        spawner.configs = [config];
        spawner.reset();
      }

      spawner.enabled = true;
      this.spawners.push(entity);

      configIndex++;
    });

    this.waveState = 'active';
  }

  private updateActive(world: World): void {
    // Check if all spawners are done
    let allComplete = true;

    for (const spawnerEntity of this.spawners) {
      const spawner = spawnerEntity.getComponent(Spawner);
      if (!spawner) continue;

      const config = spawner.getCurrentConfig();
      if (!config) continue;

      // Check if spawner has completed its count
      if (config.count) {
        const spawned = spawner.activeEntities.length;
        if (spawned < config.count) {
          allComplete = false;
          break;
        }
      } else {
        // No count limit, never completes
        allComplete = false;
        break;
      }
    }

    if (allComplete) {
      // Check if all spawned entities are dead
      const anyAlive = this.spawners.some((spawnerEntity) => {
        const spawner = spawnerEntity.getComponent(Spawner);
        return spawner?.activeEntities.some((e) => e.active) ?? false;
      });

      if (!anyAlive) {
        this.completeWave();
      }
    }
  }

  private completeWave(): void {
    const wave = this.waves[this.currentWaveIndex];
    console.log(`Wave ${wave.waveNumber} Complete!`);

    // Disable spawners
    for (const spawnerEntity of this.spawners) {
      const spawner = spawnerEntity.getComponent(Spawner);
      if (spawner) {
        spawner.enabled = false;
      }
    }

    this.waveState = 'complete';
    this.timeSinceWaveEnd = 0;
  }

  private updateComplete(deltaTime: number): void {
    const wave = this.waves[this.currentWaveIndex];
    this.timeSinceWaveEnd += deltaTime;

    if (this.timeSinceWaveEnd >= (wave.completionDelay ?? 3)) {
      this.nextWave();
    }
  }

  private nextWave(): void {
    this.currentWaveIndex++;

    if (this.currentWaveIndex >= this.waves.length) {
      console.log('All waves complete!');
      this.currentWaveIndex = 0; // Loop or stop
      // Could emit AllWavesCompleteEvent here
    }

    this.waveState = 'waiting';
    this.timeSinceWaveEnd = 0;
  }

  addWave(wave: WaveConfig): void {
    this.waves.push(wave);
  }

  startFromBeginning(): void {
    this.currentWaveIndex = 0;
    this.waveState = 'waiting';
    this.timeSinceWaveEnd = 0;
  }

  getCurrentWave(): number {
    return this.waves[this.currentWaveIndex]?.waveNumber ?? 0;
  }

  getWaveState(): string {
    return this.waveState;
  }
}
```

### 4. Entity Pool

```typescript
// spawn/EntityPool.ts
export class EntityPool {
  private prefabName: string;
  private pool: Entity[] = [];
  private active: Entity[] = [];
  private prefabManager: EntityPrefabManager;
  private world: World;

  constructor(
    prefabName: string,
    prefabManager: EntityPrefabManager,
    world: World,
    prewarmCount: number = 0
  ) {
    this.prefabName = prefabName;
    this.prefabManager = prefabManager;
    this.world = world;

    if (prewarmCount > 0) {
      this.prewarm(prewarmCount);
    }
  }

  prewarm(count: number): void {
    for (let i = 0; i < count; i++) {
      const entity = this.prefabManager.instantiate(this.prefabName, this.world);
      if (entity) {
        entity.active = false;
        this.pool.push(entity);
      }
    }
  }

  spawn(position: Vector3): Entity | null {
    let entity: Entity | null = null;

    // Try to get from pool
    if (this.pool.length > 0) {
      entity = this.pool.pop()!;
      entity.active = true;
    } else {
      // Create new entity
      entity = this.prefabManager.instantiate(this.prefabName, this.world);
    }

    if (entity) {
      const transform = entity.getComponent(Transform);
      if (transform) {
        transform.position.copy(position);
      }

      this.active.push(entity);
    }

    return entity;
  }

  despawn(entity: Entity): void {
    const index = this.active.indexOf(entity);
    if (index !== -1) {
      this.active.splice(index, 1);
      entity.active = false;

      // Reset entity state
      const health = entity.getComponent(Health);
      if (health) {
        health.current = health.max;
      }

      const velocity = entity.getComponent(Velocity);
      if (velocity) {
        velocity.set(0, 0, 0);
      }

      this.pool.push(entity);
    }
  }

  despawnAll(): void {
    for (const entity of this.active) {
      entity.active = false;
      this.pool.push(entity);
    }
    this.active = [];
  }

  getActiveCount(): number {
    return this.active.length;
  }

  getPoolSize(): number {
    return this.pool.length;
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic spawner
const spawner = world.createEntity();
spawner.addComponent(new Transform(new Vector3(10, 0, 10)));
spawner.addComponent(new SpawnPoint(2)); // Random within 2 units

const spawnerComponent = spawner.addComponent(new Spawner([
  {
    prefabName: 'enemy_goblin',
    interval: 3, // Spawn every 3 seconds
    maxActive: 5, // Max 5 active at once
  },
]));

// Example 2: Wave system
const waveSystem = new WaveSystem(spawnSystem, [
  {
    waveNumber: 1,
    delay: 3,
    completionDelay: 5,
    spawnConfigs: [
      {
        prefabName: 'enemy_goblin',
        count: 5,
        interval: 2,
      },
    ],
  },
  {
    waveNumber: 2,
    delay: 3,
    completionDelay: 5,
    spawnConfigs: [
      {
        prefabName: 'enemy_goblin',
        count: 8,
        interval: 1.5,
      },
      {
        prefabName: 'enemy_orc',
        count: 2,
        interval: 5,
      },
    ],
  },
  {
    waveNumber: 3,
    delay: 5,
    completionDelay: 5,
    spawnConfigs: [
      {
        prefabName: 'enemy_boss',
        count: 1,
        interval: 0,
      },
    ],
  },
]);

systemManager.add(waveSystem);
waveSystem.startFromBeginning();

// Example 3: Entity pool
const bulletPool = new EntityPool('bullet', prefabManager, world, 50);

// Spawn bullet
function fireBullet(position: Vector3, direction: Vector3): void {
  const bullet = bulletPool.spawn(position);
  if (bullet) {
    const velocity = bullet.getComponent(Velocity);
    if (velocity) {
      velocity.copy(direction.multiplyScalar(20));
    }

    // Despawn after 3 seconds
    setTimeout(() => {
      bulletPool.despawn(bullet);
    }, 3000);
  }
}

// Example 4: Multiple spawn points
const spawnPositions = [
  new Vector3(-10, 0, -10),
  new Vector3(10, 0, -10),
  new Vector3(-10, 0, 10),
  new Vector3(10, 0, 10),
];

for (const pos of spawnPositions) {
  const spawnPoint = world.createEntity();
  spawnPoint.addComponent(new Transform(pos));
  spawnPoint.addComponent(new SpawnPoint(1));
  spawnPoint.addComponent(new Spawner([
    {
      prefabName: 'enemy_random',
      interval: 5,
      maxActive: 3,
    },
  ]));
}

// Example 5: Respawn player
function setupPlayerRespawn(player: Entity): void {
  const respawnPoint = new Vector3(0, 0, 0);
  const respawnDelay = 3;

  eventBus.on(DeathEvent, (event) => {
    if (event.entity === player) {
      setTimeout(() => {
        const transform = player.getComponent(Transform);
        const health = player.getComponent(Health);

        if (transform && health) {
          transform.position.copy(respawnPoint);
          health.current = health.max;
          player.active = true;
        }
      }, respawnDelay * 1000);
    }
  });
}

// Example 6: Endless spawning
const endlessSpawner = world.createEntity();
endlessSpawner.addComponent(new Transform(new Vector3(0, 0, 20)));
endlessSpawner.addComponent(new SpawnPoint(10));
endlessSpawner.addComponent(new Spawner([
  {
    prefabName: 'enemy_weak',
    interval: 2,
    maxActive: 10,
    // No count = spawn forever
  },
]));
```

## Checklist

- [ ] Create spawn points
- [ ] Configure spawners
- [ ] Set up entity prefabs
- [ ] Implement wave system
- [ ] Add entity pooling
- [ ] Test spawn rates
- [ ] Handle max active limits
- [ ] Add spawn effects
- [ ] Profile spawn performance
- [ ] Test on mobile

## Common Pitfalls

1. **No pooling**: Performance issues from create/destroy
2. **No max active**: Too many entities
3. **Wrong spawn position**: Entities spawn in walls
4. **No spawn cooldown**: Instant spawn flood
5. **Forgetting cleanup**: Memory leaks
6. **Fixed spawn points**: Predictable and boring
7. **No wave system**: Can't scale difficulty

## Performance Tips

### Spawn Optimization
- Always use entity pooling
- Prewarm pools before gameplay
- Limit spawns per frame
- Use spatial partitioning
- Despawn distant entities

### Memory Optimization
- Pool entities
- Reuse prefab instances
- Limit pool sizes
- Clear unused pools

### Mobile Considerations
- Lower spawn rates
- Fewer max active entities
- Smaller pool sizes
- Simpler spawn logic
- Despawn off-screen entities

## Related Skills

- `ecs-serialization` - Entity prefabs
- `ecs-performance` - Entity pooling
- `collision-system` - Spawn collision checks
- `ai-system` - Spawned AI
- `physics-system` - Spawned physics

## References

- Object pooling patterns
- Wave-based gameplay design
- Spawn point distribution
- Entity lifecycle management
