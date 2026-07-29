---
name: threejs-webgpu-tsl
description: Three.js WebGPU renderer and TSL (Three Shader Language) - node materials, compute shaders, GPU particles, storage buffers, WebGPU post-processing. Use when targeting WebGPURenderer, writing TSL instead of GLSL, moving particles or physics to compute shaders, or deciding whether to migrate from WebGL.
---

# Three.js WebGPU & TSL

APIs below are verified against three r185. TSL exports come from `three/tsl`,
renderer and node material classes from `three/webgpu`.

## Quick Start

```javascript
import { WebGPURenderer, MeshStandardNodeMaterial } from "three/webgpu";
import { color, positionLocal, sin, time } from "three/tsl";

const renderer = new WebGPURenderer({ antialias: true });

// MANDATORY: requests the GPU adapter and device.
// Without it, rendering fails silently.
await renderer.init();

renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

const material = new MeshStandardNodeMaterial();
material.colorNode = color(1, 0, 0).mul(sin(time).mul(0.5).add(0.5));

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

No bundler config or polyfills needed — `three/webgpu` is a standard export.

## Should You Migrate?

WebGPU is **not universally faster**. Profile your specific case. Migrate when
you hit one of these walls:

| Symptom                                     | WebGPU gain      |
| ------------------------------------------- | ---------------- |
| Hundreds of objects, draw-call bound         | 2-10x            |
| Particle system plateauing (~50k on CPU)     | Millions         |
| Physics/simulation eating main thread        | Large (compute)  |
| Long post-processing chain stuttering        | Moderate         |
| Smooth WebGL scene under budget              | **None — stay**  |

### Browser support

- Chrome/Edge: v113+ (2023)
- Firefox: v141+ (Windows), v145+ (macOS ARM)
- Safari: v26+ (September 2025)

All major browsers ship WebGPU. `WebGPURenderer` falls back to a WebGL 2
backend automatically when it is unavailable — one renderer, one code path, no
separate WebGL build.

### forceWebGL

```javascript
const renderer = new WebGPURenderer({ forceWebGL: true });
```

Forces the WebGL 2 backend on a WebGPU-capable machine. Use for testing
fallback behaviour, diffing shader compilation, or reaching a WebGL extension
with no WebGPU equivalent. For a **WebGL-only production build, import
`WebGLRenderer` directly** — it ships far less code.

### Feature detection

Not every WebGPU feature is universally available. Check before relying on one:

```javascript
const adapter = await navigator.gpu?.requestAdapter();
if (!adapter) {
  // No WebGPU at all — WebGPURenderer will use WebGL 2, or bail out here.
  return;
}

const hasFloat32Filtering = adapter.features.has("float32-filterable");
const hasTimestamps = adapter.features.has("timestamp-query");
```

## TSL (Three Shader Language)

TSL is a node graph authored in JavaScript that compiles to **WGSL on WebGPU
and GLSL on WebGL**. Prefer it over raw shader source: raw GLSL/WGSL means
maintaining two shader codebases, and TSL handles uniforms, attributes and
varyings for you.

```javascript
import {
  positionLocal, normalLocal, normalWorld, positionViewDirection,
  uv, time, uniform, float, vec3, color, mix, step, sin, texture,
} from "three/tsl";
```

Note: the view direction node is `positionViewDirection` (there is no
`viewDirection` export).

### Node material slots

Node materials take nodes on named slots instead of requiring a custom shader:

```javascript
import { MeshStandardNodeMaterial } from "three/webgpu";
import { positionLocal, normalLocal, vertexColor, mx_noise_float } from "three/tsl";

const material = new MeshStandardNodeMaterial();

const noise = mx_noise_float(positionLocal.mul(3));
material.positionNode = positionLocal.add(normalLocal.mul(noise));
material.colorNode = vertexColor;
material.roughnessNode = noise.remap(-1, 1, 0.2, 0.9);
material.emissiveNode = color(0x220044);
```

Available node materials: `MeshBasicNodeMaterial`, `MeshLambertNodeMaterial`,
`MeshPhongNodeMaterial`, `MeshStandardNodeMaterial`, `MeshPhysicalNodeMaterial`,
`MeshToonNodeMaterial`, `MeshNormalNodeMaterial`, `MeshMatcapNodeMaterial`,
`PointsNodeMaterial`, `SpriteNodeMaterial`, `LineBasicNodeMaterial`,
`ShadowNodeMaterial`, `VolumeNodeMaterial`.

### Reusable functions with Fn

`Fn` wraps node logic into a callable that compiles once and is reused across
materials:

```javascript
import { Fn, float } from "three/tsl";

const fresnel = Fn(([normal, viewDir, power]) => {
  const dotNV = normal.dot(viewDir).saturate();
  return float(1).sub(dotNV).pow(power);
});

material.emissiveNode = fresnel(normalWorld, positionViewDirection, 3.0).mul(
  color(0x44aaff),
);
```

### Built-in noise (no dependencies)

TSL ships the MaterialX noise library — do not pull in a third-party GLSL noise
snippet:

```javascript
import {
  mx_noise_float, mx_noise_vec3, mx_fractal_noise_float,
  mx_worley_noise_float, mx_cell_noise_float,
} from "three/tsl";

const n = mx_noise_float(positionLocal.mul(scale));
const fbm = mx_fractal_noise_float(positionLocal, 4, 2.0, 0.5); // octaves, lacunarity, gain
const rgb = mx_noise_vec3(uv().mul(10));
```

### Uniforms

```javascript
import { uniform } from "three/tsl";

const uSpeed = uniform(1.0);
material.positionNode = positionLocal.add(sin(time.mul(uSpeed)));

// Update from JS — no needsUpdate, no uniform lookup
uSpeed.value = 2.5;
```

## Compute Shaders

CPU particle updates plateau around 50,000 particles on typical hardware.
Compute shaders scale to millions because the data never leaves the GPU.

### GPU-persistent buffers with instancedArray

`instancedArray` allocates a buffer that lives in GPU memory **across frames**,
eliminating the CPU→GPU upload that bottlenecks traditional particle systems.

```javascript
import { Fn, instancedArray, instanceIndex, deltaTime, hash, vec3 } from "three/tsl";
import { SpriteNodeMaterial } from "three/webgpu";

const count = 1_000_000;
const positions = instancedArray(count, "vec3");
const velocities = instancedArray(count, "vec3");

// One-time seeding pass
const init = Fn(() => {
  positions.element(instanceIndex).assign(vec3(0, 0, 0));
  velocities.element(instanceIndex).assign(
    vec3(hash(instanceIndex), hash(instanceIndex.add(1)), 0).sub(0.5),
  );
})().compute(count);

// Per-frame update pass
const update = Fn(() => {
  const pos = positions.element(instanceIndex);
  const vel = velocities.element(instanceIndex);
  vel.addAssign(vec3(0, -9.8, 0).mul(deltaTime));
  pos.addAssign(vel.mul(deltaTime));
})().compute(count);

await renderer.computeAsync(init);

const material = new SpriteNodeMaterial();
material.positionNode = positions.toAttribute(); // read the buffer as geometry

renderer.setAnimationLoop(async () => {
  await renderer.computeAsync(update);
  await renderer.renderAsync(scene, camera);
});
```

### Physics on the GPU

The same shape covers constraint solving, boids, and collision:

```javascript
const physics = Fn(() => {
  const pos = positions.element(instanceIndex);
  const vel = velocities.element(instanceIndex);

  // forces, collisions, constraints...
  vel.mulAssign(0.99); // damping
  pos.addAssign(vel.mul(deltaTime));
})().compute(particleCount);

renderer.compute(physics);
```

### Storage textures (read-write in compute)

Regular textures are read-only in a shader. `StorageTexture` + `textureStore`
gives you a writable target — the basis for fluid sim, image processing, and
procedural terrain.

```javascript
import { StorageTexture } from "three/webgpu";
import { Fn, textureStore, instanceIndex, uvec2, vec2, vec4, mx_noise_float } from "three/tsl";

const size = 512;
const heightmap = new StorageTexture(size, size);

const generate = Fn(() => {
  const coord = uvec2(instanceIndex.mod(size), instanceIndex.div(size));
  const height = mx_noise_float(vec2(coord).mul(0.01)).mul(amplitude);
  textureStore(heightmap, coord, vec4(height, 0, 0, 1));
})().compute(size * size);

renderer.compute(generate);
```

### Workgroup shared memory

When threads in a workgroup read the same data repeatedly, stage it in shared
memory — 10-100x faster than global memory for those access patterns.

```javascript
import { workgroupArray, workgroupBarrier, invocationLocalIndex } from "three/tsl";

const shared = workgroupArray("float", 256);

const reduce = Fn(() => {
  shared.element(invocationLocalIndex).assign(input.element(instanceIndex));
  workgroupBarrier(); // all threads must reach here before any reads
  // ...now every thread can read the whole tile
})().compute(count, [256]);
```

### Indirect draws (GPU-driven rendering)

Let a compute pass decide what gets drawn — per-frame frustum culling and LOD
selection without a CPU round trip. Required for rendering millions of
instances.

```javascript
import { IndirectStorageBufferAttribute } from "three/webgpu";
import { Fn, If, atomicAdd, instanceIndex } from "three/tsl";

// [vertexCount, instanceCount, firstVertex, firstInstance]
const indirect = new IndirectStorageBufferAttribute(4, "uint");

const cull = Fn(() => {
  // frustum test / LOD pick, then bump the visible instance count
  If(visible, () => {
    atomicAdd(indirectBuffer.element(1), 1);
  });
})().compute(instanceCount);
```

### Binding model

WebGPU batches resources into **bind groups** rather than binding each uniform
individually. The optimal layout groups frequently-updated uniforms (time,
camera) separately from static data (textures, material constants) so a frame
switches as few bind groups as possible. **Three.js does this for you** — you
only need to care when writing raw WGSL.

### Minimize buffer writes per frame

Buffer writes carry real overhead in WebGPU. Batch them:

```javascript
// Bad: one write per particle
particles.forEach((p) => p.buffer.update());

// Good: one write for the whole set
const data = new Float32Array(particles.length * 4);
particles.forEach((p, i) => data.set(p.data, i * 4));
batchBuffer.update(data);
```

For data that changes every frame, keep it in an `instancedArray` and update it
in a compute pass — then it never crosses the bus at all.

## render vs renderAsync

```javascript
// Simple scene, no compute
renderer.setAnimationLoop(() => renderer.render(scene, camera));

// Compute-heavy: await so compute passes finish before dependent render passes
renderer.setAnimationLoop(async () => {
  await renderer.computeAsync(simulation);
  await renderer.renderAsync(scene, camera);
});
```

## WebGPU Post-Processing

For WebGPU use the native node pipeline, not `EffectComposer`. Effect nodes live
in `three/addons/tsl/display/*`, **not** in `three/tsl`:

```javascript
import { PostProcessing } from "three/webgpu";
import { pass } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";
import { fxaa } from "three/addons/tsl/display/FXAANode.js";

const postProcessing = new PostProcessing(renderer);

const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode();

postProcessing.outputNode = fxaa(scenePassColor.add(bloom(scenePassColor, 0.8, 0.4, 0.85)));

renderer.setAnimationLoop(() => postProcessing.render());
```

Other available effect nodes: `dof` (DepthOfFieldNode), `ao` (GTAONode),
`smaa`, `ssr`, `denoise`, `motionBlur`, `lut3D`, `sobel`, `dotScreen`,
`chromaticAberration`, `film`, `afterImage`, `pixelation`, `outline`.

## Debugging

- `chrome://gpu` — WebGPU status and initialization errors.
- `chrome://flags` → enable **WebGPU Developer Features** for deeper validation.
- Validation errors appear in the console with a stack trace pointing at the
  offending call.
- DevTools **Performance** panel traces GPU work.
- `renderer.info` works the same as on WebGLRenderer — see `threejs-performance`.

## Gotchas

1. **`await renderer.init()` is mandatory.** Skipping it fails silently.
2. **Node materials only.** `MeshStandardMaterial` works, but `colorNode` and
   friends require the `*NodeMaterial` variants.
3. **`three/webgpu` is a separate bundle.** Don't mix `import * as THREE from "three"`
   with node materials — import classes from `three/webgpu`.
4. **Effect nodes are addons.** `bloom`, `fxaa` etc. are not `three/tsl` exports.
5. **CSM on WebGPU** uses `CSMShadowNode`, not `CSM` — see `threejs-lighting`.

## See Also

- `threejs-shaders` - GLSL, ShaderMaterial, WebGL shader patterns
- `threejs-performance` - Draw call budgets, profiling, `renderer.info`
- `threejs-postprocessing` - WebGL EffectComposer pipeline
- `threejs-materials` - Built-in (non-node) material types
