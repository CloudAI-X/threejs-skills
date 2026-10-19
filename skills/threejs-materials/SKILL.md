---
name: threejs-materials
description: "Choosing and configuring three.js materials - unlit, diffuse, specular, PBR, toon, point and line materials - shared properties like blending and stencil, environment response, and the WebGPU node-material and TSL equivalents. Use when styling meshes or tuning material behavior."
---

# Three.js Materials

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);

// PBR material - the default choice for lit, realistic surfaces
const material = new THREE.MeshStandardMaterial({
  color: 0x00aaff,
  roughness: 0.35,
  metalness: 0.1,
});
const mesh = new THREE.Mesh(new THREE.SphereGeometry(1, 48, 32), material);
scene.add(mesh);
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1)); // lit materials render black without lights
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 3, 3);
scene.add(light);

function animate() {
  mesh.rotation.y += 0.01;
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Material Types Overview

| Material | Use case | Lighting |
| -------------------- | ------------------------------------- | ------------------ |
| MeshBasicMaterial | Unlit color, textures, wireframes | None |
| MeshLambertMaterial | Matte diffuse surfaces | Diffuse |
| MeshPhongMaterial | Shiny plastic, cheap specular | Diffuse + specular |
| MeshStandardMaterial | PBR metal/rough workflow | PBR |
| MeshPhysicalMaterial | PBR plus clearcoat, transmission, sheen, iridescence | PBR |
| MeshToonMaterial | Cel shading | Toon steps |
| MeshMatcapMaterial | Lit look baked into a matcap texture, no lights needed | Baked |
| ShadowMaterial | Transparent surface that shows only the shadows it receives | Shadows only |
| PointsMaterial | Point clouds | None |
| LineBasicMaterial / LineDashedMaterial | Lines | None |
| ShaderMaterial / RawShaderMaterial | Custom GLSL, WebGLRenderer only | Custom |

Color data textures (map, emissiveMap, specularMap, sheenColorMap, specularColorMap, LDR matcap, a canvas or video used as color) need `texture.colorSpace = THREE.SRGBColorSpace`, or colors render washed out. Scalar maps (normal, bump, displacement, roughness, metalness, ao, alpha, toon gradientMap) keep the default. HDR loaders set their own color space.

## MeshBasicMaterial

No lighting calculations. Fast, always visible. Supports `envMap` with `reflectivity` (0 to 1) and `combine` (`THREE.MultiplyOperation`, `THREE.MixOperation`, `THREE.AddOperation`); `specularMap` (sRGB) masks the env-map contribution.

```javascript
const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace;

const material = new THREE.MeshBasicMaterial({
  color: 0xffffff, // multiplied with the map
  map,
  transparent: true,
  opacity: 0.5,
  side: THREE.DoubleSide, // FrontSide (default), BackSide, DoubleSide
  fog: true, // affected by scene fog
});
```

## MeshLambertMaterial and MeshPhongMaterial

Lambert is diffuse-only. Phong adds a specular highlight; it is the cheap alternative when Standard is too heavy.

```javascript
const lambert = new THREE.MeshLambertMaterial({
  color: 0x00ff00,
  emissive: 0x111111, // self-illumination color
  emissiveIntensity: 1,
});

const phong = new THREE.MeshPhongMaterial({
  color: 0x0000ff,
  specular: 0x111111, // highlight color
  shininess: 30, // higher = sharper highlight
  flatShading: false, // flat vs smooth normals
});
```

Both also support `map`, `emissiveMap`, `specularMap` (sRGB; masks the env-map reflection, and the highlight on Phong), `normalMap`, `bumpMap`, `displacementMap`, `aoMap`, `envMap`, `envMapIntensity` and `envMapRotation`.

## MeshStandardMaterial (PBR)

Physically-based rendering. Recommended for realistic results.

```javascript
const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace;
const ormMap = await new THREE.TextureLoader().loadAsync("orm.jpg"); // glTF ORM: ao red, roughness green, metalness blue; scalar: keep default

const material = new THREE.MeshStandardMaterial({
  color: 0xffffff, // multiplied with the map
  roughness: 1.0, // 0 = mirror, 1 = fully diffuse; multiplied by roughnessMap (green)
  metalness: 1.0, // 0 = dielectric, 1 = metal; multiplied by metalnessMap (blue), so keep 1 with a map
  map,
  roughnessMap: ormMap,
  metalnessMap: ormMap,
  normalMap: null, // surface detail; scale with normalScale (a Vector2)
  aoMap: ormMap, // ambient occlusion (red); intensity with aoMapIntensity
  displacementMap: null, // vertex displacement; scale with displacementScale, offset with displacementBias
  emissive: 0x000000,
  emissiveIntensity: 1,
  emissiveMap: null, // sRGB
  envMap: null, // or rely on scene.environment (see below)
  fog: true,
});
```

`lightMap` (with `lightMapIntensity`) bakes indirect light the same way aoMap bakes occlusion; an HDR or EXR lightMap is `LinearSRGBColorSpace`, which the loaders set.

### UV channels: aoMap and lightMap read the channel you pick

Every UV-mapped texture samples the attribute selected by `texture.channel`: 0 reads `uv` (the default), 1 reads `uv1`, 2 reads `uv2`, 3 reads `uv3`. aoMap and lightMap need no second UV set - they sample `uv` unless you change `channel`. GLTFLoader loads `TEXCOORD_1` as `uv1` and sets `channel` itself.

```javascript
const aoMap = await new THREE.TextureLoader().loadAsync("ao.jpg"); // scalar: keep default
const material = new THREE.MeshStandardMaterial({ aoMap }); // samples uv: no extra attribute needed

// Alternative for a separate unwrap (a baked AO or light map): copy your baker's u,v pairs into uv1
const bakedUv = new THREE.Float32BufferAttribute(mesh.geometry.attributes.position.count * 2, 2); // zero-filled until you copy them in
mesh.geometry.setAttribute("uv1", bakedUv); mesh.material = material;
aoMap.channel = 1; // this material now samples uv1 instead of uv
```

## MeshPhysicalMaterial (Advanced PBR)

Extends MeshStandardMaterial with layer and volume features. Texture variants exist for most of them: `clearcoatMap`, `clearcoatRoughnessMap`, `clearcoatNormalMap`, `transmissionMap`, `thicknessMap`, `iridescenceMap`, `iridescenceThicknessMap`, `sheenColorMap`, `sheenRoughnessMap`, `anisotropyMap`, `specularIntensityMap`, `specularColorMap`.

```javascript
const material = new THREE.MeshPhysicalMaterial({
  // Clearcoat (car paint, lacquer)
  clearcoat: 1.0, // 0 = none, 1 = full coat
  clearcoatRoughness: 0.1,
  // Transmission (glass, water) - refracts the opaque objects and scene.background behind it
  transmission: 1.0, // 0 = opaque, 1 = fully transmissive
  thickness: 0.5, // volume thickness for refraction
  attenuationDistance: 1, // world units; Infinity (the default) = no absorption
  attenuationColor: new THREE.Color(0xffaa66), // white light turns this color after attenuationDistance
  ior: 1.5, // index of refraction, 1 to 2.333
  // Sheen (fabric, velvet)
  sheen: 1.0,
  sheenColor: new THREE.Color(0xffffff),
  sheenRoughness: 0.5,
  // Iridescence (soap bubbles, oil slicks)
  iridescence: 1.0,
  iridescenceIOR: 1.3,
  iridescenceThicknessRange: [100, 400], // nanometers
  // Anisotropy (brushed metal); direction with anisotropyRotation
  anisotropy: 1.0,
  // Dispersion (prismatic glass); retroreflectivity (road signs, safety gear)
  dispersion: 0.0,
  retroreflectivity: 0.0,
  // Specular overrides: specularIntensity, specularColor (plus their maps)
});
```

Glass - transmission with a smooth surface and something behind it to refract:

```javascript
const glass = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  metalness: 0,
  roughness: 0,
  transmission: 1,
  thickness: 0.5,
  ior: 1.5,
});
// transmission refracts the opaque objects and background behind it; scene.environment adds its reflections
```

## MeshToonMaterial

Cel shading. The gradient map is non-color data - keep its default color space.

```javascript
const steps = new Uint8Array([120, 200]); // two shading steps
const gradientMap = new THREE.DataTexture(steps, 2, 1, THREE.RedFormat);
gradientMap.minFilter = THREE.NearestFilter; // hard steps; LinearFilter blends them
gradientMap.magFilter = THREE.NearestFilter;
gradientMap.needsUpdate = true;

const material = new THREE.MeshToonMaterial({ color: 0x00ff00, gradientMap });
```

## MeshNormalMaterial and MeshDepthMaterial

```javascript
// Normals encoded as RGB - debugging and flat illustrative looks
const normals = new THREE.MeshNormalMaterial({ flatShading: false, wireframe: false });

// Depth as grayscale (BasicDepthPacking, the default); RGBADepthPacking packs it for reading back in a shader
const depth = new THREE.MeshDepthMaterial();
```

## PointsMaterial

For point clouds. `size` is in pixels; with `sizeAttenuation` (the default, perspective cameras only) it is multiplied by half the canvas height and divided by the point's view-space depth.

```javascript
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.Float32BufferAttribute([0, 0, 0, 1, 0.5, 0, 0.5, 1, 0], 3));

const material = new THREE.PointsMaterial({
  color: 0xffffff,
  size: 0.1,
  sizeAttenuation: true, // scale with distance
  alphaTest: 0.5, // with a map or alphaMap: discard fragments below this alpha
});

const points = new THREE.Points(geometry, material);
scene.add(points);
```

## LineBasicMaterial and LineDashedMaterial

WebGL and WebGPU always draw line primitives 1 px wide; `linewidth`, `linecap` and `linejoin` only affect the SVGRenderer. For thick lines under WebGLRenderer use `Line2` with `LineGeometry` and `LineMaterial` from `three/addons/lines/` (`linewidth` in pixels); LineMaterial is a ShaderMaterial, so WebGPURenderer needs `Line2` from `three/addons/lines/webgpu/Line2.js`, which uses `Line2NodeMaterial`.

```javascript
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.Float32BufferAttribute([-1, 0, 0, 1, 0, 0], 3));

const solid = new THREE.LineBasicMaterial({ color: 0xffffff }); // always 1 px

const dashed = new THREE.LineDashedMaterial({ color: 0xffffff, dashSize: 0.5, gapSize: 0.25 });
const line = new THREE.Line(geometry, dashed);
line.computeLineDistances(); // required, or the line draws solid
scene.add(line);
```

## ShaderMaterial (WebGLRenderer only)

Custom GLSL with three.js uniforms injected for you. It always compiles as GLSL ES 3.00: `texture()`, `textureLod()` and `dFdx` work, and `texture2D` / `gl_FragColor` remain as aliases - leave `glslVersion` unset. ShaderMaterial, RawShaderMaterial and `onBeforeCompile` have no WebGPURenderer path; there, use node materials and TSL.

```javascript
const timer = new THREE.Timer();
timer.connect(document); // pause when the tab is hidden

const material = new THREE.ShaderMaterial({
  uniforms: { time: { value: 0 } },
  vertexShader: `
    varying vec2 vUv;
    uniform float time;
    void main() {
      vUv = uv;
      vec3 pos = position;
      pos.z += sin(pos.x * 10.0 + time) * 0.1;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    varying vec2 vUv;
    uniform float time;
    void main() {
      vec3 col = 0.5 + 0.5 * cos(time + vUv.xyx * 3.0);
      gl_FragColor = vec4(col, 1.0);
    }
  `,
});

function animate() {
  timer.update(); // first thing every frame, no argument
  material.uniforms.time.value = timer.getElapsed();
  renderer.render(scene, camera);
}
```

Declared for you in the shader: `modelMatrix`, `modelViewMatrix`, `projectionMatrix`, `viewMatrix`, `normalMatrix`, `cameraPosition`, and the attributes `position`, `normal`, `uv` (plus `instanceMatrix` on an InstancedMesh, which you apply yourself: `projectionMatrix * modelViewMatrix * instanceMatrix * vec4(pos, 1.0)`, and `color` with vertexColors). The extra uv sets and `tangent` are not declared - add `in vec2 uv1;` yourself.

## RawShaderMaterial (WebGLRenderer only)

No declarations are prepended (only `#version` from glslVersion, the `SHADER_TYPE` and `SHADER_NAME` defines, and your `defines`), but the renderer still sets `projectionMatrix`, `modelViewMatrix`, `viewMatrix`, `normalMatrix`, `modelMatrix` and `cameraPosition` - declare them in the shader and never pass them in `uniforms`: a same-named entry is overwritten every draw (`modelMatrix`, `modelViewMatrix`, `normalMatrix`) or replaces the camera's value (`projectionMatrix`, `viewMatrix`, `cameraPosition`).

```javascript
const material = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3, // without it the shader is GLSL ES 1.00, which has no texture()
  vertexShader: `
    precision highp float;
    in vec3 position;
    uniform mat4 projectionMatrix;
    uniform mat4 modelViewMatrix;
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    precision highp float;
    out highp vec4 fragColor; // your own output variable
    void main() {
      fragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

## Common Properties

```javascript
const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });

material.visible = true;
material.transparent = false; // true for opacity below 1; toggling it after the first render needs needsUpdate
material.opacity = 1.0;
material.alphaTest = 0; // discard fragments with alpha below the value
material.side = THREE.FrontSide; // FrontSide, BackSide, DoubleSide
material.depthTest = true;
material.depthWrite = true;
material.blending = THREE.NormalBlending; // or AdditiveBlending, CustomBlending
material.polygonOffset = false; // with the two values below, fixes z-fighting
material.polygonOffsetFactor = 0;
material.polygonOffsetUnits = 0;
material.toneMapped = true; // false keeps exact colors (UI overlays)
material.dithering = false; // true reduces banding in gradients
material.vertexColors = false;
```

```javascript
// Multiply and subtractive blending require premultiplied alpha (both renderers)
const material = new THREE.MeshBasicMaterial({ color: 0x8844ff });
material.blending = THREE.SubtractiveBlending; // same for THREE.MultiplyBlending
material.premultipliedAlpha = true; // without it the renderer logs an error and sets no blend function

// Stencil is off by default in both renderers: create the renderer with { stencil: true }
const maskMaterial = new THREE.MeshBasicMaterial({ colorWrite: false, depthWrite: false });
maskMaterial.stencilWrite = true; // master switch for the stencil test and write
maskMaterial.stencilRef = 1;
maskMaterial.stencilZPass = THREE.ReplaceStencilOp; // writes 1 where the mask draws; also stencilFail, stencilZFail
const contentMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000, stencilWrite: true, stencilRef: 1 });
contentMaterial.stencilFunc = THREE.EqualStencilFunc; // draws only where the mask wrote 1
contentMaterial.stencilFuncMask = 0xff; // compare mask; stencilWriteMask masks the write (both default 0xff)
// draw the mask first: give its mesh a lower renderOrder than the content mesh
```

## Multiple Materials per Mesh

```javascript
// One material per geometry group; BoxGeometry has six, in the order +x, -x, +y, -y, +z, -z
const geometry = new THREE.BoxGeometry(1, 1, 1);
const colors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];
const mesh = new THREE.Mesh(geometry, colors.map((color) => new THREE.MeshBasicMaterial({ color })));

// Custom groups: start, count, material index. Indexed geometry counts indices (BoxGeometry has 36),
// non-indexed geometry counts vertices; ranges outside every group are not drawn
geometry.clearGroups();
geometry.addGroup(0, 18, 0);
geometry.addGroup(18, 18, 1);
```

## Environment Maps

An equirectangular HDR file lights and reflects every PBR material in the scene. Both renderers prefilter it themselves - no PMREM step needed.

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

const env = await new HDRLoader().loadAsync("environment.hdr");
env.mapping = THREE.EquirectangularReflectionMapping; // without it a background draws as a flat quad
scene.environment = env; // lights and reflects PBR materials
scene.background = env;
scene.environmentIntensity = 0.8; // scene-wide dimmer; scene.environmentRotation tilts it
```

```javascript
const cubeEnv = new THREE.CubeTextureLoader().load([
  "px.jpg", "nx.jpg", "py.jpg", "ny.jpg", "pz.jpg", "nz.jpg",
]);

const material = new THREE.MeshStandardMaterial({ envMap: cubeEnv });
material.envMapIntensity = 1.5; // scales only this material's own envMap
material.envMapRotation = new THREE.Euler(0, Math.PI / 4, 0);
```

`scene.environment` lights MeshStandardMaterial and MeshPhysicalMaterial on both renderers. With WebGLRenderer it also lights MeshLambertMaterial and MeshPhongMaterial; WebGPURenderer does not.

## Runtime Changes, Cloning, Disposal

```javascript
mesh.material.color.set(0xff0000); // uniform values: no recompile needed
mesh.material.roughness = 0.8;

// material.needsUpdate = true after changing a setting compiled into the shader:
// transparent, side, blending, flatShading, vertexColors, dithering, premultipliedAlpha, alphaHash,
// adding or removing a map slot, or shader source (alphaTest and envMap recompile by themselves)

const clone = mesh.material.clone(); // copies values; map slots share their textures, ShaderMaterial uniforms are cloned
```

Dispose a material nothing else uses: `material.dispose()` releases its GPU programs, which are deleted once no other material uses them; textures need their own `texture.dispose()`.

Performance, in order of impact:

1. Share one material instance across meshes - the renderer re-uploads material uniforms whenever consecutive draws switch material (identical settings already share one compiled program).
2. Prefer opaque rendering: transparent materials are sorted back to front by object and drawn after the opaque ones, still writing depth unless `depthWrite: false`, so stacked layers overdraw and intersecting ones misorder; for cutouts, alphaTest keeps the material opaque, so it draws in the opaque pass, correctly in any order.
3. Pick the cheapest material that looks right: Basic, then Lambert, Phong, Standard, Physical.

## WebGPU and TSL

MeshStandardMaterial and the other built-in classes run under WebGPURenderer unchanged - the renderer converts them to node materials automatically; ShaderMaterial, RawShaderMaterial, MeshDepthMaterial and MeshDistanceMaterial have no node equivalent and log `Material "..." is not compatible`. For custom shading use the node material classes from `three/webgpu` with TSL nodes from `three/tsl`; they target WebGPURenderer, and WebGLRenderer draws them only through the `WebGLNodesHandler` addon (`three/addons/tsl/WebGLNodesHandler.js`, passed to `renderer.setNodesHandler()`), a migration aid without VSM shadows, MRT, transmission or the WebGPU post-processing stack.

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

```javascript
import * as THREE from "three/webgpu";
import { texture, mix, color, float, time, positionLocal, normalLocal, sin } from "three/tsl";

// Unlit: the node equivalent of a color ShaderMaterial
const unlit = new THREE.MeshBasicNodeMaterial();
unlit.colorNode = mix(color(0x0066ff), color(0xff0066), sin(time.mul(0.5)).mul(0.5).add(0.5));

// Lit: node slots replace the fixed properties; needs lights or scene.environment
const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace;
const lit = new THREE.MeshStandardNodeMaterial();
lit.colorNode = texture(map);
lit.roughnessNode = float(0.2);

// Vertex displacement on positionLocal, which already carries skinning, morphs, instancing
const shell = new THREE.MeshStandardNodeMaterial();
shell.positionNode = positionLocal.add(normalLocal.mul(sin(positionLocal.y.mul(6))).mul(0.05));
```

```javascript
import * as THREE from "three/webgpu";
import { instancedBufferAttribute } from "three/tsl";

// The WebGPU backend draws Points at 1 px whatever the size; draw larger points as quads
const positions = new THREE.InstancedBufferAttribute(new Float32Array(1500), 3); // fill with x, y, z per point
const points = new THREE.Sprite(
  new THREE.PointsNodeMaterial({ positionNode: instancedBufferAttribute(positions), size: 4, sizeAttenuation: false }) // 4 px squares
);
points.count = 500; points.frustumCulled = false; // one quad per point; culling would test only a 0.7-unit sphere at the sprite origin
scene.add(points);
```

## Common Mistakes

- `geometry.setAttribute("uv2", geometry.attributes.uv)` → `const material = new THREE.MeshStandardMaterial({ aoMap });`: aoMap samples the attribute picked by `aoMap.channel`, 0 (`uv`) by default, so it needs no extra attribute and uv2 is read only with `aoMap.channel = 2`; only a separate unwrap goes in `uv1` with `aoMap.channel = 1`.
- `new RGBELoader()` → `new HDRLoader()`: HDRLoader was renamed from RGBELoader in r180; RGBELoader is deprecated and still loads, with a warning - import HDRLoader from "three/addons/loaders/HDRLoader.js".
- `"three/examples/jsm/loaders/HDRLoader.js"` → `"three/addons/loaders/HDRLoader.js"`: the browser import maps map `three/addons/` to examples/jsm/ and define no `three/examples/` key, so the examples path fails there.
- `material.blending = THREE.MultiplyBlending;` → `material.blending = THREE.MultiplyBlending; material.premultipliedAlpha = true;`: multiply and subtractive blending require premultiplied alpha, or the renderer logs an error and sets no blend function.
- `material.stencilMask = 0xff;` → `material.stencilFuncMask = 0xff; material.stencilWriteMask = 0xff;`: `stencilMask` does not exist; assigning it just creates an unused field.
- `new THREE.WebGLRenderer({ antialias: true })` → `new THREE.WebGLRenderer({ antialias: true, stencil: true })`: the stencil buffer is off by default, so material stencil settings are silently ignored without it.
- `material.envMapIntensity = 0.5;` → `scene.environmentIntensity = 0.5;`: to dim scene.environment use the scene property - envMapIntensity attenuates only the material's own envMap.
- `material.uniforms.time.value = clock.getElapsedTime();` → `material.uniforms.time.value = timer.getElapsed();`: Clock is deprecated in r183 - create a THREE.Timer, call timer.connect(document) once, and timer.update() (no argument) first in the loop.
- `new THREE.ShaderMaterial({ glslVersion: THREE.GLSL3, fragmentShader })` → `new THREE.ShaderMaterial({ fragmentShader })`: ShaderMaterial already compiles as GLSL ES 3.00, so `texture()` works with glslVersion unset, while GLSL3 removes the `gl_FragColor` alias and a shader that writes it fails to compile.
- `uniforms: { modelViewMatrix: { value: new THREE.Matrix4() } }` → `uniform mat4 modelViewMatrix;`: declare the built-in matrices in a RawShaderMaterial's shader source - a same-named uniforms entry is overwritten every draw (modelMatrix, modelViewMatrix, normalMatrix) or replaces the camera's value (projectionMatrix, viewMatrix, cameraPosition).
- `new THREE.LineBasicMaterial({ linewidth: 5 })` → `new Line2(new LineGeometry().setPositions([-1, 0, 0, 1, 0, 0]), new LineMaterial({ linewidth: 5 }))`: WebGLRenderer always draws 1 px line primitives; Line2 needs a LineGeometry, not a BufferGeometry, and all three come from three/addons/lines/ (under WebGPURenderer, Line2 comes from three/addons/lines/webgpu/Line2.js with Line2NodeMaterial).
- `new THREE.ShaderMaterial({ vertexShader, fragmentShader })` → `new THREE.MeshBasicNodeMaterial()`: under WebGPURenderer use node materials and TSL - ShaderMaterial and RawShaderMaterial have no node equivalent and log "is not compatible".
- `import { texture } from "three/nodes";` → `import { texture } from "three/tsl";`: `three/nodes` does not exist - node materials come from `three/webgpu`, TSL functions from `three/tsl`.
- `import * as THREE from "three"; const m = new THREE.MeshStandardNodeMaterial();` → `import * as THREE from "three/webgpu"; const m = new THREE.MeshStandardNodeMaterial();`: node materials are exported only by three/webgpu and target WebGPURenderer.
- `new THREE.PointsMaterial({ size: 5 })` → `new THREE.Sprite(new THREE.PointsNodeMaterial({ positionNode: instancedBufferAttribute(positions) }))`: the WebGPU backend draws Points at 1 px whatever the size - set sprite.count to the number of points.
- `material.positionNode = positionGeometry.add(offset);` → `material.positionNode = positionLocal.add(offset);`: positionLocal carries skinning, morph targets, batching and instancing; positionGeometry is the raw rest-pose attribute.
- `map.encoding = THREE.sRGBEncoding;` → `map.colorSpace = THREE.SRGBColorSpace;`: `encoding` and `sRGBEncoding` no longer exist, so the old line does nothing and the color map renders washed out.
- `const m = new THREE.MeshStandardNodeMaterial(); m.colorNode = color(0xff0000);` → `const m = new THREE.MeshBasicNodeMaterial(); m.colorNode = color(0xff0000);`: MeshStandardNodeMaterial is lit and renders black without lights or an environment, so an unlit effect uses MeshBasicNodeMaterial.
- `new THREE.ShaderMaterial({ extensions: { derivatives: true }, fragmentShader })` → `new THREE.ShaderMaterial({ fragmentShader })`: the WebGL1 extension flags do nothing, because GLSL ES 3.00 has `dFdx`, `gl_FragDepth` and `textureLod` built in.

## See Also

- `threejs-textures` - loading, color spaces, filters and channels in depth
- `threejs-shaders` - GLSL authoring, shader chunks and debugging under WebGLRenderer
- `threejs-lighting` - lights, shadows and how they interact with materials
