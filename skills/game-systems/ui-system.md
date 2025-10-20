---
name: ui-system
description: UI system for games including HUD elements, menus, health bars, damage numbers, and UI state management
---

# UI System

## When to Use

Use this skill when:
- Creating game HUD/UI
- Displaying health bars and stats
- Implementing menus and dialogs
- Showing floating damage numbers
- Building inventory UI
- Creating pause menus

## Core Principles

1. **Separation**: UI separate from game logic
2. **Reactive**: UI responds to state changes
3. **Performant**: Minimize DOM updates
4. **Accessible**: Keyboard and screen reader support
5. **Responsive**: Works on different screen sizes
6. **Data-Driven**: UI driven by component data

## UI System Implementation

### 1. UI Components

```typescript
// components/UIElement.ts
export interface UIElementData {
  type: 'healthBar' | 'label' | 'image' | 'button' | 'panel';
  content?: string;
  position?: { x: number; y: number };
  size?: { width: number; height: number };
  style?: Partial<CSSStyleDeclaration>;
  visible?: boolean;
  anchored?: boolean; // Follow world position
}

export class UIElement {
  data: UIElementData;
  element: HTMLElement;
  private parentElement: HTMLElement;

  constructor(data: UIElementData, parentElement: HTMLElement) {
    this.data = data;
    this.parentElement = parentElement;
    this.element = this.createElement();
    this.applyStyles();

    if (data.visible !== false) {
      this.show();
    }
  }

  private createElement(): HTMLElement {
    let element: HTMLElement;

    switch (this.data.type) {
      case 'healthBar':
        element = document.createElement('div');
        element.className = 'ui-health-bar';

        const fill = document.createElement('div');
        fill.className = 'ui-health-bar-fill';
        element.appendChild(fill);
        break;

      case 'label':
        element = document.createElement('div');
        element.className = 'ui-label';
        element.textContent = this.data.content ?? '';
        break;

      case 'image':
        element = document.createElement('img');
        element.className = 'ui-image';
        (element as HTMLImageElement).src = this.data.content ?? '';
        break;

      case 'button':
        element = document.createElement('button');
        element.className = 'ui-button';
        element.textContent = this.data.content ?? '';
        break;

      case 'panel':
        element = document.createElement('div');
        element.className = 'ui-panel';
        break;

      default:
        element = document.createElement('div');
    }

    return element;
  }

  private applyStyles(): void {
    if (this.data.position) {
      this.element.style.position = 'absolute';
      this.element.style.left = `${this.data.position.x}px`;
      this.element.style.top = `${this.data.position.y}px`;
    }

    if (this.data.size) {
      this.element.style.width = `${this.data.size.width}px`;
      this.element.style.height = `${this.data.size.height}px`;
    }

    if (this.data.style) {
      Object.assign(this.element.style, this.data.style);
    }
  }

  show(): void {
    if (!this.element.parentElement) {
      this.parentElement.appendChild(this.element);
    }
    this.element.style.display = '';
    this.data.visible = true;
  }

  hide(): void {
    this.element.style.display = 'none';
    this.data.visible = false;
  }

  remove(): void {
    if (this.element.parentElement) {
      this.element.parentElement.removeChild(this.element);
    }
  }

  updatePosition(x: number, y: number): void {
    this.element.style.left = `${x}px`;
    this.element.style.top = `${y}px`;
  }

  updateContent(content: string): void {
    if (this.data.type === 'label' || this.data.type === 'button') {
      this.element.textContent = content;
    } else if (this.data.type === 'image') {
      (this.element as HTMLImageElement).src = content;
    }
    this.data.content = content;
  }

  updateHealthBar(current: number, max: number): void {
    if (this.data.type === 'healthBar') {
      const fill = this.element.querySelector('.ui-health-bar-fill') as HTMLElement;
      if (fill) {
        const percent = (current / max) * 100;
        fill.style.width = `${percent}%`;

        // Color based on health
        if (percent > 60) {
          fill.style.backgroundColor = '#00ff00';
        } else if (percent > 30) {
          fill.style.backgroundColor = '#ffff00';
        } else {
          fill.style.backgroundColor = '#ff0000';
        }
      }
    }
  }

  onClick(callback: (event: MouseEvent) => void): void {
    this.element.addEventListener('click', callback);
  }
}
```

### 2. UI Manager

```typescript
// ui/UIManager.ts
export class UIManager {
  private container: HTMLElement;
  private elements = new Map<string, UIElement>();
  private camera: THREE.Camera | null = null;

  constructor(containerId: string = 'ui-container') {
    let container = document.getElementById(containerId);

    if (!container) {
      container = document.createElement('div');
      container.id = containerId;
      container.style.position = 'absolute';
      container.style.top = '0';
      container.style.left = '0';
      container.style.width = '100%';
      container.style.height = '100%';
      container.style.pointerEvents = 'none';
      document.body.appendChild(container);
    }

    this.container = container;
    this.loadStyles();
  }

  private loadStyles(): void {
    const style = document.createElement('style');
    style.textContent = `
      .ui-health-bar {
        background: rgba(0, 0, 0, 0.5);
        border: 2px solid rgba(255, 255, 255, 0.3);
        border-radius: 3px;
        overflow: hidden;
      }

      .ui-health-bar-fill {
        height: 100%;
        background: #00ff00;
        transition: width 0.3s ease, background-color 0.3s ease;
      }

      .ui-label {
        color: white;
        text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
        font-family: Arial, sans-serif;
        font-size: 14px;
        white-space: nowrap;
        user-select: none;
      }

      .ui-button {
        pointer-events: auto;
        background: rgba(0, 0, 0, 0.7);
        border: 2px solid rgba(255, 255, 255, 0.3);
        color: white;
        padding: 10px 20px;
        cursor: pointer;
        font-size: 16px;
        border-radius: 5px;
      }

      .ui-button:hover {
        background: rgba(50, 50, 50, 0.9);
        border-color: rgba(255, 255, 255, 0.5);
      }

      .ui-panel {
        pointer-events: auto;
        background: rgba(0, 0, 0, 0.8);
        border: 2px solid rgba(255, 255, 255, 0.3);
        border-radius: 10px;
        padding: 20px;
      }

      .ui-image {
        user-select: none;
        -webkit-user-drag: none;
      }
    `;
    document.head.appendChild(style);
  }

  setCamera(camera: THREE.Camera): void {
    this.camera = camera;
  }

  createElement(id: string, data: UIElementData): UIElement {
    const element = new UIElement(data, this.container);
    this.elements.set(id, element);
    return element;
  }

  getElement(id: string): UIElement | undefined {
    return this.elements.get(id);
  }

  removeElement(id: string): void {
    const element = this.elements.get(id);
    if (element) {
      element.remove();
      this.elements.delete(id);
    }
  }

  worldToScreen(worldPos: Vector3): { x: number; y: number } | null {
    if (!this.camera) return null;

    const vector = worldPos.clone().project(this.camera);

    const x = (vector.x * 0.5 + 0.5) * this.container.clientWidth;
    const y = (vector.y * -0.5 + 0.5) * this.container.clientHeight;

    return { x, y };
  }

  clear(): void {
    this.elements.forEach((element) => element.remove());
    this.elements.clear();
  }

  hide(): void {
    this.container.style.display = 'none';
  }

  show(): void {
    this.container.style.display = '';
  }
}
```

### 3. UI System

```typescript
// systems/UISystem.ts
export class UISystem extends UpdateSystem {
  priority = 70; // Late update
  private uiManager: UIManager;

  constructor(uiManager: UIManager) {
    super();
    this.uiManager = uiManager;
  }

  update(world: World, deltaTime: number): void {
    // Update health bars
    this.updateHealthBars(world);

    // Update world-anchored UI
    this.updateWorldAnchored(world);

    // Update labels
    this.updateLabels(world);
  }

  private updateHealthBars(world: World): void {
    const entities = world.query<[Health, UIHealthBar]>([Health, UIHealthBar]);

    entities.iterate((entity, [health, uiHealthBar]) => {
      const element = this.uiManager.getElement(uiHealthBar.elementId);
      if (element) {
        element.updateHealthBar(health.current, health.max);

        // Position above entity if world-anchored
        if (uiHealthBar.worldAnchored) {
          const transform = entity.getComponent(Transform);
          if (transform) {
            const offset = new Vector3(0, uiHealthBar.worldOffset, 0);
            const worldPos = transform.position.clone().add(offset);
            const screenPos = this.uiManager.worldToScreen(worldPos);

            if (screenPos) {
              element.updatePosition(
                screenPos.x - (element.data.size?.width ?? 0) / 2,
                screenPos.y
              );
            }
          }
        }
      }
    });
  }

  private updateWorldAnchored(world: World): void {
    const entities = world.query<[Transform, UILabel]>([Transform, UILabel]);

    entities.iterate((entity, [transform, label]) => {
      if (!label.worldAnchored) return;

      const element = this.uiManager.getElement(label.elementId);
      if (element) {
        const offset = new Vector3(0, label.worldOffset, 0);
        const worldPos = transform.position.clone().add(offset);
        const screenPos = this.uiManager.worldToScreen(worldPos);

        if (screenPos) {
          element.updatePosition(screenPos.x, screenPos.y);
        }
      }
    });
  }

  private updateLabels(world: World): void {
    const entities = world.query<[UILabel]>([UILabel]);

    entities.iterate((entity, [label]) => {
      if (label.needsUpdate) {
        const element = this.uiManager.getElement(label.elementId);
        if (element) {
          element.updateContent(label.text);
        }
        label.needsUpdate = false;
      }
    });
  }
}
```

### 4. UI Components for Entities

```typescript
// components/UIHealthBar.ts
export class UIHealthBar {
  elementId: string;
  worldAnchored: boolean = true;
  worldOffset: number = 2; // Units above entity

  constructor(elementId: string) {
    this.elementId = elementId;
  }
}

// components/UILabel.ts
export class UILabel {
  elementId: string;
  text: string;
  worldAnchored: boolean = false;
  worldOffset: number = 0;
  needsUpdate: boolean = false;

  constructor(elementId: string, text: string) {
    this.elementId = elementId;
    this.text = text;
  }

  setText(text: string): void {
    this.text = text;
    this.needsUpdate = true;
  }
}
```

### 5. HUD Manager

```typescript
// ui/HUDManager.ts
export class HUDManager {
  private uiManager: UIManager;
  private playerHealthId = 'player-health';
  private scoreId = 'score';
  private ammoId = 'ammo';

  constructor(uiManager: UIManager) {
    this.uiManager = uiManager;
    this.createHUD();
  }

  private createHUD(): void {
    // Player health bar
    this.uiManager.createElement(this.playerHealthId, {
      type: 'healthBar',
      position: { x: 20, y: 20 },
      size: { width: 200, height: 20 },
    });

    // Score label
    this.uiManager.createElement(this.scoreId, {
      type: 'label',
      position: { x: 20, y: 50 },
      content: 'Score: 0',
      style: { fontSize: '18px' },
    });

    // Ammo label
    this.uiManager.createElement(this.ammoId, {
      type: 'label',
      position: { x: 20, y: 75 },
      content: 'Ammo: 30/120',
      style: { fontSize: '16px' },
    });
  }

  updateHealth(current: number, max: number): void {
    const element = this.uiManager.getElement(this.playerHealthId);
    if (element) {
      element.updateHealthBar(current, max);
    }
  }

  updateScore(score: number): void {
    const element = this.uiManager.getElement(this.scoreId);
    if (element) {
      element.updateContent(`Score: ${score}`);
    }
  }

  updateAmmo(current: number, reserve: number): void {
    const element = this.uiManager.getElement(this.ammoId);
    if (element) {
      element.updateContent(`Ammo: ${current}/${reserve}`);
    }
  }

  show(): void {
    this.uiManager.show();
  }

  hide(): void {
    this.uiManager.hide();
  }
}
```

### 6. Menu System

```typescript
// ui/MenuSystem.ts
export class MenuSystem {
  private uiManager: UIManager;
  private activeMenu: string | null = null;
  private menus = new Map<string, string[]>(); // menuId -> element IDs

  constructor(uiManager: UIManager) {
    this.uiManager = uiManager;
  }

  createMenu(
    menuId: string,
    items: Array<{ label: string; onClick: () => void }>
  ): void {
    const elementIds: string[] = [];
    const centerX = window.innerWidth / 2;
    const startY = window.innerHeight / 2 - (items.length * 60) / 2;

    items.forEach((item, index) => {
      const elementId = `${menuId}-item-${index}`;
      const button = this.uiManager.createElement(elementId, {
        type: 'button',
        position: { x: centerX - 100, y: startY + index * 60 },
        size: { width: 200, height: 50 },
        content: item.label,
        visible: false,
      });

      button.onClick(item.onClick);
      elementIds.push(elementId);
    });

    this.menus.set(menuId, elementIds);
  }

  showMenu(menuId: string): void {
    // Hide current menu
    if (this.activeMenu) {
      this.hideMenu(this.activeMenu);
    }

    // Show new menu
    const elementIds = this.menus.get(menuId);
    if (elementIds) {
      elementIds.forEach((id) => {
        const element = this.uiManager.getElement(id);
        if (element) {
          element.show();
        }
      });
      this.activeMenu = menuId;
    }
  }

  hideMenu(menuId: string): void {
    const elementIds = this.menus.get(menuId);
    if (elementIds) {
      elementIds.forEach((id) => {
        const element = this.uiManager.getElement(id);
        if (element) {
          element.hide();
        }
      });

      if (this.activeMenu === menuId) {
        this.activeMenu = null;
      }
    }
  }

  hideAllMenus(): void {
    this.menus.forEach((_, menuId) => this.hideMenu(menuId));
  }
}
```

## Usage Examples

```typescript
// Example 1: Setup UI system
const uiManager = new UIManager();
const uiSystem = new UISystem(uiManager);
systemManager.add(uiSystem);

// Set camera for world-to-screen projection
uiManager.setCamera(camera);

// Example 2: Add health bar to enemy
const enemy = world.createEntity();
enemy.addComponent(new Transform(new Vector3(5, 0, 5)));
enemy.addComponent(new Health(100, 100));

const healthBarId = `enemy-${enemy.id}-health`;
uiManager.createElement(healthBarId, {
  type: 'healthBar',
  size: { width: 50, height: 5 },
  visible: true,
});

enemy.addComponent(new UIHealthBar(healthBarId));

// Example 3: Create HUD
const hudManager = new HUDManager(uiManager);

// Update HUD
function updateHUD(): void {
  const player = world.query<[Player, Health]>([Player, Health]).first();
  if (player) {
    const health = player.getComponent(Health);
    hudManager.updateHealth(health.current, health.max);
  }

  hudManager.updateScore(gameState.score);
  hudManager.updateAmmo(weapon.currentAmmo, weapon.reserveAmmo);
}

// Example 4: Create pause menu
const menuSystem = new MenuSystem(uiManager);

menuSystem.createMenu('pause', [
  { label: 'Resume', onClick: () => resumeGame() },
  { label: 'Settings', onClick: () => menuSystem.showMenu('settings') },
  { label: 'Quit', onClick: () => quitGame() },
]);

// Show pause menu
function pauseGame(): void {
  menuSystem.showMenu('pause');
  game.paused = true;
}

// Example 5: Floating damage numbers
function showDamageNumber(entity: Entity, damage: number): void {
  const transform = entity.getComponent(Transform);
  if (!transform) return;

  const id = `damage-${Date.now()}-${Math.random()}`;
  const element = uiManager.createElement(id, {
    type: 'label',
    content: `-${damage}`,
    style: {
      fontSize: '24px',
      fontWeight: 'bold',
      color: '#ff0000',
    },
  });

  const worldPos = transform.position.clone().add(new Vector3(0, 2, 0));
  const screenPos = uiManager.worldToScreen(worldPos);

  if (screenPos) {
    element.updatePosition(screenPos.x, screenPos.y);

    // Animate upward and fade out
    let elapsed = 0;
    const duration = 1;
    const startY = screenPos.y;

    const animate = () => {
      elapsed += 0.016; // ~60fps
      const t = elapsed / duration;

      if (t >= 1) {
        uiManager.removeElement(id);
        return;
      }

      element.updatePosition(screenPos.x, startY - t * 50);
      element.element.style.opacity = `${1 - t}`;

      requestAnimationFrame(animate);
    };

    animate();
  }
}

// Example 6: Inventory UI
class InventoryUI {
  private uiManager: UIManager;
  private slots: UIElement[] = [];
  private visible = false;

  constructor(uiManager: UIManager, slotCount: number) {
    this.uiManager = uiManager;
    this.createSlots(slotCount);
  }

  private createSlots(count: number): void {
    const startX = window.innerWidth / 2 - (count * 60) / 2;
    const y = window.innerHeight - 80;

    for (let i = 0; i < count; i++) {
      const slot = this.uiManager.createElement(`inv-slot-${i}`, {
        type: 'panel',
        position: { x: startX + i * 60, y },
        size: { width: 50, height: 50 },
        visible: false,
      });

      this.slots.push(slot);
    }
  }

  show(): void {
    this.slots.forEach((slot) => slot.show());
    this.visible = true;
  }

  hide(): void {
    this.slots.forEach((slot) => slot.hide());
    this.visible = false;
  }

  updateSlot(index: number, itemName: string, count: number): void {
    if (index >= this.slots.length) return;

    const slot = this.slots[index];
    // Add item icon and count
    slot.element.innerHTML = `
      <div style="text-align: center; padding: 5px;">
        <div style="font-size: 10px; color: white;">${itemName}</div>
        <div style="font-size: 12px; color: yellow;">${count}</div>
      </div>
    `;
  }
}
```

## Checklist

- [ ] Create UI manager
- [ ] Set up UI container
- [ ] Add CSS styles
- [ ] Create HUD elements
- [ ] Implement world-to-screen projection
- [ ] Add health bars
- [ ] Create menu system
- [ ] Handle UI events
- [ ] Test on different screen sizes
- [ ] Optimize UI updates

## Common Pitfalls

1. **Too many DOM updates**: Update only when changed
2. **No world-to-screen**: UI doesn't follow 3D objects
3. **Z-index issues**: UI elements overlap incorrectly
4. **No responsive design**: Breaks on mobile
5. **Blocking pointer events**: Can't click through UI
6. **Heavy CSS**: Performance issues
7. **Forgetting cleanup**: Memory leaks

## Performance Tips

### UI Optimization
- Batch DOM updates
- Use CSS transforms for animation
- Minimize reflows and repaints
- Cache screen positions
- Update only visible elements

### Memory Optimization
- Remove unused UI elements
- Reuse UI elements when possible
- Avoid creating elements in update loop
- Clear event listeners

### Mobile Considerations
- Larger touch targets (44x44px minimum)
- Simpler UI with fewer elements
- Avoid hover states
- Use viewport meta tag
- Test on actual devices

## Related Skills

- `ecs-events` - UI events
- `ecs-system-patterns` - System implementation
- `input-system` - UI interaction
- `threejs-raycasting` - World picking
- `mobile-performance` - Mobile optimization

## References

- DOM manipulation best practices
- CSS performance optimization
- Responsive design principles
- Accessibility guidelines (WCAG)
