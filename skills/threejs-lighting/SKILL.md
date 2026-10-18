---
name: threejs-lighting
description: "Light types and physical units, shadow maps, HDR environment lighting, light probes, and lighting performance, on WebGL or with WebGPU and TSL. Use when adding or tuning lights, fixing shadow acne, setting up image-based lighting, or dimming an environment map."
---

# Three.js Lighting

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(4, 3, 8);

// A single directional light: intensity is a plain multiplier, no distance falloff
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({ color: 0x808080 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x3a76c4 })
);
scene.add(mesh);

// Shadows need three switches: renderer, light, and the meshes that cast and receive
renderer.shadowMap.enabled = true;
dirLight.castShadow = true;
mesh.castShadow = true;
floor.receiveShadow = true;

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Light Types and Units

| Light | Shadows | Intensity | Notes |
|---|---|---|---|
| AmbientLight | no | multiplier | Flat fill from every direction |
| HemisphereLight | no | multiplier | Sky color from above, ground color from below |
| DirectionalLight | yes | multiplier | Parallel rays (sun); shines at its target |
| SunLight (addon) | yes, cascaded | multiplier | Shadow frustum fitted to the view automatically |
| PointLight | yes, six faces | candela | Inverse-square falloff (decay 2) |
| SpotLight | yes | candela | Cone; inverse-square falloff (decay 2) |
| RectAreaLight | no | nits | Only MeshStandardMaterial and MeshPhysicalMaterial |
| LightProbe | no | multiplier | Spherical-harmonics ambient, usually baked |

PointLight and SpotLight intensity is candela with inverse-square falloff (`decay` 2 by default): a 1-candela light 5 units away contributes 1/25, so useful values depend on distance and typically run to the hundreds (the official docs' examples use 80 to 1000). `light.power` sets the same thing in lumens. `light.distance` is a smooth cutoff range (0 means no limit). Directional, ambient and hemisphere intensities are plain multipliers with no unit. RectAreaLight intensity is nits (candela per square metre).

## Ambient and Hemisphere

```javascript
// AmbientLight(color, intensity) — a flat multiplier on every lit material
const ambient = new THREE.AmbientLight(0x404040, 0.5);
scene.add(ambient);

// HemisphereLight(skyColor, groundColor, intensity)
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b6f47, 0.6);
scene.add(hemi);
```

## DirectionalLight

```javascript
// DirectionalLight(color, intensity): parallel rays from position towards target; the distance only matters to the shadow camera
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);

// Shines from its position towards target; add the target to the scene when you move it
dirLight.target.position.set(0, 0, 0);
scene.add(dirLight.target);
scene.add(dirLight);
```

### SunLight: automatic cascaded shadows

`SunLight` (`three/addons/lights/SunLight.js`) is a parallel light with two cascaded shadow maps, suited to large scenes. It has no target: it shines from its position towards the origin, straight down by default. With WebGLRenderer it works out of the box; under WebGPURenderer register it first (see WebGPU and TSL).

```javascript
import { SunLight } from "three/addons/lights/SunLight.js";

const sun = new SunLight(0xfff2e3, 3);
sun.position.set(100, 160, 60);
sun.castShadow = true;
scene.add(sun);
```

The cascades are fitted automatically to slices of the camera's view frustum, up to `sun.shadow.camera.far` or the view camera's far plane, whichever is smaller. `shadow.camera.left/right/top/bottom` are ignored; the default `mapSize` is 1024 × 1024 per cascade. There is no shadow frustum to hand-tighten.

## PointLight and SpotLight

```javascript
// PointLight(color, intensity in candela, distance, decay)
const bulb = new THREE.PointLight(0xffdcb0, 100, 20, 2);
bulb.position.set(0, 4, 0);
scene.add(bulb);

// SpotLight(color, intensity in candela, distance, angle, penumbra, decay)
const deskLamp = new THREE.SpotLight(0xffffff, 250, 30, Math.PI / 6, 0.4, 2);
deskLamp.position.set(2, 5, 0);
deskLamp.target.position.set(0, 0, 0);
scene.add(deskLamp.target);
scene.add(deskLamp);

// Prefer lumens: power writes the candela intensity for you
bulb.power = 800;
```

`angle` is the cone half-angle in radians with an upper bound of Math.PI / 2; `penumbra` (0 to 1) softens the cone edge.

## RectAreaLight

A rectangular emitter for soft window- or strip-style lighting. It lights only MeshStandardMaterial and MeshPhysicalMaterial, and it cannot cast shadows.

```javascript
import { RectAreaLightHelper } from "three/addons/helpers/RectAreaLightHelper.js";
import { RectAreaLightUniformsLib } from "three/addons/lights/RectAreaLightUniformsLib.js";

// WebGLRenderer: add the LTC lookup textures to the renderer's uniforms, once
RectAreaLightUniformsLib.init();

// RectAreaLight(color, intensity in nits, width, height)
const windowLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
windowLight.position.set(-5, 5, 0);
windowLight.lookAt(0, 0, 0); // emits from its front face
scene.add(windowLight);

// The helper must be added as a child of the light
const helper = new RectAreaLightHelper(windowLight);
windowLight.add(helper);
```

Under WebGPURenderer, register the LTC textures with `RectAreaLightTexturesLib` instead (see WebGPU and TSL).

## Shadows

```javascript
// 1. Renderer: shadow maps are off by default; PCFShadowMap is the soft filtered default,
// VSMShadowMap blurs the map itself, BasicShadowMap is hard-edged
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap;

// 2. Light
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// 3. Objects
mesh.castShadow = true;
mesh.receiveShadow = true;
```

`shadow.radius` above 1 widens the PCF filter and the VSM blur; it has no effect with BasicShadowMap. VSM also exposes `shadow.blurSamples` (default 8), and PointLight shadows do not support VSM at all — WebGLRenderer warns and skips the light's shadow. PCFSoftShadowMap is deprecated: it warns and falls back to PCFShadowMap, which is already soft. With VSMShadowMap, every mesh that receives shadows also casts them.

### Tuning a directional shadow

```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5); // every caster must lie between near and far from here
dirLight.castShadow = true;

// The shadow camera sits at the light's position and looks at its target: near and far
// are distances from the light; left/right/top/bottom frame the area around the target
const shadowCam = dirLight.shadow.camera;
shadowCam.left = -10;
shadowCam.right = 10;
shadowCam.top = 10;
shadowCam.bottom = -10;
shadowCam.near = 0.5;
shadowCam.far = 50;
shadowCam.updateProjectionMatrix(); // needed whenever these change after the first render

dirLight.shadow.mapSize.set(2048, 2048); // powers of two

// Acne: normalBias offsets the lookup along the surface normal (light grazing a surface);
// bias offsets the depth compared against the map — start tiny, negative fights acne
dirLight.shadow.normalBias = 0.02;
dirLight.shadow.bias = -0.0001;

// Lighten shadows without adding fill lights: 0 removes, 1 is full darkness
dirLight.shadow.intensity = 0.7;
```

### Point and spot shadow cameras are derived

The renderer rebuilds these cameras on every shadow pass: a spot light's shadow fov is `2 × angle × shadow.focus` (in degrees) and its far comes from `light.distance`; a point light's far also comes from `light.distance`. The fov is always overwritten, and a manual `far` survives only while `distance` is 0. Control the cone with `angle` and `shadow.focus`, the range with `distance`.

```javascript
const deskLamp = new THREE.SpotLight(0xffffff, 250, 30);
deskLamp.castShadow = true;
deskLamp.shadow.mapSize.set(1024, 1024);
deskLamp.shadow.focus = 0.9; // shadow fov as a fraction of the light's cone
deskLamp.shadow.bias = -0.0001;
```

Visualize a directional or spot light's shadow camera with `new THREE.CameraHelper(light.shadow.camera)` added to the scene, and call the helper's `update()` after changing the camera.

### Contact shadows (fake, cheap)

A shadow map per light is overkill for a grounded look; a radial-gradient blob costs one textured quad.

```javascript
// A soft blob under the object: no shadow map, works under both renderers
const size = 128;
const canvas = document.createElement("canvas");
canvas.width = canvas.height = size;
const ctx = canvas.getContext("2d");
const gradient = ctx.createRadialGradient(size / 2, size / 2, 0, size / 2, size / 2, size / 2);
gradient.addColorStop(0, "rgba(0,0,0,0.6)");
gradient.addColorStop(1, "rgba(0,0,0,0)");
ctx.fillStyle = gradient;
ctx.fillRect(0, 0, size, size);

const shadowTexture = new THREE.CanvasTexture(canvas);
shadowTexture.colorSpace = THREE.SRGBColorSpace; // a canvas used as a color map is sRGB

const blob = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  new THREE.MeshBasicMaterial({ map: shadowTexture, transparent: true, depthWrite: false })
);
blob.rotation.x = -Math.PI / 2;
blob.position.y = 0.001; // just above the floor to avoid z-fighting
scene.add(blob);
```

For a blob that follows the object's silhouette, render depth from below into a render target and blur it — the official `webgl_shadow_contact` example does this with `HorizontalBlurShader` and `VerticalBlurShader` from `three/addons/shaders/`.

## Light Helpers

```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b6f47, 0.6);
const bulb = new THREE.PointLight(0xffdcb0, 100, 20);
const deskLamp = new THREE.SpotLight(0xffffff, 250, 30);
scene.add(dirLight, hemi, bulb, deskLamp);

// Built-in helpers are objects: add them to the scene
const dirHelper = new THREE.DirectionalLightHelper(dirLight, 5);
const pointHelper = new THREE.PointLightHelper(bulb, 0.5);
const spotHelper = new THREE.SpotLightHelper(deskLamp);
const hemiHelper = new THREE.HemisphereLightHelper(hemi, 5);
scene.add(dirHelper, pointHelper, spotHelper, hemiHelper);
```

Every light helper copies the light's color only in `update()`, and DirectionalLightHelper, SpotLightHelper and HemisphereLightHelper also re-aim only there: call `update()` after recoloring a light, and after moving or re-aiming one of those three.

## Environment Lighting (IBL)

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

// Both renderers prefilter an equirect environment themselves: assign it directly
const hdr = await new HDRLoader().loadAsync("sunset.hdr");
hdr.mapping = THREE.EquirectangularReflectionMapping;

scene.environment = hdr; // lights the scene
scene.background = hdr; // optional: also draw it behind the scene
scene.environmentIntensity = 1; // scales the lighting contribution only
scene.environmentRotation.set(0, Math.PI / 4, 0); // rotates the lighting
scene.backgroundBlurriness = 0; // blurs the background, not the lighting
scene.backgroundIntensity = 1;
scene.backgroundRotation.set(0, 0, 0); // rotates the background independently
```

Both renderers light MeshStandardMaterial and MeshPhysicalMaterial from `scene.environment`; WebGLRenderer also lights MeshLambertMaterial and MeshPhongMaterial, WebGPURenderer does not. Keep `PMREMGenerator` for environments generated from a scene; on an equirect file it only repeats the prefiltering the renderer already does:

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const pmrem = new THREE.PMREMGenerator(renderer);
const room = new RoomEnvironment();
scene.environment = pmrem.fromScene(room, 0.04).texture;
room.dispose(); // frees the room's geometries and materials
pmrem.dispose();
```

A cube texture works as an environment without any mapping line:

```javascript
const cubeEnv = await new THREE.CubeTextureLoader().loadAsync([
  "px.jpg",
  "nx.jpg",
  "py.jpg",
  "ny.jpg",
  "pz.jpg",
  "nz.jpg",
]);

scene.environment = cubeEnv;
scene.background = cubeEnv;
```

## Light Probes

```javascript
import { LightProbeGenerator } from "three/addons/lights/LightProbeGenerator.js";

// Bake ambient spherical harmonics from what a cube camera sees
const cubeCamera = new THREE.CubeCamera(0.1, 100, new THREE.WebGLCubeRenderTarget(256));
cubeCamera.update(renderer, scene);

const lightProbe = await LightProbeGenerator.fromCubeRenderTarget(renderer, cubeCamera.renderTarget);
scene.add(lightProbe);
```

`LightProbeGenerator.fromCubeTexture(cubeTexture)` derives one synchronously from an already-loaded cube texture. For position-dependent ambient, bake a 3D grid instead: `LightProbeGridWebGL` with WebGLRenderer, `LightProbeGrid` with WebGPURenderer (`three/addons/lighting/`). Under WebGPURenderer, build the cube render target with `CubeRenderTarget` from `three/webgpu`.

```javascript
import { LightProbeGridWebGL } from "three/addons/lighting/LightProbeGridWebGL.js";

// A 10 × 4 × 10 volume centered on its position, size + 1 probes per axis by default
const probeGrid = new LightProbeGridWebGL(10, 4, 10);
probeGrid.position.set(0, 2, 0);
scene.add(probeGrid);
probeGrid.bake(renderer, scene); // bake again when the static lighting changes
```

## Common Setups

```javascript
// Three-point lighting: key, fill, rim
const keyLight = new THREE.DirectionalLight(0xffffff, 2);
keyLight.position.set(5, 6, 4);

const fillLight = new THREE.DirectionalLight(0xffffff, 0.5);
fillLight.position.set(-5, 3, 5);

const backLight = new THREE.DirectionalLight(0xffffff, 1);
backLight.position.set(-4, 5, -6);

scene.add(keyLight, fillLight, backLight);
```

```javascript
import { SunLight } from "three/addons/lights/SunLight.js";

// Outdoor daylight: sun with cascaded shadows plus a sky gradient
const sun = new SunLight(0xfff2e3, 3);
sun.position.set(100, 160, 60);
sun.castShadow = true;

const sky = new THREE.HemisphereLight(0x87ceeb, 0x8b6f47, 0.5);

scene.add(sun, sky);
```

## Animating Lights

```javascript
// Timer drives per-frame time: update() first, then read it
const timer = new THREE.Timer();
timer.connect(document); // avoids huge deltas after tab switches

const lamp = new THREE.PointLight(0xffd0a0, 100, 20);
scene.add(lamp);

function animate() {
  timer.update();

  const t = timer.getElapsed();
  lamp.position.set(Math.cos(t) * 4, 3, Math.sin(t) * 4);
  lamp.intensity = 100 + Math.sin(t * 2) * 30;

  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Performance

- Both renderers compile the set of lights into every lit material: adding or removing a light (an AmbientLight only under WebGPURenderer), or toggling `castShadow`, recompiles them. Remove lights you never use; switch a light on and off at runtime through its `intensity`. Under WebGPURenderer, `DynamicLighting` keeps materials compiled when ambient, hemisphere, directional, point or spot lights come and go, as long as they cast no shadow and no spot light has a `map`, and `ClusteredLighting` scales to many point lights (see WebGPU and TSL).
- Every shadow-casting light adds depth passes (six per point light). Leave `castShadow` off where nothing needs the shadow.
- Static scenes: render a light's shadow on demand with `light.shadow.autoUpdate = false; light.shadow.needsUpdate = true;` and set `light.shadow.needsUpdate = true` again after something moves (both renderers; WebGLRenderer also has the global `renderer.shadowMap.autoUpdate`).
- Keep directional shadow frustums tight, or move to SunLight, which fits them to the view automatically.
- Lights are not filtered per mesh by layers: a light's layers are tested only against the camera's. To light selected meshes only, use `material.lightsNode` under WebGPURenderer.

## WebGPU and TSL

Light classes, shadow settings and the scene's environment properties above carry over to WebGPURenderer, except the WebGL-only pieces this section replaces: RectAreaLightUniformsLib, WebGLCubeRenderTarget (use `CubeRenderTarget`), LightProbeGridWebGL (use `LightProbeGrid`) and the global `renderer.shadowMap.autoUpdate`. SunLight must be registered, and `scene.environment` does not light Lambert or Phong materials here.

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x101010); // the WebGPU canvas is transparent by default

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(4, 3, 8);

renderer.shadowMap.enabled = true;
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.castShadow = true;
scene.add(dirLight);

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

```javascript
import * as THREE from "three/webgpu";
import { RectAreaLightTexturesLib } from "three/addons/lights/RectAreaLightTexturesLib.js";
import { SunLight } from "three/addons/lights/SunLight.js";
import { SunLightNode } from "three/addons/lights/SunLightNode.js";
import { ClusteredLighting } from "three/addons/lighting/ClusteredLighting.js";

// RectAreaLight: register the LTC textures once (replaces RectAreaLightUniformsLib)
THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init());
// SunLight: register its node implementation once
renderer.library.addLight(SunLightNode, SunLight);
// Many point lights: clustered Forward+ shading (DynamicLighting instead avoids recompiles when unshadowed lights come and go)
renderer.lighting = new ClusteredLighting();
```

```javascript
import * as THREE from "three/webgpu";
import { lights } from "three/tsl";

const keyLight = new THREE.PointLight(0x66aaff, 120, 20);
scene.add(keyLight);

// A material lit only by keyLight, ignoring every other light in the scene
const material = new THREE.MeshStandardNodeMaterial();
material.lightsNode = lights([keyLight]);
```

Colored shadows: with `renderer.shadowMap.transmitted = true`, casters write color and alpha into the shadow map, from `material.castShadowNode` (a vec4) or the alpha of `map` and `colorNode`. `material.receivedShadowNode` (an Fn of the shadow value) changes how a receiver applies its shadows and works without the flag.

## Common Mistakes

- `light.layers.set(1);` → `material.lightsNode = lights([keyLight, fillLight]);`: lights are not filtered per mesh by layers — a light's layers are tested against the camera's, so this switches the light off for the whole scene; per-material selection needs a node material, under WebGPURenderer or under WebGLRenderer through the WebGLNodesHandler addon.
- `new THREE.PointLight(0xffffff, 1, 100);` → `new THREE.PointLight(0xffffff, 100, 20);`: point and spot intensity is candela with inverse-square falloff, so 1 contributes almost nothing at range; typical values run to the hundreds.
- `renderer.useLegacyLights = true;` → `const bulb = new THREE.PointLight(0xffffff, 100, 20);`: legacy light units no longer exist and neither useLegacyLights nor physicallyCorrectLights does anything, so retune point and spot lights in candela instead.
- `renderer.shadowMap.type = THREE.PCFSoftShadowMap;` → `renderer.shadowMap.type = THREE.PCFShadowMap;`: PCFSoftShadowMap was deprecated in r182; both renderers warn and fall back to PCFShadowMap, which is already soft.
- `lightProbe.copy(LightProbeGenerator.fromCubeRenderTarget(renderer, rt));` → `lightProbe.copy(await LightProbeGenerator.fromCubeRenderTarget(renderer, rt));`: the call is async and returns a Promise, which copy() cannot read.
- `new ContactShadows({ opacity: 0.5 });` → `new THREE.Mesh(new THREE.PlaneGeometry(2, 2), new THREE.MeshBasicMaterial({ transparent: true, map: blobTexture }));`: three.js has no ContactShadows class (it is a drei component); fake the contact shadow with a radial-gradient plane.
- `scene.environment = pmremGenerator.fromEquirectangular(texture).texture;` → `texture.mapping = THREE.EquirectangularReflectionMapping; scene.environment = texture;`: the renderer prefilters an equirect environment itself, so manual PMREM adds no quality; keep PMREMGenerator for fromScene.
- `scene.background = hdr;` → `hdr.mapping = THREE.EquirectangularReflectionMapping; scene.background = hdr;`: HDRLoader returns a texture with UV mapping, so without the line the background draws as a flat screen-space image.
- `import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader.js";` → `import { HDRLoader } from "three/addons/loaders/HDRLoader.js";`: RGBELoader was renamed HDRLoader in r180, and addons must be imported from three/addons/ for the import map to resolve.
- `spotLight.shadow.camera.fov = 30;` → `spotLight.shadow.focus = 0.9;`: the shadow camera's fov is recomputed each pass as 2 × angle × focus in degrees, so control it through the light's angle and focus.
- `const lamp = new THREE.PointLight(0xffffff, 100, 20); lamp.shadow.camera.far = 50;` → `const lamp = new THREE.PointLight(0xffffff, 100, 50);`: a non-zero distance overwrites the shadow camera's far on every shadow pass, so set the shadow range through distance; a manual far is kept only while distance is 0.
- `material.envMapIntensity = 0.5;` → `scene.environmentIntensity = 0.5;`: envMapIntensity scales only a material's own envMap, not scene.environment.
- `const clock = new THREE.Clock();` → `const timer = new THREE.Timer(); timer.connect(document);`: Clock was deprecated in r183; call timer.update() each frame before getDelta() or getElapsed().
- `RectAreaLightUniformsLib.init();` → `THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init());`: under WebGPURenderer the uniforms lib is never read, and the light's node dereferences a null LTC library until the textures are registered.

## See Also

- threejs-materials — how materials respond to light (roughness, metalness, material.envMap).
- threejs-textures — loading environment maps and setting texture color spaces.
- threejs-shaders — custom GLSL effects such as fresnel rims, extending lit built-in materials with onBeforeCompile, and lit node materials in TSL.
- threejs-postprocessing — bloom and other light-adjacent effects.
- threejs-fundamentals — renderer setup, tone mapping and exposure for physically scaled lights and HDR environments.
