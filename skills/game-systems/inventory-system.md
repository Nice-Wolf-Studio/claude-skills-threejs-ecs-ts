---
name: inventory-system
description: Inventory system for games including item management, stacking, equipment, and inventory UI
---

# Inventory System

## When to Use

Use this skill when:
- Managing player inventory
- Implementing item collection
- Creating equipment systems
- Building crafting systems
- Managing item stacking
- Implementing item drops

## Core Principles

1. **Slot-Based**: Fixed or dynamic inventory slots
2. **Stackable**: Items can stack with limits
3. **Data-Driven**: Item definitions in data
4. **Type-Safe**: Strong typing for items
5. **Event-Driven**: Item pickup/use events
6. **Serializable**: Save/load inventory

## Inventory System Implementation

### 1. Item Definitions

```typescript
// items/ItemDefinition.ts
export enum ItemType {
  Consumable = 'consumable',
  Equipment = 'equipment',
  Material = 'material',
  Quest = 'quest',
}

export enum EquipmentSlot {
  Head = 'head',
  Chest = 'chest',
  Legs = 'legs',
  Weapon = 'weapon',
  Shield = 'shield',
}

export interface ItemDefinition {
  id: string;
  name: string;
  description: string;
  type: ItemType;
  maxStack: number;
  icon?: string;
  value?: number; // Sell price
  weight?: number;

  // Equipment
  equipmentSlot?: EquipmentSlot;
  armor?: number;
  damage?: number;

  // Consumable
  healAmount?: number;
  effectDuration?: number;
}

export class ItemDatabase {
  private items = new Map<string, ItemDefinition>();

  register(item: ItemDefinition): void {
    this.items.set(item.id, item);
  }

  get(id: string): ItemDefinition | undefined {
    return this.items.get(id);
  }

  getAll(): ItemDefinition[] {
    return Array.from(this.items.values());
  }
}
```

### 2. Inventory Components

```typescript
// components/Inventory.ts
export interface InventorySlot {
  itemId: string | null;
  quantity: number;
}

export class Inventory {
  slots: InventorySlot[];
  maxSlots: number;

  constructor(maxSlots: number = 20) {
    this.maxSlots = maxSlots;
    this.slots = Array.from({ length: maxSlots }, () => ({
      itemId: null,
      quantity: 0,
    }));
  }

  addItem(itemId: string, quantity: number = 1, itemDb: ItemDatabase): boolean {
    const itemDef = itemDb.get(itemId);
    if (!itemDef) return false;

    let remaining = quantity;

    // Try to stack with existing items
    if (itemDef.maxStack > 1) {
      for (const slot of this.slots) {
        if (slot.itemId === itemId && slot.quantity < itemDef.maxStack) {
          const canAdd = Math.min(remaining, itemDef.maxStack - slot.quantity);
          slot.quantity += canAdd;
          remaining -= canAdd;

          if (remaining === 0) return true;
        }
      }
    }

    // Add to empty slots
    while (remaining > 0) {
      const emptySlot = this.slots.find((s) => s.itemId === null);
      if (!emptySlot) return false; // Inventory full

      const addAmount = Math.min(remaining, itemDef.maxStack);
      emptySlot.itemId = itemId;
      emptySlot.quantity = addAmount;
      remaining -= addAmount;
    }

    return true;
  }

  removeItem(itemId: string, quantity: number = 1): boolean {
    let remaining = quantity;

    for (const slot of this.slots) {
      if (slot.itemId === itemId) {
        const removeAmount = Math.min(remaining, slot.quantity);
        slot.quantity -= removeAmount;
        remaining -= removeAmount;

        if (slot.quantity === 0) {
          slot.itemId = null;
        }

        if (remaining === 0) return true;
      }
    }

    return remaining === 0;
  }

  hasItem(itemId: string, quantity: number = 1): boolean {
    let count = 0;

    for (const slot of this.slots) {
      if (slot.itemId === itemId) {
        count += slot.quantity;
      }
    }

    return count >= quantity;
  }

  getItemCount(itemId: string): number {
    let count = 0;

    for (const slot of this.slots) {
      if (slot.itemId === itemId) {
        count += slot.quantity;
      }
    }

    return count;
  }

  isFull(): boolean {
    return this.slots.every((slot) => slot.itemId !== null);
  }

  isEmpty(): boolean {
    return this.slots.every((slot) => slot.itemId === null);
  }

  clear(): void {
    for (const slot of this.slots) {
      slot.itemId = null;
      slot.quantity = 0;
    }
  }

  getFilledSlots(): InventorySlot[] {
    return this.slots.filter((slot) => slot.itemId !== null);
  }
}

// components/Equipment.ts
export class Equipment {
  slots = new Map<EquipmentSlot, string | null>();

  constructor() {
    // Initialize all equipment slots
    for (const slot of Object.values(EquipmentSlot)) {
      this.slots.set(slot, null);
    }
  }

  equip(slot: EquipmentSlot, itemId: string): string | null {
    const previous = this.slots.get(slot) ?? null;
    this.slots.set(slot, itemId);
    return previous; // Return unequipped item
  }

  unequip(slot: EquipmentSlot): string | null {
    const itemId = this.slots.get(slot) ?? null;
    this.slots.set(slot, null);
    return itemId;
  }

  getEquipped(slot: EquipmentSlot): string | null {
    return this.slots.get(slot) ?? null;
  }

  isEquipped(itemId: string): boolean {
    return Array.from(this.slots.values()).includes(itemId);
  }

  getTotalArmor(itemDb: ItemDatabase): number {
    let total = 0;

    for (const itemId of this.slots.values()) {
      if (itemId) {
        const item = itemDb.get(itemId);
        if (item?.armor) {
          total += item.armor;
        }
      }
    }

    return total;
  }

  getTotalDamage(itemDb: ItemDatabase): number {
    let total = 0;

    for (const itemId of this.slots.values()) {
      if (itemId) {
        const item = itemDb.get(itemId);
        if (item?.damage) {
          total += item.damage;
        }
      }
    }

    return total;
  }
}
```

### 3. Inventory System

```typescript
// systems/InventorySystem.ts
export class InventorySystem extends UpdateSystem {
  priority = 40;
  private itemDb: ItemDatabase;
  private eventBus: EventBus;

  constructor(itemDb: ItemDatabase, eventBus: EventBus) {
    super();
    this.itemDb = itemDb;
    this.eventBus = eventBus;
  }

  update(world: World, deltaTime: number): void {
    // Process item pickups
    this.processItemPickups(world);

    // Update equipment stats
    this.updateEquipmentStats(world);
  }

  private processItemPickups(world: World): void {
    const items = world.query<[Transform, ItemPickup]>([Transform, ItemPickup]);
    const players = world.query<[Transform, Inventory]>([Transform, Inventory]);

    items.iterate((itemEntity, [itemTransform, pickup]) => {
      players.iterate((playerEntity, [playerTransform, inventory]) => {
        const distance = itemTransform.position.distanceTo(playerTransform.position);

        if (distance <= pickup.pickupRadius) {
          // Try to add to inventory
          if (inventory.addItem(pickup.itemId, pickup.quantity, this.itemDb)) {
            // Pickup successful
            this.eventBus.emit(new ItemPickedUpEvent(playerEntity, pickup.itemId, pickup.quantity));
            world.destroyEntity(itemEntity);
          }
        }
      });
    });
  }

  private updateEquipmentStats(world: World): void {
    const entities = world.query<[Equipment, Armor, Attack]>([Equipment, Armor, Attack]);

    entities.iterate((entity, [equipment, armor, attack]) => {
      // Update armor from equipment
      armor.value = equipment.getTotalArmor(this.itemDb);

      // Update damage from equipment
      attack.damage = equipment.getTotalDamage(this.itemDb);
    });
  }

  useItem(entity: Entity, itemId: string): boolean {
    const inventory = entity.getComponent(Inventory);
    if (!inventory || !inventory.hasItem(itemId)) return false;

    const itemDef = this.itemDb.get(itemId);
    if (!itemDef) return false;

    switch (itemDef.type) {
      case ItemType.Consumable:
        return this.useConsumable(entity, itemId, itemDef);

      case ItemType.Equipment:
        return this.equipItem(entity, itemId, itemDef);

      default:
        return false;
    }
  }

  private useConsumable(entity: Entity, itemId: string, itemDef: ItemDefinition): boolean {
    const inventory = entity.getComponent(Inventory);
    if (!inventory) return false;

    // Apply consumable effect
    if (itemDef.healAmount) {
      const health = entity.getComponent(Health);
      if (health) {
        health.heal(itemDef.healAmount);
      }
    }

    // Remove from inventory
    inventory.removeItem(itemId, 1);

    this.eventBus.emit(new ItemUsedEvent(entity, itemId));

    return true;
  }

  private equipItem(entity: Entity, itemId: string, itemDef: ItemDefinition): boolean {
    const inventory = entity.getComponent(Inventory);
    const equipment = entity.getComponent(Equipment);

    if (!inventory || !equipment || !itemDef.equipmentSlot) return false;

    // Unequip current item in slot
    const previousItem = equipment.equip(itemDef.equipmentSlot, itemId);

    // Remove from inventory
    inventory.removeItem(itemId, 1);

    // Add previous item back to inventory
    if (previousItem) {
      inventory.addItem(previousItem, 1, this.itemDb);
    }

    this.eventBus.emit(new ItemEquippedEvent(entity, itemId, itemDef.equipmentSlot));

    return true;
  }

  dropItem(entity: Entity, itemId: string, quantity: number = 1, world: World): void {
    const inventory = entity.getComponent(Inventory);
    const transform = entity.getComponent(Transform);

    if (!inventory || !transform) return;

    if (inventory.removeItem(itemId, quantity)) {
      // Create dropped item entity
      const droppedItem = world.createEntity();
      const dropTransform = droppedItem.addComponent(new Transform(transform.position.clone()));
      dropTransform.position.y += 0.5; // Slightly above ground

      const pickup = droppedItem.addComponent(new ItemPickup(itemId, quantity));

      this.eventBus.emit(new ItemDroppedEvent(entity, itemId, quantity));
    }
  }
}

// components/ItemPickup.ts
export class ItemPickup {
  itemId: string;
  quantity: number;
  pickupRadius: number = 2;

  constructor(itemId: string, quantity: number = 1) {
    this.itemId = itemId;
    this.quantity = quantity;
  }
}
```

### 4. Crafting System

```typescript
// crafting/CraftingSystem.ts
export interface CraftingRecipe {
  id: string;
  result: { itemId: string; quantity: number };
  ingredients: Array<{ itemId: string; quantity: number }>;
  craftTime?: number;
}

export class CraftingSystem {
  private recipes = new Map<string, CraftingRecipe>();
  private itemDb: ItemDatabase;

  constructor(itemDb: ItemDatabase) {
    this.itemDb = itemDb;
  }

  addRecipe(recipe: CraftingRecipe): void {
    this.recipes.set(recipe.id, recipe);
  }

  canCraft(recipeId: string, inventory: Inventory): boolean {
    const recipe = this.recipes.get(recipeId);
    if (!recipe) return false;

    // Check if has all ingredients
    for (const ingredient of recipe.ingredients) {
      if (!inventory.hasItem(ingredient.itemId, ingredient.quantity)) {
        return false;
      }
    }

    return true;
  }

  craft(recipeId: string, inventory: Inventory): boolean {
    if (!this.canCraft(recipeId, inventory)) return false;

    const recipe = this.recipes.get(recipeId)!;

    // Remove ingredients
    for (const ingredient of recipe.ingredients) {
      inventory.removeItem(ingredient.itemId, ingredient.quantity);
    }

    // Add result
    inventory.addItem(recipe.result.itemId, recipe.result.quantity, this.itemDb);

    return true;
  }

  getAvailableRecipes(inventory: Inventory): CraftingRecipe[] {
    return Array.from(this.recipes.values()).filter((recipe) =>
      this.canCraft(recipe.id, inventory)
    );
  }
}
```

## Usage Examples

```typescript
// Example 1: Setup item database
const itemDb = new ItemDatabase();

itemDb.register({
  id: 'health_potion',
  name: 'Health Potion',
  description: 'Restores 50 HP',
  type: ItemType.Consumable,
  maxStack: 10,
  healAmount: 50,
  value: 25,
});

itemDb.register({
  id: 'iron_sword',
  name: 'Iron Sword',
  description: 'A sturdy iron sword',
  type: ItemType.Equipment,
  maxStack: 1,
  equipmentSlot: EquipmentSlot.Weapon,
  damage: 15,
  value: 100,
});

// Example 2: Player inventory
const player = world.createEntity();
const inventory = player.addComponent(new Inventory(20));
const equipment = player.addComponent(new Equipment());

// Add items
inventory.addItem('health_potion', 3, itemDb);
inventory.addItem('iron_sword', 1, itemDb);

// Example 3: Use item
const inventorySystem = new InventorySystem(itemDb, eventBus);

// Use health potion
if (inventory.hasItem('health_potion')) {
  inventorySystem.useItem(player, 'health_potion');
}

// Equip sword
inventorySystem.useItem(player, 'iron_sword');

// Example 4: Item pickup
const itemDrop = world.createEntity();
itemDrop.addComponent(new Transform(new Vector3(5, 0, 5)));
itemDrop.addComponent(new ItemPickup('health_potion', 1));

// Will be picked up automatically when player gets close

// Example 5: Crafting
const crafting = new CraftingSystem(itemDb);

crafting.addRecipe({
  id: 'craft_bandage',
  result: { itemId: 'bandage', quantity: 1 },
  ingredients: [
    { itemId: 'cloth', quantity: 2 },
  ],
});

// Craft item
if (crafting.canCraft('craft_bandage', inventory)) {
  crafting.craft('craft_bandage', inventory);
}

// Example 6: Drop item
inventorySystem.dropItem(player, 'health_potion', 1, world);

// Example 7: Equipment stats
const totalArmor = equipment.getTotalArmor(itemDb);
const totalDamage = equipment.getTotalDamage(itemDb);

console.log(`Armor: ${totalArmor}, Damage: ${totalDamage}`);
```

## Checklist

- [ ] Define item types and data
- [ ] Create inventory component
- [ ] Implement item stacking
- [ ] Add equipment system
- [ ] Create item pickups
- [ ] Implement item usage
- [ ] Add crafting system
- [ ] Create inventory UI
- [ ] Test item serialization
- [ ] Profile inventory performance

## Common Pitfalls

1. **No max stack limit**: Unlimited stacking
2. **Missing item checks**: Use items not owned
3. **Equipment without inventory**: Can't unequip
4. **No pickup radius**: Hard to collect items
5. **Forgetting serialization**: Can't save inventory
6. **Missing item database**: No item definitions
7. **No full inventory check**: Items disappear

## Performance Tips

### Inventory Optimization
- Cache item lookups
- Use slot indices instead of searching
- Limit inventory updates per frame
- Pool item pickup entities
- Batch UI updates

### Memory Optimization
- Share item definitions
- Pool inventory events
- Limit max inventory slots
- Clear empty slots

### Mobile Considerations
- Simpler inventory UI
- Touch-friendly item selection
- Fewer inventory slots
- Simpler crafting recipes
- Auto-pickup for items

## Related Skills

- `ecs-serialization` - Save/load inventory
- `ui-system` - Inventory UI
- `ecs-events` - Item events
- `spawn-system` - Item drops
- `ecs-component-patterns` - Component design

## References

- Inventory system design
- Item stacking algorithms
- Equipment slot systems
- Crafting mechanics
