---
name: threejs-animation-systems
description: Three.js animation implementation with AnimationMixer, clips, actions, blending, and skeletal animation for characters
---

# Three.js Animation Systems

## When to Use

Use this skill when:
- Animating 3D models and characters
- Implementing character locomotion
- Playing animation clips from GLTF/FBX
- Blending between animations
- Creating animation state machines

## Core Principles

1. **AnimationMixer**: Central controller for all animations
2. **AnimationClips**: Reusable animation data
3. **AnimationActions**: Runtime instances of clips
4. **Weight Blending**: Smooth transitions between animations
5. **Time Control**: Speed, loops, clamping
6. **Performance**: Update only active animations

## Implementation

### 1. Animation Manager

```typescript
import * as THREE from 'three';

export class AnimationManager {
  private mixers = new Map<THREE.Object3D, THREE.AnimationMixer>();
  private actions = new Map<string, THREE.AnimationAction>();
  private currentAction: THREE.AnimationAction | null = null;

  /**
   * Create mixer for model
   */
  createMixer(model: THREE.Object3D): THREE.AnimationMixer {
    if (this.mixers.has(model)) {
      return this.mixers.get(model)!;
    }

    const mixer = new THREE.AnimationMixer(model);
    this.mixers.set(model, mixer);
    return mixer;
  }

  /**
   * Add animation clip
   */
  addClip(
    model: THREE.Object3D,
    clip: THREE.AnimationClip,
    name: string
  ): THREE.AnimationAction {
    const mixer = this.createMixer(model);
    const action = mixer.clipAction(clip);
    this.actions.set(name, action);
    return action;
  }

  /**
   * Play animation
   */
  play(
    name: string,
    options?: {
      loop?: THREE.AnimationActionLoopStyles;
      repetitions?: number;
      clampWhenFinished?: boolean;
      fadeIn?: number;
    }
  ): THREE.AnimationAction | null {
    const action = this.actions.get(name);
    if (!action) return null;

    // Configure action
    if (options?.loop !== undefined) {
      action.setLoop(options.loop, options.repetitions ?? Infinity);
    }

    if (options?.clampWhenFinished) {
      action.clampWhenFinished = true;
    }

    // Fade in
    if (options?.fadeIn && options.fadeIn > 0) {
      action.reset();
      action.fadeIn(options.fadeIn);
    }

    action.play();
    this.currentAction = action;

    return action;
  }

  /**
   * Cross-fade between animations
   */
  crossFade(
    fromName: string,
    toName: string,
    duration: number = 0.3
  ): THREE.AnimationAction | null {
    const fromAction = this.actions.get(fromName);
    const toAction = this.actions.get(toName);

    if (!fromAction || !toAction) return null;

    // Start new action
    toAction.reset();
    toAction.setEffectiveTimeScale(1);
    toAction.setEffectiveWeight(1);
    toAction.play();

    // Cross-fade
    fromAction.crossFadeTo(toAction, duration, true);

    this.currentAction = toAction;
    return toAction;
  }

  /**
   * Stop animation
   */
  stop(name: string, fadeOut: number = 0): void {
    const action = this.actions.get(name);
    if (!action) return;

    if (fadeOut > 0) {
      action.fadeOut(fadeOut);
      setTimeout(() => action.stop(), fadeOut * 1000);
    } else {
      action.stop();
    }
  }

  /**
   * Update all mixers
   */
  update(deltaTime: number): void {
    this.mixers.forEach(mixer => mixer.update(deltaTime));
  }

  /**
   * Get action by name
   */
  getAction(name: string): THREE.AnimationAction | undefined {
    return this.actions.get(name);
  }

  /**
   * Dispose all animations
   */
  dispose(): void {
    this.actions.forEach(action => action.stop());
    this.mixers.forEach(mixer => mixer.uncacheRoot(mixer.getRoot()));
    this.mixers.clear();
    this.actions.clear();
  }
}
```

### 2. Animation State Machine

```typescript
export type AnimationState = 'idle' | 'walk' | 'run' | 'jump' | 'attack';

export interface StateTransition {
  from: AnimationState;
  to: AnimationState;
  duration: number;
}

export class AnimationStateMachine {
  private currentState: AnimationState = 'idle';
  private animationManager: AnimationManager;
  private transitions: StateTransition[] = [];

  constructor(animationManager: AnimationManager) {
    this.animationManager = animationManager;
  }

  /**
   * Add state transition
   */
  addTransition(from: AnimationState, to: AnimationState, duration: number = 0.3): void {
    this.transitions.push({ from, to, duration });
  }

  /**
   * Change state with transition
   */
  setState(newState: AnimationState): void {
    if (newState === this.currentState) return;

    const transition = this.transitions.find(
      t => t.from === this.currentState && t.to === newState
    );

    const duration = transition?.duration ?? 0.3;

    this.animationManager.crossFade(this.currentState, newState, duration);
    this.currentState = newState;
  }

  getCurrentState(): AnimationState {
    return this.currentState;
  }
}
```

### 3. Character Animation Controller

```typescript
export class CharacterAnimationController {
  private animationManager: AnimationManager;
  private stateMachine: AnimationStateMachine;
  private isMoving = false;
  private isRunning = false;
  private isJumping = false;

  constructor(model: THREE.Object3D, clips: THREE.AnimationClip[]) {
    this.animationManager = new AnimationManager();

    // Add all clips
    clips.forEach(clip => {
      this.animationManager.addClip(model, clip, clip.name);
    });

    // Set up state machine
    this.stateMachine = new AnimationStateMachine(this.animationManager);

    // Define transitions
    this.stateMachine.addTransition('idle', 'walk', 0.2);
    this.stateMachine.addTransition('walk', 'idle', 0.2);
    this.stateMachine.addTransition('walk', 'run', 0.3);
    this.stateMachine.addTransition('run', 'walk', 0.3);
    this.stateMachine.addTransition('idle', 'jump', 0.1);
    this.stateMachine.addTransition('walk', 'jump', 0.1);
    this.stateMachine.addTransition('run', 'jump', 0.1);
    this.stateMachine.addTransition('jump', 'idle', 0.2);

    // Start with idle
    this.animationManager.play('idle', {
      loop: THREE.LoopRepeat,
    });
  }

  /**
   * Set movement state
   */
  setMovement(velocity: number): void {
    const wasMoving = this.isMoving;
    const wasRunning = this.isRunning;

    this.isMoving = Math.abs(velocity) > 0.1;
    this.isRunning = Math.abs(velocity) > 5;

    if (!this.isJumping) {
      if (this.isRunning && !wasRunning) {
        this.stateMachine.setState('run');
      } else if (this.isMoving && !this.isRunning && wasRunning) {
        this.stateMachine.setState('walk');
      } else if (this.isMoving && !wasMoving) {
        this.stateMachine.setState('walk');
      } else if (!this.isMoving && wasMoving) {
        this.stateMachine.setState('idle');
      }
    }
  }

  /**
   * Trigger jump
   */
  jump(): void {
    if (this.isJumping) return;

    this.isJumping = true;
    this.stateMachine.setState('jump');

    // Get jump animation duration
    const jumpAction = this.animationManager.getAction('jump');
    if (jumpAction) {
      const duration = jumpAction.getClip().duration;

      setTimeout(() => {
        this.isJumping = false;
        this.stateMachine.setState(this.isMoving ? 'walk' : 'idle');
      }, duration * 1000);
    }
  }

  /**
   * Trigger attack
   */
  attack(): void {
    const currentState = this.stateMachine.getCurrentState();

    // Play attack animation
    this.animationManager.play('attack', {
      loop: THREE.LoopOnce,
      clampWhenFinished: true,
      fadeIn: 0.1,
    });

    // Return to previous state after attack
    const attackAction = this.animationManager.getAction('attack');
    if (attackAction) {
      const duration = attackAction.getClip().duration;

      setTimeout(() => {
        this.stateMachine.setState(currentState);
      }, duration * 1000);
    }
  }

  /**
   * Update animations
   */
  update(deltaTime: number): void {
    this.animationManager.update(deltaTime);
  }

  dispose(): void {
    this.animationManager.dispose();
  }
}
```

### 4. Animation Blending

```typescript
export class AnimationBlender {
  private mixer: THREE.AnimationMixer;
  private blendedAction?: THREE.AnimationAction;

  constructor(model: THREE.Object3D) {
    this.mixer = new THREE.AnimationMixer(model);
  }

  /**
   * Blend two animations with weight
   */
  blend(
    clip1: THREE.AnimationClip,
    clip2: THREE.AnimationClip,
    weight: number // 0 = clip1, 1 = clip2
  ): void {
    const action1 = this.mixer.clipAction(clip1);
    const action2 = this.mixer.clipAction(clip2);

    action1.setEffectiveWeight(1 - weight);
    action2.setEffectiveWeight(weight);

    action1.play();
    action2.play();
  }

  /**
   * Create additive animation
   */
  createAdditiveAnimation(
    baseClip: THREE.AnimationClip,
    additiveClip: THREE.AnimationClip,
    weight: number = 1.0
  ): void {
    const baseAction = this.mixer.clipAction(baseClip);
    const additiveAction = this.mixer.clipAction(additiveClip);

    baseAction.play();

    // Set additive blending
    additiveAction.blendMode = THREE.AdditiveAnimationBlendMode;
    additiveAction.setEffectiveWeight(weight);
    additiveAction.play();
  }

  update(deltaTime: number): void {
    this.mixer.update(deltaTime);
  }
}
```

### 5. Procedural Animation

```typescript
export class ProceduralAnimation {
  /**
   * Animate object along path
   */
  static animateAlongPath(
    object: THREE.Object3D,
    path: THREE.CurvePath<THREE.Vector3>,
    duration: number,
    onUpdate?: (t: number) => void
  ): void {
    const startTime = performance.now();

    const animate = () => {
      const elapsed = (performance.now() - startTime) / 1000;
      const t = Math.min(elapsed / duration, 1);

      const point = path.getPointAt(t);
      object.position.copy(point);

      // Look ahead for rotation
      if (t < 1) {
        const lookAt = path.getPointAt(Math.min(t + 0.01, 1));
        object.lookAt(lookAt);
      }

      onUpdate?.(t);

      if (t < 1) {
        requestAnimationFrame(animate);
      }
    };

    animate();
  }

  /**
   * Bounce animation
   */
  static bounce(
    object: THREE.Object3D,
    height: number,
    duration: number
  ): void {
    const startY = object.position.y;
    const startTime = performance.now();

    const animate = () => {
      const elapsed = (performance.now() - startTime) / 1000;
      const t = (elapsed % duration) / duration;

      // Parabolic bounce
      const bounce = Math.sin(t * Math.PI);
      object.position.y = startY + bounce * height;

      requestAnimationFrame(animate);
    };

    animate();
  }

  /**
   * Rotate continuously
   */
  static rotate(
    object: THREE.Object3D,
    axis: 'x' | 'y' | 'z',
    speed: number
  ): () => void {
    const startTime = performance.now();
    let animationId: number;

    const animate = () => {
      const elapsed = (performance.now() - startTime) / 1000;

      object.rotation[axis] = elapsed * speed;

      animationId = requestAnimationFrame(animate);
    };

    animate();

    // Return stop function
    return () => cancelAnimationFrame(animationId);
  }
}
```

## Usage Examples

```typescript
// Load model with animations
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

const loader = new GLTFLoader();
loader.load('/models/character.glb', (gltf) => {
  const model = gltf.scene;
  const animations = gltf.animations;

  // Create animation controller
  const controller = new CharacterAnimationController(model, animations);

  // Update in game loop
  function animate(deltaTime: number) {
    // Update based on input
    const velocity = getPlayerVelocity();
    controller.setMovement(velocity);

    // Update animations
    controller.update(deltaTime);
  }

  // Handle input
  if (input.wasActionJustPressed('jump')) {
    controller.jump();
  }

  if (input.wasActionJustPressed('attack')) {
    controller.attack();
  }
});

// Simple animation manager
const animManager = new AnimationManager();
animManager.addClip(model, walkClip, 'walk');
animManager.addClip(model, runClip, 'run');

// Cross-fade
animManager.crossFade('walk', 'run', 0.3);

// Update
animManager.update(deltaTime);
```

## Checklist

- [ ] Create AnimationMixer for each animated model
- [ ] Load animation clips from GLTF/FBX
- [ ] Set up AnimationActions for each clip
- [ ] Implement state machine for transitions
- [ ] Configure loop modes (LoopOnce, LoopRepeat, LoopPingPong)
- [ ] Use cross-fading for smooth transitions
- [ ] Set clampWhenFinished for one-shot animations
- [ ] Update mixer every frame with delta time
- [ ] Dispose mixers when models are removed
- [ ] Test all animation transitions
- [ ] Optimize: only update active animations
- [ ] Use time scale for slow-motion effects

## Common Pitfalls

1. **Not updating mixer**: Animations don't play
2. **Wrong delta time units**: Animations too fast/slow
3. **Memory leaks**: Not disposing mixers
4. **Instant transitions**: No cross-fading looks choppy
5. **Wrong loop mode**: Animations don't repeat properly
6. **Not resetting actions**: Actions start from wrong time

## Performance Tips

- Only update active mixers (skip when paused/inactive)
- Use `AnimationAction.setEffectiveWeight(0)` to disable without stopping
- Cache AnimationActions, don't recreate each frame
- Limit number of simultaneous animations per model
- Use simpler rigs for mobile (fewer bones)
- Consider animation LOD (simpler animations at distance)
- Use `mixer.stopAllAction()` when model is far away

## Related Skills

- `threejs-model-loading` - Loading animated models
- `threejs-skeletal-animation` - Advanced bone animation
- `ecs-component-patterns` - Animation components
- `camera-system` - Syncing camera with animations
