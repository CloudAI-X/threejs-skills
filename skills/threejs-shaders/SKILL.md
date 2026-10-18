---
name: threejs-shaders
description: "Custom GLSL vertex and fragment shaders: uniforms, varyings, shader chunks, extending built-in materials, instanced attributes, plus the same effects as node materials with TSL on WebGPU. Use when writing a shader, a custom material or effect, moving vertices in a shader, or porting GLSL to TSL."
---

# Three.js Shaders

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

const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    color: { value: new THREE.Color(0x0066ff) }, // hex converted to the linear working space
  },
  vertexShader: /* glsl */ `
    varying vec2 vUv;

    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform float time;
    uniform vec3 color;
    varying vec2 vUv;

    void main() {
      float wave = sin(vUv.x * 10.0 + time * 2.0) * 0.5 + 0.5;
      gl_FragColor = vec4(color * wave, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material);
scene.add(mesh);

const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(animate);

function animate() {
  timer.update();
  material.uniforms.time.value = timer.getElapsed();
  renderer.render(scene, camera);
}
```

## ShaderMaterial vs RawShaderMaterial

Every `ShaderMaterial` compiles as GLSL ES 3.00 — three.js always prepends `#version 300 es` (WebGL 1 support is gone). `texture()`, `textureLod()`, `textureSize()` and `dFdx`/`fwidth` work with no flags; `texture2D`, `textureCube`, `attribute`, `varying` and `gl_FragColor` are `#define` aliases, so most GLSL ES 1.00 shaders compile unchanged (not ones that name a variable `texture`, write `gl_FragData` or keep an `#extension` line, which fails because three.js puts declarations before your source; delete it, the features are core). Leave `glslVersion` unset: setting `THREE.GLSL3` only removes the `gl_FragColor` alias, forcing you to declare your own output variable. ShaderMaterial injects the usual declarations:

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: /* glsl */ `
    // Declared for you: uniforms modelMatrix, modelViewMatrix, projectionMatrix,
    // viewMatrix, normalMatrix, cameraPosition; attributes position, normal, uv,
    // instanceMatrix (InstancedMesh) and color (vertexColors: true)

    varying vec3 vNormal;

    void main() {
      vNormal = normalize(normalMatrix * normal); // view-space normal
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    varying vec3 vNormal;

    void main() {
      gl_FragColor = vec4(vNormal * 0.5 + 0.5, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});
```

RawShaderMaterial gets only `#define SHADER_TYPE`, `#define SHADER_NAME` and your `defines` (plus `#version 300 es` once `glslVersion` is set): no declarations and no precision statement. Always set `glslVersion: THREE.GLSL3` (without it the shader is GLSL ES 1.00, which has no `texture()`), write `precision highp float;`, use `in`/`out` and declare your own output. Declare the matrices you need in the shader and never pass them through `uniforms`: the renderer sets `modelMatrix`, `modelViewMatrix`, `projectionMatrix`, `viewMatrix`, `normalMatrix` and `cameraPosition` for any program that declares them, and a same-named entry in `uniforms` is either overwritten every draw (`modelMatrix`, `modelViewMatrix`, `normalMatrix`) or replaces the camera's value (`projectionMatrix`, `viewMatrix`, `cameraPosition`).

```javascript
const material = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3,
  vertexShader: /* glsl */ `
    precision highp float;

    in vec3 position;

    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;

    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    precision highp float;

    out highp vec4 fragColor;

    void main() {
      fragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

## Uniforms

```javascript
const texture = await new THREE.TextureLoader().loadAsync("textures/diffuse.jpg");
texture.colorSpace = THREE.SRGBColorSpace; // a color image: decoded to linear when sampled

const material = new THREE.ShaderMaterial({
  uniforms: {
    floatValue: { value: 1.5 },
    vec3Value: { value: new THREE.Vector3(1, 2, 3) },
    colorValue: { value: new THREE.Color(0xff0000) }, // hex converted to the linear working space
    mat4Value: { value: new THREE.Matrix4() },
    mapValue: { value: texture },
  },
  // vertexShader and fragmentShader as usual
});
```

The matching GLSL declarations — sample the texture with `texture(mapValue, vUv)`:

```glsl
uniform float floatValue;
uniform vec3 vec3Value;
uniform vec3 colorValue;    // a THREE.Color arrives as vec3
uniform mat4 mat4Value;
uniform sampler2D mapValue;
```

Update a uniform by assigning `.value` from the frame loop (`material.uniforms.floatValue.value = 2.0;`); vectors and colors mutate in place (`material.uniforms.vec3Value.value.set(1, 2, 3);`). Never name a uniform `position`, `normal`, `uv`, `modelViewMatrix` or `cameraPosition` — three.js already declares those in the vertex shader, and the redeclaration is a compile error.

## Output Color and Tone Mapping

`THREE.Color` converts hex input from sRGB to the linear working color space, so a color uniform holds linear values. A ShaderMaterial writes its output as-is: tone mapping and the output color space conversion happen only if the fragment shader ends with the two includes:

```glsl
gl_FragColor = vec4(color, 1.0);
#include <tonemapping_fragment>
#include <colorspace_fragment>
```

`tonemapping_fragment` applies `renderer.toneMapping` (a no-op with `NoToneMapping`); `colorspace_fragment` encodes to `renderer.outputColorSpace` (`SRGBColorSpace` by default). Without them, mid-tone colors render darker than the same hex on a built-in material. Both chunks write `gl_FragColor`, so they require `glslVersion` unset. An 8-bit RGBA texture with `texture.colorSpace = THREE.SRGBColorSpace` decodes to linear on sampling, so sampled colors combine correctly with the includes in place. A `VideoTexture` does not: three.js uploads it without sRGB decoding and decodes it only inside built-in materials, so decode it yourself with `sRGBTransferEOTF(texture(videoMap, vUv))` (the ShaderMaterial prefix declares that function). Rendering into a render target, as inside an EffectComposer, both includes leave the color linear and `OutputPass` encodes it at the end.

## Common Shader Patterns

### Vertex Displacement

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    amplitude: { value: 0.2 },
  },
  vertexShader: /* glsl */ `
    uniform float time;
    uniform float amplitude;

    void main() {
      vec3 pos = position;
      pos.z += sin(pos.x * 5.0 + time) * amplitude;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    void main() {
      gl_FragColor = vec4(0.5, 0.8, 1.0, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2, 64, 64), material); // displacement needs subdivisions
```

### Fresnel

Use one space consistently. World space with `cameraPosition`:

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: /* glsl */ `
    varying vec3 vWorldNormal;
    varying vec3 vWorldPosition;

    void main() {
      vWorldNormal = normalize(mat3(modelMatrix) * normal); // world space, uniform scale assumed
      vWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    varying vec3 vWorldNormal;
    varying vec3 vWorldPosition;

    void main() {
      vec3 viewDirection = normalize(cameraPosition - vWorldPosition);
      float fresnel = pow(1.0 - max(dot(viewDirection, vWorldNormal), 0.0), 3.0);

      gl_FragColor = vec4(mix(vec3(0.0, 0.0, 0.5), vec3(0.5, 0.8, 1.0), fresnel), 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});
```

`normalMatrix * normal` is view space — pair it with the view-space direction to the camera, never with `cameraPosition`. For rim lighting keep both in view space: in the vertex shader `vec4 mvPosition = modelViewMatrix * vec4(position, 1.0); vViewDir = -mvPosition.xyz; vNormal = normalMatrix * normal;`, in the fragment shader `float rim = pow(1.0 - max(dot(normalize(vViewDir), normalize(vNormal)), 0.0), 4.0);`.

### Dissolve

```glsl
uniform float progress;
uniform sampler2D noiseMap;
varying vec2 vUv;

void main() {
  float noise = texture(noiseMap, vUv).r;
  if (noise < progress) discard;

  float edge = smoothstep(progress, progress + 0.1, noise);
  gl_FragColor = vec4(mix(vec3(1.0, 0.5, 0.0), vec3(0.5), edge), 1.0);
  #include <tonemapping_fragment>
  #include <colorspace_fragment>
}
```

## Instanced Meshes and Custom Attributes

Three.js declares `position`, `normal`, `uv`, `instanceMatrix` (for an InstancedMesh), `instanceColor` (once `setColorAt` has been called) and `color` (with `vertexColors: true`). Anything else — a custom per-vertex attribute, `uv1` to `uv3`, `tangent` — you declare yourself with `attribute` (the alias for `in`).

```javascript
const instanceCount = 100;
const geometry = new THREE.PlaneGeometry(0.2, 0.2);
const offsets = new Float32Array(instanceCount * 3);
// fill offsets with one vec3 per instance
geometry.setAttribute("aOffset", new THREE.InstancedBufferAttribute(offsets, 3));

const material = new THREE.ShaderMaterial({
  vertexShader: /* glsl */ `
    attribute vec3 aOffset;

    void main() {
      // instanceMatrix is declared by three.js; the shader must apply it
      gl_Position = projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position + aOffset, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});

const mesh = new THREE.InstancedMesh(geometry, material, instanceCount);
mesh.frustumCulled = false; // the bounds come from instanceMatrix and ignore aOffset
```

For a per-vertex attribute use `geometry.setAttribute("aOffset", new THREE.BufferAttribute(new Float32Array(geometry.attributes.position.count * 3), 3));` — the shader declaration is the same.

## Extending Built-in Materials

`onBeforeCompile` rewrites the shader strings of a built-in material. It runs once per program, before includes are resolved.

```javascript
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

material.onBeforeCompile = (shader) => {
  shader.uniforms.time = { value: 0 };
  material.userData.shader = shader; // keep the reference for updates

  shader.vertexShader = "uniform float time;\n" + shader.vertexShader;
  shader.vertexShader = shader.vertexShader.replace(
    "#include <begin_vertex>",
    `
    #include <begin_vertex>
    transformed.y += sin(position.x * 10.0 + time) * 0.1;
    `,
  );
};

// in the frame loop, with timer.update() first as in the Quick Start:
// if (material.userData.shader) material.userData.shader.uniforms.time.value = timer.getElapsed();
```

Injection points in the built-in shaders: vertex `#include <begin_vertex>` (declares `vec3 transformed = vec3(position)`), `#include <beginnormal_vertex>` (`vec3 objectNormal`) and `#include <project_vertex>` (sets `gl_Position`); fragment `#include <color_fragment>` (`diffuseColor` after map and vertex colors), `#include <opaque_fragment>` (writes `gl_FragColor` before tone mapping) and `#include <fog_fragment>` (after tone mapping, color space and fog).

If `onBeforeCompile` produces different shaders for different materials (for example depending on `material.userData` values), also define `material.customProgramCacheKey()` returning a distinct string per variant, or the renderer reuses one cached program for all of them.

## Shader Chunks

`#include <chunk>` works in ShaderMaterial and RawShaderMaterial source and reuses three.js helper functions; `perspectiveDepthToViewZ` below comes from the `packing` chunk.

```javascript
const fragmentShader = /* glsl */ `
  #include <packing>

  uniform sampler2D depthTexture;
  uniform float cameraNear; // set from camera.near
  uniform float cameraFar; // set from camera.far
  varying vec2 vUv;

  void main() {
    float viewZ = perspectiveDepthToViewZ(texture(depthTexture, vUv).r, cameraNear, cameraFar);
    gl_FragColor = vec4(vec3(-viewZ / cameraFar), 1.0);
  }
`;
```

Only use chunk keys that exist in this release (log `Object.keys(THREE.ShaderChunk)` to check): there is no `output_fragment` chunk — the final output chunk is `opaque_fragment`.

## Debugging

```javascript
// While renderer.debug.checkShaderErrors is true (the default), a failed program logs each failing shader's errors
// with the numbered source lines around the first one. A handler replaces that report and runs only on a compile or link failure:
renderer.debug.onShaderError = (gl, program, vertexShader, fragmentShader) => {
  console.error(gl.getProgramInfoLog(program));
  for (const shader of [vertexShader, fragmentShader]) {
    const log = gl.getShaderInfoLog(shader);
    if (log) console.error(log, gl.getShaderSource(shader)); // the fully assembled shader
  }
};
```

Debug visually by writing the value under test to the output: `gl_FragColor = vec4(vUv, 0.0, 1.0);` for UVs, `gl_FragColor = vec4(vNormal * 0.5 + 0.5, 1.0);` for normals.

## WebGPU and TSL

`ShaderMaterial`, `RawShaderMaterial` and `onBeforeCompile` run only under WebGLRenderer. Under WebGPURenderer a ShaderMaterial logs `Material "ShaderMaterial" is not compatible` and falls back to a plain node material, and `onBeforeCompile` is never called — build the effect as a node material with TSL instead. WebGPURenderer falls back to WebGL 2 by itself when WebGPU is unavailable, so one TSL code path covers both backends.

### Node materials

An unlit effect uses `MeshBasicNodeMaterial`; `MeshStandardNodeMaterial` is lit and renders black without lights or an environment. A ShaderMaterial maps to `MeshBasicNodeMaterial` (unlit) or to the node slots of `MeshStandardNodeMaterial` (lit); replacing the whole fragment output uses `fragmentNode`. The familiar slots exist as node properties: `colorNode`, `opacityNode`, `emissiveNode`, `normalNode`, `positionNode`.

```javascript
import * as THREE from "three/webgpu";
import { color, mix, sin, time, uv } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111827); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 2);

const material = new THREE.MeshBasicNodeMaterial();
material.colorNode = mix(color(0x004466), color(0x66ddff), sin(uv().x.mul(20).add(time)).mul(0.5).add(0.5));

scene.add(new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material));

renderer.setAnimationLoop(animate);

function animate() {
  renderer.render(scene, camera);
}
```

### Uniforms and time

`uniform(value)` creates an updatable uniform; `time` is a built-in node that updates itself every frame and replaces a hand-fed time uniform.

```javascript
import * as THREE from "three/webgpu";
import { abs, mx_fractal_noise_float, time, uniform, uv, vec3 } from "three/tsl";

const material = new THREE.MeshBasicNodeMaterial();

const amplitude = uniform(0.5);
material.colorNode = vec3(abs(mx_fractal_noise_float(uv().mul(10).add(time.mul(0.2)))).mul(amplitude));

// update from JS without touching the node graph:
amplitude.value = 1.0;
```

### Textures and discard

`texture(map, uvNode)` samples a texture (`texture(map)` uses the mesh UVs); `.discard()` on a condition drops the fragment, as `discard` does in GLSL. The Dissolve shader above as TSL:

```javascript
import * as THREE from "three/webgpu";
import { Fn, color, mix, smoothstep, texture, uniform, uv } from "three/tsl";

const noiseMap = await new THREE.TextureLoader().loadAsync("textures/noise.png"); // scalar data: keep NoColorSpace
const progress = uniform(0); // animate progress.value from 0 to 1

const material = new THREE.MeshBasicNodeMaterial();
material.colorNode = Fn(() => {
  const n = texture(noiseMap, uv()).r;
  n.lessThan(progress).discard();
  return mix(color(1, 0.5, 0), color(0.5, 0.5, 0.5), smoothstep(progress, progress.add(0.1), n));
})();
```

### Vertex displacement

`positionNode` replaces the transformed local position. Build it on `positionLocal`, which already carries skinning, morph targets, displacement maps, batching and InstancedMesh transforms; `positionGeometry` is the raw attribute, for rest-pose input only. Displacement needs a subdivided mesh.

```javascript
import * as THREE from "three/webgpu";
import { normalLocal, positionLocal, sin, time } from "three/tsl";

const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(1, 2, 3);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x223344, 1)); // lit material: black without lights or scene.environment

const material = new THREE.MeshStandardNodeMaterial({ flatShading: true }); // normals from the displaced surface; vertex normals ignore positionNode
material.positionNode = positionLocal.add(normalLocal.mul(sin(positionLocal.y.mul(4).add(time))).mul(0.2));

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2, 96, 96), material);
mesh.rotation.x = -Math.PI / 3; // tilt so the displacement along the normal is visible
scene.add(mesh);
```

### Reusable logic

Write reusable logic as a TSL `Fn`; it compiles to both GLSL and WGSL. `varying(node, "name")` passes a value from the vertex stage to the fragment stage.

```javascript
import * as THREE from "three/webgpu";
import { Fn, color, mix, positionWorld, sin, varying } from "three/tsl";

const heightFade = Fn(([y]) => mix(color(0x003344), color(0x88eeff), sin(y.mul(3)).mul(0.5).add(0.5)));

const material = new THREE.MeshBasicNodeMaterial();
const vWorldY = varying(positionWorld.y, "vWorldY"); // interpolated from vertex to fragment stage
material.colorNode = heightFade(vWorldY);
```

### Porting GLSL

Rewrite GLSL functions as `Fn`s instead of pasting shader strings: `glslFn` runs only on the WebGL 2 backend and `wgslFn` only on the WebGPU backend, so neither is portable. Compute shaders dispatch through `renderer.compute()` after `await renderer.init()`.

### Debugging

`await renderer.debug.getShaderAsync(scene, camera, mesh)` returns the generated `{ vertexShader, fragmentShader }` for one mesh.

## Common Mistakes

- `const clock = new THREE.Clock();` → `const timer = new THREE.Timer(); timer.connect(document);`: Clock is deprecated in r183; call timer.update() first in each frame, then timer.getElapsed() or timer.getDelta().
- `"#include <output_fragment>"` → `"#include <opaque_fragment>"`: output_fragment does not exist, so a string replace targeting it silently matches nothing.
- `fragmentShader: "uniform vec3 color;\nvoid main() {\n  gl_FragColor = vec4(color, 1.0);\n}"` → `fragmentShader: "uniform vec3 color;\nvoid main() {\n  gl_FragColor = vec4(color, 1.0);\n  #include <tonemapping_fragment>\n  #include <colorspace_fragment>\n}"`: color uniforms hold linear values and without the two includes, each on its own line, the output is never encoded, so mid-tones render darker than on built-in materials.
- `const renderer = new THREE.WebGPURenderer(); const material = new THREE.ShaderMaterial({ vertexShader, fragmentShader });` → `const material = new THREE.MeshBasicNodeMaterial(); material.colorNode = color(0xff0000);`: under WebGPURenderer ShaderMaterial logs "not compatible" and onBeforeCompile never runs; these APIs work only under WebGLRenderer.
- `material.positionNode = positionGeometry.add(offset)` → `material.positionNode = positionLocal.add(offset)`: positionLocal already carries skinning, morph targets, batching and instanced transforms; positionGeometry is the raw attribute.
- `import { time } from "three/nodes"` → `import { time } from "three/tsl"`: the three/nodes module does not exist; TSL names come from three/tsl, node materials from three/webgpu.
- `import { toVarying } from "three/tsl"` → `varying(node, "vName")`: the exported function is varying(); toVarying exists only as the method form node.toVarying("vName").
- `import { uniforms, PI2 } from "three/tsl"` → `import { uniformArray, TWO_PI } from "three/tsl"`: uniformArray() was renamed from uniforms() in r168 and TWO_PI from PI2 in r181.
- `amplitude.label("amp")` → `amplitude.setName("amp")`: setName() was renamed from label() in r179; label() still sets the name but logs a deprecation warning.
- `texture(map).uv(uvNode)` → `texture(map).sample(uvNode)`: TextureNode.sample() was renamed from uv() in r172.
- `import { screen, viewportTopLeft, directionToColor } from "three/tsl"` → `import { blendScreen, viewportUV, packNormalToRGB } from "three/tsl"`: blendScreen was renamed from screen() in r171, viewportUV from viewportTopLeft in r168 and packNormalToRGB from directionToColor() in r185.
- `rangeFog(color, near, far)` → `fog(color, rangeFogFactor(near, far))`: three/tsl no longer exports rangeFog(); fog() takes a factor node.
- `const t = uniform(0); t.value = timer.getElapsed();` → `time`: the built-in time node updates itself every frame, so the uniform round-trip is unnecessary.
- `const n = glslFn(glslSource);` → `const n = Fn(([p]) => mx_noise_float(p));`: glslFn runs only on the WebGL 2 backend and wgslFn only on the WebGPU backend; a TSL Fn compiles for both.
- `uniforms: { modelViewMatrix: { value: new THREE.Matrix4() } }` → `uniform mat4 modelViewMatrix;`: the renderer sets the built-in matrices for any program that declares them, so a uniforms entry is overwritten every draw (modelMatrix, modelViewMatrix, normalMatrix) or replaces the camera's value (projectionMatrix, viewMatrix, cameraPosition); the GLSL declaration is for a RawShaderMaterial, since a ShaderMaterial already declares the matrices and redeclaring one fails to compile.
- `new THREE.ShaderMaterial({ transparent: true, opacity: 0.5 })` → `new THREE.ShaderMaterial({ transparent: true, uniforms: { opacity: { value: 0.5 } } })`: ShaderMaterial never uploads material.opacity, so declare uniform float opacity and write it to the alpha channel.
- `uniforms: { position: { value: new THREE.Vector3() } }` → `uniforms: { offset: { value: new THREE.Vector3() } }`: position collides with the attribute three.js declares in every ShaderMaterial vertex shader.
- `new THREE.ShaderMaterial({ glslVersion: THREE.GLSL3, vertexShader, fragmentShader })` → `new THREE.ShaderMaterial({ vertexShader, fragmentShader })`: ShaderMaterial always compiles as GLSL ES 3.00, and GLSL3 only removes the gl_FragColor alias, so a shader that still writes gl_FragColor stops compiling.
- `new THREE.RawShaderMaterial({ vertexShader, fragmentShader })` → `new THREE.RawShaderMaterial({ vertexShader, fragmentShader, glslVersion: THREE.GLSL3 })`: without glslVersion a RawShaderMaterial compiles as GLSL ES 1.00, which has no texture().
- `new THREE.ShaderMaterial({ vertexShader, fragmentShader, extensions: { derivatives: true, fragDepth: true } })` → `new THREE.ShaderMaterial({ vertexShader, fragmentShader })`: the WebGL 1 extension keys were removed in r163 and are ignored; dFdx, gl_FragDepth and textureLod are core in GLSL ES 3.00, and extensions now holds only clipCullDistance and multiDraw.
- `blending: THREE.MultiplyBlending` → `blending: THREE.MultiplyBlending, premultipliedAlpha: true`: MultiplyBlending and SubtractiveBlending set no blend function and log an error unless premultiplied alpha is on, and a ShaderMaterial with alpha below 1 must then premultiply its own output with #include <premultiplied_alpha_fragment>.
- `gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);` → `gl_Position = projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position, 1.0);`: on an InstancedMesh three.js declares instanceMatrix but the shader must apply it, or every instance draws at the mesh's own transform.
- `varying vec2 vUv1; void main() { vUv1 = uv1; }` → `attribute vec2 uv1; varying vec2 vUv1; void main() { vUv1 = uv1; }`: three.js declares position, normal and uv for a ShaderMaterial; uv1 to uv3 and tangent need your own declaration.
- `dot(normalize(cameraPosition - vWorldPosition), normalize(normalMatrix * normal))` → `dot(normalize(cameraPosition - vWorldPosition), normalize(mat3(modelMatrix) * normal))`: normalMatrix produces view-space normals while cameraPosition is world space, so the term drifts as the camera rotates.
- `material.onBeforeCompile = (shader) => console.log(shader.fragmentShader);` → `renderer.debug.onShaderError = (gl, program, vs, fs) => console.error(gl.getShaderSource(fs));`: onBeforeCompile runs before includes are resolved and the prefix is added, while onShaderError receives the assembled shaders when a program fails to compile or link.

## See Also

- `threejs-materials` — built-in materials and their texture maps, when a standard material already does the job
- `threejs-textures` — loading textures, color spaces and UV channels for maps sampled in shaders
- `threejs-postprocessing` — full-screen passes (bloom, outline) instead of per-object shaders
