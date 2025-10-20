---
name: level-system
description: Level and progression system including experience, leveling, skill trees, and stat progression
---

# Level and Progression System

## When to Use

Use this skill when:
- Implementing character leveling
- Creating experience systems
- Building skill trees
- Managing stat progression
- Implementing unlockables
- Creating progression curves

## Core Principles

1. **Balanced Progression**: Fair XP curves
2. **Rewarding**: Clear benefits from leveling
3. **Data-Driven**: Configure progression in data
4. **Event-Driven**: Level-up events
5. **Flexible**: Support different progression types
6. **Serializable**: Save/load progress

## Level System Implementation

### 1. Experience and Level Components

```typescript
// components/Experience.ts
export class Experience {
  current: number = 0;
  level: number = 1;
  maxLevel: number = 100;

  constructor(level: number = 1, current: number = 0) {
    this.level = level;
    this.current = current;
  }

  getRequiredXP(level: number): number {
    // Exponential curve: XP needed = 100 * level^1.5
    return Math.floor(100 * Math.pow(level, 1.5));
  }

  getXPForCurrentLevel(): number {
    return this.getRequiredXP(this.level);
  }

  getXPProgress(): number {
    const required = this.getXPForCurrentLevel();
    return this.current / required;
  }

  addXP(amount: number): number[] {
    this.current += amount;
    const levelsGained: number[] = [];

    // Check for level ups
    while (this.current >= this.getXPForCurrentLevel() && this.level < this.maxLevel) {
      this.current -= this.getXPForCurrentLevel();
      this.level++;
      levelsGained.push(this.level);
    }

    // Cap current XP at required for max level
    if (this.level >= this.maxLevel) {
      this.current = 0;
    }

    return levelsGained;
  }

  getXPToNextLevel(): number {
    return this.getXPForCurrentLevel() - this.current;
  }
}

// components/Stats.ts
export interface StatValues {
  strength: number;
  agility: number;
  intelligence: number;
  vitality: number;
}

export class Stats {
  base: StatValues = {
    strength: 10,
    agility: 10,
    intelligence: 10,
    vitality: 10,
  };

  bonuses: StatValues = {
    strength: 0,
    agility: 0,
    intelligence: 0,
    vitality: 0,
  };

  availablePoints: number = 0;

  constructor(base?: Partial<StatValues>) {
    if (base) {
      Object.assign(this.base, base);
    }
  }

  getTotal(stat: keyof StatValues): number {
    return this.base[stat] + this.bonuses[stat];
  }

  getAllTotals(): StatValues {
    return {
      strength: this.getTotal('strength'),
      agility: this.getTotal('agility'),
      intelligence: this.getTotal('intelligence'),
      vitality: this.getTotal('vitality'),
    };
  }

  addPoint(stat: keyof StatValues): boolean {
    if (this.availablePoints > 0) {
      this.base[stat]++;
      this.availablePoints--;
      return true;
    }
    return false;
  }

  addBonus(stat: keyof StatValues, amount: number): void {
    this.bonuses[stat] += amount;
  }

  removeBonus(stat: keyof StatValues, amount: number): void {
    this.bonuses[stat] -= amount;
  }
}
```

### 2. Level System

```typescript
// systems/LevelSystem.ts
export class LevelSystem extends UpdateSystem {
  priority = 38;
  private eventBus: EventBus;

  constructor(eventBus: EventBus) {
    super();
    this.eventBus = eventBus;
  }

  update(world: World, deltaTime: number): void {
    // Listen for XP gain events (handled by event system)
    // Update stats based on level
    this.updateStatsFromLevel(world);
  }

  private updateStatsFromLevel(world: World): void {
    const entities = world.query<[Experience, Stats, Health, Attack]>([
      Experience,
      Stats,
      Health,
      Attack,
    ]);

    entities.iterate((entity, [experience, stats, health, attack]) => {
      // Update health based on vitality
      const vitality = stats.getTotal('vitality');
      health.max = 100 + vitality * 10;

      // Update damage based on strength
      const strength = stats.getTotal('strength');
      attack.damage = 10 + strength * 2;
    });
  }

  grantXP(entity: Entity, amount: number): void {
    const experience = entity.getComponent(Experience);
    if (!experience) return;

    const levelsGained = experience.addXP(amount);

    // Emit XP gained event
    this.eventBus.emit(new XPGainedEvent(entity, amount));

    // Handle level ups
    for (const newLevel of levelsGained) {
      this.onLevelUp(entity, newLevel);
    }
  }

  private onLevelUp(entity: Entity, newLevel: number): void {
    const stats = entity.getComponent(Stats);
    if (stats) {
      // Grant stat points on level up
      stats.availablePoints += 5;
    }

    // Emit level up event
    this.eventBus.emit(new LevelUpEvent(entity, newLevel));

    // Full heal on level up
    const health = entity.getComponent(Health);
    if (health) {
      health.current = health.max;
    }
  }
}
```

### 3. Skill Tree System

```typescript
// progression/SkillTree.ts
export interface SkillNode {
  id: string;
  name: string;
  description: string;
  icon?: string;
  maxRank: number;
  prerequisites?: string[]; // Required skill IDs
  requiredLevel?: number;
  cost?: number; // Skill points needed
}

export class SkillTree {
  private nodes = new Map<string, SkillNode>();
  private unlocked = new Map<string, number>(); // skillId -> rank

  addNode(node: SkillNode): void {
    this.nodes.set(node.id, node);
  }

  canUnlock(skillId: string, level: number, availablePoints: number): boolean {
    const node = this.nodes.get(skillId);
    if (!node) return false;

    // Check level requirement
    if (node.requiredLevel && level < node.requiredLevel) {
      return false;
    }

    // Check cost
    if (node.cost && availablePoints < node.cost) {
      return false;
    }

    // Check current rank
    const currentRank = this.unlocked.get(skillId) ?? 0;
    if (currentRank >= node.maxRank) {
      return false;
    }

    // Check prerequisites
    if (node.prerequisites) {
      for (const prereqId of node.prerequisites) {
        if (!this.unlocked.has(prereqId)) {
          return false;
        }
      }
    }

    return true;
  }

  unlock(skillId: string): boolean {
    const node = this.nodes.get(skillId);
    if (!node) return false;

    const currentRank = this.unlocked.get(skillId) ?? 0;
    this.unlocked.set(skillId, currentRank + 1);

    return true;
  }

  getRank(skillId: string): number {
    return this.unlocked.get(skillId) ?? 0;
  }

  isUnlocked(skillId: string): boolean {
    return this.unlocked.has(skillId);
  }

  getUnlockedSkills(): string[] {
    return Array.from(this.unlocked.keys());
  }

  reset(): void {
    this.unlocked.clear();
  }
}

// components/Skills.ts
export class Skills {
  tree: SkillTree;
  points: number = 0;

  constructor() {
    this.tree = new SkillTree();
  }

  unlockSkill(skillId: string, level: number): boolean {
    if (!this.tree.canUnlock(skillId, level, this.points)) {
      return false;
    }

    const node = this.tree['nodes'].get(skillId);
    if (node?.cost) {
      this.points -= node.cost;
    }

    return this.tree.unlock(skillId);
  }

  addPoints(amount: number): void {
    this.points += amount;
  }
}
```

### 4. Achievement System

```typescript
// progression/AchievementSystem.ts
export interface Achievement {
  id: string;
  name: string;
  description: string;
  icon?: string;
  hidden?: boolean;
  rewards?: {
    xp?: number;
    items?: Array<{ itemId: string; quantity: number }>;
  };
}

export class AchievementManager {
  private achievements = new Map<string, Achievement>();
  private unlocked = new Set<string>();
  private progress = new Map<string, number>();
  private eventBus: EventBus;

  constructor(eventBus: EventBus) {
    this.eventBus = eventBus;
  }

  register(achievement: Achievement): void {
    this.achievements.set(achievement.id, achievement);
  }

  unlock(achievementId: string, entity: Entity): void {
    if (this.unlocked.has(achievementId)) return;

    const achievement = this.achievements.get(achievementId);
    if (!achievement) return;

    this.unlocked.add(achievementId);

    // Grant rewards
    if (achievement.rewards?.xp) {
      const experience = entity.getComponent(Experience);
      if (experience) {
        experience.addXP(achievement.rewards.xp);
      }
    }

    this.eventBus.emit(new AchievementUnlockedEvent(entity, achievementId));
  }

  setProgress(achievementId: string, progress: number): void {
    this.progress.set(achievementId, progress);

    // Check if achievement should be unlocked
    if (progress >= 1) {
      // Achievement complete
    }
  }

  getProgress(achievementId: string): number {
    return this.progress.get(achievementId) ?? 0;
  }

  isUnlocked(achievementId: string): boolean {
    return this.unlocked.has(achievementId);
  }

  getUnlockedAchievements(): Achievement[] {
    return Array.from(this.unlocked)
      .map((id) => this.achievements.get(id))
      .filter((a): a is Achievement => a !== undefined);
  }

  getUnlockedCount(): number {
    return this.unlocked.size;
  }

  getTotalCount(): number {
    return this.achievements.size;
  }
}
```

## Usage Examples

```typescript
// Example 1: Basic leveling
const player = world.createEntity();
const experience = player.addComponent(new Experience(1, 0));
const stats = player.addComponent(new Stats({ strength: 15, vitality: 12 }));

const levelSystem = new LevelSystem(eventBus);
systemManager.add(levelSystem);

// Grant XP
levelSystem.grantXP(player, 150);

// Check level
console.log(`Level: ${experience.level}`);
console.log(`XP: ${experience.current}/${experience.getXPForCurrentLevel()}`);

// Example 2: Stat allocation
eventBus.on(LevelUpEvent, (event) => {
  const stats = event.entity.getComponent(Stats);
  if (stats) {
    console.log(`Level up! ${stats.availablePoints} points available`);
  }
});

// Allocate points
if (stats.availablePoints > 0) {
  stats.addPoint('strength');
  stats.addPoint('vitality');
}

// Example 3: Skill tree
const skills = player.addComponent(new Skills());

// Define skill tree
skills.tree.addNode({
  id: 'fireball',
  name: 'Fireball',
  description: 'Cast a fireball',
  maxRank: 3,
  requiredLevel: 5,
  cost: 1,
});

skills.tree.addNode({
  id: 'meteor',
  name: 'Meteor',
  description: 'Summon a meteor',
  maxRank: 1,
  prerequisites: ['fireball'],
  requiredLevel: 10,
  cost: 2,
});

// Unlock skill
if (skills.tree.canUnlock('fireball', experience.level, skills.points)) {
  skills.unlockSkill('fireball', experience.level);
}

// Example 4: XP from kills
eventBus.on(DeathEvent, (event) => {
  // Grant XP to killer
  if (event.source) {
    const sourceExp = event.source.getComponent(Experience);
    if (sourceExp) {
      const xpAmount = event.entity.getComponent(Experience)?.level * 10 ?? 50;
      levelSystem.grantXP(event.source, xpAmount);
    }
  }
});

// Example 5: Achievements
const achievementManager = new AchievementManager(eventBus);

achievementManager.register({
  id: 'first_kill',
  name: 'First Blood',
  description: 'Defeat your first enemy',
  rewards: { xp: 100 },
});

achievementManager.register({
  id: 'level_10',
  name: 'Experienced',
  description: 'Reach level 10',
  rewards: { xp: 500 },
});

eventBus.on(DeathEvent, (event) => {
  if (event.source) {
    achievementManager.unlock('first_kill', event.source);
  }
});

eventBus.on(LevelUpEvent, (event) => {
  if (event.level >= 10) {
    achievementManager.unlock('level_10', event.entity);
  }
});

// Example 6: Progression curves
class ProgressionCurve {
  static linear(level: number, baseXP: number): number {
    return baseXP * level;
  }

  static exponential(level: number, baseXP: number, exponent: number = 1.5): number {
    return Math.floor(baseXP * Math.pow(level, exponent));
  }

  static logarithmic(level: number, baseXP: number): number {
    return Math.floor(baseXP * Math.log(level + 1) * 10);
  }
}

// Example 7: Prestige system
class PrestigeSystem {
  private prestigeLevel = 0;
  private prestigeBonuses: StatValues = {
    strength: 0,
    agility: 0,
    intelligence: 0,
    vitality: 0,
  };

  prestige(entity: Entity): void {
    const experience = entity.getComponent(Experience);
    const stats = entity.getComponent(Stats);

    if (!experience || !stats) return;

    // Requirements
    if (experience.level < experience.maxLevel) return;

    // Grant permanent bonuses
    this.prestigeLevel++;
    this.prestigeBonuses.strength += 5;
    this.prestigeBonuses.vitality += 5;

    // Reset level
    experience.level = 1;
    experience.current = 0;

    // Reset stats but keep prestige bonuses
    stats.base = {
      strength: 10 + this.prestigeBonuses.strength,
      agility: 10 + this.prestigeBonuses.agility,
      intelligence: 10 + this.prestigeBonuses.intelligence,
      vitality: 10 + this.prestigeBonuses.vitality,
    };
  }
}
```

## Checklist

- [ ] Create experience component
- [ ] Define level progression curve
- [ ] Implement stat system
- [ ] Add level-up rewards
- [ ] Create skill tree
- [ ] Implement achievements
- [ ] Add XP sources
- [ ] Create progression UI
- [ ] Test balance
- [ ] Serialize progression

## Common Pitfalls

1. **Unbalanced curves**: Too fast/slow leveling
2. **No level cap**: Unlimited growth
3. **Missing rewards**: No incentive to level
4. **Complex skill trees**: Hard to understand
5. **No stat reset**: Can't fix mistakes
6. **Forgetting serialization**: Progress lost
7. **Missing validation**: Can cheat stats

## Performance Tips

### Progression Optimization
- Cache XP calculations
- Update stats only on change
- Limit achievement checks
- Use event-driven XP grants
- Batch stat updates

### Memory Optimization
- Pool progression events
- Limit skill tree size
- Cache stat totals
- Serialize efficiently

### Mobile Considerations
- Simpler skill trees
- Fewer stats to track
- Clearer progression UI
- Touch-friendly stat allocation
- Auto-save progress

## Related Skills

- `ecs-serialization` - Save/load progress
- `ecs-events` - Level-up events
- `ui-system` - Progression UI
- `health-combat-system` - Stat effects
- `inventory-system` - Level rewards

## References

- XP curve design
- Skill tree systems
- RPG stat mechanics
- Achievement systems
