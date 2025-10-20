---
name: touch-input-handling
description: Comprehensive touch input system for mobile Three.js games including gestures, virtual joystick, and multi-touch support
---

# Touch Input Handling

## When to Use

Use this skill when:
- Building mobile Three.js games
- Implementing touch controls
- Creating virtual joysticks
- Supporting multi-touch gestures
- Converting mouse code to touch

## Core Principles

1. **Touch Events**: Use touchstart/touchmove/touchend
2. **Gesture Detection**: Recognize swipes, pinch, rotate
3. **Virtual Controls**: Joysticks and buttons for mobile
4. **Multi-touch**: Support multiple simultaneous touches
5. **Prevent Default**: Stop unwanted scrolling/zooming
6. **Normalize Input**: Abstract mouse and touch to unified API

## Implementation

### 1. Touch Manager

```typescript
export interface TouchPoint {
  id: number;
  x: number;
  y: number;
  startX: number;
  startY: number;
  deltaX: number;
  deltaY: number;
  force: number;
}

export class TouchManager {
  private touches = new Map<number, TouchPoint>();
  private canvas: HTMLElement;
  private callbacks = {
    onTouchStart: new Set<(touch: TouchPoint) => void>(),
    onTouchMove: new Set<(touch: TouchPoint) => void>(),
    onTouchEnd: new Set<(touch: TouchPoint) => void>(),
    onSwipe: new Set<(direction: 'left' | 'right' | 'up' | 'down', distance: number) => void>(),
    onPinch: new Set<(scale: number, center: { x: number; y: number }) => void>(),
  };

  constructor(canvas: HTMLElement) {
    this.canvas = canvas;
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.canvas.addEventListener('touchstart', this.onTouchStart, { passive: false });
    this.canvas.addEventListener('touchmove', this.onTouchMove, { passive: false });
    this.canvas.addEventListener('touchend', this.onTouchEnd, { passive: false });
    this.canvas.addEventListener('touchcancel', this.onTouchEnd, { passive: false });
  }

  private onTouchStart = (event: TouchEvent): void => {
    event.preventDefault();

    for (let i = 0; i < event.changedTouches.length; i++) {
      const touch = event.changedTouches[i];
      const rect = this.canvas.getBoundingClientRect();

      const touchPoint: TouchPoint = {
        id: touch.identifier,
        x: touch.clientX - rect.left,
        y: touch.clientY - rect.top,
        startX: touch.clientX - rect.left,
        startY: touch.clientY - rect.top,
        deltaX: 0,
        deltaY: 0,
        force: (touch as any).force || 1.0,
      };

      this.touches.set(touch.identifier, touchPoint);

      this.callbacks.onTouchStart.forEach(cb => cb(touchPoint));
    }
  };

  private onTouchMove = (event: TouchEvent): void => {
    event.preventDefault();

    for (let i = 0; i < event.changedTouches.length; i++) {
      const touch = event.changedTouches[i];
      const touchPoint = this.touches.get(touch.identifier);

      if (touchPoint) {
        const rect = this.canvas.getBoundingClientRect();
        const newX = touch.clientX - rect.left;
        const newY = touch.clientY - rect.top;

        touchPoint.deltaX = newX - touchPoint.x;
        touchPoint.deltaY = newY - touchPoint.y;
        touchPoint.x = newX;
        touchPoint.y = newY;
        touchPoint.force = (touch as any).force || 1.0;

        this.callbacks.onTouchMove.forEach(cb => cb(touchPoint));
      }
    }

    // Detect gestures
    this.detectGestures(event);
  };

  private onTouchEnd = (event: TouchEvent): void => {
    event.preventDefault();

    for (let i = 0; i < event.changedTouches.length; i++) {
      const touch = event.changedTouches[i];
      const touchPoint = this.touches.get(touch.identifier);

      if (touchPoint) {
        // Detect swipe
        const dx = touchPoint.x - touchPoint.startX;
        const dy = touchPoint.y - touchPoint.startY;
        const distance = Math.sqrt(dx * dx + dy * dy);

        if (distance > 50) {
          const angle = Math.atan2(dy, dx);
          const direction = this.getSwipeDirection(angle);
          this.callbacks.onSwipe.forEach(cb => cb(direction, distance));
        }

        this.callbacks.onTouchEnd.forEach(cb => cb(touchPoint));
        this.touches.delete(touch.identifier);
      }
    }
  };

  private getSwipeDirection(angle: number): 'left' | 'right' | 'up' | 'down' {
    const deg = (angle * 180) / Math.PI;

    if (deg >= -45 && deg <= 45) return 'right';
    if (deg > 45 && deg < 135) return 'down';
    if (deg >= 135 || deg <= -135) return 'left';
    return 'up';
  }

  private detectGestures(event: TouchEvent): void {
    // Pinch gesture (2 fingers)
    if (event.touches.length === 2) {
      const touch1 = event.touches[0];
      const touch2 = event.touches[1];

      const dist = Math.hypot(
        touch2.clientX - touch1.clientX,
        touch2.clientY - touch1.clientY
      );

      const centerX = (touch1.clientX + touch2.clientX) / 2;
      const centerY = (touch1.clientY + touch2.clientY) / 2;

      // Store initial distance for scale calculation
      if (!this['initialPinchDist']) {
        this['initialPinchDist'] = dist;
      }

      const scale = dist / this['initialPinchDist'];

      this.callbacks.onPinch.forEach(cb => cb(scale, { x: centerX, y: centerY }));
    } else {
      delete this['initialPinchDist'];
    }
  }

  on(
    event: 'touchStart' | 'touchMove' | 'touchEnd' | 'swipe' | 'pinch',
    callback: any
  ): () => void {
    const eventMap = {
      touchStart: this.callbacks.onTouchStart,
      touchMove: this.callbacks.onTouchMove,
      touchEnd: this.callbacks.onTouchEnd,
      swipe: this.callbacks.onSwipe,
      pinch: this.callbacks.onPinch,
    };

    eventMap[event].add(callback);

    return () => {
      eventMap[event].delete(callback);
    };
  }

  getTouches(): TouchPoint[] {
    return Array.from(this.touches.values());
  }

  getTouchCount(): number {
    return this.touches.size;
  }

  dispose(): void {
    this.canvas.removeEventListener('touchstart', this.onTouchStart);
    this.canvas.removeEventListener('touchmove', this.onTouchMove);
    this.canvas.removeEventListener('touchend', this.onTouchEnd);
    this.canvas.removeEventListener('touchcancel', this.onTouchEnd);
  }
}
```

### 2. Virtual Joystick

```typescript
export class VirtualJoystick {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private active = false;
  private baseX = 0;
  private baseY = 0;
  private stickX = 0;
  private stickY = 0;
  private maxDistance = 50;
  private touchId: number | null = null;

  constructor(container: HTMLElement, x: number, y: number) {
    this.canvas = document.createElement('canvas');
    this.canvas.width = 150;
    this.canvas.height = 150;
    this.canvas.style.position = 'absolute';
    this.canvas.style.left = `${x}px`;
    this.canvas.style.bottom = `${y}px`;
    this.canvas.style.pointerEvents = 'none';
    this.canvas.style.zIndex = '1000';

    container.appendChild(this.canvas);

    this.ctx = this.canvas.getContext('2d')!;

    this.baseX = this.canvas.width / 2;
    this.baseY = this.canvas.height / 2;
    this.stickX = this.baseX;
    this.stickY = this.baseY;

    this.setupTouchListeners(container);
    this.draw();
  }

  private setupTouchListeners(container: HTMLElement): void {
    container.addEventListener('touchstart', this.onTouchStart, { passive: false });
    container.addEventListener('touchmove', this.onTouchMove, { passive: false });
    container.addEventListener('touchend', this.onTouchEnd, { passive: false });
  }

  private onTouchStart = (event: TouchEvent): void => {
    if (this.touchId !== null) return;

    const touch = event.touches[0];
    const rect = this.canvas.getBoundingClientRect();

    // Check if touch is near joystick
    const dx = touch.clientX - (rect.left + this.baseX);
    const dy = touch.clientY - (rect.top + this.baseY);
    const dist = Math.sqrt(dx * dx + dy * dy);

    if (dist < this.maxDistance * 2) {
      this.active = true;
      this.touchId = touch.identifier;
      event.preventDefault();
    }
  };

  private onTouchMove = (event: TouchEvent): void => {
    if (this.touchId === null) return;

    for (let i = 0; i < event.touches.length; i++) {
      const touch = event.touches[i];

      if (touch.identifier === this.touchId) {
        const rect = this.canvas.getBoundingClientRect();

        let dx = touch.clientX - (rect.left + this.baseX);
        let dy = touch.clientY - (rect.top + this.baseY);

        const dist = Math.sqrt(dx * dx + dy * dy);

        if (dist > this.maxDistance) {
          const angle = Math.atan2(dy, dx);
          dx = Math.cos(angle) * this.maxDistance;
          dy = Math.sin(angle) * this.maxDistance;
        }

        this.stickX = this.baseX + dx;
        this.stickY = this.baseY + dy;

        this.draw();
        event.preventDefault();
        break;
      }
    }
  };

  private onTouchEnd = (event: TouchEvent): void => {
    for (let i = 0; i < event.changedTouches.length; i++) {
      if (event.changedTouches[i].identifier === this.touchId) {
        this.active = false;
        this.touchId = null;
        this.stickX = this.baseX;
        this.stickY = this.baseY;
        this.draw();
        break;
      }
    }
  };

  private draw(): void {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // Draw base
    this.ctx.beginPath();
    this.ctx.arc(this.baseX, this.baseY, this.maxDistance, 0, Math.PI * 2);
    this.ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
    this.ctx.fill();
    this.ctx.strokeStyle = 'rgba(255, 255, 255, 0.8)';
    this.ctx.lineWidth = 2;
    this.ctx.stroke();

    // Draw stick
    this.ctx.beginPath();
    this.ctx.arc(this.stickX, this.stickY, 25, 0, Math.PI * 2);
    this.ctx.fillStyle = this.active
      ? 'rgba(100, 150, 255, 0.8)'
      : 'rgba(255, 255, 255, 0.5)';
    this.ctx.fill();
    this.ctx.strokeStyle = 'rgba(255, 255, 255, 1)';
    this.ctx.stroke();
  }

  /**
   * Get normalized joystick direction (-1 to 1)
   */
  getDirection(): { x: number; y: number } {
    const dx = this.stickX - this.baseX;
    const dy = this.stickY - this.baseY;

    return {
      x: dx / this.maxDistance,
      y: dy / this.maxDistance,
    };
  }

  /**
   * Get angle in radians
   */
  getAngle(): number {
    const dx = this.stickX - this.baseX;
    const dy = this.stickY - this.baseY;
    return Math.atan2(dy, dx);
  }

  /**
   * Get magnitude (0 to 1)
   */
  getMagnitude(): number {
    const dx = this.stickX - this.baseX;
    const dy = this.stickY - this.baseY;
    const dist = Math.sqrt(dx * dx + dy * dy);
    return Math.min(dist / this.maxDistance, 1);
  }

  isActive(): boolean {
    return this.active;
  }

  dispose(): void {
    this.canvas.remove();
  }
}
```

### 3. Unified Input System

```typescript
export interface InputState {
  pointer: { x: number; y: number };
  buttons: Map<string, boolean>;
  axes: Map<string, { x: number; y: number }>;
}

export class UnifiedInputSystem {
  private state: InputState = {
    pointer: { x: 0, y: 0 },
    buttons: new Map(),
    axes: new Map(),
  };

  private touchManager: TouchManager;
  private joystick?: VirtualJoystick;

  constructor(canvas: HTMLElement, enableJoystick: boolean = true) {
    this.touchManager = new TouchManager(canvas);

    if (enableJoystick) {
      this.joystick = new VirtualJoystick(canvas, 80, 80);
    }

    this.setupListeners();
  }

  private setupListeners(): void {
    // Touch events
    this.touchManager.on('touchStart', (touch: TouchPoint) => {
      this.state.pointer = { x: touch.x, y: touch.y };
      this.state.buttons.set('primary', true);
    });

    this.touchManager.on('touchMove', (touch: TouchPoint) => {
      this.state.pointer = { x: touch.x, y: touch.y };
    });

    this.touchManager.on('touchEnd', () => {
      this.state.buttons.set('primary', false);
    });

    this.touchManager.on('swipe', (direction, distance) => {
      this.state.buttons.set(`swipe-${direction}`, true);

      // Clear swipe after frame
      setTimeout(() => {
        this.state.buttons.set(`swipe-${direction}`, false);
      }, 100);
    });

    // Mouse events (desktop fallback)
    window.addEventListener('keydown', (e) => {
      this.state.buttons.set(e.key.toLowerCase(), true);
    });

    window.addEventListener('keyup', (e) => {
      this.state.buttons.set(e.key.toLowerCase(), false);
    });
  }

  update(): void {
    // Update joystick axis
    if (this.joystick) {
      const dir = this.joystick.getDirection();
      this.state.axes.set('movement', { x: dir.x, y: -dir.y }); // Flip Y
    }

    // Update keyboard axis (WASD)
    const keyX =
      (this.getButton('d') ? 1 : 0) +
      (this.getButton('a') ? -1 : 0);
    const keyY =
      (this.getButton('w') ? 1 : 0) +
      (this.getButton('s') ? -1 : 0);

    if (keyX !== 0 || keyY !== 0) {
      const len = Math.sqrt(keyX * keyX + keyY * keyY);
      this.state.axes.set('movement', {
        x: keyX / len,
        y: keyY / len,
      });
    }
  }

  getButton(name: string): boolean {
    return this.state.buttons.get(name) || false;
  }

  getAxis(name: string): { x: number; y: number } {
    return this.state.axes.get(name) || { x: 0, y: 0 };
  }

  getPointer(): { x: number; y: number } {
    return this.state.pointer;
  }

  dispose(): void {
    this.touchManager.dispose();
    this.joystick?.dispose();
  }
}
```

## Usage Examples

```typescript
// Touch manager
const touchManager = new TouchManager(canvas);

touchManager.on('touchStart', (touch) => {
  console.log('Touch started:', touch.x, touch.y);
});

touchManager.on('swipe', (direction, distance) => {
  console.log('Swiped:', direction, distance);
});

touchManager.on('pinch', (scale, center) => {
  camera.zoom *= scale;
});

// Virtual joystick
const joystick = new VirtualJoystick(container, 80, 80);

function animate() {
  const dir = joystick.getDirection();
  const magnitude = joystick.getMagnitude();

  player.position.x += dir.x * magnitude * speed;
  player.position.z += dir.y * magnitude * speed;

  requestAnimationFrame(animate);
}

// Unified input (mouse + touch + keyboard)
const input = new UnifiedInputSystem(canvas);

function gameLoop() {
  input.update();

  const movement = input.getAxis('movement');
  player.velocity.set(movement.x * speed, 0, movement.y * speed);

  if (input.getButton('primary') || input.getButton(' ')) {
    player.jump();
  }

  requestAnimationFrame(gameLoop);
}
```

## Checklist

- [ ] Call preventDefault() to stop scrolling
- [ ] Handle touchstart/touchmove/touchend
- [ ] Support multi-touch (track touch IDs)
- [ ] Implement swipe gesture detection
- [ ] Create virtual joystick for movement
- [ ] Normalize touch coordinates to canvas
- [ ] Add mouse events as fallback
- [ ] Detect pinch/zoom gestures
- [ ] Implement button debouncing
- [ ] Test on actual mobile devices
- [ ] Support both portrait and landscape
- [ ] Clean up event listeners on dispose

## Common Pitfalls

1. **Not calling preventDefault()**: Page scrolls during gameplay
2. **Ignoring touch IDs**: Multi-touch doesn't work
3. **Not testing on real devices**: Emulator behavior differs
4. **Forgetting mouse fallback**: Desktop testing difficult
5. **Poor joystick positioning**: Blocks game view
6. **No visual feedback**: Users don't know where to touch

## Performance Tips

- Use passive: false only when needed
- Throttle touchmove events if processing is expensive
- Pool touch point objects
- Cache getBoundingClientRect() results
- Use CSS transforms for joystick (hardware accelerated)
- Minimize canvas redraws for virtual controls

## Related Skills

- `raycasting` - Touch object picking
- `input-system` - ECS input component
- `camera-controls` - Touch camera controls
- `mobile-performance` - Touch input optimization
