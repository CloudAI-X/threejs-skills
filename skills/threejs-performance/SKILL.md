---
name: threejs-performance
description: Three.js performance and memory - draw call budgets, instancing vs batching, geometry merging, frustum culling, GPU resource disposal and leak hunting, object pooling, profiling with renderer.info / stats-gl / Spector.js, three-mesh-bvh raycasting, context loss. Use when a scene is slow, memory grows over time, frame rate drops, or before shipping a 3D scene.
---

# Three.js Performance & Memory

Two rules govern almost everything here:

1. **Draw calls are the budget.** Under 100 per frame, most devices hold 60fps.
   Above 500, even powerful GPUs struggle.
2. **Three.js does not garbage-collect GPU resources.** Every geometry,
   material, texture and render target you create must be disposed explicitly.

## Measure First

Never optimize without a number. `renderer.info` is free and always available:

```javascript
setInterval(() => {
  console.log("calls:", renderer.info.render.calls);
  console.log("triangles:", renderer.info.render.triangles);
  console.log("geometries:", renderer.info.memory.geometries);
  console.log("textures:", renderer.info.memory.textures);
  console.log("programs:", renderer.info.programs.length);
}, 1000);
```

Read it as two separate signals:

- **`render.*` should be low.** That's your per-frame cost.
- **`memory.*` should be flat.** A count that climbs while the scene looks
  unchanged is a leak, full stop.

## Draw Call Optimization

### InstancedMesh — same geometry, many transforms

1,000 trees as 1,000 meshes is 1,000 draw calls. As an `InstancedMesh` it is one.

```javascript
const count = 1000;
const mesh = new THREE.InstancedMesh(geometry, material, count);
const dummy = new THREE.Object3D();

for (let i = 0; i < count; i++) {
  dummy.position.set(x[i], y[i], z[i]);
  dummy.rotation.y = Math.random() * Math.PI;
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true;
scene.add(mesh);
```

### BatchedMesh — different geometries, one material

`BatchedMesh` (r156+) packs multiple *distinct* geometries sharing one material
into a single draw call — the variation `InstancedMesh` can't give you.

```javascript
// (maxInstanceCount, maxVertexCount, maxIndexCount, material)
const batch = new THREE.BatchedMesh(1000, 30_000, 60_000, sharedMaterial);

const boxId = batch.addGeometry(boxGeometry);
const coneId = batch.addGeometry(coneGeometry);

const instanceA = batch.addInstance(boxId);
const instanceB = batch.addInstance(coneId);

batch.setMatrixAt(instanceA, matrixA);
batch.setColorAt(instanceB, new THREE.Color(0xff0000));
batch.setVisibleAt(instanceA, true);

scene.add(batch);
```

Pair it with **array textures** (`DataArrayTexture`, sampled by layer index in
the shader) to get per-instance texture variation without extra binds.

### Share materials

Three.js batches meshes with identical materials. A unique material per object
defeats that *and* forces a separate shader program compile.

```javascript
// Bad: 500 materials, 500 programs, no batching
meshes.forEach((m) => (m.material = new THREE.MeshStandardMaterial({ color: "red" })));

// Good: one material, one program
const shared = new THREE.MeshStandardMaterial({ color: "red" });
meshes.forEach((m) => (m.material = shared));
```

Three.js reuses shader programs across materials whose compiled source and
uniform layout match, so gratuitous variation (different `defines`, differing
map presence) proliferates programs. Watch `renderer.info.programs.length`.

### Merge static geometry

For geometry that never moves independently, merge at load time:

```javascript
import { mergeGeometries } from "three/addons/utils/BufferGeometryUtils.js";

const merged = mergeGeometries([geo1, geo2, geo3]);
const mesh = new THREE.Mesh(merged, sharedMaterial);
```

Trade-off: you lose per-object culling, transforms and raycast granularity. Merge
things that live and die together; instance things that move independently.

### Frustum culling

Objects outside the camera frustum are culled automatically and cost no draw
call. It is free — as long as bounding volumes are accurate.

```javascript
mesh.frustumCulled = true; // default

// Disable for things that must never disappear
skybox.frustumCulled = false;
particleSystem.frustumCulled = false; // shader-displaced verts break the bounds

// Recompute after mutating positions, or culling will be wrong
geometry.computeBoundingSphere();
geometry.computeBoundingBox();

// Manual test
const frustum = new THREE.Frustum();
const m = new THREE.Matrix4().multiplyMatrices(
  camera.projectionMatrix,
  camera.matrixWorldInverse,
);
frustum.setFromProjectionMatrix(m);
if (frustum.intersectsObject(mesh)) { /* visible */ }
```

Common bug: a vertex shader displaces geometry beyond its bounding sphere, so
the object pops out of view at the screen edge. Either enlarge the bounds or set
`frustumCulled = false`.

## Memory Management

### Full disposal

A single 4K texture costs 64MB+ of VRAM. Materials hold textures on many named
slots, so walk them:

```javascript
function disposeMesh(mesh) {
  mesh.geometry?.dispose();

  const materials = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
  for (const material of materials) {
    if (!material) continue;
    for (const value of Object.values(material)) {
      if (value?.isTexture) {
        // GLTF textures are ImageBitmaps — dispose() alone leaks the bitmap
        value.source?.data?.close?.();
        value.dispose();
      }
    }
    material.dispose();
  }

  mesh.removeFromParent();
}

function disposeSubtree(root) {
  root.traverse((child) => {
    if (child.isMesh || child.isPoints || child.isLine) disposeMesh(child);
  });
  root.removeFromParent();
}
```

The `source.data.close()` step matters specifically for textures that arrived as
`ImageBitmap` (which is how `GLTFLoader` decodes them). Without it the bitmap
survives `texture.dispose()` and leaks.

### Render targets

Post-processing render targets hold framebuffer memory until released:

```javascript
renderTarget.dispose();
composer.dispose();
```

Resizing by allocating a new target without disposing the old one is a classic
slow leak — `composer.setSize()` handles this for you; hand-rolled targets don't.

### Object pooling

For entities spawned and destroyed constantly (bullets, impacts, particles),
allocation churn causes GC pauses. Pool them and pre-warm during loading.

```javascript
class ObjectPool {
  constructor(factory, reset, initialSize = 20) {
    this.factory = factory;
    this.reset = reset;
    this.pool = [];
    for (let i = 0; i < initialSize; i++) {
      const obj = factory();
      obj.visible = false;
      this.pool.push(obj);
    }
  }

  acquire() {
    const obj = this.pool.pop() ?? this.factory();
    obj.visible = true;
    return obj;
  }

  release(obj) {
    this.reset(obj);
    obj.visible = false;
    this.pool.push(obj);
  }
}

const bullets = new ObjectPool(
  () => new THREE.Mesh(bulletGeometry, bulletMaterial),
  (b) => b.position.set(0, 0, 0),
  50,
);
```

### Cache textures

Load each texture once; share the instance everywhere.

```javascript
const textureCache = new Map();

function getTexture(url) {
  if (!textureCache.has(url)) textureCache.set(url, textureLoader.load(url));
  return textureCache.get(url);
}
```

Three.js also has a built-in file-level cache: `THREE.Cache.enabled = true`.

## Fast Raycasting with three-mesh-bvh

Default raycasting tests every triangle. A BVH turns that into a tree walk —
80,000+ polygons at 60fps.

```javascript
import {
  computeBoundsTree, disposeBoundsTree, acceleratedRaycast,
} from "three-mesh-bvh";

// Patch once, globally
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;
THREE.Mesh.prototype.raycast = acceleratedRaycast;

mesh.geometry.computeBoundsTree();

// Remember to release it with the geometry
mesh.geometry.disposeBoundsTree();
```

Also set `raycaster.firstHitOnly = true` when you only need the nearest hit —
it lets the BVH prune aggressively.

## Profiling Tools

| Tool                | What it tells you                                    |
| ------------------- | ---------------------------------------------------- |
| `renderer.info`     | Draw calls, triangles, live geometry/texture counts   |
| `stats-gl`          | FPS + **GPU time**, works on WebGL and WebGPU         |
| `lil-gui`           | Live tuning of any numeric parameter                  |
| Spector.js          | Frame capture: every draw call, bind, shader program  |
| DevTools Performance| Long frames, GC pauses, blocking JS on the main thread|
| `r3f-perf`          | Per-component cost in React Three Fiber               |

```javascript
import Stats from "stats-gl";

const stats = new Stats({ trackGPU: true });
document.body.appendChild(stats.dom);
await stats.init(renderer);

renderer.setAnimationLoop(() => {
  stats.begin();
  renderer.render(scene, camera);
  stats.end();
  stats.update();
});
```

```javascript
import GUI from "lil-gui";

const gui = new GUI();
gui.add(light, "intensity", 0, 2);
gui.add(material, "roughness", 0, 1);
gui.addColor(material, "color");
gui.add(bloomPass, "strength", 0, 3);
```

Profile **real sessions**, not synthetic loops — GC pauses and asset decode
stalls only show up under realistic load.

### Attribute the frame

When the frame is long but you don't know to what, time the subsystems:

```javascript
function animate() {
  const t0 = performance.now();
  physics.update();
  const t1 = performance.now();
  controls.update();
  const t2 = performance.now();
  renderer.render(scene, camera);
  const t3 = performance.now();

  console.log(`physics ${(t1 - t0).toFixed(2)}ms  controls ${(t2 - t1).toFixed(2)}ms  render ${(t3 - t2).toFixed(2)}ms`);
}
```

Caveat: `renderer.render()` returns as soon as commands are *submitted*, not when
the GPU finishes. If render time looks suspiciously cheap while frames are long,
you are GPU-bound — use `stats-gl` with `trackGPU`, or a GPU timing query.

### GPU timing queries (WebGPU)

```javascript
const adapter = await navigator.gpu.requestAdapter();
if (adapter.features.has("timestamp-query")) {
  const device = await adapter.requestDevice({
    requiredFeatures: ["timestamp-query"],
  });
  // ...create timestamp query sets
}
```

## The Render Loop

Prefer `setAnimationLoop` over `requestAnimationFrame`: it handles WebXR sessions
automatically and gives clean start/stop.

```javascript
renderer.setAnimationLoop((time) => {
  renderer.render(scene, camera);
});

renderer.setAnimationLoop(null); // stop
```

## Handle Context Loss

The GPU context can be lost on tab backgrounding, driver reset, or memory
pressure. Unhandled, the canvas goes black permanently.

```javascript
renderer.domElement.addEventListener("webglcontextlost", (event) => {
  event.preventDefault(); // required, or the context will never be restored
  renderer.setAnimationLoop(null);
});

renderer.domElement.addEventListener("webglcontextrestored", () => {
  // Textures and geometries are re-uploaded by three.js; restart the loop
  renderer.setAnimationLoop(animate);
});
```

## Triage Checklist

Work top to bottom — each step is cheaper than the one below it.

1. `renderer.info.render.calls` over 100? → instance, batch, merge, share materials.
2. `renderer.info.memory.*` climbing? → you have a disposal leak.
3. Mobile and GPU-bound? → clamp `setPixelRatio(Math.min(devicePixelRatio, 2))`.
4. Shadows expensive? → see `threejs-lighting` (shadow map size, `autoUpdate`).
5. Post-processing expensive? → see `threejs-postprocessing` (pass merging, half-res).
6. Big assets / slow first paint? → see `threejs-asset-pipeline`.
7. Still slow with few draw calls? → you're fill-rate or shader bound; see
   `threejs-shaders`.
8. Hit a hard ceiling on a compute-heavy scene? → see `threejs-webgpu-tsl`.

## See Also

- `threejs-asset-pipeline` - Compression, LOD, loading strategy
- `threejs-lighting` - Shadow and light cost
- `threejs-postprocessing` - Effect pipeline cost
- `threejs-shaders` - Shader-level optimization
- `threejs-react-three-fiber` - R3F-specific re-render and disposal rules
- `threejs-webgpu-tsl` - When and why to migrate to WebGPU
