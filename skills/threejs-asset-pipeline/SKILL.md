---
name: threejs-asset-pipeline
description: "Shrinking glTF assets for three.js with gltf-transform: inspect, validate and optimize commands, Draco, meshopt and quantization, KTX2, WebP and AVIF textures, the loader setup each output needs, and gltfjsx for React. Use when compressing, optimizing or validating 3D models before loading them."
---

# Three.js Asset Pipeline

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

Compress on the command line, then load the result. Run `inspect` before and after to see what the pass saved:

```bash
npx @gltf-transform/cli inspect model.glb
npx @gltf-transform/cli optimize model.glb model.opt.glb --compress draco --texture-compress webp --texture-size 2048
npx @gltf-transform/cli inspect model.opt.glb
```

Draco output needs a DRACOLoader attached; WebP textures decode in the browser:

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const sun = new THREE.DirectionalLight(0xffffff, 2); // glTF PBR materials render black without lights or an environment
sun.position.set(3, 5, 2);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x444444, 1.5));

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(2, 1.5, 3);
camera.lookAt(0, 0, 0);

const gltf = await new GLTFLoader()
  .setDRACOLoader(new DRACOLoader()) // the Draco WASM decoder loads from the three package
  .loadAsync("model.opt.glb");
scene.add(gltf.scene);

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Inspect and Validate

Never guess where the bytes are. `inspect` prints an overview (generator, `extensionsUsed`, `extensionsRequired`) and one table each for scenes, meshes, materials, textures and animations, so a geometry-heavy file and a texture-heavy file are obvious at a glance.

```bash
npx @gltf-transform/cli inspect model.glb
npx @gltf-transform/cli inspect model.opt.glb --format csv
npx @gltf-transform/cli validate model.glb --ignore ACCESSOR_WEIGHTS_NON_NORMALIZED
```

`extensionsRequired` is the runtime contract: `KHR_draco_mesh_compression`, `EXT_meshopt_compression` and `KHR_texture_basisu` each map to a piece of loader wiring below, while `KHR_mesh_quantization`, `EXT_mesh_gpu_instancing`, `EXT_texture_webp` and `EXT_texture_avif` need nothing at runtime. `validate` runs the official glTF validator and reports errors, warnings and hints; it exits nonzero when errors are found, so it works as a CI gate. The commands in this skill that write a file log the input and output sizes when they finish; `inspect` the output to see which table the savings came from.

## The optimize Command

`optimize` chains the common passes in one run: dedup, instance, palette, flatten, join, weld, simplify, resample, prune and sparse data, then texture compression, and geometry compression last. With no flags it applies meshopt compression and recompresses textures in their original format. Defaults are aggressive but generic — `inspect` first, then tune:

```bash
# draco geometry + webp textures, capped at 2048px
npx @gltf-transform/cli optimize model.glb model.opt.glb --compress draco --texture-compress webp --texture-size 2048
# GPU textures instead of download size, keep every triangle
npx @gltf-transform/cli optimize model.glb model.opt.glb --texture-compress ktx2 --simplify false
```

Main flags (booleans take an explicit value, like `--simplify false`):

- `--compress draco|meshopt|quantize` — geometry and animation compression (default `meshopt`).
- `--texture-compress ktx2|webp|avif` — texture codec (default `auto`, recompress in the original format).
- `--texture-size <px>` — cap texture dimensions, default 2048.
- `--simplify <bool>`, `--simplify-ratio <0-1>`, `--simplify-error <fraction>`, `--simplify-lock-border <bool>` — mesh simplification; ratio 0 and error 0.0001 by default.
- `--instance <bool>`, `--instance-min <count>` — GPU instancing for repeated meshes.
- `--join <bool>`, `--join-meshes <bool>`, `--join-named <bool>`, `--flatten <bool>` — draw-call reduction; `--join` requires `--flatten`.
- `--weld <bool>` — merge duplicate vertices; `draco` welds by itself even with `--weld false`, while `simplify` welds only unindexed primitives, so keep `--weld true` when simplifying.
- `--prune <bool>`, `--resample <bool>`, `--palette <bool>`, `--sparse <bool>`, `--meshopt-level medium|high`.

With `--texture-compress ktx2`, `optimize` picks bitstreams itself: UASTC for the normal, occlusion and metallic-roughness slots, ETC1S for everything else. For alternatives outside this skill's lane, gltfpack covers similar ground; gltf-transform is enough here.

## Geometry Compression

Three codecs with different tradeoffs — pick per model, not per project:

- **Draco** (`--compress draco`, `draco` command): compresses triangle-mesh geometry only — animation, morph targets, textures, points and lines are not compressed. Runtime: DRACOLoader.
- **Meshopt** (`--compress meshopt`, `meshopt` command): compresses point, line and triangle geometry, morph targets and animation tracks, and decodes very quickly; pair it with lossless gzip or brotli over the wire. Runtime: MeshoptDecoder.
- **Quantize** (`quantize` command): stores float attributes as 8- or 16-bit integers under KHR_mesh_quantization — smaller on disk and in GPU memory, and no decoder to wire; GLTFLoader reads quantized attributes natively.

```bash
npx @gltf-transform/cli draco model.glb model.opt.glb --method edgebreaker --quantize-position 14
npx @gltf-transform/cli meshopt model.glb model.opt.glb --level high
npx @gltf-transform/cli quantize model.glb model.opt.glb
npx @gltf-transform/cli gzip model.opt.glb
```

`gzip` writes `model.opt.glb.gz` next to its input; serve it with `Content-Encoding: gzip` and the browser decompresses it transparently. It adds little on top of Draco or JPEG textures, and earns the most on top of meshopt and quantize.

Make `draco`, `meshopt` or `optimize` the last command in a chain: every other command that rewrites the model decodes Draco and meshopt geometry when it reads the file (it warns "Decoded KHR_draco_mesh_compression. Further compression will be lossy.") and writes it back uncompressed.

## Texture Compression

Two different problems: KTX2 (Basis Universal) stays compressed in GPU memory wherever the GPU supports one of its transcode targets (KTX2Loader falls back to uncompressed RGBA otherwise) and uploads fast — it optimizes GPU memory and performance — while WebP and AVIF shrink the download and are decompressed to raw pixels in GPU memory, which can occupy 4-8x more there.

```bash
# KTX2: ETC1S for color (smaller, lossier), UASTC for normal maps (larger, cleaner)
npx @gltf-transform/cli etc1s model.glb model.opt.glb --quality 128
npx @gltf-transform/cli uastc model.glb model.opt.glb --level 4
# Download size: WebP or AVIF, quality 1-100
npx @gltf-transform/cli webp model.glb model.opt.glb --quality 80
# Cap dimensions of PNG or JPEG textures; never enlarges them
npx @gltf-transform/cli resize model.glb model.opt.glb --width 1024 --height 1024
```

KTX2 output — `etc1s`, `uastc` or `optimize --texture-compress ktx2` — shells out to the `ktx` command from KTX-Software (4.4.0 or newer), so install that first; WebP and AVIF are encoded in-process. The single `etc1s` and `uastc` commands convert every texture to one bitstream; the per-slot split (UASTC for normal, occlusion and metallic-roughness, ETC1S for the rest) is what `optimize --texture-compress ktx2` does for you, and `--slots` selects texture roles by name when you need a custom split. Runtime: KTX2 needs KTX2Loader with `detectSupport`; WebP and AVIF decode in the browser.

## Scene Optimization Commands

Run these when `inspect` shows the problem is draw calls or vertex count rather than bytes:

```bash
npx @gltf-transform/cli weld model.glb model.opt.glb
npx @gltf-transform/cli simplify model.glb model.opt.glb --ratio 0.5 --error 0.001
npx @gltf-transform/cli join model.glb model.opt.glb
npx @gltf-transform/cli instance model.glb model.opt.glb --min 5
npx @gltf-transform/cli dedup model.glb model.opt.glb
npx @gltf-transform/cli prune model.glb model.opt.glb
```

- `weld` merges bitwise-identical vertices and indexes the primitive (points are skipped). `draco` welds by itself; `simplify` welds only unindexed primitives and `meshopt` none, and meshopt's reorder step skips unindexed primitives (it warns when none are indexed), so run `weld` before `simplify` and `meshopt`.
- `simplify` removes triangles, aiming for `--ratio` (fraction of vertices to keep) while staying under `--error` (fraction of mesh extent); `--lock-border true` preserves open edges. It is lossy.
- `join` merges compatible primitives to cut draw calls; it runs dedup and flatten itself first. In scenes that reuse one mesh heavily it inflates vertex count — use `instance` there instead.
- `instance` writes EXT_mesh_gpu_instancing for meshes referenced by several nodes (`--min` defaults to 5); GLTFLoader renders those batches as an InstancedMesh with no runtime setup. It skips any file that contains an animation, and instanced meshes must share materials.
- `dedup` merges redundant accessors, materials, meshes, skins and textures left by exporters.
- `prune` deletes properties no Scene references, including animations that target nothing attached to a scene.

## Loader Setup for Each Output

The pipeline choice decides the loader wiring. The Draco decoder and the Basis transcoder load from the three package by default — call `setDecoderPath` or `setTranscoderPath` only when you host those files yourself, and then pin the path to the three release you ship. Create one KTX2Loader for the app and share it between GLTFLoaders: each active instance loads its own transcoder and workers, and three.js warns when several are active.

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js";

// Wire every codec the pipeline may emit; a missing one throws at load time
const gltfLoader = new GLTFLoader()
  .setDRACOLoader(new DRACOLoader()) // KHR_draco_mesh_compression
  .setMeshoptDecoder(MeshoptDecoder) // EXT_meshopt_compression
  .setKTX2Loader(new KTX2Loader().detectSupport(renderer)); // KHR_texture_basisu

const gltf = await gltfLoader.loadAsync("model.opt.glb");
scene.add(gltf.scene);
```

What each missing piece throws: a Draco file without a DRACOLoader ("No DRACOLoader instance provided"), a meshopt file without a decoder ("setMeshoptDecoder must be called before loading compressed files"), a KTX2 file without a KTX2Loader ("setKTX2Loader must be called before loading KTX2 textures"), and a KTX2Loader that never ran `detectSupport` ("Missing initialization with `.detectSupport( renderer )`"). WebP and AVIF textures, quantized attributes and EXT_mesh_gpu_instancing need none of this.

## Scripted Pipelines

When the CLI stops fitting — per-project defaults, CI, many files — script the same transforms with the SDK. A Node script, run with `node pipeline.mjs`:

```javascript
// deps: @gltf-transform/core @gltf-transform/extensions @gltf-transform/functions draco3dgltf
import { NodeIO } from "@gltf-transform/core";
import { ALL_EXTENSIONS } from "@gltf-transform/extensions";
import { dedup, prune, weld, draco } from "@gltf-transform/functions";
import draco3d from "draco3dgltf";

const io = new NodeIO()
  .registerExtensions(ALL_EXTENSIONS) // an extension left out is dropped on read, or throws if required
  .registerDependencies({
    "draco3d.decoder": await draco3d.createDecoderModule(), // reads Draco input
    "draco3d.encoder": await draco3d.createEncoderModule(),
  });

const document = await io.read("model.glb");
await document.transform(
  dedup(),
  weld(), // shared vertices, before compression
  prune(), // drop what no Scene references
  draco({ method: "edgebreaker", quantizePosition: 14 }),
);
await io.write("model.opt.glb", document);
```

Compression runs when the I/O class writes the file, so it stays the last step. Register every extension: the I/O writes only registered extensions, so WebP textures or quantized attributes without theirs make an invalid file. For meshopt instead, add `"meshopt.encoder": MeshoptEncoder` to `registerDependencies` after `await MeshoptEncoder.ready` (MeshoptEncoder from the meshoptimizer package) and pass `meshopt({ encoder: MeshoptEncoder })`; texture compression in scripts takes `textureCompress({ encoder, targetFormat: "webp" })` with sharp as the encoder.

## React Output with gltfjsx

`gltfjsx` turns a model into a React Three Fiber component — declarative JSX you can edit, instead of a loaded black box:

```bash
npx gltfjsx model.glb --transform
```

This writes `Model.jsx` (a component for @react-three/fiber; `--types` for a .tsx) and, because of `--transform`, also rewrites the asset: `model-transformed.glb` gets palette materials, joined meshes, welded and pruned geometry, Draco compression, and textures re-encoded as WebP, capped at `--resolution` (1024 by default) — normal maps are re-encoded as JPEG, capped at 2048 px or `--resolution`, whichever is larger — and the generated component loads that file. Add `--simplify` with `--ratio` and `--error` for simplification. The component side (Canvas, Suspense, useGLTF) is the threejs-react-three-fiber skill.

## WebGPU and TSL

The loaders and addons above are identical under WebGPURenderer, which renders the loaded materials as node materials by itself (`mesh.material` stays the class GLTFLoader created). One ordering rule: `KTX2Loader.detectSupport(renderer)` needs the renderer's backend, so create the loader after `await renderer.init()`.

```javascript
import * as THREE from "three/webgpu";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
// renderer is an initialized WebGPURenderer (its init() was awaited)
const gltfLoader = new GLTFLoader()
  .setKTX2Loader(new KTX2Loader().detectSupport(renderer));
const gltf = await gltfLoader.loadAsync("model.opt.glb");
scene.add(gltf.scene);
```

## Common Mistakes

- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"` → `import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"`: the three/examples/jsm specifier does not resolve under the import map three.js documents.
- `npx @gltf-transform/cli optimize model.glb out.glb --compress ktx2` → `npx @gltf-transform/cli optimize model.glb out.glb --compress draco --texture-compress ktx2`: texture codecs belong to `--texture-compress`; `--compress` takes draco, meshopt or quantize.
- `npx @gltf-transform/cli draco model.glb a.glb && npx @gltf-transform/cli webp a.glb out.glb --quality 80` → `npx @gltf-transform/cli webp model.glb a.glb --quality 80 && npx @gltf-transform/cli draco a.glb out.glb`: every command except `draco`, `meshopt` and `optimize` decodes Draco and meshopt geometry when it reads a file and writes it uncompressed, so geometry compression runs last.
- `new GLTFLoader().loadAsync("draco.glb")` → `new GLTFLoader().setDRACOLoader(new DRACOLoader()).loadAsync("draco.glb")`: loading a KHR_draco_mesh_compression file without a DRACOLoader throws "No DRACOLoader instance provided".
- `new GLTFLoader().loadAsync("meshopt.glb")` → `new GLTFLoader().setMeshoptDecoder(MeshoptDecoder).loadAsync("meshopt.glb")`: an EXT_meshopt_compression file throws "setMeshoptDecoder must be called before loading compressed files".
- `new GLTFLoader().loadAsync("ktx2.glb")` → `new GLTFLoader().setKTX2Loader(new KTX2Loader().detectSupport(renderer)).loadAsync("ktx2.glb")`: a KHR_texture_basisu file throws "setKTX2Loader must be called before loading KTX2 textures".
- `setKTX2Loader(new KTX2Loader())` → `setKTX2Loader(new KTX2Loader().detectSupport(renderer))`: a KTX2Loader that never ran detectSupport throws "Missing initialization with `.detectSupport( renderer )`".
- `dracoLoader.setDecoderPath(cdnPathFromAnotherRelease)` → `new GLTFLoader().setDRACOLoader(new DRACOLoader())`: the Draco decoder loads from the three package by default, and a path pinned to a different release mismatches the loader.
- `dracoLoader.setDecoderConfig({ type: "js" })` → `new DRACOLoader()`: setDecoderConfig is deprecated in r185; without it DRACOLoader uses the WASM decoder wherever WebAssembly exists.
- `ktx2Loader.detectSupportAsync(renderer)` → `await renderer.init(); ktx2Loader.detectSupport(renderer);`: detectSupportAsync is deprecated in r181; call detectSupport once the renderer is initialized.
- `npx @gltf-transform/cli dequantize model.glb out.glb` → `npx @gltf-transform/cli quantize model.glb out.glb`: GLTFLoader reads KHR_mesh_quantization attributes with no runtime setup, so dequantize only when your own code needs float vertex data in model units: quantized attributes hold normalized integers, and quantized positions are scaled back by their node's transform.

## See Also

- `threejs-loaders` - the runtime side: loading progress, caching, cloning skinned models and disposal of what the pipeline produces.
- `threejs-react-three-fiber` - the React app around a gltfjsx component: Canvas, useGLTF, Suspense and drei helpers.
- `threejs-performance` - measuring what the pipeline bought: draw calls, GPU memory and renderer.info.
