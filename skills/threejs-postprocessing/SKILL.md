---
name: threejs-postprocessing
description: "Three.js post-processing: composer pass chains, built-in renderer effects, and the WebGPU/TSL node pipeline — bloom, depth of field, ambient occlusion, custom screen effects. Use when adding visual effects, color grading, blur, glow, or screen-space shaders, on WebGL or with WebGPU and TSL."
---

# Three.js Post-Processing

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

// A bright emissive sphere gives bloom something to pick up
const mesh = new THREE.Mesh(
  new THREE.SphereGeometry(1, 32, 16),
  new THREE.MeshStandardMaterial({ color: 0x222222, emissive: 0xffffff, emissiveIntensity: 2 }),
);
scene.add(mesh);

// The composer replaces renderer.render(). Render the scene, then effects,
// then OutputPass last: it applies renderer.toneMapping and outputColorSpace once.
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
// bloomPass.strength, .radius and .threshold are live properties; threshold is the luminance above which things glow
composer.addPass(bloomPass);
composer.addPass(new OutputPass());

function animate() {
  composer.render(); // not renderer.render()
}

renderer.setAnimationLoop(animate);
```

## Pass Order

The composer's buffers hold linear colors; tone mapping and the sRGB conversion happen once, in `OutputPass`, using the renderer's settings. Build chains in this order: `RenderPass` → linear-space effects (bloom, depth of field, SSAO, outline, ...) → `SMAAPass`, after the effects whose edges it should smooth → `OutputPass` → `FXAAPass` (`three/addons/postprocessing/FXAAPass.js`) only when FXAA is used, because FXAA needs sRGB input. Without FXAA, `OutputPass` is the last pass.

`SSAOPass` darkens the image produced by the previous pass, so a `RenderPass` must come before it. `RenderPixelatedPass` renders the scene itself and replaces `RenderPass`. `pass.enabled = false` skips a pass without rebuilding the chain.

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { SMAAPass } from "three/addons/postprocessing/SMAAPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85));
composer.addPass(new SMAAPass()); // no arguments; the composer sizes it
composer.addPass(new OutputPass());
```

## Composer Setup

- The composer copies the renderer's pixel ratio at construction. Call `renderer.setPixelRatio()` before constructing it, or `composer.setPixelRatio()` afterwards.
- The composer's buffers are not multisampled; `antialias: true` on the renderer does not reach them. For MSAA on the scene render and the composer's buffers, pass your own buffer with `samples` (CSS-pixel size) and size the composer once:

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";

const composer = new EffectComposer(
  renderer,
  new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight, {
    type: THREE.HalfFloatType,
    samples: 4, // MSAA on the composer's two buffers
  }),
);
composer.setSize(window.innerWidth, window.innerHeight); // scales the buffer by the pixel ratio
```

- On resize, call `camera.aspect = w / h; camera.updateProjectionMatrix(); renderer.setSize(w, h); composer.setSize(w, h);` — the composer resizes its buffers and every pass, pixel-ratio included. Never resize a pass by hand.
- `composer.render()` drives time-based passes (FilmPass) from its own internal timer; pass a delta in seconds only if you manage time yourself.
- The composer assigns `renderToScreen` to the last enabled pass every frame. Set `composer.renderToScreen = false` only to keep the result off screen, as selective bloom does below.

## Selective Bloom

Bloom only meshes marked with a layer: paint everything else black for a bloom-only composer that renders off screen, then mix its texture into a final composer that renders the clean scene. `scene.background` is not a mesh: set it to `null` around `bloomComposer.render()`, or it is added to the final image a second time; on resize, call `setSize` on both composers.

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const BLOOM_LAYER = 1;
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_LAYER); // test mask: matches any object enabled on this layer

mesh.layers.enable(BLOOM_LAYER); // add the layer; set() would remove the mesh from layer 0

// Paint non-bloomed meshes black, then restore them
const darkMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
const materials = new Map();

function darkenNonBloomed(object) {
  if (object.isMesh && bloomLayer.test(object.layers) === false) {
    materials.set(object, object.material);
    object.material = darkMaterial;
  }
}

function restoreMaterial(object) {
  if (materials.has(object)) {
    object.material = materials.get(object);
    materials.delete(object);
  }
}

// Bloom-only composer; with renderToScreen off, its result stays in renderTarget2
const bloomComposer = new EffectComposer(renderer);
bloomComposer.renderToScreen = false;
bloomComposer.addPass(new RenderPass(scene, camera));
bloomComposer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85));

// Final composer: the clean scene plus the bloom texture, mixed by a ShaderPass
const finalComposer = new EffectComposer(renderer);
finalComposer.addPass(new RenderPass(scene, camera));
finalComposer.addPass(
  new ShaderPass(
    new THREE.ShaderMaterial({
      uniforms: {
        baseTexture: { value: null }, // filled from the previous pass
        bloomTexture: { value: bloomComposer.renderTarget2.texture },
      },
      vertexShader: /* glsl */ `
        varying vec2 vUv;
        void main() {
          vUv = uv;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: /* glsl */ `
        uniform sampler2D baseTexture;
        uniform sampler2D bloomTexture;
        varying vec2 vUv;
        void main() {
          gl_FragColor = texture2D(baseTexture, vUv) + texture2D(bloomTexture, vUv);
        }
      `,
    }),
    "baseTexture",
  ),
);
finalComposer.addPass(new OutputPass());

function animate() {
  const background = scene.background;
  scene.background = null; // not a mesh: it would bloom and be added twice
  scene.traverse(darkenNonBloomed); // black scene: only marked meshes bloom
  bloomComposer.render();
  scene.traverse(restoreMaterial);
  scene.background = background;
  finalComposer.render(); // clean scene + bloom to the screen
}

renderer.setAnimationLoop(animate);
```

## Depth of Field

```javascript
import { BokehPass } from "three/addons/postprocessing/BokehPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera)); // BokehPass reads this color image

const bokehPass = new BokehPass(scene, camera, {
  focus: 10.0, // view-space depth that stays sharp, in world units
  aperture: 0.025, // how far out of focus things blur
  maxblur: 0.01, // cap on the blur radius
});
composer.addPass(bokehPass);
composer.addPass(new OutputPass());

// Refocus at runtime: distanceTo matches the view-space depth for objects near the view center
bokehPass.uniforms.focus.value = camera.position.distanceTo(mesh.position);
```

## Ambient Occlusion

```javascript
import { SSAOPass } from "three/addons/postprocessing/SSAOPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);

// SSAOPass darkens the previous image, so the scene render must come first
composer.addPass(new RenderPass(scene, camera));

const ssaoPass = new SSAOPass(scene, camera, window.innerWidth, window.innerHeight);
ssaoPass.kernelRadius = 16;
ssaoPass.minDistance = 0.005;
ssaoPass.maxDistance = 0.1;
composer.addPass(ssaoPass);
composer.addPass(new OutputPass());

// Debug views: ssaoPass.output = SSAOPass.OUTPUT.SSAO (or Blur, Depth, Normal); the default shows the shaded scene
```

## Outline

```javascript
import * as THREE from "three";
import { OutlinePass } from "three/addons/postprocessing/OutlinePass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const outlinePass = new OutlinePass(new THREE.Vector2(window.innerWidth, window.innerHeight), scene, camera);
outlinePass.selectedObjects = [mesh]; // only these are outlined
outlinePass.edgeStrength = 3;
outlinePass.visibleEdgeColor.set(0xffffff);
outlinePass.hiddenEdgeColor.set(0x190a05); // parts hidden behind other objects
composer.addPass(outlinePass);
composer.addPass(new OutputPass());
```

## More Passes

Film grain, glitch, halftone and vignette are drop-in filters for the middle of the chain. `ColorCorrectionShader` works like the vignette: a `ShaderPass` with `powRGB` / `mulRGB` / `addRGB` uniform vectors. Its default powRGB of (2, 2, 2) darkens the image; start from (1, 1, 1) for neutral.

```javascript
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";
import { FilmPass } from "three/addons/postprocessing/FilmPass.js";
import { GlitchPass } from "three/addons/postprocessing/GlitchPass.js";
import { HalftonePass } from "three/addons/postprocessing/HalftonePass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { VignetteShader } from "three/addons/shaders/VignetteShader.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const filmPass = new FilmPass(0.35, false); // grain intensity, grayscale; animates from the composer's delta
composer.addPass(filmPass);

const glitchPass = new GlitchPass();
glitchPass.goWild = true; // glitch every frame; false (the default) glitches at random intervals
composer.addPass(glitchPass);

const halftonePass = new HalftonePass({ shape: 1, radius: 4, scatter: 0, greyscale: false }); // shape: 1 dot, 2 ellipse, 3 line, 4 square, 5 diamond
composer.addPass(halftonePass);

const vignettePass = new ShaderPass(VignetteShader); // uniforms: offset and darkness
vignettePass.uniforms.offset.value = 1.0;
vignettePass.uniforms.darkness.value = 1.0;
composer.addPass(vignettePass);

composer.addPass(new OutputPass());
```

`RenderPixelatedPass` renders the scene itself, so it replaces `RenderPass` instead of following it:

```javascript
import { RenderPixelatedPass } from "three/addons/postprocessing/RenderPixelatedPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
const pixelatedPass = new RenderPixelatedPass(6, scene, camera); // 6 = pixel size
pixelatedPass.normalEdgeStrength = 0.3; // edges where the surface normal changes (creases)
pixelatedPass.depthEdgeStrength = 0.4; // depth-discontinuity edges
composer.addPass(pixelatedPass);
composer.addPass(new OutputPass());
```

## Custom Passes

Wrap any `ShaderMaterial`-style shader object in a `ShaderPass`. The `tDiffuse` uniform receives the previous pass output. Keep custom effects before `OutputPass` while they work in linear space; place one after it only if it needs sRGB input, like FXAA.

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const WaveShader = {
  uniforms: {
    tDiffuse: { value: null },
    time: { value: 0 },
    intensity: { value: 1.0 },
  },
  vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform sampler2D tDiffuse;
    uniform float time;
    uniform float intensity;
    varying vec2 vUv;
    void main() {
      vec2 uv = vUv;
      uv.x += sin(uv.y * 10.0 + time) * 0.01 * intensity; // wave distortion
      gl_FragColor = texture2D(tDiffuse, uv);
    }
  `,
};

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
const wavePass = new ShaderPass(WaveShader);
composer.addPass(wavePass);
composer.addPass(new OutputPass());

// Drive time uniforms from a Timer
const timer = new THREE.Timer();
timer.connect(document); // keeps deltas sane in inactive tabs

function animate() {
  timer.update();
  wavePass.uniforms.time.value = timer.getElapsed();
  composer.render();
}

renderer.setAnimationLoop(animate);
```

## Renderer Built-In Effects (No Composer)

`renderer.setEffects()` runs ordinary passes on internal HDR buffers after a plain `renderer.render()` — no composer, and tone mapping and output color space are applied automatically after the effects. It requires `outputBufferType: THREE.HalfFloatType` (or `FloatType`), takes no `OutputPass` or `FXAAPass`, anti-aliases through `antialias: true`, and resizes with `renderer.setSize()`. Effects receive no delta time here: FilmPass's time uniform becomes NaN and its grain is lost, so run time-driven passes through EffectComposer.

```javascript
import * as THREE from "three";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";

const renderer = new THREE.WebGLRenderer({ antialias: true, outputBufferType: THREE.HalfFloatType });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
renderer.setEffects([bloomPass]);

function animate() {
  renderer.render(scene, camera); // bloom runs on the internal HDR buffers after the scene render
}

renderer.setAnimationLoop(animate);
```

## WebGPU and TSL

Under WebGPURenderer, post-processing is a TSL node graph on a `RenderPipeline`, not a pass chain: render the scene with `pass()`, build effects from `three/addons/tsl/display/*.js`, and assign the final node to `renderPipeline.outputNode`. `pass()` uses the renderer's MSAA samples, so `antialias: true` works here, except for the `ao()` pre-pass, which must be `pass(scene, camera, { samples: 0 })`, and `traa`, which needs a renderer without `antialias`; `pass()` also follows `renderer.setSize()`, so the pipeline needs no resize call of its own.

```javascript
import * as THREE from "three/webgpu";
import { pass } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);
scene.add(new THREE.Mesh(new THREE.SphereGeometry(1, 32, 16), new THREE.MeshBasicNodeMaterial({ color: 0xffffff })));
const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode("output");

// bloom() returns only the glow, so add it to the scene color
renderPipeline.outputNode = scenePassColor.add(bloom(scenePassColor, 1.5, 0.4, 0.85));

function animate() {
  renderPipeline.render(); // instead of renderer.render()
}

renderer.setAnimationLoop(animate);
```

Filters return the finished image and replace the output node: `dof`, `film`, `rgbShift`, `chromaticAberration`, `afterImage`, `dotScreen`, `lut3D` and `smaa` (linear input) take the scene color node, `pixelationPass(scene, camera, 6)` renders the scene itself, and `traa` needs a velocity MRT and a renderer without `antialias`. Like `bloom`, `outline(scene, camera, { selectedObjects })` returns only edge masks (`.visibleEdge`, `.hiddenEdge`) that you color and add to the scene color, and `ssao(depthNode, normalNode, camera)` returns only the occlusion factor in `.r`, which multiplies the scene color.

```javascript
import * as THREE from "three/webgpu";
import { pass, renderOutput } from "three/tsl";
import { dof } from "three/addons/tsl/display/DepthOfFieldNode.js";
import { fxaa } from "three/addons/tsl/display/FXAANode.js";

const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode("output");
// A filter replaces the image: node, viewZ, focus distance, focal length, bokeh scale
const dofNode = dof(scenePassColor, scenePass.getViewZNode(), 4.5, 1.2, 2.5);

// FXAA runs in sRGB: disable the automatic output transform, then convert the DOF result before FXAA
renderPipeline.outputColorTransform = false;
renderPipeline.outputNode = fxaa(renderOutput(dofNode));

// After the first frame, a changed outputNode or outputColorTransform only takes effect with:
renderPipeline.needsUpdate = true;
```

`ao()` needs depth and view-space normals from a separate, single-sampled pre-pass, and its AO lives in the red channel:

```javascript
import * as THREE from "three/webgpu";
import { pass, mrt, normalView, screenUV, builtinAOContext } from "three/tsl";
import { ao } from "three/addons/tsl/display/GTAONode.js";

const renderPipeline = new THREE.RenderPipeline(renderer);
// Pre-pass with view-space normals in the output MRT attachment
const prePass = pass(scene, camera, { samples: 0 }); // ao() cannot read a multisampled depth texture
prePass.setMRT(mrt({ output: normalView }));

const aoPass = ao(prePass.getTextureNode("depth"), prePass.getTextureNode(), camera);

// Apply the AO to a normal scene pass
const scenePass = pass(scene, camera);
scenePass.contextNode = builtinAOContext(aoPass.getTextureNode().sample(screenUV).r);
renderPipeline.outputNode = scenePass;
```

A custom effect is plain TSL over the scene texture instead of a GLSL `ShaderPass`; this is the wave from Custom Passes, animated by the `time` node:

```javascript
import * as THREE from "three/webgpu";
import { pass, screenUV, sin, time, vec2 } from "three/tsl";

const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePassColor = pass(scene, camera).getTextureNode("output");
const offset = vec2(sin(screenUV.y.mul(10).add(time)).mul(0.01), 0);
renderPipeline.outputNode = scenePassColor.sample(screenUV.add(offset));
```

Selective bloom needs no layer trick here: add `emissive` (from `three/tsl`) to the pass MRT with `scenePass.setMRT(mrt({ output, emissive }))` and bloom only that attachment: `renderPipeline.outputNode = scenePassColor.add(bloom(scenePass.getTextureNode("emissive")))`.

## Common Mistakes

- `three/examples/jsm/postprocessing/EffectComposer.js` → `three/addons/postprocessing/EffectComposer.js`: the examples/jsm specifier does not resolve under the import map.
- `composer.addPass(new ShaderPass(GammaCorrectionShader))` → `composer.addPass(new OutputPass())`: GammaCorrectionShader only applies the sRGB transfer function, while OutputPass applies renderer tone mapping, exposure and output color space and ends every chain; only passes that need sRGB input, such as FXAAPass, follow it.
- `composer.addPass(bloomPass); composer.render();` → `composer.addPass(bloomPass); composer.addPass(new OutputPass()); composer.render();`: the composer's buffers are linear, and without OutputPass tone mapping and the sRGB conversion are skipped or applied unevenly.
- `composer.addPass(fxaaPass); composer.addPass(outputPass);` → `composer.addPass(outputPass); composer.addPass(fxaaPass);`: FXAA needs sRGB input so it follows OutputPass, while SMAA works in linear space and must run before it.
- `new ShaderPass(FXAAShader)` → `new FXAAPass()`: FXAAPass sets its own resolution uniform from the composer size (pixel ratio included); a hand-set value is wrong on HiDPI screens.
- `new FilmPass(0.35, 0.5, 648, false)` → `new FilmPass(0.35, false)`: the signature is (intensity, grayscale); the scanline arguments do not exist.
- `new HalftonePass(width, height, params)` → `new HalftonePass(params)`: the width and height arguments were removed in r175; options go in the single params object.
- `new SMAAPass(width, height)` → `new SMAAPass()`: the size arguments were removed in r175; the composer sizes the pass.
- `new UnrealBloomPass(new THREE.Vector2(window.innerWidth / 2, window.innerHeight / 2), 1.5, 0.4, 0.85)` → `new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85)`: addPass immediately resizes the pass to the composer size, undoing the half-resolution trick.
- `bloomPass.resolution.set(width, height)` → `composer.setSize(width, height)`: UnrealBloomPass reads its resolution only in the constructor; the composer resizes the pass and its mips.
- `bloomPass.renderToScreen = false` → `bloomComposer.renderToScreen = false`: the composer reassigns renderToScreen on every enabled pass each frame, so only the composer-level switch keeps the result off screen.
- `requestAnimationFrame(animate)` → `renderer.setAnimationLoop(animate)`: both renderers' docs advise it over a manual loop, and it hands frames to the WebXR session when one starts.
- `clock.getElapsedTime()` → `timer.getElapsed()`: Clock is deprecated in r183; create a THREE.Timer, call timer.connect(document) once and timer.update() before reading it each frame.
- `composer.render(); renderer.render(scene, camera);` → `bloomComposer.render(); finalComposer.render();`: the second renderer.render clears the canvas and erases the composer output; selective bloom mixes the bloom texture in through a second composer.
- `const renderer = new THREE.WebGLRenderer({ antialias: true }); renderer.setEffects([bloomPass]);` → `const renderer = new THREE.WebGLRenderer({ antialias: true, outputBufferType: THREE.HalfFloatType }); renderer.setEffects([bloomPass]);`: setEffects() requires a HalfFloatType or FloatType output buffer; with the default it logs an error and applies no effects.
- `renderer.setEffects([bloomPass, new OutputPass(), new FXAAPass()])` → `renderer.setEffects([bloomPass])`: the built-in path applies tone mapping and color space once, after the effects, so an OutputPass in the list applies them twice (it warns) and an FXAAPass without one gets linear input; anti-alias with `antialias: true` instead.
- `import { bloom } from "three/tsl"` → `import { bloom } from "three/addons/tsl/display/BloomNode.js"`: three/tsl exports pass, mrt and renderOutput; the effect nodes live in the addons.
- `new THREE.PostProcessing(renderer)` → `new THREE.RenderPipeline(renderer)`: renamed from PostProcessing in r183; the old name survives only as a deprecated alias that warns.
- `renderPipeline.outputNode = bloom(scenePassColor)` → `renderPipeline.outputNode = scenePassColor.add(bloom(scenePassColor))`: bloom() returns only the glow, so on its own the scene is never drawn.
- `renderPipeline.outputNode = fxaa(scenePassColor)` → `renderPipeline.outputColorTransform = false; renderPipeline.outputNode = fxaa(renderOutput(scenePass));`: FXAA needs sRGB input, but the default outputColorTransform tone-maps and converts after it.
- `renderPipeline.outputNode = scenePassColor.add(dof(scenePassColor, scenePass.getViewZNode()))` → `renderPipeline.outputNode = dof(scenePassColor, scenePass.getViewZNode())`: dof returns the finished image, so adding the scene color draws the scene twice.
- `await renderPipeline.renderAsync()` → `renderPipeline.render()`: renderAsync is deprecated in r181; call `await renderer.init()` once at setup and render synchronously.
- `aoPass.distanceFallOff.value = 2` → `aoPass.radius.value = 0.2`: distanceExponent and distanceFallOff are deprecated in r186 and no longer change the AO, so tune radius and scale instead.
- `new EffectComposer(webgpuRenderer)` → `new THREE.RenderPipeline(webgpuRenderer)`: EffectComposer and its passes run only under WebGLRenderer; WebGPURenderer effects are TSL nodes on a RenderPipeline.

## See Also

- `threejs-fundamentals` — renderer creation, sizing, color spaces and the animation loop.
- `threejs-shaders` — GLSL ShaderMaterial authoring and the TSL node language for custom effects.
- `threejs-textures` — render targets and using a post-processed image as a texture.
