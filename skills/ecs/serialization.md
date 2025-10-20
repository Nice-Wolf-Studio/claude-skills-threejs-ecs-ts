---
name: ecs-serialization
description: Serializing and deserializing ECS worlds including save/load systems, component serialization, entity prefabs, and network synchronization
---

# ECS Serialization

## When to Use

Use this skill when:
- Implementing save/load functionality
- Creating entity prefabs/templates
- Network synchronization
- Level editor export/import
- Debugging and inspection
- Game state snapshots

## Core Principles

1. **Component-Based**: Serialize components, not entities
2. **Type-Safe**: Preserve component types during deserialization
3. **Efficient**: Minimize serialized data size
4. **Versioned**: Handle schema changes gracefully
5. **Selective**: Serialize only what's needed
6. **Deterministic**: Same input produces same output

## Serialization System Implementation

### 1. Component Serialization

```typescript
// core/Serializable.ts
export interface ISerializable {
  serialize(): SerializedData;
  deserialize(data: SerializedData): void;
}

export type SerializedData = Record<string, any>;

// Base serializable component
export abstract class SerializableComponent implements ISerializable {
  abstract serialize(): SerializedData;
  abstract deserialize(data: SerializedData): void;

  getTypeName(): string {
    return this.constructor.name;
  }
}

// Example: Transform component
import { Vector3, Quaternion } from 'three';

export class Transform extends SerializableComponent {
  position = new Vector3();
  rotation = new Quaternion();
  scale = new Vector3(1, 1, 1);

  serialize(): SerializedData {
    return {
      position: this.position.toArray(),
      rotation: this.rotation.toArray(),
      scale: this.scale.toArray(),
    };
  }

  deserialize(data: SerializedData): void {
    if (data.position) this.position.fromArray(data.position);
    if (data.rotation) this.rotation.fromArray(data.rotation);
    if (data.scale) this.scale.fromArray(data.scale);
  }
}

// Example: Health component
export class Health extends SerializableComponent {
  current: number = 100;
  max: number = 100;
  regenRate: number = 0;

  serialize(): SerializedData {
    return {
      current: this.current,
      max: this.max,
      regenRate: this.regenRate,
    };
  }

  deserialize(data: SerializedData): void {
    this.current = data.current ?? this.current;
    this.max = data.max ?? this.max;
    this.regenRate = data.regenRate ?? this.regenRate;
  }
}
```

### 2. Entity Serialization

```typescript
// core/EntitySerializer.ts
export interface SerializedEntity {
  id: string;
  components: Array<{
    type: string;
    data: SerializedData;
  }>;
}

export class EntitySerializer {
  private componentRegistry = new Map<string, ComponentConstructor<any>>();

  registerComponent(type: ComponentConstructor<any>): void {
    this.componentRegistry.set(type.name, type);
  }

  serializeEntity(entity: Entity): SerializedEntity {
    const components = entity.getAllComponents();
    const serializedComponents: Array<{ type: string; data: SerializedData }> = [];

    for (const component of components) {
      if ('serialize' in component && typeof component.serialize === 'function') {
        serializedComponents.push({
          type: component.constructor.name,
          data: component.serialize(),
        });
      }
    }

    return {
      id: entity.id,
      components: serializedComponents,
    };
  }

  deserializeEntity(data: SerializedEntity, world: World): Entity {
    const entity = world.createEntity(data.id);

    for (const componentData of data.components) {
      const ComponentType = this.componentRegistry.get(componentData.type);
      if (!ComponentType) {
        console.warn(`Unknown component type: ${componentData.type}`);
        continue;
      }

      const component = new ComponentType();
      if ('deserialize' in component && typeof component.deserialize === 'function') {
        component.deserialize(componentData.data);
      }

      entity.addComponent(component);
    }

    return entity;
  }
}
```

### 3. World Serialization

```typescript
// core/WorldSerializer.ts
export interface SerializedWorld {
  version: string;
  timestamp: number;
  entities: SerializedEntity[];
  metadata?: Record<string, any>;
}

export class WorldSerializer {
  private entitySerializer: EntitySerializer;
  private version: string = '1.0.0';

  constructor(entitySerializer: EntitySerializer) {
    this.entitySerializer = entitySerializer;
  }

  serializeWorld(world: World, metadata?: Record<string, any>): SerializedWorld {
    const entities = world.getAllEntities();
    const serializedEntities: SerializedEntity[] = [];

    for (const entity of entities) {
      serializedEntities.push(this.entitySerializer.serializeEntity(entity));
    }

    return {
      version: this.version,
      timestamp: Date.now(),
      entities: serializedEntities,
      metadata: metadata ?? {},
    };
  }

  deserializeWorld(data: SerializedWorld, world: World): void {
    // Check version compatibility
    if (!this.isCompatibleVersion(data.version)) {
      throw new Error(`Incompatible save version: ${data.version}`);
    }

    // Clear existing world
    world.clear();

    // Deserialize all entities
    for (const entityData of data.entities) {
      this.entitySerializer.deserializeEntity(entityData, world);
    }
  }

  private isCompatibleVersion(version: string): boolean {
    // Simple version check (implement proper semver comparison)
    return version === this.version;
  }
}
```

### 4. Save/Load System

```typescript
// systems/SaveLoadSystem.ts
export interface SaveData {
  world: SerializedWorld;
  playerData?: Record<string, any>;
  gameSettings?: Record<string, any>;
}

export class SaveLoadSystem {
  private worldSerializer: WorldSerializer;
  private saveSlots = new Map<string, SaveData>();

  constructor(worldSerializer: WorldSerializer) {
    this.worldSerializer = worldSerializer;
  }

  // Save game state
  save(slotName: string, world: World, additionalData?: Record<string, any>): void {
    const saveData: SaveData = {
      world: this.worldSerializer.serializeWorld(world, additionalData),
    };

    // Save to slot
    this.saveSlots.set(slotName, saveData);

    // Persist to storage
    this.persistToStorage(slotName, saveData);
  }

  // Load game state
  load(slotName: string, world: World): boolean {
    let saveData = this.saveSlots.get(slotName);

    if (!saveData) {
      // Try loading from storage
      saveData = this.loadFromStorage(slotName);
      if (!saveData) return false;
    }

    try {
      this.worldSerializer.deserializeWorld(saveData.world, world);
      return true;
    } catch (error) {
      console.error('Failed to load save:', error);
      return false;
    }
  }

  // List available saves
  listSaves(): string[] {
    const saves: string[] = [];

    // From memory
    this.saveSlots.forEach((_, key) => saves.push(key));

    // From storage
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i);
      if (key?.startsWith('save_')) {
        const slotName = key.substring(5);
        if (!saves.includes(slotName)) {
          saves.push(slotName);
        }
      }
    }

    return saves;
  }

  // Delete save
  deleteSave(slotName: string): void {
    this.saveSlots.delete(slotName);
    localStorage.removeItem(`save_${slotName}`);
  }

  private persistToStorage(slotName: string, data: SaveData): void {
    try {
      const json = JSON.stringify(data);
      localStorage.setItem(`save_${slotName}`, json);
    } catch (error) {
      console.error('Failed to persist save:', error);
    }
  }

  private loadFromStorage(slotName: string): SaveData | null {
    try {
      const json = localStorage.getItem(`save_${slotName}`);
      if (!json) return null;

      return JSON.parse(json);
    } catch (error) {
      console.error('Failed to load save:', error);
      return null;
    }
  }

  // Export save as downloadable file
  exportSave(slotName: string): void {
    const saveData = this.saveSlots.get(slotName);
    if (!saveData) return;

    const json = JSON.stringify(saveData, null, 2);
    const blob = new Blob([json], { type: 'application/json' });
    const url = URL.createObjectURL(blob);

    const a = document.createElement('a');
    a.href = url;
    a.download = `${slotName}.json`;
    a.click();

    URL.revokeObjectURL(url);
  }

  // Import save from file
  async importSave(file: File, slotName: string): Promise<boolean> {
    try {
      const text = await file.text();
      const saveData: SaveData = JSON.parse(text);

      this.saveSlots.set(slotName, saveData);
      this.persistToStorage(slotName, saveData);

      return true;
    } catch (error) {
      console.error('Failed to import save:', error);
      return false;
    }
  }
}
```

### 5. Entity Prefabs

```typescript
// prefabs/EntityPrefab.ts
export interface EntityPrefabData {
  name: string;
  components: Array<{
    type: string;
    data: SerializedData;
  }>;
}

export class EntityPrefabManager {
  private prefabs = new Map<string, EntityPrefabData>();
  private entitySerializer: EntitySerializer;

  constructor(entitySerializer: EntitySerializer) {
    this.entitySerializer = entitySerializer;
  }

  // Create prefab from entity
  createPrefab(name: string, entity: Entity): void {
    const serialized = this.entitySerializer.serializeEntity(entity);
    this.prefabs.set(name, {
      name,
      components: serialized.components,
    });
  }

  // Instantiate entity from prefab
  instantiate(name: string, world: World): Entity | null {
    const prefab = this.prefabs.get(name);
    if (!prefab) {
      console.warn(`Prefab not found: ${name}`);
      return null;
    }

    const entity = this.entitySerializer.deserializeEntity(
      {
        id: world.generateEntityId(),
        components: prefab.components,
      },
      world
    );

    return entity;
  }

  // Save prefab to JSON
  savePrefab(name: string): string | null {
    const prefab = this.prefabs.get(name);
    if (!prefab) return null;

    return JSON.stringify(prefab, null, 2);
  }

  // Load prefab from JSON
  loadPrefab(json: string): boolean {
    try {
      const prefab: EntityPrefabData = JSON.parse(json);
      this.prefabs.set(prefab.name, prefab);
      return true;
    } catch (error) {
      console.error('Failed to load prefab:', error);
      return false;
    }
  }

  // List all prefabs
  listPrefabs(): string[] {
    return Array.from(this.prefabs.keys());
  }

  // Delete prefab
  deletePrefab(name: string): void {
    this.prefabs.delete(name);
  }
}
```

### 6. Network Synchronization

```typescript
// network/NetworkSerializer.ts
export interface NetworkSnapshot {
  frame: number;
  entities: Array<{
    id: string;
    components: SerializedData;
  }>;
}

export class NetworkSerializer {
  private entitySerializer: EntitySerializer;
  private dirtyEntities = new Set<Entity>();

  constructor(entitySerializer: EntitySerializer) {
    this.entitySerializer = entitySerializer;
  }

  // Mark entity as dirty (needs sync)
  markDirty(entity: Entity): void {
    this.dirtyEntities.add(entity);
  }

  // Create snapshot of dirty entities
  createSnapshot(frame: number): NetworkSnapshot {
    const entities: Array<{ id: string; components: SerializedData }> = [];

    for (const entity of this.dirtyEntities) {
      entities.push({
        id: entity.id,
        components: this.entitySerializer.serializeEntity(entity).components.reduce(
          (acc, comp) => {
            acc[comp.type] = comp.data;
            return acc;
          },
          {} as SerializedData
        ),
      });
    }

    this.dirtyEntities.clear();

    return { frame, entities };
  }

  // Apply snapshot to world
  applySnapshot(snapshot: NetworkSnapshot, world: World): void {
    for (const entityData of snapshot.entities) {
      const entity = world.getEntity(entityData.id);
      if (!entity) continue;

      // Update components
      for (const [componentType, componentData] of Object.entries(entityData.components)) {
        const ComponentType = this.entitySerializer['componentRegistry'].get(componentType);
        if (!ComponentType) continue;

        let component = entity.getComponent(ComponentType);
        if (!component) {
          component = new ComponentType();
          entity.addComponent(component);
        }

        if ('deserialize' in component) {
          component.deserialize(componentData);
        }
      }
    }
  }

  // Delta compression (only send changed data)
  createDeltaSnapshot(
    frame: number,
    previousSnapshot: NetworkSnapshot
  ): NetworkSnapshot {
    const snapshot = this.createSnapshot(frame);

    // TODO: Implement delta compression
    // Compare with previous snapshot and only include changes

    return snapshot;
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic save/load
import { SaveLoadSystem } from './systems/SaveLoadSystem';

const entitySerializer = new EntitySerializer();
entitySerializer.registerComponent(Transform);
entitySerializer.registerComponent(Health);
entitySerializer.registerComponent(Velocity);

const worldSerializer = new WorldSerializer(entitySerializer);
const saveLoadSystem = new SaveLoadSystem(worldSerializer);

// Save game
saveLoadSystem.save('slot1', world, {
  playerName: 'Player1',
  level: 5,
});

// Load game
if (saveLoadSystem.load('slot1', world)) {
  console.log('Game loaded successfully');
}

// Example 2: Entity prefabs
import { EntityPrefabManager } from './prefabs/EntityPrefabManager';

const prefabManager = new EntityPrefabManager(entitySerializer);

// Create enemy prefab from existing entity
const enemyTemplate = world.createEntity();
enemyTemplate.addComponent(new Transform());
enemyTemplate.addComponent(new Health(50, 50, 0));
enemyTemplate.addComponent(new Enemy());

prefabManager.createPrefab('goblin', enemyTemplate);

// Spawn enemies from prefab
for (let i = 0; i < 10; i++) {
  const enemy = prefabManager.instantiate('goblin', world);
  if (enemy) {
    const transform = enemy.getComponent(Transform);
    transform.position.set(Math.random() * 10, 0, Math.random() * 10);
  }
}

// Example 3: Export/import saves
// Export
saveLoadSystem.exportSave('slot1');

// Import
const fileInput = document.createElement('input');
fileInput.type = 'file';
fileInput.onchange = async (e) => {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (file) {
    const success = await saveLoadSystem.importSave(file, 'imported');
    if (success) {
      console.log('Save imported');
    }
  }
};
fileInput.click();

// Example 4: Versioned serialization
export class VersionedComponent extends SerializableComponent {
  private static VERSION = 2;

  value: number = 0;
  newField: string = ''; // Added in version 2

  serialize(): SerializedData {
    return {
      version: VersionedComponent.VERSION,
      value: this.value,
      newField: this.newField,
    };
  }

  deserialize(data: SerializedData): void {
    const version = data.version ?? 1;

    this.value = data.value ?? 0;

    // Handle version migration
    if (version >= 2) {
      this.newField = data.newField ?? '';
    } else {
      this.newField = 'migrated'; // Default for old saves
    }
  }
}

// Example 5: Selective serialization
export class SelectivelySerializedComponent extends SerializableComponent {
  // Serialized
  persistentData: number = 0;

  // Not serialized (runtime only)
  private cachedValue: number = 0;
  private tempBuffer: Float32Array = new Float32Array(100);

  serialize(): SerializedData {
    return {
      persistentData: this.persistentData,
      // cachedValue and tempBuffer are NOT serialized
    };
  }

  deserialize(data: SerializedData): void {
    this.persistentData = data.persistentData ?? 0;
    // Recalculate runtime data
    this.recalculateCache();
  }

  private recalculateCache(): void {
    this.cachedValue = this.persistentData * 2;
  }
}

// Example 6: Auto-save system
class AutoSaveSystem extends UpdateSystem {
  private saveLoadSystem: SaveLoadSystem;
  private saveInterval = 60; // 60 seconds
  private timeSinceLastSave = 0;

  constructor(saveLoadSystem: SaveLoadSystem) {
    super();
    this.saveLoadSystem = saveLoadSystem;
  }

  update(world: World, deltaTime: number): void {
    this.timeSinceLastSave += deltaTime;

    if (this.timeSinceLastSave >= this.saveInterval) {
      this.saveLoadSystem.save('autosave', world);
      console.log('Auto-saved');
      this.timeSinceLastSave = 0;
    }
  }
}
```

## Serialization Patterns

### Pattern 1: Compression

```typescript
import pako from 'pako';

export class CompressedSerializer extends WorldSerializer {
  serializeWorld(world: World, metadata?: Record<string, any>): SerializedWorld {
    const serialized = super.serializeWorld(world, metadata);

    // Compress the JSON
    const json = JSON.stringify(serialized);
    const compressed = pako.deflate(json);

    return {
      ...serialized,
      metadata: {
        ...serialized.metadata,
        compressed: true,
        originalSize: json.length,
      },
      entities: compressed as any, // Store compressed data
    };
  }

  deserializeWorld(data: SerializedWorld, world: World): void {
    if (data.metadata?.compressed) {
      // Decompress
      const compressed = data.entities as any;
      const json = pako.inflate(compressed, { to: 'string' });
      data = JSON.parse(json);
    }

    super.deserializeWorld(data, world);
  }
}
```

### Pattern 2: Incremental Saves

```typescript
export class IncrementalSaveSystem extends SaveLoadSystem {
  private lastFullSave: SerializedWorld | null = null;

  save(slotName: string, world: World, additionalData?: Record<string, any>): void {
    const currentState = this.worldSerializer.serializeWorld(world, additionalData);

    if (!this.lastFullSave) {
      // First save is full
      super.save(slotName, world, additionalData);
      this.lastFullSave = currentState;
    } else {
      // Save only changes
      const delta = this.computeDelta(this.lastFullSave, currentState);
      this.saveDelta(slotName + '_delta', delta);
    }
  }

  private computeDelta(
    previous: SerializedWorld,
    current: SerializedWorld
  ): Partial<SerializedWorld> {
    // Compute differences between saves
    // Implementation depends on requirements
    return {};
  }
}
```

### Pattern 3: Async Serialization

```typescript
export class AsyncSerializer extends WorldSerializer {
  async serializeWorldAsync(
    world: World,
    metadata?: Record<string, any>
  ): Promise<SerializedWorld> {
    return new Promise((resolve) => {
      // Use requestIdleCallback or web worker
      requestIdleCallback(() => {
        const result = this.serializeWorld(world, metadata);
        resolve(result);
      });
    });
  }

  async deserializeWorldAsync(data: SerializedWorld, world: World): Promise<void> {
    return new Promise((resolve) => {
      requestIdleCallback(() => {
        this.deserializeWorld(data, world);
        resolve();
      });
    });
  }
}
```

## Checklist

- [ ] Implement serialize() for all components
- [ ] Implement deserialize() for all components
- [ ] Register all component types
- [ ] Test save/load functionality
- [ ] Handle version migrations
- [ ] Compress large saves
- [ ] Validate loaded data
- [ ] Handle corrupted saves gracefully
- [ ] Implement auto-save
- [ ] Test with different save slots

## Common Pitfalls

1. **Forgetting to register components**: Deserialize fails silently
2. **Circular references**: JSON.stringify fails
3. **Large save files**: No compression
4. **No version handling**: Breaks when schema changes
5. **Serializing runtime data**: Waste space on cached values
6. **Synchronous saves**: Causes frame drops
7. **No validation**: Corrupted saves crash game

## Performance Tips

### Serialization Optimization
- Compress large saves (gzip, brotli)
- Use binary formats (MessagePack, Protocol Buffers)
- Serialize asynchronously
- Only serialize changed data (delta compression)
- Use efficient data structures

### Memory Optimization
- Stream large files instead of loading all at once
- Clear old saves periodically
- Limit number of auto-save slots
- Use IndexedDB for large saves (not localStorage)

### Network Optimization
- Delta compression for network sync
- Prioritize important entities
- Batch updates
- Use binary protocols (WebSocket binary)

## Related Skills

- `ecs-architecture` - Overall ECS structure
- `ecs-component-patterns` - Component design
- `ecs-events` - State change events
- `typescript-game-types` - Type-safe serialization
- `ecs-performance` - Optimization techniques

## References

- JSON serialization best practices
- Protocol Buffers / MessagePack
- Save system design patterns
- Network synchronization techniques
