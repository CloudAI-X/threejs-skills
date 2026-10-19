---
name: threejs-textures
description: "Three.js textures: image loading, color spaces, UV channels and transforms, texture types from canvas to HDR and KTX2, render targets, texture memory. Use when working with images on materials, UV coordinates, cubemaps, render targets or texture optimization; on WebGL or with WebGPU and TSL."
---

# Three.js Textures

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 2);

// Color images must be tagged sRGB or they render too bright and washed out
const texture = new THREE.TextureLoader().load("diffuse.jpg");
texture.colorSpace = THREE.SRGBColorSpace;

const mesh = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  new THREE.MeshBasicMaterial({ map: texture }),
);
scene.add(mesh);

renderer.setAnimationLoop(animate);

function animate() {
  mesh.rotation.z += 0.01;
  renderer.render(scene, camera);
}
```

## Loading

```javascript
// load() returns the texture at once; its image arrives asynchronously
const texture = new THREE.TextureLoader().load("diffuse.jpg");
texture.colorSpace = THREE.SRGBColorSpace;
mesh.material.map = texture; // before the first render; afterwards also set mesh.material.needsUpdate = true

// When the next step needs the loaded image, await it
const colorMap = await new THREE.TextureLoader().loadAsync("diffuse.jpg");
colorMap.colorSpace = THREE.SRGBColorSpace;
```

TextureLoader's `onProgress` parameter is unsupported and never fires; a `LoadingManager` reports progress as each file finishes:

```javascript
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => console.log(`${loaded}/${total}: ${url}`);
const loader = new THREE.TextureLoader(manager);
```

## Color Space

Color management is on by default and every texture starts at `NoColorSpace`:

- LDR color images get `SRGBColorSpace`: `map`, `emissiveMap`, `specularMap`, sheen color and specular color maps, LDR matcaps, and canvas or video used as color.
- Scalar data keeps `NoColorSpace`: normal, bump, displacement, roughness, metalness, ambient-occlusion and alpha maps.
- HDR environments are `LinearSRGBColorSpace`; the HDR loaders set that themselves. GLTFLoader sets the color spaces of the textures it loads.

```javascript
const colorMap = new THREE.TextureLoader().load("color.jpg");
colorMap.colorSpace = THREE.SRGBColorSpace; // authored color image

const normalMap = new THREE.TextureLoader().load("normal.jpg");
// normalMap stays at NoColorSpace: it is data, not color
```

## Transform: Wrap, Repeat, Offset, Rotation

```javascript
const texture = new THREE.TextureLoader().load("diffuse.jpg");

// Tiling requires repeat wrapping; the default clamps edge pixels
texture.wrapS = THREE.RepeatWrapping;
texture.wrapT = THREE.RepeatWrapping;
texture.repeat.set(4, 4); // tile 4x4

texture.offset.set(0.5, 0.5); // shift in UV units
texture.center.set(0.5, 0.5); // pivot for rotation
texture.rotation = Math.PI / 4; // radians
```

`MirroredRepeatWrapping` tiles with a mirror flip on each repeat.

## Filtering, Mipmaps, Anisotropy

```javascript
const texture = new THREE.TextureLoader().load("diffuse.jpg");

// Minification (many texture pixels per screen pixel)
texture.minFilter = THREE.LinearMipmapLinearFilter; // default: trilinear blend between mip levels
texture.minFilter = THREE.NearestFilter; // hard pixel look, samples only the base level

// Magnification (texture magnified on screen)
texture.magFilter = THREE.LinearFilter; // default
texture.magFilter = THREE.NearestFilter; // pixelated

// Mipmaps are generated whenever generateMipmaps is true, whatever the filters.
// Turn generation off to save memory (UI sprites, data textures):
texture.generateMipmaps = false;
texture.minFilter = THREE.LinearFilter; // and do not request a mipmap filter

// Sharper at grazing angles; clamp to what the hardware supports
texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
```

## Texture Types

Base class from any image element:

```javascript
const image = new Image();
image.src = "sprite.png";

const imageTexture = new THREE.Texture(image);
imageTexture.colorSpace = THREE.SRGBColorSpace;
image.onload = () => (imageTexture.needsUpdate = true); // upload once the pixels exist
```

`DataTexture` from raw typed-array pixels. Its defaults differ from `Texture`: `NearestFilter` for both filters, `flipY` false, mipmaps off:

```javascript
const size = 256;
const data = new Uint8Array(size * size * 4); // RGBA, one byte per channel
for (let i = 0; i < size; i++) {
  for (let j = 0; j < size; j++) {
    const index = (i * size + j) * 4;
    data[index] = i;
    data[index + 1] = j;
    data[index + 2] = 128;
    data[index + 3] = 255;
  }
}
const dataTexture = new THREE.DataTexture(data, size, size);
dataTexture.needsUpdate = true;
```

`CanvasTexture` for dynamically drawn textures (sets `needsUpdate` on construction):

```javascript
const canvas = document.createElement("canvas");
canvas.width = 256;
canvas.height = 256;
const ctx = canvas.getContext("2d");
ctx.fillStyle = "#404040";
ctx.fillRect(0, 0, 256, 256);
ctx.fillStyle = "#ffffff";
ctx.font = "48px sans-serif";
ctx.fillText("Hello", 40, 150);

const canvasTexture = new THREE.CanvasTexture(canvas);
canvasTexture.colorSpace = THREE.SRGBColorSpace; // canvas colors are sRGB

// after each redraw of the canvas:
canvasTexture.needsUpdate = true;
```

`VideoTexture` for a playing `HTMLVideoElement` (re-uploads every frame, mipmaps off):

```javascript
const video = document.createElement("video");
video.src = "video.mp4";
video.loop = true;
video.muted = true;
video.play();

const videoTexture = new THREE.VideoTexture(video);
videoTexture.colorSpace = THREE.SRGBColorSpace;
```

`VideoFrameTexture` for WebCodecs: each decoded frame is pushed by hand:

```javascript
const frameTexture = new THREE.VideoFrameTexture();
frameTexture.colorSpace = THREE.SRGBColorSpace;
const decoder = new VideoDecoder({
  output(frame) {
    if (frameTexture.image instanceof VideoFrame) frameTexture.image.close(); // free the replaced frame, or decoding stalls
    frameTexture.setFrame(frame);
  },
  error: (e) => console.error(e),
});
```

`HTMLTexture` draws an HTML element through the browser's HTML-in-Canvas API: the renderer moves the element into its own canvas and re-uploads it on paint events; where the browser lacks that API the texture stays empty.

## Compressed Textures (KTX2)

```javascript
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";

const ktx2Loader = new KTX2Loader();
// No setTranscoderPath call: the transcoder loads from the three package.
// Set one only when self-hosting the decoders, pinned to this release.
ktx2Loader.detectSupport(renderer); // required before loading; works with both renderers

const compressedMap = await ktx2Loader.loadAsync("diffuse.ktx2");
// colorSpace comes from the file's transfer function; nothing to set by hand
```

Basis Universal KTX2 transcodes to a compressed format the GPU supports: smaller uploads and less memory than PNG or JPEG, which upload uncompressed (where the GPU supports none of the targets, KTX2Loader also falls back to uncompressed RGBA). Compressed textures cannot be flipped or mipmapped at upload: `flipY` stays false, so author the file in the orientation your UVs expect, and mipmaps must be embedded in the file.

## Environment Maps

Cube maps (the loader tags the result sRGB itself):

```javascript
const cubeTexture = new THREE.CubeTextureLoader().load([
  "px.jpg", "nx.jpg", // +X, -X
  "py.jpg", "ny.jpg", // +Y, -Y
  "pz.jpg", "nz.jpg", // +Z, -Z
]);

scene.background = cubeTexture;
scene.environment = cubeTexture; // lights Standard and Physical materials (WebGLRenderer also Lambert and Phong)
```

HDR equirectangular files — set `mapping`, then assign; both renderers prefilter the environment themselves:

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

const env = await new HDRLoader().loadAsync("environment.hdr");
env.mapping = THREE.EquirectangularReflectionMapping; // without it the background draws as a flat screen quad
scene.environment = env;
scene.background = env;
```

`EXRLoader` (`three/addons/loaders/EXRLoader.js`) loads `.exr` files the same way and `UltraHDRLoader` loads Ultra HDR JPEGs; both set their own linear color space.

Scene-level controls:

```javascript
scene.environmentIntensity = 0.5; // dim the image-based lighting
scene.environmentRotation.y = Math.PI / 4;

scene.backgroundBlurriness = 0.5; // 0-1
scene.backgroundIntensity = 1.0;
scene.backgroundRotation.y = Math.PI;
```

Both renderers convert an equirectangular environment into a cube map with a larger texture internally (better sampling, more memory); when memory matters, ship a lower-resolution equirect file.

`PMREMGenerator` is for generated environments — a scene such as `RoomEnvironment` captured with `fromScene`:

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const pmrem = new THREE.PMREMGenerator(renderer);
const room = new RoomEnvironment();
scene.environment = pmrem.fromScene(room, 0.04).texture;
room.dispose(); // frees the room's geometries and materials
pmrem.dispose();
```

Do not run a loaded equirect file through PMREMGenerator "for better reflections" — direct assignment with `mapping` set produces the same prefiltering.

## Render Targets

```javascript
const renderTarget = new THREE.WebGLRenderTarget(512, 512);
const rtScene = new THREE.Scene(); // must not contain the mesh that samples the target
rtScene.add(new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshNormalMaterial())); // unlit, visible without lights

renderer.setRenderTarget(renderTarget);
renderer.render(rtScene, camera);
renderer.setRenderTarget(null); // back to the canvas

// the output is a texture like any other
mesh.material.map = renderTarget.texture;
```

Depth buffer as a texture:

```javascript
const depthTarget = new THREE.WebGLRenderTarget(512, 512);
depthTarget.depthTexture = new THREE.DepthTexture(512, 512); // default type UnsignedIntType
```

MSAA and multiple attachments:

```javascript
// MSAA: resolved into .texture automatically after rendering
const msaaTarget = new THREE.WebGLRenderTarget(512, 512, { samples: 4 });

// MRT: N color attachments, read back through .textures
const mrtTarget = new THREE.WebGLRenderTarget(512, 512, { count: 2 });
const firstAttachment = mrtTarget.textures[0];
const secondAttachment = mrtTarget.textures[1];
```

## Dynamic Reflections

```javascript
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter,
});
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);

mesh.material.envMap = cubeRenderTarget.texture;

// each frame: hide the reflective mesh so it does not see itself
mesh.visible = false;
cubeCamera.position.copy(mesh.position);
cubeCamera.update(renderer, scene);
mesh.visible = true;
```

## UV Channels

Every UV-mapped material texture samples the UV attribute chosen by `texture.channel`: `0` reads `uv` (the default), `1` reads `uv1`, `2` reads `uv2`, `3` reads `uv3`. So `aoMap` and `lightMap` work on the default `uv` set; a separately unwrapped set goes in `uv1` with the texture pointed at it:

```javascript
const aoTexture = new THREE.TextureLoader().load("ao.jpg");

const geometry = new THREE.BoxGeometry(1, 1, 1);
const uvAttribute = geometry.attributes.uv;
const aoUvs = new Float32Array(uvAttribute.count * 2); // fill with the second unwrap
geometry.setAttribute("uv1", new THREE.BufferAttribute(aoUvs, 2));

const material = new THREE.MeshStandardMaterial({ aoMap: aoTexture });
material.aoMap.channel = 1; // sample "uv1" instead of "uv"
const aoMesh = new THREE.Mesh(geometry, material);
scene.add(aoMesh);
```

## Texture Atlas

One image, many sprites, each selected with offset and repeat:

```javascript
const atlas = await new THREE.TextureLoader().loadAsync("atlas.png");
atlas.colorSpace = THREE.SRGBColorSpace;
// filtering and mipmaps sample across sprite borders: leave padding around each sprite

// one clone per sprite: clones share the image source, so WebGLRenderer uploads it once
function spriteTexture(row, col, gridSize = 2) {
  const sprite = atlas.clone();
  sprite.offset.set(col / gridSize, 1 - (row + 1) / gridSize);
  sprite.repeat.set(1 / gridSize, 1 / gridSize);
  return sprite;
}

mesh.material.map = spriteTexture(0, 0); // top-left sprite of a 2x2 sheet
```

## Standard Material Slots

Channel conventions worth knowing: `aoMap` reads red, `roughnessMap` reads green, `metalnessMap` reads blue (so one packed occlusion-roughness-metalness image works); `alphaMap` reads green.

```javascript
const colorMap = new THREE.TextureLoader().load("color.jpg");
const emissiveMap = new THREE.TextureLoader().load("emissive.jpg");
const normalMap = new THREE.TextureLoader().load("normal.jpg");
colorMap.colorSpace = THREE.SRGBColorSpace;
emissiveMap.colorSpace = THREE.SRGBColorSpace;
// normalMap keeps NoColorSpace, as do roughness, metalness, ao, alpha and displacement maps

const material = new THREE.MeshStandardMaterial({
  map: colorMap,
  normalMap: normalMap,
  normalScale: new THREE.Vector2(1, 1),
  emissiveMap: emissiveMap,
  emissive: 0xffffff, // non-black, so the emissive map contributes
  roughness: 1, // scalar multipliers for the matching maps
  metalness: 0,
});

material.aoMap = new THREE.TextureLoader().load("ao.jpg");
material.aoMapIntensity = 1;
```

`alphaMap` takes effect with `transparent: true` (blended) or `alphaTest` (cut out); `displacementMap` scales with `displacementScale` and `displacementBias` and wants a subdivided mesh; `normalMapType` defaults to `TangentSpaceNormalMap`, set `ObjectSpaceNormalMap` for object-space normal maps.

## Disposal and Memory

`texture.dispose()` frees the GPU upload once no other undisposed texture shares it (the atlas clones above share one); the image object remains. Dispose only textures that no other material still uses:

```javascript
function disposeMaterial(material) {
  for (const key of Object.keys(material)) {
    const value = material[key];
    if (value && value.isTexture) {
      value.dispose();
    }
  }
  material.dispose();
}
```

```javascript
console.log(renderer.info.memory.textures); // texture count currently on the GPU
console.log(renderer.capabilities.maxTextureSize); // largest supported dimension
```

Share one texture instance across materials rather than loading per mesh. Non-power-of-two sizes are fine: WebGL 2 supports mipmaps and repeat wrapping on them.

## WebGPU and TSL

```javascript
import * as THREE from "three/webgpu";
import { texture, uv } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas starts transparent

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 2);

const map = new THREE.TextureLoader().load("diffuse.jpg");
map.colorSpace = THREE.SRGBColorSpace;
map.anisotropy = renderer.getMaxAnisotropy(); // no capabilities object under WebGPU

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), new THREE.MeshBasicNodeMaterial());
mesh.material.colorNode = texture(map, uv().mul(0.5).add(0.25)); // custom UV transform in TSL: the center half, zoomed 2x
scene.add(mesh);

renderer.setAnimationLoop(animate);

function animate() {
  mesh.rotation.z += 0.01;
  renderer.render(scene, camera);
}
```

TSL sampling: `texture(map)` uses `uv()` plus the texture's own offset/repeat/rotation; an explicit node takes over the coordinates entirely. The classic slots (`map`, `normalMap`, `roughnessMap`, `aoMap`) work unchanged on node materials; `colorNode` is for custom sampling.

```javascript
import * as THREE from "three/webgpu";
import { texture, uv } from "three/tsl";
const map = new THREE.TextureLoader().load("diffuse.jpg");
map.colorSpace = THREE.SRGBColorSpace;
mesh.material.colorNode = texture(map); // uv() plus the texture's offset/repeat/rotation
mesh.material.colorNode = texture(map, uv(1)); // explicit node: samples the "uv1" attribute
```

`KTX2Loader.detectSupport(renderer)` queries device features, so call it after `await renderer.init()`. For dynamic reflections use `CubeRenderTarget` (not `WebGLCubeRenderTarget`) with the same hide-update-show CubeCamera pattern.

```javascript
import * as THREE from "three/webgpu";

const cubeRenderTarget = new THREE.CubeRenderTarget(256); // replaces WebGLCubeRenderTarget
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);
mesh.material.envMap = cubeRenderTarget.texture;
mesh.visible = false; // hide so the mesh does not reflect itself
cubeCamera.position.copy(mesh.position);
cubeCamera.update(renderer, scene);
mesh.visible = true;
```

## Common Mistakes

- `geometry.setAttribute("uv2", geometry.attributes.uv)` → `material.aoMap = aoTexture;`: aoMap samples the attribute picked by `texture.channel` (default 0, `uv`), so it needs no extra attribute and `uv2` is read only by textures whose `channel` is 2; only a separate baked unwrap goes in `uv1`, with `aoTexture.channel = 1`.
- `new RGBELoader()` → `new HDRLoader()`: renamed from RGBELoader to HDRLoader in r180; the old class still runs but logs a deprecation warning.
- `import { EXRLoader } from "three/examples/jsm/loaders/EXRLoader.js"` → `import { EXRLoader } from "three/addons/loaders/EXRLoader.js"`: the browser import maps map `three/addons/` to examples/jsm/ and define no `three/examples/` key, so the examples path fails to resolve there.
- `texture.encoding = THREE.sRGBEncoding` → `texture.colorSpace = THREE.SRGBColorSpace`: `encoding` and the `sRGBEncoding` constant were removed; `colorSpace` replaced them.
- `const texture = loader.load("color.jpg")` → `const texture = loader.load("color.jpg"); texture.colorSpace = THREE.SRGBColorSpace;`: an LDR color image left at `NoColorSpace` renders too bright and washed out.
- `texture.minFilter = THREE.LinearFilter; // no mipmaps` → `texture.generateMipmaps = false;`: mipmaps are generated whenever `generateMipmaps` is true, whatever the filter, so LinearFilter alone still uploads them.
- `texture.generateMipmaps = false; texture.wrapS = THREE.ClampToEdgeWrapping; // non-power-of-two size` → `texture.wrapS = THREE.RepeatWrapping;`: WebGL 2 supports mipmaps and repeat wrapping on non-power-of-two textures, so nothing has to be disabled for them.
- `texture.anisotropy = renderer.capabilities.getMaxAnisotropy()` → `texture.anisotropy = renderer.getMaxAnisotropy()`: WebGPURenderer has no `capabilities` object; this is the call under WebGPU.
- `material.envMapIntensity = 0.5` → `scene.environmentIntensity = 0.5`: `envMapIntensity` scales only the material's own `envMap`, not `scene.environment`.
- `scene.environment = pmrem.fromEquirectangular(hdr).texture` → `hdr.mapping = THREE.EquirectangularReflectionMapping; scene.environment = hdr;`: the renderer prefilters an equirectangular environment itself, so manual PMREM of a loaded file adds nothing.
- `scene.background = hdr` → `hdr.mapping = THREE.EquirectangularReflectionMapping; scene.background = hdr;`: without `mapping`, an equirectangular texture draws as a flat screen-space quad instead of a surround.
- `ktx2Loader.detectSupportAsync(renderer)` → `await renderer.init(); ktx2Loader.detectSupport(renderer);`: the async variant was deprecated in r181, and support detection needs the initialized device.
- `ktx2Loader.setTranscoderPath("libs/basis/")` → `const ktx2Loader = new KTX2Loader(); ktx2Loader.detectSupport(renderer);`: the transcoder loads from the package, so a path is only needed when self-hosting, pinned to three@0.186.0.
- `new THREE.WebGLMultipleRenderTargets(width, height, 2)` → `new THREE.WebGLRenderTarget(width, height, { count: 2 })`: removed in r162; the attachments are `renderTarget.textures[0..count-1]`.
- `new THREE.WebGLCubeRenderTarget(256)` → `new THREE.CubeRenderTarget(256)`: under WebGPURenderer use `CubeRenderTarget` imported from `three/webgpu`; `WebGLCubeRenderTarget` is WebGL-only.
- `loader.load(url, onLoad, onProgress, onError)` → `manager.onProgress = (url, loaded, total) => console.log(url, loaded, total)`: TextureLoader's `onProgress` parameter is unsupported and never fires.
- `mesh.material.map = texture; // after the mesh has rendered` → `mesh.material.map = texture; mesh.material.needsUpdate = true;`: WebGLRenderer builds a material's shader for the maps it has at its first render, so a `map` added later stays invisible until the shader is rebuilt, which `needsUpdate` forces.
- `normalMap.colorSpace = THREE.LinearSRGBColorSpace` → `normalMap.colorSpace = THREE.NoColorSpace`: normal, roughness, metalness, ao, alpha and displacement maps hold data, not color, so they keep the default `NoColorSpace`.
- `const texture = await new Promise((resolve, reject) => loader.load(url, resolve, undefined, reject))` → `const texture = await loader.loadAsync(url)`: `loadAsync` is that promise, built into the `Loader` base class.
- `cubeCamera.update(renderer, scene)` → `mesh.visible = false; cubeCamera.update(renderer, scene); mesh.visible = true;`: while visible, the reflective mesh is drawn into the cube map that its own `envMap` reads.

## See Also

- `threejs-materials` — material properties, blending, and physical materials beyond texture slots.
- `threejs-loaders` — loading managers, glTF models, and the loader family generally.
- `threejs-shaders` — custom GLSL texture sampling under WebGLRenderer.
- `threejs-lighting` — environment lighting and light units.
- `threejs-asset-pipeline` — encoding a model's textures to KTX2, WebP or AVIF with gltf-transform.
