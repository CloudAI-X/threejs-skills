---
name: threejs-asset-pipeline
description: Three.js asset optimization and loading strategy - Draco, Meshopt, KTX2/Basis (UASTC vs ETC1S), gltf-transform, texture atlasing, LOD, decoder path setup, lazy loading, code splitting, progressive loading, web workers, Core Web Vitals. Use when models or textures are too large, first paint is slow, VRAM is high, or preparing 3D assets for production.
---

# Three.js Asset Pipeline

Two facts drive every decision here:

- **Texture memory is decompressed memory.** A 200KB PNG becomes 20MB+ in VRAM,
  because PNG/JPEG are decoded fully before upload. GPU-compressed formats
  (KTX2/Basis) stay compressed on the GPU — roughly 10x less VRAM.
- **Bytes on the wire block the first frame.** A 3D hero section that isn't
  code-split and lazy-loaded will wreck LCP even if the scene itself is fast.

## Optimize the Asset

### gltf-transform — the one tool to learn

```bash
npm install -g @gltf-transform/cli

# The catch-all: prune, dedupe, resample, compress geometry and textures
gltf-transform optimize model.glb output.glb \
  --texture-compress ktx2 \
  --compress draco
```

Individual operations when you want control:

```bash
gltf-transform draco model.glb out.glb --method edgebreaker  # geometry
gltf-transform meshopt model.glb out.glb                      # geometry (alt)
gltf-transform uastc model.glb out.glb                        # textures, high quality
gltf-transform etc1s model.glb out.glb                        # textures, small
gltf-transform prune model.glb out.glb                        # drop unused data
gltf-transform dedup model.glb out.glb                        # merge duplicates
gltf-transform resize model.glb out.glb --width 1024 --height 1024
gltf-transform inspect model.glb                              # what's actually in it
```

Run `inspect` first — most oversized GLBs are one forgotten 4K texture, not the
geometry.

### Geometry: Draco vs Meshopt

| | Draco | Meshopt |
|---|---|---|
| Compression | 90-95% reduction | Comparable (with gzip/brotli) |
| Decode speed | Slower | Faster |
| Decoder size | ~200KB WASM | ~30KB |
| Runs in worker | Yes | Yes |

Draco compresses hardest; Meshopt decodes faster with a much smaller decoder,
which often wins on mobile and on time-to-first-frame. Test both against your
actual model — the answer varies with topology.

### Textures: UASTC vs ETC1S

Both are Basis Universal modes inside a KTX2 container, transcoded at load time
to whatever the device supports (ASTC, BC7, ETC2, …).

- **UASTC** — higher quality, larger files. Use for **normal maps** (banding in
  a normal map is visible as shading artifacts) and hero/close-up textures.
- **ETC1S** — much smaller, acceptable quality. Use for **diffuse/albedo**,
  environment, and secondary textures.

Starting point: UASTC for normals, ETC1S for everything else.

Shopify's **gltf-compressor** gives an interactive visual A/B — hold `C` to see
the original, release for the compressed version. That answers "how far can I
push this before it looks bad" far faster than re-exporting.

### Texture atlasing

Every distinct texture is a bind. On mobile GPUs bind overhead is real. Combine
small textures into an atlas and remap UVs, or use a `DataArrayTexture` (array
texture) sampled by layer index — which pairs naturally with `BatchedMesh`.

### Level of detail

Swap high-poly for low-poly by distance. Typically 30-40% frame time back in
large scenes.

```javascript
const lod = new THREE.LOD();
lod.addLevel(highDetailMesh, 0);
lod.addLevel(mediumDetailMesh, 50);
lod.addLevel(lowDetailMesh, 100);
scene.add(lod);
```

In React Three Fiber use drei's `<Detailed distances={[0, 50, 100]}>`.

## Wire Up the Decoders

Compressed assets need decoder/transcoder files served alongside your app. This
is the #1 source of "it works locally, breaks in production".

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js";

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath("/draco/"); // copy from three/examples/jsm/libs/draco/
dracoLoader.preload(); // fetch the WASM before the first model needs it

const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath("/basis/"); // three/examples/jsm/libs/basis/
ktx2Loader.detectSupport(renderer); // MUST run before loading KTX2

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);
gltfLoader.setKTX2Loader(ktx2Loader);
gltfLoader.setMeshoptDecoder(MeshoptDecoder);
```

Checklist:

- Copy the decoder folders into your public/static directory as a build step, or
  point at a versioned CDN.
- `ktx2Loader.detectSupport(renderer)` is mandatory — without it the transcoder
  doesn't know which GPU format to target.
- `dracoLoader.preload()` avoids a WASM fetch on the critical path.
- Decoding runs in Web Workers, so it does not block the main thread.
- Dispose loaders (`dracoLoader.dispose()`) if you tear the app down.

## Loading Strategy & Core Web Vitals

### Code-split the library

`three` plus addons is a large bundle. Keep it out of the initial chunk:

```javascript
async function startScene() {
  const THREE = await import("three");
  const { GLTFLoader } = await import("three/addons/loaders/GLTFLoader.js");
  // ...
}
```

### Lazy load below the fold

Don't initialize a renderer the user hasn't scrolled to.

```javascript
const observer = new IntersectionObserver(
  (entries) => {
    if (entries[0].isIntersecting) {
      startScene();
      observer.disconnect();
    }
  },
  { rootMargin: "200px" }, // start slightly early so it's ready on arrival
);
observer.observe(canvasContainer);
```

### Preload above-the-fold assets

```html
<link rel="preload" href="/model.glb" as="fetch" crossorigin />
<link rel="preload" href="/texture.ktx2" as="fetch" crossorigin />
```

### Progressive loading

Show something immediately, upgrade in the background:

```javascript
const lowRes = await loadModel("/model-low.glb");
scene.add(lowRes);

loadModel("/model-high.glb").then((highRes) => {
  scene.add(highRes);
  scene.remove(lowRes);
  disposeSubtree(lowRes); // don't leak the placeholder
});
```

### Placeholder geometry

```javascript
const placeholder = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshBasicMaterial({ color: 0x808080, wireframe: true }),
);
scene.add(placeholder);

loadModel("/model.glb").then((model) => {
  scene.remove(placeholder);
  placeholder.geometry.dispose();
  placeholder.material.dispose();
  scene.add(model);
});
```

### Stream large scenes

For open worlds, load and unload by proximity rather than up front:

```javascript
function updateVisibleChunks(cameraPosition) {
  const near = getChunksNear(cameraPosition, LOAD_RADIUS);
  for (const chunk of near) if (!chunk.loaded) loadChunk(chunk);
  for (const chunk of loadedChunks) {
    if (!near.includes(chunk)) unloadChunk(chunk); // and dispose it
  }
}
```

### Offload heavy work to Web Workers

Anything CPU-heavy that isn't rendering — physics stepping, procedural
generation, mesh post-processing, pathfinding — belongs off the main thread.

```javascript
const worker = new Worker(new URL("./physics-worker.js", import.meta.url), {
  type: "module",
});
worker.postMessage({ positions, velocities }, [positions.buffer]); // transfer, don't copy
worker.onmessage = ({ data }) => applyResults(data);
```

Use transferable objects (`ArrayBuffer`) so large typed arrays move rather than
being cloned. Note: Draco/KTX2 decoding is already worker-based.

### Track progress

```javascript
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => setProgress((loaded / total) * 100);
manager.onLoad = () => hideLoadingScreen();
manager.onError = (url) => console.error("failed:", url);

const gltfLoader = new GLTFLoader(manager);
```

In R3F, `useProgress()` from drei plus `<Suspense>` covers this — see
`threejs-react-three-fiber`.

## Budgets

Rough production targets for a web 3D scene:

| Asset            | Budget                                  |
| ---------------- | --------------------------------------- |
| Total GLB        | < 5MB (< 2MB for a landing-page hero)   |
| Single texture   | 1024-2048px; 4K only for hero surfaces  |
| Texture format   | KTX2 — never ship PNG/JPEG for PBR maps |
| Triangles        | < 500k desktop, < 150k mobile           |
| Draw calls       | < 100 (see `threejs-performance`)       |

## See Also

- `threejs-loaders` - Loader APIs, async patterns, error handling
- `threejs-performance` - Draw calls, disposal, profiling
- `threejs-textures` - Texture configuration and memory
- `threejs-react-three-fiber` - Suspense, preloading, drei helpers
