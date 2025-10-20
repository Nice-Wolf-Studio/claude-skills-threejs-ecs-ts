---
name: health-combat-system
description: Health and combat system including damage, healing, armor, status effects, and combat mechanics
---

# Health and Combat System

## When to Use

Use this skill when:
- Implementing health and damage
- Creating combat mechanics
- Adding status effects
- Building armor/defense systems
- Implementing healing
- Creating damage types

## Core Principles

1. **Event-Driven**: Use events for damage/healing
2. **Extensible**: Support multiple damage types
3. **Balanced**: Clear combat formulas
4. **Feedback**: Visual/audio damage feedback
5. **Data-Driven**: Configure via components
6. **Invulnerability**: Temporary damage immunity

## Health and Combat Implementation

### 1. Health Components

```typescript
// components/Health.ts
export class Health {
  current: number;
  max: number;
  regenRate: number = 0; // HP per second
  isInvulnerable: boolean = false;
  invulnerabilityDuration: number = 0;

  constructor(max: number, current?: number, regenRate: number = 0) {
    this.max = max;
    this.current = current ?? max;
    this.regenRate = regenRate;
  }

  isDead(): boolean {
    return this.current <= 0;
  }

  isFullHealth(): boolean {
    return this.current >= this.max;
  }

  getHealthPercent(): number {
    return this.current / this.max;
  }

  damage(amount: number): void {
    if (this.isInvulnerable) return;

    this.current = Math.max(0, this.current - amount);
  }

  heal(amount: number): void {
    this.current = Math.min(this.max, this.current + amount);
  }

  setInvulnerable(duration: number): void {
    this.isInvulnerable = true;
    this.invulnerabilityDuration = duration;
  }
}

// components/Armor.ts
export class Armor {
  value: number; // Flat damage reduction
  percentage: number = 0; // 0-1, percentage reduction

  constructor(value: number = 0, percentage: number = 0) {
    this.value = value;
    this.percentage = percentage;
  }

  calculateReduction(damage: number): number {
    // Apply percentage reduction first
    let reduced = damage * (1 - this.percentage);

    // Then flat reduction
    reduced = Math.max(0, reduced - this.value);

    return reduced;
  }
}

// components/Attack.ts
export class Attack {
  damage: number;
  damageType: string = 'physical';
  range: number;
  cooldown: number = 0;
  cooldownTime: number;
  attackSpeed: number = 1; // Attacks per second

  constructor(
    damage: number,
    range: number,
    cooldownTime: number = 1,
    damageType: string = 'physical'
  ) {
    this.damage = damage;
    this.range = range;
    this.cooldownTime = cooldownTime;
    this.damageType = damageType;
  }

  canAttack(): boolean {
    return this.cooldown <= 0;
  }

  startCooldown(): void {
    this.cooldown = this.cooldownTime;
  }
}
```

### 2. Status Effects

```typescript
// components/StatusEffect.ts
export enum StatusEffectType {
  Poison = 'poison',
  Burn = 'burn',
  Freeze = 'freeze',
  Slow = 'slow',
  Stun = 'stun',
  Regeneration = 'regeneration',
  Strength = 'strength',
  Weakness = 'weakness',
}

export interface StatusEffectData {
  type: StatusEffectType;
  duration: number;
  tickInterval?: number; // For DoT effects
  value?: number; // Damage/heal per tick or stat modifier
}

export class StatusEffect {
  type: StatusEffectType;
  duration: number;
  remainingTime: number;
  tickInterval: number;
  timeSinceLastTick: number = 0;
  value: number;
  stacks: number = 1;
  maxStacks: number = 1;

  constructor(data: StatusEffectData) {
    this.type = data.type;
    this.duration = data.duration;
    this.remainingTime = data.duration;
    this.tickInterval = data.tickInterval ?? 1;
    this.value = data.value ?? 0;
  }

  update(deltaTime: number): boolean {
    this.remainingTime -= deltaTime;
    this.timeSinceLastTick += deltaTime;

    return this.remainingTime > 0;
  }

  shouldTick(): boolean {
    return this.timeSinceLastTick >= this.tickInterval;
  }

  tick(): void {
    this.timeSinceLastTick = 0;
  }

  addStack(): void {
    if (this.stacks < this.maxStacks) {
      this.stacks++;
    }
  }

  getEffectiveValue(): number {
    return this.value * this.stacks;
  }
}

export class StatusEffects {
  private effects = new Map<StatusEffectType, StatusEffect>();

  add(effect: StatusEffect): void {
    const existing = this.effects.get(effect.type);

    if (existing) {
      // Refresh duration and add stack
      existing.remainingTime = Math.max(existing.remainingTime, effect.duration);
      existing.addStack();
    } else {
      this.effects.set(effect.type, effect);
    }
  }

  remove(type: StatusEffectType): void {
    this.effects.delete(type);
  }

  has(type: StatusEffectType): boolean {
    return this.effects.has(type);
  }

  get(type: StatusEffectType): StatusEffect | undefined {
    return this.effects.get(type);
  }

  getAll(): StatusEffect[] {
    return Array.from(this.effects.values());
  }

  clear(): void {
    this.effects.clear();
  }
}
```

### 3. Combat System

```typescript
// systems/CombatSystem.ts
export class CombatSystem extends UpdateSystem {
  priority = 35;
  private eventBus: EventBus;

  constructor(eventBus: EventBus) {
    super();
    this.eventBus = eventBus;
  }

  update(world: World, deltaTime: number): void {
    // Update attack cooldowns
    this.updateCooldowns(world, deltaTime);

    // Update invulnerability
    this.updateInvulnerability(world, deltaTime);

    // Process attacks
    this.processAttacks(world);

    // Update status effects
    this.updateStatusEffects(world, deltaTime);

    // Health regeneration
    this.updateHealthRegen(world, deltaTime);

    // Check for deaths
    this.checkDeaths(world);
  }

  private updateCooldowns(world: World, deltaTime: number): void {
    const entities = world.query<[Attack]>([Attack]);

    entities.iterate((entity, [attack]) => {
      if (attack.cooldown > 0) {
        attack.cooldown = Math.max(0, attack.cooldown - deltaTime);
      }
    });
  }

  private updateInvulnerability(world: World, deltaTime: number): void {
    const entities = world.query<[Health]>([Health]);

    entities.iterate((entity, [health]) => {
      if (health.isInvulnerable) {
        health.invulnerabilityDuration -= deltaTime;

        if (health.invulnerabilityDuration <= 0) {
          health.isInvulnerable = false;
          health.invulnerabilityDuration = 0;
        }
      }
    });
  }

  private processAttacks(world: World): void {
    const attackers = world.query<[Transform, Attack]>([Transform, Attack]);
    const targets = world.query<[Transform, Health]>([Transform, Health]);

    attackers.iterate((attacker, [attackerTransform, attack]) => {
      if (!attack.canAttack()) return;

      // Find targets in range
      targets.iterate((target, [targetTransform, health]) => {
        if (target === attacker) return;

        const distance = attackerTransform.position.distanceTo(targetTransform.position);

        if (distance <= attack.range) {
          // Apply damage
          this.applyDamage(attacker, target, attack.damage, attack.damageType);
          attack.startCooldown();
        }
      });
    });
  }

  private applyDamage(
    source: Entity,
    target: Entity,
    baseDamage: number,
    damageType: string
  ): void {
    const health = target.getComponent(Health);
    if (!health || health.isInvulnerable) return;

    // Apply armor reduction
    let finalDamage = baseDamage;
    const armor = target.getComponent(Armor);

    if (armor) {
      finalDamage = armor.calculateReduction(baseDamage);
    }

    // Apply damage
    health.damage(finalDamage);

    // Emit damage event
    this.eventBus.emit(new DamageEvent(target, source, finalDamage, damageType));

    // Grant brief invulnerability
    health.setInvulnerable(0.1); // 100ms
  }

  private updateStatusEffects(world: World, deltaTime: number): void {
    const entities = world.query<[Health, StatusEffects]>([Health, StatusEffects]);

    entities.iterate((entity, [health, statusEffects]) => {
      const effects = statusEffects.getAll();

      for (const effect of effects) {
        // Update effect duration
        const stillActive = effect.update(deltaTime);

        if (!stillActive) {
          statusEffects.remove(effect.type);
          continue;
        }

        // Apply effect
        if (effect.shouldTick()) {
          this.applyStatusEffect(entity, health, effect);
          effect.tick();
        }
      }
    });
  }

  private applyStatusEffect(entity: Entity, health: Health, effect: StatusEffect): void {
    const value = effect.getEffectiveValue();

    switch (effect.type) {
      case StatusEffectType.Poison:
      case StatusEffectType.Burn:
        health.damage(value);
        break;

      case StatusEffectType.Regeneration:
        health.heal(value);
        break;

      case StatusEffectType.Stun:
        // Handled by movement/AI systems
        break;

      case StatusEffectType.Slow:
        // Handled by movement system
        break;
    }
  }

  private updateHealthRegen(world: World, deltaTime: number): void {
    const entities = world.query<[Health]>([Health]);

    entities.iterate((entity, [health]) => {
      if (health.regenRate > 0 && !health.isFullHealth()) {
        health.heal(health.regenRate * deltaTime);
      }
    });
  }

  private checkDeaths(world: World): void {
    const entities = world.query<[Health]>([Health]);

    entities.iterate((entity, [health]) => {
      if (health.isDead() && entity.active) {
        this.eventBus.emit(new DeathEvent(entity, null));
        entity.active = false;
      }
    });
  }
}
```

### 4. Damage Types System

```typescript
// combat/DamageTypeSystem.ts
export interface DamageResistance {
  physical: number; // 0-1 (0 = immune, 1 = full damage)
  fire: number;
  ice: number;
  poison: number;
  lightning: number;
}

export class DamageResistances {
  resistances: DamageResistance = {
    physical: 1,
    fire: 1,
    ice: 1,
    poison: 1,
    lightning: 1,
  };

  getResistance(damageType: string): number {
    return this.resistances[damageType as keyof DamageResistance] ?? 1;
  }

  setResistance(damageType: string, value: number): void {
    if (damageType in this.resistances) {
      this.resistances[damageType as keyof DamageResistance] = Math.max(0, Math.min(1, value));
    }
  }

  calculateDamage(baseDamage: number, damageType: string): number {
    return baseDamage * this.getResistance(damageType);
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic combat setup
const player = world.createEntity();
player.addComponent(new Health(100, 100, 5)); // 100 HP, 5 HP/s regen
player.addComponent(new Armor(10, 0.2)); // 10 flat reduction, 20% reduction
player.addComponent(new Attack(15, 2, 0.5)); // 15 damage, 2 range, 0.5s cooldown

const enemy = world.createEntity();
enemy.addComponent(new Health(50));
enemy.addComponent(new Attack(10, 1.5, 1));

// Example 2: Apply status effect
const statusEffects = entity.getComponent(StatusEffects) ||
  entity.addComponent(new StatusEffects());

// Apply poison
statusEffects.add(new StatusEffect({
  type: StatusEffectType.Poison,
  duration: 5,
  tickInterval: 1,
  value: 2, // 2 damage per second
}));

// Apply regeneration
statusEffects.add(new StatusEffect({
  type: StatusEffectType.Regeneration,
  duration: 10,
  tickInterval: 1,
  value: 5, // 5 HP per second
}));

// Example 3: Damage resistances
const resistances = enemy.addComponent(new DamageResistances());
resistances.setResistance('fire', 0.5); // 50% fire resistance
resistances.setResistance('ice', 1.5); // 150% ice damage (weakness)
resistances.setResistance('physical', 0); // Immune to physical

// Example 4: Healing potion
function useHealthPotion(entity: Entity): void {
  const health = entity.getComponent(Health);
  if (health) {
    health.heal(50);
    eventBus.emit(new HealEvent(entity, 50));
  }
}

// Example 5: Critical hits
class CriticalHitSystem extends UpdateSystem {
  private critChance = 0.2; // 20%
  private critMultiplier = 2;

  update(world: World, deltaTime: number): void {
    // Listen to damage events
    eventBus.on(DamageEvent, (event) => {
      if (Math.random() < this.critChance) {
        // Apply additional crit damage
        const critDamage = event.amount * (this.critMultiplier - 1);
        const health = event.target.getComponent(Health);
        if (health) {
          health.damage(critDamage);
          eventBus.emit(new CriticalHitEvent(event.target, event.source, critDamage));
        }
      }
    });
  }
}

// Example 6: Combo system
class ComboSystem extends UpdateSystem {
  private combos = new Map<Entity, { count: number; lastHitTime: number }>();
  private comboWindow = 2; // 2 seconds

  update(world: World, deltaTime: number): void {
    eventBus.on(DamageEvent, (event) => {
      if (!event.source) return;

      const now = performance.now() / 1000;
      const combo = this.combos.get(event.source) ?? { count: 0, lastHitTime: 0 };

      // Check if within combo window
      if (now - combo.lastHitTime < this.comboWindow) {
        combo.count++;
      } else {
        combo.count = 1;
      }

      combo.lastHitTime = now;
      this.combos.set(event.source, combo);

      // Apply combo multiplier
      if (combo.count > 1) {
        const multiplier = 1 + (combo.count - 1) * 0.1; // +10% per combo
        const bonusDamage = event.amount * (multiplier - 1);
        const health = event.target.getComponent(Health);
        if (health) {
          health.damage(bonusDamage);
        }
      }
    });
  }
}

// Example 7: Lifesteal
class LifestealSystem extends UpdateSystem {
  private lifestealPercent = 0.2; // 20%

  update(world: World, deltaTime: number): void {
    eventBus.on(DamageEvent, (event) => {
      if (!event.source) return;

      const sourceHealth = event.source.getComponent(Health);
      if (sourceHealth) {
        const healAmount = event.amount * this.lifestealPercent;
        sourceHealth.heal(healAmount);
      }
    });
  }
}
```

## Checklist

- [ ] Add Health components
- [ ] Implement Attack components
- [ ] Create combat system
- [ ] Add armor/resistance system
- [ ] Implement status effects
- [ ] Add healing mechanics
- [ ] Create damage events
- [ ] Add visual feedback
- [ ] Test combat balance
- [ ] Profile performance

## Common Pitfalls

1. **No invulnerability**: Hit multiple times per frame
2. **Wrong damage calculation**: Armor broken
3. **No death check**: Entities with negative HP
4. **Forgotten cooldowns**: Instant attacks
5. **Status effect stacking**: Unlimited stacks
6. **No resistance system**: All damage same
7. **Missing events**: No damage feedback

## Performance Tips

### Combat Optimization
- Update cooldowns once per frame
- Cache damage calculations
- Limit status effect checks
- Use spatial queries for range checks
- Batch damage events

### Memory Optimization
- Pool damage events
- Reuse status effect objects
- Limit active status effects
- Clear expired effects immediately

### Mobile Considerations
- Simpler damage calculations
- Fewer status effects
- Lower tick rates for DoT
- Simpler resistance system
- Fewer damage types

## Related Skills

- `ecs-events` - Combat events
- `ecs-system-patterns` - System implementation
- `audio-system` - Combat sounds
- `ui-system` - Health bars
- `collision-system` - Hit detection

## References

- Combat system design
- Damage calculation formulas
- Status effect patterns
- RPG mechanics design
