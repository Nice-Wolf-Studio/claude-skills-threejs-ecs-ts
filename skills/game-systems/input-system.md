---
name: input-system
description: Unified input system for ECS games supporting keyboard, mouse, touch, and gamepad with action mapping and input buffering
---

# Input System

## When to Use

Use this skill when:
- Building input handling for ECS games
- Supporting multiple input devices
- Implementing action mapping
- Creating input buffering for combos
- Building cross-platform controls

## Core Principles

1. **Device Abstraction**: Unified API for all input types
2. **Action Mapping**: Map inputs to game actions
3. **Input Buffering**: Store recent inputs for combos
4. **Frame-Perfect Input**: Sample input at consistent rate
5. **Priority Handling**: Process inputs in correct order

## Implementation

### 1. Input Component

```typescript
// components/Input.ts
import { Component } from '../core/Component';

export interface InputMapping {
  action: string;
  keys?: string[];
  buttons?: number[];
  touches?: boolean;
  axis?: { positive: string; negative: string };
}

export class Input implements Component {
  // Raw input state
  keys = new Set<string>();
  mouseButtons = new Set<number>();
  mouseX = 0;
  mouseY = 0;
  mouseDeltaX = 0;
  mouseDeltaY = 0;

  touches: Array<{ id: number; x: number; y: number }> = [];

  gamepadButtons = new Map<number, boolean>();
  gamepadAxes = new Map<number, number>();

  // Action states
  actions = new Map<string, boolean>();
  actionJustPressed = new Map<string, boolean>();
  actionJustReleased = new Map<string, boolean>();

  // Input buffer for combos
  buffer: Array<{ action: string; timestamp: number }> = [];
  bufferDuration = 200; // ms

  // Mappings
  mappings: InputMapping[] = [];

  constructor() {}

  registerMapping(mapping: InputMapping): void {
    this.mappings.push(mapping);
  }

  isActionActive(action: string): boolean {
    return this.actions.get(action) || false;
  }

  wasActionJustPressed(action: string): boolean {
    return this.actionJustPressed.get(action) || false;
  }

  wasActionJustReleased(action: string): boolean {
    return this.actionJustReleased.get(action) || false;
  }

  getAxis(action: string): number {
    const mapping = this.mappings.find(m => m.action === action && m.axis);

    if (mapping?.axis) {
      const positive = this.keys.has(mapping.axis.positive) ? 1 : 0;
      const negative = this.keys.has(mapping.axis.negative) ? -1 : 0;
      return positive + negative;
    }

    return 0;
  }

  addToBuffer(action: string, timestamp: number): void {
    this.buffer.push({ action, timestamp });

    // Clean old entries
    this.buffer = this.buffer.filter(
      entry => timestamp - entry.timestamp < this.bufferDuration
    );
  }

  checkCombo(actions: string[], currentTime: number): boolean {
    if (this.buffer.length < actions.length) return false;

    // Check if buffer contains sequence
    let bufferIndex = this.buffer.length - 1;

    for (let i = actions.length - 1; i >= 0; i--) {
      if (bufferIndex < 0) return false;

      while (
        bufferIndex >= 0 &&
        this.buffer[bufferIndex].action !== actions[i]
      ) {
        bufferIndex--;
      }

      if (bufferIndex < 0) return false;
      bufferIndex--;
    }

    return true;
  }
}
```

### 2. Input System

```typescript
// systems/InputSystem.ts
import { System } from '../core/System';
import { Input } from '../components/Input';
import { EntityManager } from '../core/EntityManager';
import { ComponentManager } from '../core/ComponentManager';

export class InputSystem extends System {
  private canvas: HTMLElement;
  private prevKeys = new Set<string>();
  private prevMouseButtons = new Set<number>();

  constructor(
    entities: EntityManager,
    components: ComponentManager,
    canvas: HTMLElement
  ) {
    super(entities, components);
    this.canvas = canvas;
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    // Keyboard
    window.addEventListener('keydown', this.onKeyDown);
    window.addEventListener('keyup', this.onKeyUp);

    // Mouse
    this.canvas.addEventListener('mousedown', this.onMouseDown);
    this.canvas.addEventListener('mouseup', this.onMouseUp);
    this.canvas.addEventListener('mousemove', this.onMouseMove);

    // Touch
    this.canvas.addEventListener('touchstart', this.onTouchStart, {
      passive: false,
    });
    this.canvas.addEventListener('touchmove', this.onTouchMove, {
      passive: false,
    });
    this.canvas.addEventListener('touchend', this.onTouchEnd, {
      passive: false,
    });

    // Prevent context menu
    this.canvas.addEventListener('contextmenu', e => e.preventDefault());
  }

  private onKeyDown = (e: KeyboardEvent): void => {
    const inputComponents = this.getInputComponents();
    inputComponents.forEach(input => {
      input.keys.add(e.key.toLowerCase());
    });
  };

  private onKeyUp = (e: KeyboardEvent): void {
    const inputComponents = this.getInputComponents();
    inputComponents.forEach(input => {
      input.keys.delete(e.key.toLowerCase());
    });
  };

  private onMouseDown = (e: MouseEvent): void {
    const inputComponents = this.getInputComponents();
    inputComponents.forEach(input => {
      input.mouseButtons.add(e.button);
    });
  };

  private onMouseUp = (e: MouseEvent): void {
    const inputComponents = this.getInputComponents();
    inputComponents.forEach(input => {
      input.mouseButtons.delete(e.button);
    });
  };

  private onMouseMove = (e: MouseEvent): void {
    const inputComponents = this.getInputComponents();
    inputComponents.forEach(input => {
      input.mouseDeltaX = e.movementX;
      input.mouseDeltaY = e.movementY;
      input.mouseX = e.clientX;
      input.mouseY = e.clientY;
    });
  };

  private onTouchStart = (e: TouchEvent): void => {
    e.preventDefault();
    const inputComponents = this.getInputComponents();

    inputComponents.forEach(input => {
      for (let i = 0; i < e.touches.length; i++) {
        const touch = e.touches[i];
        input.touches.push({
          id: touch.identifier,
          x: touch.clientX,
          y: touch.clientY,
        });
      }
    });
  };

  private onTouchMove = (e: TouchEvent): void => {
    e.preventDefault();
    const inputComponents = this.getInputComponents();

    inputComponents.forEach(input => {
      for (let i = 0; i < e.touches.length; i++) {
        const touch = e.touches[i];
        const existing = input.touches.find(t => t.id === touch.identifier);

        if (existing) {
          existing.x = touch.clientX;
          existing.y = touch.clientY;
        }
      }
    });
  };

  private onTouchEnd = (e: TouchEvent): void => {
    e.preventDefault();
    const inputComponents = this.getInputComponents();

    inputComponents.forEach(input => {
      for (let i = 0; i < e.changedTouches.length; i++) {
        const touch = e.changedTouches[i];
        const index = input.touches.findIndex(t => t.id === touch.identifier);

        if (index !== -1) {
          input.touches.splice(index, 1);
        }
      }
    });
  };

  private getInputComponents(): Input[] {
    const components: Input[] = [];

    for (const entity of this.entities.getAll()) {
      const input = this.components.get(entity, Input);
      if (input) {
        components.push(input);
      }
    }

    return components;
  }

  update(deltaTime: number): void {
    const currentTime = performance.now();

    for (const entity of this.entities.getAll()) {
      const input = this.components.get(entity, Input);
      if (!input) continue;

      // Clear just pressed/released states
      input.actionJustPressed.clear();
      input.actionJustReleased.clear();

      // Update actions from mappings
      for (const mapping of input.mappings) {
        let isActive = false;

        // Check keyboard keys
        if (mapping.keys) {
          isActive = mapping.keys.some(key => input.keys.has(key.toLowerCase()));
        }

        // Check mouse buttons
        if (mapping.buttons) {
          isActive = isActive || mapping.buttons.some(btn => input.mouseButtons.has(btn));
        }

        // Check touch
        if (mapping.touches && input.touches.length > 0) {
          isActive = true;
        }

        // Check gamepad buttons
        // ... similar logic for gamepad

        const wasActive = input.actions.get(mapping.action) || false;

        // Detect state changes
        if (isActive && !wasActive) {
          input.actionJustPressed.set(mapping.action, true);
          input.addToBuffer(mapping.action, currentTime);
        } else if (!isActive && wasActive) {
          input.actionJustReleased.set(mapping.action, true);
        }

        input.actions.set(mapping.action, isActive);
      }

      // Reset mouse delta
      input.mouseDeltaX = 0;
      input.mouseDeltaY = 0;
    }
  }

  dispose(): void {
    window.removeEventListener('keydown', this.onKeyDown);
    window.removeEventListener('keyup', this.onKeyUp);
    this.canvas.removeEventListener('mousedown', this.onMouseDown);
    this.canvas.removeEventListener('mouseup', this.onMouseUp);
    this.canvas.removeEventListener('mousemove', this.onMouseMove);
    this.canvas.removeEventListener('touchstart', this.onTouchStart);
    this.canvas.removeEventListener('touchmove', this.onTouchMove);
    this.canvas.removeEventListener('touchend', this.onTouchEnd);
  }
}
```

### 3. Player Input Component

```typescript
// components/PlayerInput.ts
export class PlayerInput extends Input {
  constructor() {
    super();

    // Register default mappings
    this.registerMapping({
      action: 'moveForward',
      keys: ['w', 'arrowup'],
      axis: { positive: 'w', negative: 's' },
    });

    this.registerMapping({
      action: 'moveBackward',
      keys: ['s', 'arrowdown'],
    });

    this.registerMapping({
      action: 'moveLeft',
      keys: ['a', 'arrowleft'],
    });

    this.registerMapping({
      action: 'moveRight',
      keys: ['d', 'arrowright'],
    });

    this.registerMapping({
      action: 'jump',
      keys: [' ', 'space'],
      touches: true,
    });

    this.registerMapping({
      action: 'shoot',
      keys: ['enter'],
      buttons: [0], // Left mouse button
    });

    this.registerMapping({
      action: 'reload',
      keys: ['r'],
    });
  }
}
```

### 4. Player Movement System

```typescript
// systems/PlayerMovementSystem.ts
import { System } from '../core/System';
import { PlayerInput } from '../components/PlayerInput';
import { Transform } from '../components/Transform';
import { Velocity } from '../components/Velocity';

export class PlayerMovementSystem extends System {
  update(deltaTime: number): void {
    for (const result of this.query(PlayerInput, Transform, Velocity)) {
      const input = result.components[0] as PlayerInput;
      const transform = result.components[1] as Transform;
      const velocity = result.components[2] as Velocity;

      const speed = 5;

      // Get input axes
      const moveX = input.getAxis('moveRight') - input.getAxis('moveLeft');
      const moveZ = input.getAxis('moveForward') - input.getAxis('moveBackward');

      // Normalize diagonal movement
      const length = Math.sqrt(moveX * moveX + moveZ * moveZ);
      const normalizedX = length > 0 ? moveX / length : 0;
      const normalizedZ = length > 0 ? moveZ / length : 0;

      // Apply velocity
      velocity.vx = normalizedX * speed;
      velocity.vz = normalizedZ * speed;

      // Jump
      if (input.wasActionJustPressed('jump')) {
        velocity.vy = 10;
      }

      // Check combo (e.g., double jump)
      if (input.checkCombo(['jump', 'jump'], performance.now())) {
        velocity.vy = 15;
        console.log('Double jump!');
      }
    }
  }
}
```

## Usage Examples

```typescript
// Setup input system
const canvas = document.getElementById('canvas') as HTMLElement;
const inputSystem = new InputSystem(entityManager, componentManager, canvas);
world.addSystem(inputSystem);

// Create player with input
const player = world.createEntity();
const playerInput = new PlayerInput();
world.addComponent(player, PlayerInput, playerInput);
world.addComponent(player, Transform, new Transform());
world.addComponent(player, Velocity, new Velocity());

// Custom action mapping
playerInput.registerMapping({
  action: 'dash',
  keys: ['shift'],
});

// Check input in system
function update() {
  if (playerInput.wasActionJustPressed('dash')) {
    console.log('Dash!');
  }

  if (playerInput.isActionActive('shoot')) {
    console.log('Shooting...');
  }

  // Check combo
  if (playerInput.checkCombo(['dash', 'jump'], performance.now())) {
    console.log('Dash jump combo!');
  }
}

// Cleanup
inputSystem.dispose();
```

## Checklist

- [ ] Set up keyboard event listeners
- [ ] Set up mouse event listeners
- [ ] Set up touch event listeners
- [ ] Implement action mapping system
- [ ] Add input buffering for combos
- [ ] Support multiple input devices
- [ ] Handle just pressed/released events
- [ ] Normalize diagonal movement
- [ ] Add axis input support
- [ ] Implement gamepad support (optional)
- [ ] Dispose event listeners on cleanup
- [ ] Test on all target platforms

## Common Pitfalls

1. **Not preventing default**: Touch causes scrolling
2. **No diagonal normalization**: Faster diagonal movement
3. **Forgetting to clear states**: Actions stick
4. **Not handling mobile**: No touch support
5. **Missing cleanup**: Event listener leaks
6. **Global input state**: Multiple players conflict

## Performance Tips

- Pool input event objects
- Cache input components
- Use Set for key tracking (O(1) lookup)
- Throttle mouse move events if expensive
- Batch input processing
- Use passive event listeners where possible
- Clear input buffer regularly

## Related Skills

- `touch-input-handling` - Mobile touch controls
- `ecs-component-patterns` - Component design
- `player-controller` - Player movement logic
- `combat-system` - Action handling
