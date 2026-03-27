---
name: threejs-r183-migration
description: Migration guide for Three.js r183 covering breaking changes, deprecations, and new features
---

# Three.js r183 Migration Guide

## When to Use

Use this skill when:
- Upgrading a project from Three.js r182 or earlier to r183
- Starting a new project on r183 and wanting to use latest APIs
- Debugging issues after a Three.js version bump
- Reviewing code for deprecated API usage

## Core Principles

1. **Check Deprecations First**: Address breaking changes before new features
2. **Test Incrementally**: Upgrade and test one area at a time
3. **Renderer-Aware**: Some changes only affect WebGPU, not WebGL
4. **Performance Wins**: Several r183 changes are free performance improvements
5. **Backward Compatibility**: Most changes have migration paths, not hard breaks

## Breaking and Deprecated Changes

### 1. Clock Deprecated — Use Timer

`THREE.Clock` is deprecated in r183. Replace with `Timer` from addons:

```typescript
// ❌ BEFORE (deprecated)
import * as THREE from 'three';

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  const elapsed = clock.getElapsedTime();

  mixer.update(delta);
  renderer.render(scene, camera);
}

// ✅ AFTER (r183+)
import { Timer } from 'three/addons/misc/Timer.js';

const timer = new Timer();

function animate() {
  requestAnimationFrame(animate);
  timer.update(); // Must call explicitly each frame

  const delta = timer.getDelta();
  const elapsed = timer.getElapsed();

  mixer.update(delta);
  renderer.render(scene, camera);
}
```

Key differences:
- `Timer` requires explicit `timer.update()` call before reading values
- `Timer` is not affected by page visibility changes (no large delta spike when tab regains focus)
- Method is `getElapsed()` not `getElapsedTime()`

### 2. shadowMap.color Renamed to shadowMap.transmitted (WebGPU)

Only affects WebGPU renderer:

```typescript
// ❌ BEFORE
renderer.shadowMap.color;

// ✅ AFTER (r183+ WebGPU)
renderer.shadowMap.transmitted;
```

### 3. Line2NodeMaterial.useColor Renamed (WebGPU)

```typescript
// ❌ BEFORE
lineMaterial.useColor = true;

// ✅ AFTER (r183+)
lineMaterial.vertexColors = true;
```

### 4. Scriptable Node Removed from TSL

The `Scriptable` node has been removed from TSL (Three.js Shading Language). Use standard TSL node functions instead:

```typescript
// ❌ BEFORE — Scriptable node
// ScriptableNode is no longer available

// ✅ AFTER — Use Fn / tslFn for custom node logic
import { Fn, float, vec4 } from 'three/tsl';

const customEffect = Fn(([input]) => {
  return vec4(input.rgb, float(1.0));
});
```

### 5. Camera Scale Excluded from View Matrix

Camera scale is now excluded from the view matrix calculation. If you relied on scaling the camera:

```typescript
// ❌ BEFORE — scaling camera for zoom effect
camera.scale.set(2, 2, 2); // No longer affects rendering

// ✅ AFTER — use proper zoom mechanisms
camera.zoom = 2;
camera.updateProjectionMatrix();
// Or use OrbitControls.dolly() for orbit cameras
```

## New Features

### 1. BatchedMesh — Per-Instance Opacity and Wireframe

`BatchedMesh` now supports per-instance opacity and wireframe rendering:

```typescript
const batchedMesh = new THREE.BatchedMesh(
  maxGeometryCount,
  maxVertexCount,
  maxIndexCount
);

const geoId = batchedMesh.addGeometry(geometry);
const instanceId = batchedMesh.addInstance(geoId);

// Per-instance color (already existed)
batchedMesh.setColorAt(instanceId, new THREE.Color(1, 0, 0));

// r183+: per-instance opacity and wireframe now supported
batchedMesh.setVisibleAt(instanceId, true);
```

### 2. OrbitControls — Programmatic Methods

New methods exposed for programmatic camera control:

```typescript
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);

// r183+ programmatic methods
controls.rotate(azimuthAngle, polarAngle);
controls.pan(deltaX, deltaY);
controls.dolly(zoomScale);

// r183+ cursor customization
controls.cursorStyle = 'grab';
```

### 3. Lambert/Phong — scene.environment IBL

`MeshLambertMaterial` and `MeshPhongMaterial` now respond to `scene.environment`:

```typescript
scene.environment = hdrEnvironmentMap;

// These now pick up environment IBL (r183+)
const lambert = new THREE.MeshLambertMaterial({ color: 0x00ff00 });
const phong = new THREE.MeshPhongMaterial({ color: 0xff0000 });
```

This is a significant upgrade for mobile-optimized materials that previously could not benefit from environment lighting.

### 4. MeshPhysicalMaterial — Clearcoat for Area Lights

Clearcoat now correctly interacts with rectangular area lights:

```typescript
const material = new THREE.MeshPhysicalMaterial({
  clearcoat: 1.0,
  clearcoatRoughness: 0.1,
});

const areaLight = new THREE.RectAreaLight(0xffffff, 5, 4, 2);
// Clearcoat reflection of area light now renders correctly
```

### 5. GLTFLoader — MeshOpt Compression

Native support for `KHR_meshopt_compression`:

```typescript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { MeshoptDecoder } from 'three/examples/jsm/libs/meshopt_decoder.module.js';

const loader = new GLTFLoader();
loader.setMeshoptDecoder(MeshoptDecoder);

const gltf = await loader.loadAsync('/model.glb');
```

### 6. BezierInterpolant

New interpolant for smooth Bezier curve interpolation in animation keyframe tracks.

### 7. Matrix4 Optimizations

`Matrix4.decompose()` now caches the determinant, and `Matrix4.invert()` is optimized. These are internal improvements — no code changes needed, but heavy matrix operations will be faster.

### 8. KTX2Loader Improvements

- BC3 alpha transparency fixes
- ISO 21496-1 gainmap metadata support (HDR from SDR+gainmap)

### 9. VRMLLoader — Camera Support

VRML files with camera definitions now have cameras properly imported.

## Migration Checklist

- [ ] Search codebase for `new THREE.Clock` and replace with `Timer`
- [ ] Search for `clock.getDelta()` / `clock.getElapsedTime()` and update
- [ ] If using WebGPU: search for `shadowMap.color` and rename to `shadowMap.transmitted`
- [ ] If using WebGPU: search for `useColor` on Line2NodeMaterial, rename to `vertexColors`
- [ ] If using TSL: remove any `Scriptable` node usage, refactor to `Fn`/`tslFn`
- [ ] Check for camera scale usage in rendering (now excluded from view matrix)
- [ ] Consider adding MeshOpt decoder for GLTFLoader
- [ ] Consider upgrading mobile materials to use `scene.environment`
- [ ] Test shadow rendering (WebGPU) after upgrade
- [ ] Run full test suite and check for visual regressions

## Performance Improvements (Free)

These r183 changes require no code modifications:
- `Matrix4.decompose()` determinant caching
- `Matrix4.invert()` optimization
- KTX2Loader BC3 alpha fixes
- Lambert/Phong IBL (if `scene.environment` is already set)

## Common Pitfalls

1. **Forgetting `timer.update()`**: Timer returns 0 delta without explicit update call
2. **Using `getElapsedTime()` instead of `getElapsed()`**: Timer method name is different from Clock
3. **Camera scale regression**: Zoom effects via camera scale silently break
4. **WebGPU-only changes**: Don't apply WebGPU renames to WebGL code paths
5. **MeshOpt decoder not registered**: GLTFLoader will fail on meshopt-compressed files without `setMeshoptDecoder()`

## Related Skills

- `threejs-scene-setup` - Scene initialization with Timer
- `threejs-animation-systems` - Animation timing with Timer
- `threejs-camera-controls` - OrbitControls new methods
- `threejs-material-systems` - Lambert/Phong IBL, clearcoat
- `threejs-model-loading` - MeshOpt compression
- `threejs-instancing-advanced` - BatchedMesh improvements
- `threejs-shadows` - Shadow map property rename
- `threejs-texture-management` - KTX2Loader fixes
- `threejs-math-utilities` - Matrix4 optimizations
- `threejs-post-processing` - TSL Scriptable node removal
- `threejs-best-practices` - Timer recommendation
