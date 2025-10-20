---
name: threejs-raycasting
description: Mouse and touch interaction using raycasting for object picking, collision detection, and UI interaction in Three.js
---

# Three.js Raycasting and Interaction

## When to Use

Use this skill when:
- Implementing mouse/touch object picking
- Detecting clicks on 3D objects
- Creating interactive UI in 3D space
- Performing collision/intersection tests
- Implementing drag-and-drop mechanics

## Core Principles

1. **Raycaster Reuse**: Create once, reuse for all casts
2. **Layer Filtering**: Use layers to limit raycast targets
3. **Recursive vs Non-recursive**: Choose based on scene depth
4. **Touch Normalization**: Convert touch to NDC coordinates
5. **Performance**: Limit raycast frequency and targets

## Implementation

### 1. Basic Raycaster

```typescript
import * as THREE from 'three';

export class RaycastManager {
  private raycaster = new THREE.Raycaster();
  private pointer = new THREE.Vector2();
  private camera: THREE.Camera;
  private domElement: HTMLElement;

  constructor(camera: THREE.Camera, domElement: HTMLElement) {
    this.camera = camera;
    this.domElement = domElement;
  }

  /**
   * Update pointer from mouse event
   */
  updatePointerFromMouse(event: MouseEvent): void {
    const rect = this.domElement.getBoundingClientRect();

    this.pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    this.pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
  }

  /**
   * Update pointer from touch event
   */
  updatePointerFromTouch(event: TouchEvent): void {
    if (event.touches.length === 0) return;

    const rect = this.domElement.getBoundingClientRect();
    const touch = event.touches[0];

    this.pointer.x = ((touch.clientX - rect.left) / rect.width) * 2 - 1;
    this.pointer.y = -((touch.clientY - rect.top) / rect.height) * 2 + 1;
  }

  /**
   * Cast ray and return intersections
   */
  cast(
    objects: THREE.Object3D[],
    recursive: boolean = true
  ): THREE.Intersection[] {
    this.raycaster.setFromCamera(this.pointer, this.camera);
    return this.raycaster.intersectObjects(objects, recursive);
  }

  /**
   * Get first intersection
   */
  castFirst(
    objects: THREE.Object3D[],
    recursive: boolean = true
  ): THREE.Intersection | null {
    const intersections = this.cast(objects, recursive);
    return intersections.length > 0 ? intersections[0] : null;
  }

  /**
   * Cast ray from world position and direction
   */
  castFromRay(
    origin: THREE.Vector3,
    direction: THREE.Vector3,
    objects: THREE.Object3D[],
    recursive: boolean = true
  ): THREE.Intersection[] {
    this.raycaster.set(origin, direction.normalize());
    return this.raycaster.intersectObjects(objects, recursive);
  }

  /**
   * Set raycast layers
   */
  setLayers(layers: number[]): void {
    this.raycaster.layers.disableAll();
    layers.forEach(layer => this.raycaster.layers.enable(layer));
  }

  /**
   * Set raycast distance limit
   */
  setNearFar(near: number, far: number): void {
    this.raycaster.near = near;
    this.raycaster.far = far;
  }

  getRaycaster(): THREE.Raycaster {
    return this.raycaster;
  }
}
```

### 2. Click Handler

```typescript
export interface ClickableObject {
  object: THREE.Object3D;
  onClick: (intersection: THREE.Intersection) => void;
  onHover?: (intersection: THREE.Intersection) => void;
  onHoverEnd?: () => void;
}

export class ClickHandler {
  private raycastManager: RaycastManager;
  private clickables: ClickableObject[] = [];
  private hoveredObject: ClickableObject | null = null;
  private isDragging = false;
  private dragThreshold = 5;
  private mouseDownPos = new THREE.Vector2();

  constructor(
    camera: THREE.Camera,
    domElement: HTMLElement
  ) {
    this.raycastManager = new RaycastManager(camera, domElement);
    this.setupEventListeners(domElement);
  }

  private setupEventListeners(domElement: HTMLElement): void {
    domElement.addEventListener('mousedown', this.onMouseDown);
    domElement.addEventListener('mousemove', this.onMouseMove);
    domElement.addEventListener('mouseup', this.onMouseUp);
    domElement.addEventListener('touchstart', this.onTouchStart);
    domElement.addEventListener('touchmove', this.onTouchMove);
    domElement.addEventListener('touchend', this.onTouchEnd);
  }

  private onMouseDown = (event: MouseEvent): void => {
    this.isDragging = false;
    this.mouseDownPos.set(event.clientX, event.clientY);
  };

  private onMouseMove = (event: MouseEvent): void => {
    const dist = new THREE.Vector2(event.clientX, event.clientY)
      .sub(this.mouseDownPos)
      .length();

    if (dist > this.dragThreshold) {
      this.isDragging = true;
    }

    this.raycastManager.updatePointerFromMouse(event);
    this.updateHover();
  };

  private onMouseUp = (event: MouseEvent): void => {
    if (this.isDragging) {
      this.isDragging = false;
      return;
    }

    this.raycastManager.updatePointerFromMouse(event);
    this.handleClick();
  };

  private onTouchStart = (event: TouchEvent): void => {
    if (event.touches.length === 1) {
      this.isDragging = false;
      const touch = event.touches[0];
      this.mouseDownPos.set(touch.clientX, touch.clientY);
    }
  };

  private onTouchMove = (event: TouchEvent): void => {
    if (event.touches.length === 1) {
      const touch = event.touches[0];
      const dist = new THREE.Vector2(touch.clientX, touch.clientY)
        .sub(this.mouseDownPos)
        .length();

      if (dist > this.dragThreshold) {
        this.isDragging = true;
      }
    }
  };

  private onTouchEnd = (event: TouchEvent): void => {
    if (this.isDragging) {
      this.isDragging = false;
      return;
    }

    if (event.changedTouches.length > 0) {
      this.raycastManager.updatePointerFromTouch(event);
      this.handleClick();
    }
  };

  private handleClick(): void {
    const objects = this.clickables.map(c => c.object);
    const intersection = this.raycastManager.castFirst(objects, true);

    if (intersection) {
      const clickable = this.clickables.find(c =>
        this.isChildOf(intersection.object, c.object)
      );

      if (clickable) {
        clickable.onClick(intersection);
      }
    }
  }

  private updateHover(): void {
    const objects = this.clickables.map(c => c.object);
    const intersection = this.raycastManager.castFirst(objects, true);

    if (intersection) {
      const clickable = this.clickables.find(c =>
        this.isChildOf(intersection.object, c.object)
      );

      if (clickable !== this.hoveredObject) {
        if (this.hoveredObject?.onHoverEnd) {
          this.hoveredObject.onHoverEnd();
        }

        this.hoveredObject = clickable || null;

        if (clickable?.onHover) {
          clickable.onHover(intersection);
        }
      }
    } else {
      if (this.hoveredObject?.onHoverEnd) {
        this.hoveredObject.onHoverEnd();
      }
      this.hoveredObject = null;
    }
  }

  private isChildOf(child: THREE.Object3D, parent: THREE.Object3D): boolean {
    if (child === parent) return true;
    if (!child.parent) return false;
    return this.isChildOf(child.parent, parent);
  }

  addClickable(clickable: ClickableObject): void {
    this.clickables.push(clickable);
  }

  removeClickable(object: THREE.Object3D): void {
    this.clickables = this.clickables.filter(c => c.object !== object);
  }

  dispose(): void {
    const domElement = this.raycastManager['domElement'];
    domElement.removeEventListener('mousedown', this.onMouseDown);
    domElement.removeEventListener('mousemove', this.onMouseMove);
    domElement.removeEventListener('mouseup', this.onMouseUp);
    domElement.removeEventListener('touchstart', this.onTouchStart);
    domElement.removeEventListener('touchmove', this.onTouchMove);
    domElement.removeEventListener('touchend', this.onTouchEnd);
  }
}
```

### 3. Drag and Drop

```typescript
export class DragDropManager {
  private raycastManager: RaycastManager;
  private dragPlane = new THREE.Plane();
  private offset = new THREE.Vector3();
  private intersection = new THREE.Vector3();
  private draggingObject: THREE.Object3D | null = null;
  private draggables = new Set<THREE.Object3D>();

  constructor(camera: THREE.Camera, domElement: HTMLElement) {
    this.raycastManager = new RaycastManager(camera, domElement);
    this.setupEventListeners(domElement);
  }

  private setupEventListeners(domElement: HTMLElement): void {
    domElement.addEventListener('mousedown', this.onMouseDown);
    domElement.addEventListener('mousemove', this.onMouseMove);
    domElement.addEventListener('mouseup', this.onMouseUp);
    domElement.addEventListener('touchstart', this.onTouchStart);
    domElement.addEventListener('touchmove', this.onTouchMove);
    domElement.addEventListener('touchend', this.onTouchEnd);
  }

  private onMouseDown = (event: MouseEvent): void => {
    this.raycastManager.updatePointerFromMouse(event);
    this.startDrag();
  };

  private onMouseMove = (event: MouseEvent): void => {
    this.raycastManager.updatePointerFromMouse(event);
    this.updateDrag();
  };

  private onMouseUp = (): void => {
    this.endDrag();
  };

  private onTouchStart = (event: TouchEvent): void => {
    event.preventDefault();
    this.raycastManager.updatePointerFromTouch(event);
    this.startDrag();
  };

  private onTouchMove = (event: TouchEvent): void => {
    event.preventDefault();
    this.raycastManager.updatePointerFromTouch(event);
    this.updateDrag();
  };

  private onTouchEnd = (): void => {
    this.endDrag();
  };

  private startDrag(): void {
    const objects = Array.from(this.draggables);
    const intersection = this.raycastManager.castFirst(objects, false);

    if (intersection) {
      this.draggingObject = intersection.object;

      // Set drag plane perpendicular to camera
      const camera = this.raycastManager['camera'];
      const normal = new THREE.Vector3();
      camera.getWorldDirection(normal);
      this.dragPlane.setFromNormalAndCoplanarPoint(
        normal,
        intersection.point
      );

      // Calculate offset
      this.offset.copy(intersection.point).sub(this.draggingObject.position);
    }
  }

  private updateDrag(): void {
    if (!this.draggingObject) return;

    const raycaster = this.raycastManager.getRaycaster();

    if (raycaster.ray.intersectPlane(this.dragPlane, this.intersection)) {
      this.draggingObject.position.copy(this.intersection).sub(this.offset);
    }
  }

  private endDrag(): void {
    this.draggingObject = null;
  }

  addDraggable(object: THREE.Object3D): void {
    this.draggables.add(object);
  }

  removeDraggable(object: THREE.Object3D): void {
    this.draggables.delete(object);
  }

  dispose(): void {
    const domElement = this.raycastManager['domElement'];
    domElement.removeEventListener('mousedown', this.onMouseDown);
    domElement.removeEventListener('mousemove', this.onMouseMove);
    domElement.removeEventListener('mouseup', this.onMouseUp);
    domElement.removeEventListener('touchstart', this.onTouchStart);
    domElement.removeEventListener('touchmove', this.onTouchMove);
    domElement.removeEventListener('touchend', this.onTouchEnd);
  }
}
```

## Usage Examples

```typescript
// Basic raycasting
const raycastManager = new RaycastManager(camera, renderer.domElement);

canvas.addEventListener('click', (event) => {
  raycastManager.updatePointerFromMouse(event);
  const intersections = raycastManager.cast([mesh1, mesh2, mesh3]);

  if (intersections.length > 0) {
    console.log('Clicked:', intersections[0].object.name);
  }
});

// Click handling
const clickHandler = new ClickHandler(camera, renderer.domElement);

clickHandler.addClickable({
  object: playerMesh,
  onClick: (intersection) => {
    console.log('Player clicked at:', intersection.point);
  },
  onHover: () => {
    playerMesh.material.emissive.set(0x444444);
  },
  onHoverEnd: () => {
    playerMesh.material.emissive.set(0x000000);
  },
});

// Drag and drop
const dragDropManager = new DragDropManager(camera, renderer.domElement);
dragDropManager.addDraggable(box);

// Cleanup
clickHandler.dispose();
dragDropManager.dispose();
```

## Checklist

- [ ] Reuse raycaster instance
- [ ] Normalize pointer coordinates to NDC (-1 to 1)
- [ ] Use layers to filter raycast targets
- [ ] Limit raycast to visible objects only
- [ ] Use non-recursive raycast when possible
- [ ] Implement drag threshold for click vs drag
- [ ] Handle both mouse and touch events
- [ ] Dispose event listeners on cleanup
- [ ] Use object pools for large scenes
- [ ] Set near/far limits on raycaster

## Common Pitfalls

1. **Not normalizing coordinates**: Raycasts miss targets
2. **Raycasting every frame**: Performance issues
3. **Not checking recursive flag**: Misses nested objects
4. **Forgetting touch events**: No mobile support
5. **Not removing event listeners**: Memory leaks
6. **Raycasting invisible objects**: Wasted performance

## Performance Tips

- Only raycast on click/touch, not every frame
- Use layers to limit raycast targets (10-100x faster)
- Set `recursive: false` for flat scene graphs
- Use spatial partitioning for large scenes
- Implement object culling before raycasting
- Cache raycast results when objects are static
- Use coarse then fine raycasting for precision

## Related Skills

- `threejs-scene-setup` - Scene structure for raycasting
- `touch-input-handling` - Touch gesture support
- `input-system` - Unified input abstraction
- `ui-system` - In-game UI interaction
