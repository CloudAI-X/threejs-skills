---
name: threejs-loaders
description: "Loading three.js assets: glTF with Draco, meshopt and KTX2 compression, other model formats, textures, HDR and EXR environments, progress, caching, cloning and disposal. Use when loading 3D models, textures or environment maps, or managing loading progress, on WebGL or with WebGPU and TSL."
---

# Three.js Loaders

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.environment = new THREE.PMREMGenerator(renderer).fromScene(new RoomEnvironment(), 0.04).texture; // glTF PBR materials render black without lights or an environment

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(2, 1.5, 3);
camera.lookAt(0, 0, 0);

const gltf = await new GLTFLoader().loadAsync("model.glb");
scene.add(gltf.scene);

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## LoadingManager

One manager coordinates several loaders and reports combined progress. `loadAsync` still rejects on failure; `manager.onError` fires as well.

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const manager = new THREE.LoadingManager();

manager.onStart = (url, loaded, total) => {
  console.log(`Started loading: ${url}`);
};

manager.onProgress = (url, loaded, total) => {
  console.log(`Loading: ${((loaded / total) * 100).toFixed(1)}%`);
};

manager.onLoad = () => {
  console.log("All assets loaded");
};

manager.onError = (url) => {
  console.error(`Error loading: ${url}`);
};

const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);

// onLoad fires when every tracked item has finished
await Promise.all([
  gltfLoader.loadAsync("model.glb"),
  textureLoader.loadAsync("texture.jpg"),
  textureLoader.loadAsync("normal.jpg"),
]);
```

`manager.abort()` cancels in-flight requests for loaders that load through FileLoader (models, HDR, EXR, KTX2). Loaders that load through an HTML image element, such as TextureLoader, ignore it.

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const manager = new THREE.LoadingManager();
const timeoutId = setTimeout(() => manager.abort(), 30000);

try {
  const gltf = await new GLTFLoader(manager).loadAsync("model.glb");
  scene.add(gltf.scene);
} catch (error) {
  console.error(error);
} finally {
  clearTimeout(timeoutId);
}
```

## Texture Loading

`TextureLoader.load()` returns the texture immediately and fills it in when the image decodes; assign it right away and the first render picks it up. Adding a map to a material that has already rendered without one needs `material.needsUpdate = true`. `onProgress` is never called for image loading.

```javascript
new THREE.TextureLoader().load(
  "texture.jpg",
  (map) => {
    map.colorSpace = THREE.SRGBColorSpace; // a color map; scalar maps keep NoColorSpace
    mesh.material.map = map;
    mesh.material.needsUpdate = true; // the material rendered without a map before
  },
  undefined, // onProgress: not supported for image loading
  (error) => {
    console.error("Could not load texture", error);
  },
);
```

### Texture configuration

```javascript
const map = new THREE.TextureLoader().load("color.jpg");
map.colorSpace = THREE.SRGBColorSpace; // LDR color images; scalar maps keep NoColorSpace

map.wrapS = THREE.RepeatWrapping; // ClampToEdgeWrapping (default), RepeatWrapping, MirroredRepeatWrapping
map.wrapT = THREE.RepeatWrapping;
map.repeat.set(2, 2);
map.offset.set(0.5, 0.5);
map.rotation = Math.PI / 4;
map.center.set(0.5, 0.5);

map.minFilter = THREE.LinearMipmapLinearFilter; // default
map.magFilter = THREE.LinearFilter; // default; NearestFilter for a pixelated look
map.anisotropy = renderer.capabilities.getMaxAnisotropy(); // sharper textures at grazing angles

mesh.material.map = map;
// wrap, filters, anisotropy, colorSpace and flipY (true by default) apply at upload: set them before the first render or set map.needsUpdate = true; repeat, offset, rotation and center apply every frame
```

Color spaces: LDR color images (albedo, emissive) are `SRGBColorSpace`; scalar maps (normal, bump, roughness, metalness, ao, alpha) keep the default `NoColorSpace`. GLTFLoader sets the correct color spaces for the textures it loads, and the HDR loaders below set `LinearSRGBColorSpace` themselves. A texture you load yourself for a glTF model's material needs `map.flipY = false` to match the textures GLTFLoader loads.

### CubeTextureLoader

For skyboxes and cube environments, six faces in +X, -X, +Y, -Y, +Z, -Z order; the loader sets sRGB itself.

```javascript
const cubeTexture = await new THREE.CubeTextureLoader().loadAsync([
  "px.jpg",
  "nx.jpg",
  "py.jpg",
  "ny.jpg",
  "pz.jpg",
  "nz.jpg",
]);

scene.background = cubeTexture;
scene.environment = cubeTexture;
```

## HDR Environments

`HDRLoader` (Radiance `.hdr`) and `EXRLoader` return floating-point equirectangular textures. Both renderers prefilter an equirectangular environment themselves, so no PMREM step is needed for a file — set `mapping` and assign it.

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";
import { EXRLoader } from "three/addons/loaders/EXRLoader.js";

const env = await new HDRLoader().loadAsync("environment.hdr");
env.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = env;
scene.background = env; // without mapping, a background texture draws as a flat screen quad

const exr = await new EXRLoader().loadAsync("environment.exr");
exr.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = exr;
```

Dim the environment with `scene.environmentIntensity` (and rotate it with `scene.environmentRotation`); `material.envMapIntensity` scales only a material's own `envMap`, which stays null on models lit by `scene.environment`.

`UltraHDRLoader` reads gain-map JPEGs — HDR dynamic range at JPEG file sizes — the same way:

```javascript
import { UltraHDRLoader } from "three/addons/loaders/UltraHDRLoader.js";

const env = await new UltraHDRLoader().loadAsync("environment.jpg");
env.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = env;
```

Use `PMREMGenerator` for generated environments, not files:

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const pmremGenerator = new THREE.PMREMGenerator(renderer);
const room = new RoomEnvironment();
scene.environment = pmremGenerator.fromScene(room, 0.04).texture;
room.dispose(); pmremGenerator.dispose(); // the room's geometries and materials, then the generator
```

## glTF and GLB

The primary format for web delivery. The result carries `scene`, `animations`, `cameras`, `asset` (version, generator) and `userData` (authoring custom data).

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const gltf = await new GLTFLoader().loadAsync("model.glb");
scene.add(gltf.scene);

if (gltf.animations.length > 0) {
  const mixer = new THREE.AnimationMixer(gltf.scene);
  mixer.clipAction(gltf.animations[0]).play();
  const timer = new THREE.Timer();
  timer.connect(document);

  function animate() {
    timer.update();
    mixer.update(timer.getDelta());
    renderer.render(scene, camera);
  }

  renderer.setAnimationLoop(animate);
}
```

### Draco, meshopt and KTX2 compression

The Draco decoder and the Basis transcoder load from the three package by default; call `setDecoderPath` / `setTranscoderPath` only when you host them yourself, and then pin the path to the three release you ship. A file that requires `EXT_meshopt_compression` or `KHR_meshopt_compression` fails to load unless `setMeshoptDecoder` is set.

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js";

const gltfLoader = new GLTFLoader()
  .setDRACOLoader(new DRACOLoader())
  .setMeshoptDecoder(MeshoptDecoder)
  .setKTX2Loader(new KTX2Loader().detectSupport(renderer));

const gltf = await gltfLoader.loadAsync("compressed-model.glb");
scene.add(gltf.scene);
```

### Processing loaded content

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const gltf = await new GLTFLoader().loadAsync("model.glb");
const model = gltf.scene;

model.traverse((child) => {
  if (child.isMesh) {
    child.castShadow = true; // shadows also need renderer.shadowMap.enabled = true and a light with castShadow
    child.receiveShadow = true;
  }
});

model.getObjectByName("Head"); // look up named nodes

scene.environmentIntensity = 0.5; // dim scene.environment for this scene

// Center the model and scale it into a unit box
const box = new THREE.Box3().setFromObject(model);
const center = box.getCenter(new THREE.Vector3());
const size = box.getSize(new THREE.Vector3());
const scale = 1 / Math.max(size.x, size.y, size.z);
model.scale.setScalar(scale);
model.position.copy(center).multiplyScalar(-scale); // the offset is scaled too

scene.add(model);
```

## Other Model Formats

### OBJ + MTL

```javascript
import { MTLLoader } from "three/addons/loaders/MTLLoader.js";
import { OBJLoader } from "three/addons/loaders/OBJLoader.js";

const materials = await new MTLLoader().loadAsync("model.mtl");
materials.preload();

const object = await new OBJLoader().setMaterials(materials).loadAsync("model.obj");
scene.add(object);
```

### FBX

```javascript
import { FBXLoader } from "three/addons/loaders/FBXLoader.js";

const object = await new FBXLoader().loadAsync("model.fbx");
object.scale.setScalar(0.01); // FBX files are often exported in centimeters

const mixer = new THREE.AnimationMixer(object); // call mixer.update(timer.getDelta()) every frame, as in the glTF example
if (object.animations.length > 0) mixer.clipAction(object.animations[0]).play();

scene.add(object);
```

### STL

```javascript
import { STLLoader } from "three/addons/loaders/STLLoader.js";

const geometry = await new STLLoader().loadAsync("model.stl");
scene.add(new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: 0x888888 })));
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1));
scene.add(new THREE.DirectionalLight(0xffffff, 2)); // lit materials render black without lights; this one shines down from (0, 1, 0)
```

### PLY

The loader honors the file's data types, so `double` properties arrive as attributes backed by a `Float64Array`, which WebGLRenderer cannot upload; convert them to `Float32BufferAttribute` before rendering.

```javascript
import { PLYLoader } from "three/addons/loaders/PLYLoader.js";

const geometry = await new PLYLoader().loadAsync("model.ply");

for (const name of Object.keys(geometry.attributes)) {
  const attribute = geometry.getAttribute(name);
  if (attribute.array instanceof Float64Array) {
    geometry.setAttribute(name, new THREE.Float32BufferAttribute(attribute.array, attribute.itemSize));
  }
}

if (!geometry.hasAttribute("normal")) geometry.computeVertexNormals(); // keep the file's own normals
const material = new THREE.MeshStandardMaterial({ vertexColors: geometry.hasAttribute("color") }); // vertexColors without a color attribute renders black
scene.add(new THREE.Mesh(geometry, material));
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1));
scene.add(new THREE.DirectionalLight(0xffffff, 2)); // lit materials render black without lights; this one shines down from (0, 1, 0)
```

### USD

```javascript
import { USDLoader } from "three/addons/loaders/USDLoader.js";

const model = await new USDLoader().loadAsync("model.usdz"); // USD, USDA, USDC and USDZ archives
scene.add(model);
```

## Async Patterns and Custom Sources

Load independent assets in parallel:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

const [model, environment, colorMap] = await Promise.all([
  new GLTFLoader().loadAsync("model.glb"),
  new HDRLoader().loadAsync("environment.hdr"),
  new THREE.TextureLoader().loadAsync("color.jpg"),
]);

environment.mapping = THREE.EquirectangularReflectionMapping;
colorMap.colorSpace = THREE.SRGBColorSpace;

scene.add(model.scene);
scene.environment = environment;
```

Parse data you already hold (a downloaded file, a file input, a database record):

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const response = await fetch("model.glb");
const buffer = await response.arrayBuffer();

const gltf = await new GLTFLoader().parseAsync(buffer, "");
scene.add(gltf.scene);
```

Blob URLs and data URLs work as loader URLs:

```javascript
async function loadTextureFromBlob(blob) {
  const url = URL.createObjectURL(blob);
  const texture = await new THREE.TextureLoader().loadAsync(url);
  URL.revokeObjectURL(url);
  return texture;
}
```

Prefix paths and rewrite URLs through the manager:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.setPath("assets/models/"); // prefix for the URL you pass to load
loader.setResourcePath("assets/textures/"); // where the file's own references resolve
const gltf = await loader.loadAsync("model.glb"); // loads assets/models/model.glb

const manager = new THREE.LoadingManager();
manager.setURLModifier((url) => url.replace("assets/", "https://cdn.example.com/assets/"));
const cdnGltf = await new GLTFLoader(manager).loadAsync("assets/models/model.glb"); // fetched from https://cdn.example.com/assets/models/model.glb
```

Enable the shared cache to serve repeated URLs from memory:

```javascript
THREE.Cache.enabled = true;
THREE.Cache.clear(); // empty it when you need to force reloads
```

Keep payloads small — Draco or meshopt for geometry, KTX2 for textures, produced as in threejs-asset-pipeline — and show a placeholder mesh while the real model loads, swapping it in once loading resolves.

## Cloning and Disposal

`Object3D.clone()` on a skinned model shares the skeleton: every copy stays bound to the original's bones, so copies follow the original and ignore their own animation. Use `SkeletonUtils.clone()` for instances.

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const gltf = await new GLTFLoader().loadAsync("character.glb");
scene.add(gltf.scene);

const copy = SkeletonUtils.clone(gltf.scene);
scene.add(copy);
```

`object.dispose()` frees renderer-side state for that object only (WebGPU bindings, InstancedMesh buffers) and does not visit children or dispose geometry, materials or textures — traverse the subtree:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const gltf = await new GLTFLoader().loadAsync("model.glb");
const model = gltf.scene;
scene.add(model);

// ... later
scene.remove(model);
model.traverse((child) => {
  child.dispose(); // per-object renderer state: InstancedMesh buffers, WebGPU bindings
  if (child.geometry) { // meshes, lines and points
    child.geometry.dispose();
    const materials = Array.isArray(child.material) ? child.material : [child.material];
    for (const material of materials) {
      for (const value of Object.values(material)) {
        if (value?.isTexture) value.dispose();
      }
      material.dispose();
    }
  }
});
```

## WebGPU and TSL

```javascript
import * as THREE from "three/webgpu";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";
const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111827); // the WebGPU canvas is transparent by default
scene.environment = new THREE.PMREMGenerator(renderer).fromScene(new RoomEnvironment(), 0.04).texture;
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(2, 1.5, 3);
camera.lookAt(0, 0, 0);
const gltf = await new GLTFLoader().loadAsync("model.glb"); // loaded materials become node materials automatically
scene.add(gltf.scene);
function animate() { renderer.render(scene, camera); } renderer.setAnimationLoop(animate);
```

- `ktx2Loader.detectSupport(renderer)` and the `PMREMGenerator` methods (`fromScene`, `fromEquirectangular`, `fromCubemap`) throw before `await renderer.init()`. Max anisotropy is `renderer.getMaxAnisotropy()`; `renderer.capabilities` does not exist under WebGPURenderer.
- Gaussian splats: `SPLATLoader`, `SPZLoader`, `KSPLATLoader` and `GaussianSplatPLYLoader` load data for `new GaussianSplat(data)` from `three/addons/objects/GaussianSplat.js`, which requires WebGPURenderer.

## Common Mistakes

- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"` → `import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"`: the `three/examples/jsm/` specifier does not resolve under the import map three.js documents.
- `import { RGBELoader } from "three/addons/loaders/RGBELoader.js"` → `import { HDRLoader } from "three/addons/loaders/HDRLoader.js"`: renamed from RGBELoader in r180; the old class warns on every construction.
- `new Promise((resolve, reject) => loader.load(url, resolve, undefined, reject))` → `loader.loadAsync(url)`: every loader already implements loadAsync, which rejects on error.
- `material.map = new THREE.TextureLoader().load("color.jpg")` → `const map = new THREE.TextureLoader().load("color.jpg"); map.colorSpace = THREE.SRGBColorSpace; material.map = map;`: a loaded color map defaults to NoColorSpace and renders washed out.
- `normalMap.colorSpace = THREE.LinearSRGBColorSpace` → `normalMap.colorSpace = THREE.NoColorSpace`: scalar maps (normal, bump, roughness, metalness, ao, alpha) keep NoColorSpace; LinearSRGBColorSpace is for linear color data.
- `texture.encoding = THREE.sRGBEncoding` → `texture.colorSpace = THREE.SRGBColorSpace`: encoding and sRGBEncoding no longer exist, so the old line leaves the map at NoColorSpace and it renders washed out.
- `child.material.envMapIntensity = 0.5` → `scene.environmentIntensity = 0.5`: envMapIntensity scales only material.envMap, which stays null when the model is lit by scene.environment.
- `scene.environment = pmremGenerator.fromEquirectangular(hdrTexture).texture` → `hdrTexture.mapping = THREE.EquirectangularReflectionMapping; scene.environment = hdrTexture;`: both renderers run that PMREM step themselves for an equirectangular texture, so the manual pass only adds code and a render target to dispose.
- `scene.background = hdrTexture` → `hdrTexture.mapping = THREE.EquirectangularReflectionMapping; scene.background = hdrTexture;`: without the mapping, a background texture draws as a flat screen quad.
- `ktx2Loader.setTranscoderPath(pathFromAnotherRelease)` → `const ktx2Loader = new KTX2Loader().detectSupport(renderer)`: the Basis transcoder and the Draco decoder load from the three package by default; a path pinned to another release mismatches the loader.
- `ktx2Loader.detectSupportAsync(renderer)` → `await renderer.init(); ktx2Loader.detectSupport(renderer);`: detectSupportAsync is deprecated in r181; call detectSupport once the renderer is initialized.
- `renderer.capabilities.getMaxAnisotropy()` → `renderer.getMaxAnisotropy()`: capabilities exists on WebGLRenderer only; under WebGPURenderer use the renderer method.
- `scene.add(new THREE.Mesh(await new PLYLoader().loadAsync("scan.ply"), material))` → `const geometry = await new PLYLoader().loadAsync("scan.ply"); for (const [name, a] of Object.entries(geometry.attributes)) if (a.array instanceof Float64Array) geometry.setAttribute(name, new THREE.Float32BufferAttribute(a.array, a.itemSize)); scene.add(new THREE.Mesh(geometry, material));`: PLYLoader keeps double properties as Float64 attributes, which WebGLRenderer cannot upload (it throws Unsupported buffer data format).
- `gltf.scene.clone()` → `SkeletonUtils.clone(gltf.scene)`: Object3D.clone() shares the skeleton, so every copy stays bound to the original's bones and follows its animation.
- `gltf.scene.dispose()` → `gltf.scene.traverse((child) => { child.geometry?.dispose(); for (const material of [child.material ?? []].flat()) { for (const value of Object.values(material)) if (value?.isTexture) value.dispose(); material.dispose(); } child.dispose(); })`: Object3D.dispose() frees renderer state for that object only, never its geometry, materials or textures, and material.dispose() does not free the material's textures.
- `fetch(url, { signal: controller.signal })` → `setTimeout(() => manager.abort(), 30000)`: loaders are cancelled through their LoadingManager, which aborts in-flight FileLoader requests.
- `new USDZLoader()` → `new USDLoader()`: deprecated in r179; USDLoader reads USD, USDA, USDC and USDZ.

## See Also

- `threejs-fundamentals` - renderer setup, resize handling and the app scaffold around loaded assets
- `threejs-textures` and `threejs-materials` - texture properties and color spaces, and adjusting the materials loaded models come with
- `threejs-asset-pipeline` - compressing and optimizing glTF files before they reach these loaders
- `threejs-animation` - AnimationMixer, clips and skinned-mesh playback
