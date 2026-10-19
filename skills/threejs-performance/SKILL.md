---
name: threejs-performance
description: "Measuring and optimizing three.js speed: render statistics, memory counters, fewer draw calls via merging, instancing, batching and LOD, fill rate, shadows, CPU work per frame, shader stalls and GPU compute for particles, WebGL or WebGPU with TSL. Use when a scene is slow, janks or exceeds memory."
---

# Three.js Performance

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.add(new THREE.HemisphereLight(0xffffff, 0x445566, 1));
const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(3, 6, 4);
scene.add(sun);

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 8);

const mesh = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 16), new THREE.MeshStandardMaterial({ color: 0x4488ff }));
scene.add(mesh);

function animate() {
  mesh.rotation.y += 0.01;
  renderer.render(scene, camera);
  // the counters reset at the start of every render() call, so these cover the frame just drawn
  console.log(`${renderer.info.render.calls} calls, ${renderer.info.render.triangles} triangles, ${renderer.info.memory.textures} textures`);
}

renderer.setAnimationLoop(animate);
```

## Measure First

Optimize what the numbers name, not what a hunch suspects. WebGLRenderer's `renderer.info` object is read after `render()`:

```javascript
function animate() {
  renderer.render(scene, camera);
  console.table({
    calls: renderer.info.render.calls, // draw calls submitted by this render
    triangles: renderer.info.render.triangles,
    lines: renderer.info.render.lines,
    points: renderer.info.render.points,
    geometries: renderer.info.memory.geometries, // geometry buffers live on the GPU
    textures: renderer.info.memory.textures,
    programs: renderer.info.programs.length, // compiled shader programs in the cache
  });
}

renderer.setAnimationLoop(animate);
```

Details worth knowing when reading these:

- The counters reset at the start of every `render()` call while `info.autoReset` is true (the default). Reading them before `render()` reports the previous frame, and a frame with several `render()` calls (picking, reflections, post passes) shows only the last one's numbers.
- Shadow-map drawing runs through the same counters: each shadow-casting light re-draws its casters from the light's view inside `render()`, and those draws appear in `calls` and `triangles`. Toggling a light's `castShadow` is directly visible here.
- `info.memory.geometries` and `info.memory.textures` count uploads, not bytes; they drop when the matching `dispose()` runs. Geometries and textures upload once and are shared by every mesh that references them.

When a frame really has several `render()` calls, accumulate over the whole frame yourself:

```javascript
renderer.info.autoReset = false; // render() stops clearing the counters

function animate() {
  renderer.render(scene, camera);
  console.log(renderer.info.render.calls); // every render of this frame, summed
  renderer.info.reset(); // once per frame, after the last render
}

renderer.setAnimationLoop(animate);
```

`Timer` gives the frame interval; average it over a second of frames, single frames jitter too much to compare. The interval never drops below one display refresh (16.7 ms at 60 Hz), so it shows dropped frames but not headroom; time the CPU side of the frame with `performance.now()` around the work:

```javascript
const timer = new THREE.Timer();
timer.connect(document); // pauses while the tab is hidden

let interval = 0;
let cpu = 0;
let frames = 0;

function animate() {
  timer.update();
  interval += timer.getDelta(); // seconds since the previous frame, never less than one display refresh
  const start = performance.now();
  renderer.render(scene, camera);
  cpu += performance.now() - start; // CPU time spent in render(); the GPU executes the commands later
  frames++;
  if (frames === 60) {
    console.log(`${((interval / frames) * 1000).toFixed(2)} ms per frame, ${(cpu / frames).toFixed(2)} ms CPU in render()`);
    interval = 0;
    cpu = 0;
    frames = 0;
  }
}

renderer.setAnimationLoop(animate);
```

For anything deeper than frame time, use the browser profiler — the Chrome DevTools Performance panel shows long main-thread tasks, scripting versus rendering time, and GPU work on its GPU track. The `stats-gl` package shows a live per-frame statistics overlay when you want the numbers always visible.

## Draw Calls

Each visible mesh queues one draw call, or one per geometry group when the material is an array, and two settings multiply that: a `transparent` material with `side: THREE.DoubleSide` draws each mesh twice, back faces then front faces (`forceSinglePass: true` makes it one), and while any MeshPhysicalMaterial with `transmission` above 0 is in view, every opaque object is drawn a second time into the transmission buffer. Without those two, a thousand single-material meshes is a thousand draw calls; `info.render.calls` is the number to watch. The classic reducers:

**Merge static geometry.** Many meshes that never move and share one material become one mesh and one call:

```javascript
import * as BufferGeometryUtils from "three/addons/utils/BufferGeometryUtils.js";

const rockGeometry = new THREE.DodecahedronGeometry(1, 0);
const transform = new THREE.Matrix4();
const parts = [];
for (let i = 0; i < 200; i++) {
  const baked = rockGeometry.clone();
  transform.setPosition((Math.random() - 0.5) * 40, 0, (Math.random() - 0.5) * 40);
  baked.applyMatrix4(transform); // bake the placement into the vertices
  parts.push(baked);
}
const merged = BufferGeometryUtils.mergeGeometries(parts); // inputs need matching attributes and index state
scene.add(new THREE.Mesh(merged, new THREE.MeshNormalMaterial())); // 200 rocks, one call — but one bounding sphere for all of them
```

Merging trades culling granularity for the call count: the merged mesh has a single bounding sphere, so frustum culling is all-or-nothing. Keep world-sized clusters merged per cluster, not one merge for a whole level.

**Instance repeats.** One geometry drawn many times with per-instance transforms stays one call:

```javascript
const tree = new THREE.ConeGeometry(0.5, 2, 6);
const trees = new THREE.InstancedMesh(tree, new THREE.MeshNormalMaterial(), 500);
const placement = new THREE.Object3D();
for (let i = 0; i < 500; i++) {
  placement.position.set((Math.random() - 0.5) * 40, 1, (Math.random() - 0.5) * 40);
  placement.rotation.y = Math.random() * 6.28;
  placement.updateMatrix();
  trees.setMatrixAt(i, placement.matrix);
}
// culling tests one sphere around all instances (computed lazily on the first cull test), so all 500 draw whenever it touches the view;
// after moving instances later, set trees.instanceMatrix.needsUpdate = true and call trees.computeBoundingSphere()
scene.add(trees);
```

**Batch different geometries.** Many geometries sharing one material pack into shared buffers:

```javascript
const material = new THREE.MeshNormalMaterial();
const batchedMesh = new THREE.BatchedMesh(500, 20000, 20000, material);
const coneId = batchedMesh.addGeometry(new THREE.ConeGeometry(0.5, 2, 6));
const boxId = batchedMesh.addGeometry(new THREE.BoxGeometry(1, 1, 1));
const placement = new THREE.Matrix4();
for (let i = 0; i < 500; i++) {
  const instanceId = batchedMesh.addInstance(i % 2 ? coneId : boxId); // a geometry renders only through an instance
  placement.makeTranslation((Math.random() - 0.5) * 40, 1, (Math.random() - 0.5) * 40);
  batchedMesh.setMatrixAt(instanceId, placement);
}
scene.add(batchedMesh);
```

Unlike InstancedMesh, it culls and sorts each instance on its own (`perObjectFrustumCulled` and `sortObjects`, both on by default). WebGLRenderer multi-draws these in one call where the browser supports `WEBGL_multi_draw`, one draw per instance otherwise; the WebGPU backend issues one draw per instance. `threejs-geometry` covers both APIs in depth.

**Drop detail with distance.** `LOD` keeps every level as a child and, inside every `render()` (its `autoUpdate` is on by default), shows only the level for the current camera distance, so far objects submit fewer triangles — the call count stays the same:

```javascript
const detailMaterial = new THREE.MeshNormalMaterial();
const lod = new THREE.LOD();
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 64, 32), detailMaterial), 0); // distances are camera-to-object
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 16, 8), detailMaterial), 12);
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 8, 4), detailMaterial), 30);
scene.add(lod);
```

**Let culling skip whole calls.** Frustum culling is on by default: before a mesh is queued, its geometry's bounding sphere is tested against the camera frustum, and a failed test removes the draw entirely. Two ways to lose it silently: editing vertices on the CPU without `geometry.computeBoundingSphere()` leaves a stale sphere — one left smaller than the moved vertices hides the mesh while it is still in view — and moving vertices in a shader leaves the CPU sphere untouched, so set `mesh.frustumCulled = false` on that one mesh, not scene-wide.

## GPU Memory

`info.memory.textures` counts textures; the byte math is yours. An RGBA8 texture costs 4 bytes per pixel, and with the default mip chain WebGPURenderer's byte estimate multiplies by 1.333 — a 2048 by 2048 color map is about 22 MB of VRAM. Three levers:

- **Compressed KTX2.** A Basis Universal KTX2 texture (ETC1S or UASTC) is transcoded to a block-compressed format the GPU supports (ASTC, BC7, BC1 or BC3, ETC2, ETC1 or PVRTC) and stays compressed in VRAM, instead of expanding to 4 bytes per pixel like a PNG or JPEG does; when none of them fits the file (ASTC takes UASTC files only, ETC1 has no alpha, PVRTC needs power-of-two sizes), KTX2Loader falls back to uncompressed RGBA. Load it with KTX2Loader:

```javascript
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";

const ktx2Loader = new KTX2Loader().detectSupport(renderer);
const map = await ktx2Loader.loadAsync("color.ktx2"); // a CompressedTexture for a Basis Universal file
map.colorSpace = THREE.SRGBColorSpace; // KTX2Loader already sets it from the file's transfer function; this line only matters for a color file tagged linear
mesh.material.map = map;
mesh.material.needsUpdate = true; // the material rendered without a map before
```

- **Size.** A 1024 texture is a quarter of a 2048 one. Author color maps at the size they appear on screen, and let mipmapping handle distance.
- **Disposal.** `geometry.dispose()` frees its GPU buffers at once; `texture.dispose()` frees the GPU texture once no other undisposed texture shares it (a `texture.clone()` shares the image, and its GPU copy while wrap, filter, format and color space stay the same). Nothing counts meshes, so dispose only what nothing still renders — a mesh that still uses a disposed geometry or texture uploads it again on its next render:

```javascript
scene.remove(mesh);
mesh.geometry.dispose(); // frees the GPU buffers now, even if another mesh still uses this geometry

// material.dispose() releases its shader programs (deleted once no material uses them), never its textures
for (const value of Object.values(mesh.material)) {
  if (value?.isTexture) value.dispose();
}
mesh.material.dispose();
mesh.dispose(); // renderer-side state kept for this one object

console.log(renderer.info.memory.textures); // the count drops as each texture frees
```

To free a whole subtree, traverse it — `object.dispose()` does not visit children. Under WebGPURenderer `memory.geometries` and `memory.textures` stay counts, and the `*Size` fields and `memory.total` add byte estimates; see the WebGPU section.

## Fill Rate

Fill rate is fragment work: pixels shaded per frame. The drawing buffer is `width × ratio` by `height × ratio` pixels, so shading cost scales with the pixel ratio squared — a 3× display shades 9× the pixels of 1×. Cap it:

```javascript
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

**Overdraw and transparency.** Opaque geometry draws first (sorted by group, render order, material, then distance), transmissive next, transparent last — transparent back-to-front so blending composes correctly. Stacked transparent layers cannot reject each other: every layer behind another transparent surface still shades its pixels, and a full-screen transparent overlay shades every pixel of the screen once more on top of the scene. Prefer opaque materials where the look allows, and keep transparent quads few and small. `renderer.info` will not show overdraw — calls and triangles stay identical — only frame time moves.

**Shadow maps.** Each shadow-casting light renders its casters into a depth texture every frame while shadow updates are on. The defaults are small (512 by 512); raise them knowingly, and freeze them for static scenery:

```javascript
const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(3, 6, 4);
sun.castShadow = true;
sun.shadow.mapSize.set(1024, 1024);
scene.add(sun);

renderer.shadowMap.enabled = true;

// static scenery: draw the shadow maps once instead of every frame
renderer.shadowMap.autoUpdate = false;
renderer.shadowMap.needsUpdate = true; // render them one more time, then freeze

// per light instead of globally: sun.shadow.autoUpdate = false; sun.shadow.needsUpdate = true;
```

Each shadow-casting light adds its own pass, and those draws appear in `info.render.calls`: it re-draws every `castShadow` object inside its shadow camera's frustum, a PointLight once per cube face the object falls in (up to six), and with `VSMShadowMap` the `receiveShadow` objects too, plus two blur draws. Tight shadow camera bounds (`sun.shadow.camera.left`, `right`, `top`, `bottom`, then `sun.shadow.camera.updateProjectionMatrix()`) cut the casters drawn; the map size sets each pass's fill cost.

## CPU Work in the Loop

**Static transforms.** Every `render()` calls `scene.updateMatrixWorld()`, which visits every object and recomposes its local matrix from position, rotation and scale when `matrixAutoUpdate` is true (the default). For scenery that never moves, compose once:

```javascript
mesh.matrixAutoUpdate = false;
mesh.updateMatrix(); // bake position/rotation/scale into mesh.matrix before the first render

// later, when this object does move:
mesh.position.set(1, 2, 3);
mesh.updateMatrix(); // manual now: the renderer no longer recomposes it
```

**Raycasting.** `intersectObjects(objects, true)` walks every descendant of every object and tests each one: a layers test first, then the geometry's bounding sphere, then triangles. Narrow the input instead of filtering the output:

```javascript
const raycaster = new THREE.Raycaster();
raycaster.layers.set(1); // only objects enabled on layer 1 are tested, before any geometry work

const pickables = [];
scene.traverse((child) => {
  if (child.isMesh && child.userData.pickable) {
    child.layers.enable(1); // adds layer 1 while keeping layer 0 membership
    pickables.push(child);
  }
});

const hits = raycaster.intersectObjects(pickables, false); // an explicit list, no recursion
```

A pointermove handler that raycasts runs on every event, so this runs hot on scenes with picking. For large static geometry, the `three-mesh-bvh` package accelerates the triangle tests.

**No allocations in the loop.** Objects created per frame become garbage, and collection pauses show up as periodic frame spikes. Reuse vectors and matrices declared outside the loop:

```javascript
const timer = new THREE.Timer();
timer.connect(document);

const step = new THREE.Vector3(); // allocated once, reused every frame
const target = new THREE.Vector3(0, 1, 0);

function animate() {
  timer.update();
  step.subVectors(target, mesh.position).multiplyScalar(Math.min(timer.getDelta() * 2, 1));
  mesh.position.add(step);
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Shader Compile Stalls

Programs build lazily: the first frame a material renders under conditions it has not seen before — a different light count, fog switched on for it — stalls while its shader compiles. Build them during loading instead:

```javascript
// during loading, before the first frame: compile everything the scene will render with
await renderer.compileAsync(scene, camera); // uses KHR_parallel_shader_compile where available

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera); // material programs are ready; shadow-depth programs still build on the first shadow render
});
```

`renderer.compile(scene, camera)` issues the same builds but returns at once, so the first render still waits for any program that has not finished; prefer the async form. Changing a setting that is compiled into the shader — `transparent`, `side`, `blending`, `flatShading`, `vertexColors`, `dithering`, `premultipliedAlpha`, `alphaHash`, or adding or removing a map — needs `material.needsUpdate = true` and builds a new program, so avoid flipping these at runtime (`threejs-materials` lists them all).

## WebGPU and TSL

```javascript
import * as THREE from "three/webgpu";
const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 4, 18);
function animate() {
  renderer.render(scene, camera);
  // autoReset clears per-frame counters at each loop tick; render.calls counts render passes since the app started
  console.log(`${renderer.info.render.drawCalls} draw calls, ${renderer.info.memory.total} bytes of GPU memory`);
}
renderer.setAnimationLoop(animate);
```

Compute moves particle work off the CPU: no per-particle loop, no buffer re-upload per frame, one dispatch updates every particle:

```javascript
import * as THREE from "three/webgpu";
import { Fn, cos, hash, instanceIndex, instancedArray, sin, time, TWO_PI, vec3 } from "three/tsl";
const count = 100000;
const positionBuffer = instancedArray(count, "vec3"); // a storage buffer living on the GPU
const simulate = Fn(() => {
  const rand = hash(instanceIndex);
  const angle = rand.mul(TWO_PI).add(time); // time updates itself every frame
  positionBuffer.element(instanceIndex).assign(vec3(cos(angle).mul(8), sin(rand.mul(97).add(time.mul(2))).mul(2).add(2), sin(angle).mul(8)));
});
const simulateCompute = simulate().compute(count); // one compute thread per particle
const material = new THREE.SpriteNodeMaterial();
material.positionNode = positionBuffer.toAttribute(); // the material reads the storage buffer directly
material.colorNode = vec3(1, 0.7, 0.3);
const particles = new THREE.Sprite(material);
particles.count = count; // the sprite draws once per buffer element
particles.frustumCulled = false; // culling tests one small sphere at the sprite origin, not the particles
scene.add(particles);
function animate() {
  renderer.compute(simulateCompute); // GPU updates the buffer, then the draw reads it
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

- The WebGPU counters have their own names: `info.render.drawCalls` per frame, `info.render.calls` (render passes: each `render()` and internal ones such as the output pass) and `info.calls` (render and compute calls) cumulative since the app started, `info.compute.frameCalls` per frame, and `info.memory.total` — a byte estimate over vertex, index and storage buffers, textures, uniform buffers and programs (`memory.texturesSize`, `memory.attributesSize` and the other `*Size` fields hold the parts). A compressed texture counts as 1 byte there, a fallback estimate, so KTX2 memory is under-reported.
- The second block's loop replaces the first block's `animate`; `renderer.compute()` submits directly because `await renderer.init()` has already run.

## Common Mistakes

- `function animate() { requestAnimationFrame(animate); renderer.render(scene, camera); }` → `renderer.setAnimationLoop(animate);`: a WebXR session runs only the callback given to setAnimationLoop, through the session's own frame loop, and under WebGPURenderer setAnimationLoop also awaits init() before the first frame.
- `const clock = new THREE.Clock(); const dt = clock.getDelta(); const t = clock.getElapsedTime();` → `const timer = new THREE.Timer(); timer.update(); const dt = timer.getDelta(); const t = timer.getElapsed();`: Clock is deprecated in r183, and getElapsedTime() calls getDelta() internally, so a later getDelta() in the same frame returns about zero.
- `console.log(renderer.info.render.calls); renderer.render(scene, camera);` → `renderer.render(scene, camera); console.log(renderer.info.render.calls);`: the counters reset at the start of render(), so reading before it reports the previous frame.
- `renderer.info.autoReset = false` → `renderer.info.autoReset = false; function animate() { renderer.render(scene, camera); console.log(renderer.info.render.calls); renderer.info.reset(); }`: with autoReset off, nothing clears the counters unless reset() runs once per frame after the last render(), so they accumulate across frames.
- `webgpuRenderer.info.render.calls` → `webgpuRenderer.info.render.drawCalls`: under WebGPURenderer, render.calls counts render passes since the app started, while drawCalls counts this frame's draw calls; WebGLRenderer has no drawCalls, and its render.calls is the per-render() draw count.
- `renderer.compile(scene, camera)` → `await renderer.compileAsync(scene, camera)`: compile() queues the builds and returns, so a render right after it still blocks on unfinished programs, while compileAsync() resolves only once KHR_parallel_shader_compile reports every program complete, where the browser exposes that extension.
- `renderer.shadowMap.type = THREE.PCFSoftShadowMap` → `renderer.shadowMap.type = THREE.PCFShadowMap`: PCFSoftShadowMap is deprecated in r182 and falls back to PCF with a console warning.
- `mesh.material.dispose()` → `mesh.material.dispose(); for (const value of Object.values(mesh.material)) if (value?.isTexture) value.dispose();`: material.dispose() releases its shader programs, never the material's textures, so info.memory.textures only drops when each texture.dispose() runs.
- `gltf.scene.dispose()` → `gltf.scene.traverse((child) => { child.geometry?.dispose(); for (const material of [child.material ?? []].flat()) { for (const value of Object.values(material)) if (value?.isTexture) value.dispose(); material.dispose(); } child.dispose(); })`: Object3D.dispose() frees renderer state for that one object only, never its children, geometry, materials or textures.
- `const hits = raycaster.intersectObjects(scene.children, true)` → `raycaster.layers.set(1); const hits = raycaster.intersectObjects(pickables, false);`: the recursive call walks and tests every descendant on every event, while the layers check runs before each object's geometry test and an explicit list never visits the rest of the scene.
- `scene.traverse((child) => (child.frustumCulled = false))` → `geometry.computeBoundingSphere(); // after moving vertices on the CPU`: culling skips whole draw calls through each geometry's bounding sphere, so disabling it scene-wide draws everything every frame.
- `function animate() { mesh.position.add(new THREE.Vector3(0, 0.01, 0)); }` → `const offset = new THREE.Vector3(0, 0.01, 0); function animate() { mesh.position.add(offset); }`: every per-frame allocation becomes garbage that the collector must pause to reclaim, which shows up as periodic frame spikes.
- `const dt = uniform(0); /* fed from the timer each frame */` → `const dt = deltaTime;`: the built-in time and deltaTime nodes are uniforms the renderer already updates on every render, so a hand-fed copy duplicates them and needs its own Timer.

## See Also

- `threejs-fundamentals` - renderer construction, the render loop and resize handling around everything measured here.
- `threejs-geometry` - the full instancing, batching, merging and attribute APIs behind the draw-call cuts.
- `threejs-materials` - which material settings compile into the shader and force a rebuild.
- `threejs-loaders` - the KTX2, Draco and meshopt loader wiring for the compressed files these levers produce.
- `threejs-lighting` - light types, units and shadow setup whose costs this skill measures.
- `threejs-postprocessing` - the composer's own buffers, MSAA and pixel-ratio settings.
- `threejs-asset-pipeline` - the gltf-transform commands that produce KTX2 textures and compressed, instanced or joined glTF files.
