---
name: threejs-fundamentals
description: "Three.js scene setup: scenes and fog, cameras, the renderer and tone mapping, the animation loop, object hierarchies, transforms, core math and cleanup, on WebGL or with WebGPU and TSL. Use when creating a scene from scratch, configuring a camera or renderer, handling resize, or arranging objects."
---

# Three.js Fundamentals

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.5, 5);

const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x3a86ff }),
);
scene.add(cube);
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 4, 5); // shines from its position toward the origin
scene.add(light);
scene.add(new THREE.AmbientLight(0xffffff, 0.4));

const timer = new THREE.Timer();
timer.connect(document); // freezes the delta while the tab is hidden

function animate() {
  timer.update(); // advance the timer first thing each frame
  cube.rotation.y += timer.getDelta(); // seconds since the previous frame
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);

window.addEventListener("resize", () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

Without a bundler, the browser loads three.js as ES modules through an import map; this is the map for the WebGLRenderer examples above:

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.186.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.0/examples/jsm/"
  }
}
</script>
```

For WebGPURenderer work, map all four keys — `three.tsl.js` imports `three/webgpu`, so the map fails without that entry:

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.186.0/build/three.webgpu.js",
    "three/webgpu": "https://cdn.jsdelivr.net/npm/three@0.186.0/build/three.webgpu.js",
    "three/tsl": "https://cdn.jsdelivr.net/npm/three@0.186.0/build/three.tsl.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.0/examples/jsm/"
  }
}
</script>
```

## Scene

```javascript
const scene = new THREE.Scene();

scene.background = new THREE.Color(0x111318); // solid color
scene.fog = new THREE.Fog(0x111318, 1, 100); // linear fog: near, far
scene.fog = new THREE.FogExp2(0x111318, 0.02); // exponential fog: density (pick one)
```

- `scene.background` also accepts a `CubeTexture`, or a `Texture` whose `mapping` is `THREE.EquirectangularReflectionMapping`; a plain unmapped texture is drawn as a flat screen-filling quad, not a skybox.
- `scene.environment` lights `MeshStandardMaterial` and `MeshPhysicalMaterial` on both renderers, and `MeshLambertMaterial` and `MeshPhongMaterial` on WebGLRenderer only. `scene.environmentIntensity` scales it — `material.envMapIntensity` scales only `material.envMap`. See threejs-loaders for HDR environments.

## Cameras

### PerspectiveCamera

```javascript
// PerspectiveCamera(fov in degrees, aspect, near, far)
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0); // rotates the camera to face the target

camera.fov = 60;
camera.updateProjectionMatrix(); // required after changing fov, aspect, near or far
```

### OrthographicCamera

```javascript
// OrthographicCamera(left, right, top, bottom, near, far); frustum size in world units
const aspect = window.innerWidth / window.innerHeight;
const frustumHeight = 10;
const camera = new THREE.OrthographicCamera(
  (-frustumHeight * aspect) / 2,
  (frustumHeight * aspect) / 2,
  frustumHeight / 2,
  -frustumHeight / 2,
  0.1,
  1000,
);
// on resize: recompute left and right from the new aspect, then call camera.updateProjectionMatrix()
```

### ArrayCamera

```javascript
// Renders the scene once per sub-camera, each into its own viewport.
// Viewport units under WebGLRenderer are drawing-buffer pixels (CSS size × pixel ratio);
// WebGPURenderer multiplies CSS pixels by the pixel ratio itself.
const width = window.innerWidth;
const height = window.innerHeight;
const dpr = renderer.getPixelRatio();
const cameras = [];
for (let i = 0; i < 4; i++) {
  const subCamera = new THREE.PerspectiveCamera(75, width / height, 0.1, 100);
  subCamera.viewport = new THREE.Vector4(
    ((i % 2) * width * dpr) / 2,
    (Math.floor(i / 2) * height * dpr) / 2,
    (width * dpr) / 2,
    (height * dpr) / 2,
  );
  subCamera.position.set(-2 + (i % 2) * 4, 1.5, 6);
  subCamera.lookAt(0, 0, 0);
  subCamera.updateMatrixWorld(); // sub-cameras are not scene-graph children; update them when they move
  cameras.push(subCamera);
}
const arrayCamera = new THREE.ArrayCamera(cameras);
arrayCamera.position.set(0, 1.5, 6); // WebGLRenderer culls every viewport with this camera's frustum (fov 50, aspect 1 by default); WebGPURenderer uses each sub-camera's frustum
renderer.render(scene, arrayCamera);
```

### CubeCamera

```javascript
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256);
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);

const chrome = new THREE.Mesh(
  new THREE.SphereGeometry(1, 32, 16),
  new THREE.MeshStandardMaterial({ color: 0xffffff, metalness: 1, roughness: 0 }),
);
chrome.material.envMap = cubeRenderTarget.texture;
scene.add(chrome);

// every frame: hide the reflective mesh, or it is drawn into the map it samples
chrome.visible = false;
cubeCamera.position.copy(chrome.position);
cubeCamera.update(renderer, scene);
chrome.visible = true;
```

With WebGPURenderer, build the target with `CubeRenderTarget` from `three/webgpu` instead of `WebGLCubeRenderTarget`.

## WebGLRenderer

```javascript
const renderer = new THREE.WebGLRenderer({
  canvas: document.querySelector("#canvas"), // reuse an existing canvas; omit to create one
  antialias: true, // MSAA
  alpha: true, // let the page background show through the canvas
  stencil: true, // stencil is off by default; pass this for stencil techniques
  powerPreference: "high-performance",
  preserveDrawingBuffer: true, // keep the drawing buffer for readPixels / screenshots
});

renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // cap the fill rate on high-DPI displays
renderer.setSize(window.innerWidth, window.innerHeight);

renderer.toneMapping = THREE.ACESFilmicToneMapping; // or AgXToneMapping, NeutralToneMapping; NoToneMapping is the default
renderer.toneMappingExposure = 1.0;
// renderer.outputColorSpace is SRGBColorSpace by default; leave it alone

renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // the default, already soft; VSMShadowMap and BasicShadowMap are the alternatives

renderer.setClearColor(0x000000, 0); // clear alpha 0 keeps the alpha: true canvas transparent
renderer.render(scene, camera);
```

## Objects

### Object3D

Base class of Mesh, Group, Light and Camera.

```javascript
const obj = new THREE.Object3D();
const child = new THREE.Object3D();

obj.position.set(1, 2, 3); // translation
obj.rotation.set(0, Math.PI / 4, 0); // Euler angles in radians; kept in sync with quaternion
obj.quaternion.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 4); // the same rotation as a quaternion
obj.scale.set(1, 1, 1);

obj.getWorldPosition(new THREE.Vector3()); // world-space queries take a target vector to write into
obj.getWorldQuaternion(new THREE.Quaternion());
obj.getWorldDirection(new THREE.Vector3()); // the +Z axis in world space; cameras return their view direction (−Z)

obj.add(child); // reparents child; also removes it from its previous parent
obj.attach(child); // reparents and keeps child's world transform; add() keeps the local one
obj.remove(child);
obj.visible = false; // hides the whole subtree

obj.layers.enable(1); // ADD layer 1; set(1) would replace the mask and leave layer 0
obj.layers.disable(1);
obj.layers.test(camera.layers); // true when the object shares a layer with the camera

obj.traverse((descendant) => {
  if (descendant.isMesh) descendant.castShadow = true; // visits obj itself, then all descendants
});

// matrix updates run automatically while matrixAutoUpdate is true (the default)
obj.matrixAutoUpdate = false;
obj.updateMatrix(); // composes obj.matrix and flags the world matrix dirty
// when writing obj.matrix directly instead, flag the world matrix yourself:
obj.matrixWorldNeedsUpdate = true;
```

### Group and Mesh

```javascript
const group = new THREE.Group(); // an Object3D that exists to hold children
const ball = new THREE.Mesh(new THREE.SphereGeometry(0.5), new THREE.MeshStandardMaterial());
group.add(ball);
group.rotation.y = Math.PI / 4; // transforms the whole subtree
scene.add(group);
```

```javascript
const mesh = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshStandardMaterial());
scene.add(mesh);

// one material per geometry group; BoxGeometry has six (+x, −x, +y, −y, +z, −z)
// a group whose materialIndex has no material is not drawn
const faceColors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];
const dice = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  faceColors.map((color) => new THREE.MeshStandardMaterial({ color })),
);

mesh.castShadow = true; // this object casts shadows
mesh.receiveShadow = true; // this object shows shadows
mesh.frustumCulled = true; // default: skipped when its bounding sphere leaves the camera frustum
mesh.renderOrder = 10; // renders later within the same transparency bucket
```

## Coordinate System

Right-handed, +Y up: +X right, +Y up, +Z toward the viewer. A camera at the origin with no rotation looks down −Z.

```javascript
const axesHelper = new THREE.AxesHelper(5); // red X, green Y, blue Z; 5 = axis length
scene.add(axesHelper);
```

## Math Utilities

### Vector3

```javascript
const a = new THREE.Vector3(1, 0, 0);
const b = new THREE.Vector3(0, 1, 0);
const c = a.clone(); // new vector with the same values

a.copy(b); // a becomes b
a.add(b); // in place: a = a + b
a.sub(b);
a.multiplyScalar(2);
a.divideScalar(2);
a.lerp(b, 0.5); // move halfway toward b
a.normalize(); // unit length
a.negate();
a.cross(b); // cross product, written into a (right-hand rule)

const len = a.length();
const lenSq = a.lengthSq(); // compare lengths without a square root
const dist = a.distanceTo(b);
const dot = a.dot(b); // > 0 same half-space, < 0 opposite, 0 perpendicular
const angle = a.angleTo(b); // radians

a.applyQuaternion(new THREE.Quaternion());
a.applyMatrix4(new THREE.Matrix4());
a.project(camera); // world position to normalized device coordinates
a.unproject(camera); // normalized device coordinates back to world
```

### Matrix4, Quaternion, Euler

```javascript
const m = new THREE.Matrix4();
m.identity();
m.makeTranslation(1, 2, 3);
m.makeRotationY(Math.PI / 4);
m.makeScale(2, 2, 2);

const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale = new THREE.Vector3(1, 1, 1);
m.compose(position, quaternion, scale); // build from transform components
m.decompose(position, quaternion, scale); // read them back

m.multiply(new THREE.Matrix4()); // m = m * other (other applied first)
m.premultiply(new THREE.Matrix4()); // m = other * m
m.invert();
m.lookAt(new THREE.Vector3(0, 0, 1), new THREE.Vector3(), new THREE.Vector3(0, 1, 0)); // eye, target, up

const q = new THREE.Quaternion();
q.setFromEuler(new THREE.Euler(0, Math.PI / 2, 0, "YXZ")); // Euler order matters, XYZ is the default
q.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 4);
q.setFromRotationMatrix(m);
q.slerp(new THREE.Quaternion(), 0.5); // spherical interpolation, written into q
q.invert();
```

### Color

```javascript
const color = new THREE.Color(0xff0000); // hex number
const sky = new THREE.Color("#87ceeb"); // also color names and "rgb(255, 0, 0)"

color.setHex(0x0000ff);
color.setRGB(1, 0.5, 0); // 0–1 in the linear working color space; hex and CSS strings are sRGB
color.setRGB(1, 128 / 255, 0, THREE.SRGBColorSpace); // sRGB components, the same color as 0xff8000
color.setHSL(0.5, 1, 0.5); // hue, saturation, lightness in 0–1, also linear unless a color space is passed
color.lerp(sky, 0.5); // interpolate toward the target, written into color
color.multiplyScalar(2); // scale the components
```

### MathUtils

```javascript
const clamped = THREE.MathUtils.clamp(15, 1, 10); // 10
const mixed = THREE.MathUtils.lerp(0, 10, 0.5); // 5
const mapped = THREE.MathUtils.mapLinear(0.5, 0, 1, 0, 100); // 50
const radians = THREE.MathUtils.degToRad(90);
const degrees = THREE.MathUtils.radToDeg(Math.PI);
const random = THREE.MathUtils.randFloat(1, 10);
const eased = THREE.MathUtils.smoothstep(0.5, 0, 1); // eased 0–1 curve
```

## Loading Manager

```javascript
const manager = new THREE.LoadingManager();
manager.onStart = (url, loaded, total) => console.log(`started ${url}`);
manager.onProgress = (url, loaded, total) => console.log(`${loaded}/${total} ${url}`);
manager.onLoad = () => console.log("all items loaded");
manager.onError = (url) => console.error(`failed ${url}`);

const loader = new THREE.TextureLoader(manager); // any loader takes a manager
```

## Cleanup

```javascript
scene.remove(mesh);
// dispose geometry, materials and textures only when nothing else still uses them
mesh.geometry.dispose();
const materials = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
for (const material of materials) {
  for (const value of Object.values(material)) {
    if (value?.isTexture) value.dispose(); // material.dispose() leaves its textures alone
  }
  material.dispose();
}
mesh.dispose(); // frees renderer state kept for this object only

renderer.setAnimationLoop(null); // stop the loop
renderer.dispose(); // frees the renderer's GPU resources; the WebGL context stays alive
renderer.forceContextLoss(); // also release the context when the canvas is discarded
```

`Object3D.dispose()` releases per-object renderer state (WebGPURenderer bindings, WebGLRenderer instanced-mesh buffers); it does not visit children or free geometry, material or textures. To tear down a subtree, `traverse` it and dispose each mesh as above. A custom Object3D subclass that overrides `dispose()` calls `super.dispose()`. `timer.dispose()` disconnects a connected timer.

## Performance

- Frustum culling tests a bounding sphere, not the bounding box: `geometry.boundingSphere` for a plain mesh, the object's own `boundingSphere` for InstancedMesh, SkinnedMesh and BatchedMesh. Both are computed once, so after editing positions call `geometry.computeBoundingSphere()`, and after moving instances or bones far from where they started call `object.computeBoundingSphere()`, or visible objects get culled.
- Merge static geometry that shares a material into one mesh, and use `InstancedMesh` for many copies of one object (threejs-geometry).
- Switch meshes by distance with `LOD`; the renderer calls `update(camera)` for you each frame.
- On WebGLRenderer, `renderer.info.render.calls` counts the draw calls of the last `render()` call (with several passes per frame, set `renderer.info.autoReset = false` and call `renderer.info.reset()` once per frame), and `renderer.info.memory` the live geometries and textures; check them after merging meshes or disposing objects.

```javascript
import * as BufferGeometryUtils from "three/addons/utils/BufferGeometryUtils.js";

// inputs must carry the same set of attributes
const merged = BufferGeometryUtils.mergeGeometries([
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.SphereGeometry(1, 32, 16),
]);

const lod = new THREE.LOD();
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 32, 16)), 0); // distance 0: full detail
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 12, 8)), 20);
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 6, 4)), 60);
scene.add(lod);
```

## WebGPU and TSL

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // must complete before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318); // the WebGPU canvas is transparent by default

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.5, 5);

const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardNodeMaterial({ color: 0x3a86ff }),
);
scene.add(cube);
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 4, 5); // shines from its position toward the origin
scene.add(light);
scene.add(new THREE.AmbientLight(0xffffff, 0.4));

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  cube.rotation.y += timer.getDelta();
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

- `renderer.render()` throws until `await renderer.init()` has completed. `setAnimationLoop` would run the init too, but await `init()` right after construction, before setup code touches the renderer: the WebGL 2 fallback replaces `renderer.backend` during `init()`.
- WebGPURenderer falls back to WebGL 2 by itself when WebGPU is unavailable; `new THREE.WebGPURenderer({ forceWebGL: true })` forces the fallback, and after `await renderer.init()`, `renderer.backend.isWebGPUBackend` reports which backend runs.
- Node materials and TSL target WebGPURenderer (either backend). WebGLRenderer renders them only through the `WebGLNodesHandler` addon (`three/addons/tsl/WebGLNodesHandler.js`, passed to `renderer.setNodesHandler()`), a migration aid without VSM shadows, MRT, transmission or the WebGPU post-processing stack. ShaderMaterial and `onBeforeCompile` are WebGLRenderer-only. Custom shading starts in threejs-shaders.
- Built-in materials such as `MeshStandardMaterial` are converted to their node versions automatically; use the `*NodeMaterial` classes when you set node slots.
- There is no `renderer.capabilities` object here: anisotropy comes from `renderer.getMaxAnisotropy()`.
- GPU readback stays asynchronous: `await renderer.getArrayBufferAsync(attribute)`.

## Common Mistakes

- `renderer.shadowMap.type = THREE.PCFSoftShadowMap` → `renderer.shadowMap.type = THREE.PCFShadowMap`: deprecated in r182; both renderers now warn and fall back to PCFShadowMap, which is already soft.
- `const clock = new THREE.Clock()` → `const timer = new THREE.Timer(); timer.connect(document);`: Clock is deprecated in r183 and warns on construction; call `timer.update()` once per frame before `timer.getDelta()` or `timer.getElapsed()`.
- `requestAnimationFrame(animate)` → `renderer.setAnimationLoop(animate)`: both renderers' docs advise it over a manual loop, WebXR frames run only through it, and on WebGPURenderer `render()` throws before the async init that `setAnimationLoop` performs.
- `subCamera.viewport = new THREE.Vector4(0, 0, 0.5, 0.5)` → `subCamera.viewport = new THREE.Vector4(x * dpr, y * dpr, width * dpr, height * dpr)`: ArrayCamera viewports are drawing-buffer pixels under WebGLRenderer (WebGPURenderer scales CSS pixels by the pixel ratio itself), so fractions draw nothing.
- `cubeCamera.update(renderer, scene)` → `mesh.visible = false; cubeCamera.update(renderer, scene); mesh.visible = true;`: a visible reflective mesh is rendered into the very cube map its material samples.
- `scene.background = texture` → `texture.mapping = THREE.EquirectangularReflectionMapping; scene.background = texture`: without an equirect or cube mapping the texture draws as a flat screen-filling quad, not a skybox.
- `import { mergeGeometries } from "three/examples/jsm/utils/BufferGeometryUtils.js"` → `import * as BufferGeometryUtils from "three/addons/utils/BufferGeometryUtils.js"`: the `three/examples/jsm/` specifier does not resolve under the standard import map, which maps `three/addons/` instead.
- `<script src="three.min.js"></script>` → `<script type="module">import * as THREE from "three";</script>`: `build/three.js` and `build/three.min.js` were removed in r161, so in the browser three.js loads as an ES module behind an import map.
- `mesh.layers.set(1)` → `mesh.layers.enable(1)`: `set` replaces the layer mask and takes the mesh off layer 0, where the default camera and lights live.
- `renderer.outputEncoding = THREE.sRGBEncoding` → `renderer.outputColorSpace = THREE.SRGBColorSpace`: the encoding API no longer exists, and the color-space property already defaults to `SRGBColorSpace`, so the assignment is usually unnecessary.
- `renderer.physicallyCorrectLights = true` → `const bulb = new THREE.PointLight(0xffffff, 100)`: physically based light units are always on and the `physicallyCorrectLights` and `useLegacyLights` switches no longer exist, so delete the line and give point and spot lights candela intensities that suit their distance (threejs-lighting).
- `material.envMapIntensity = 0.5` → `scene.environmentIntensity = 0.5`: `envMapIntensity` scales only the material's own `envMap`, not `scene.environment`.
- `mesh.dispose()` → `mesh.dispose(); mesh.geometry.dispose(); for (const value of Object.values(mesh.material)) if (value?.isTexture) value.dispose(); mesh.material.dispose();`: the object-level dispose frees only per-object renderer state, never the geometry, material or textures, and material.dispose() leaves the textures alone too.
- `import { WebGLCubeRenderTarget } from "three/webgpu"` → `import { CubeRenderTarget } from "three/webgpu"`: `three/webgpu` does not export WebGLCubeRenderTarget, which does not work with WebGPURenderer; keep it for WebGLRenderer.
- `await renderer.renderAsync(scene, camera)` → `await renderer.init(); renderer.render(scene, camera)`: the async render methods are deprecated in r181; initialize once, then render synchronously in the loop.

## See Also

- `threejs-geometry` — creating and editing BufferGeometry, instancing, merging, bounding volumes.
- `threejs-materials` — choosing and configuring materials and their texture slots.
- `threejs-textures` — loading textures, color spaces, UV channels.
- `threejs-lighting` — light types, units, and shadow configuration.
- `threejs-animation` — the AnimationMixer and clip system for keyframed motion.
- `threejs-interaction` — raycasting, pointer events, and camera controls.
- `threejs-loaders` — glTF and other file loaders, async loading patterns.
- `threejs-shaders` — ShaderMaterial and GLSL on WebGLRenderer, TSL node materials on WebGPURenderer.
- `threejs-postprocessing` — EffectComposer passes and the WebGPU render pipeline.
- `threejs-performance` — measuring frame cost and cutting draw calls, memory and fill rate.
- `threejs-react-three-fiber` — the same scene graph written as React components.
