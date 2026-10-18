---
name: threejs-animation
description: "Keyframe, skeletal and morph-target animation, procedural motion and clip blending, on WebGL or with WebGPU and TSL. Use when animating objects, playing glTF animations, posing bones, driving morph targets, or mixing and crossfading clips."
---

# Three.js Animation

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
camera.position.set(0, 0.5, 3);

const mesh = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshNormalMaterial());
scene.add(mesh);

const timer = new THREE.Timer();
timer.connect(document); // also keeps deltas small after the tab was hidden

function animate() {
  timer.update(); // first thing each frame: fixes getDelta()/getElapsed() for this frame
  const delta = timer.getDelta();
  const elapsed = timer.getElapsed();

  mesh.rotation.y += delta; // frame-rate-independent spin
  mesh.position.y = Math.sin(elapsed * 2) * 0.25; // bob on absolute time

  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## How the System Fits Together

- **AnimationClip**: keyframe data — tracks of times and values, no playback state.
- **AnimationMixer**: plays clips on one root object and all its descendants.
- **AnimationAction**: playback control for one clip on one mixer.

Each frame, `mixer.update(delta)` samples the active clips and writes the values into ordinary object properties. The system runs on the CPU and never touches the renderer, so it drives models the same way under either renderer.

## AnimationClip and Keyframe Tracks

```javascript
// One track per animated property. values.length must be times.length * valueSize.
const times = [0, 1, 2]; // keyframe times in seconds
const values = [0, 1, 0]; // one scalar per keyframe for a Number track

const track = new THREE.NumberKeyframeTrack(".position[y]", times, values); // the root's position.y

const clip = new THREE.AnimationClip("bounce", 2, [track]); // duration in seconds
// pass -1 as the duration to derive it from the tracks
```

Track names follow `nodeName.objectName[objectIndex].propertyName[propertyIndex]`; all parts except the property are optional. `.position` binds the root's position, `.material.opacity` a material property, `.position[y]` one component, `.morphTargetInfluences[smile]` one morph target, `Hips.position` a named descendant — that is how bones are addressed in a skinned model. Object names may be `material`, `materials`, `bones` or `map`.

```javascript
// Number track (value size 1); create the material with transparent: true, or opacity is ignored
new THREE.NumberKeyframeTrack(".material.opacity", [0, 1], [1, 0]);

// Vector track (value size 3: x, y, z per keyframe)
new THREE.VectorKeyframeTrack(".position", [0, 1, 2], [0, 0, 0, 1, 2, 0, 0, 0, 0]);

// Rotation track (value size 4): quaternion x, y, z, w per keyframe, not Euler angles
const q2 = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI);
new THREE.QuaternionKeyframeTrack(
  ".quaternion",
  [0, 1],
  [0, 0, 0, 1, q2.x, q2.y, q2.z, q2.w], // identity, then q2
);

// Color track (value size 3)
new THREE.ColorKeyframeTrack(".material.color", [0, 1], [1, 0, 0, 0, 1, 0]);

// Boolean track (always discrete)
new THREE.BooleanKeyframeTrack(".visible", [0, 1], [true, false]);
```

### Interpolation

```javascript
const track = new THREE.VectorKeyframeTrack(
  ".position",
  [0, 1, 2],
  [0, 0, 0, 1, 2, 0, 0, 0, 0],
);

track.setInterpolation(THREE.InterpolateLinear); // default
track.setInterpolation(THREE.InterpolateSmooth); // cubic through the keyframes
track.setInterpolation(THREE.InterpolateDiscrete); // step, no in-between values

// Bezier takes absolute (time, value) control points, two numbers per keyframe per
// component: a stride-1 two-keyframe track needs four entries per array
const morphTrack = new THREE.NumberKeyframeTrack(
  ".morphTargetInfluences[smile]",
  [0, 1],
  [0, 1],
);
morphTrack.setInterpolation(THREE.InterpolateBezier);
morphTrack.settings = { inTangents: [0, 0, 1, 1], outTangents: [0, 1, 1, 0] };
```

## AnimationMixer

```javascript
const track = new THREE.NumberKeyframeTrack(".rotation[y]", [0, 2], [0, Math.PI * 2]);
const clip = new THREE.AnimationClip("spin", 2, [track]);

// One mixer per animated root; it reaches the root and all descendants
const mixer = new THREE.AnimationMixer(mesh);
const action = mixer.clipAction(clip); // cached: calling it again for this clip and root returns the same action
action.play();

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  mixer.update(timer.getDelta()); // required: advances every active action
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);

// Events fire on the mixer while it updates; "finished" needs LoopOnce or finite repetitions
mixer.addEventListener("finished", (e) => console.log("finished:", e.action.getClip().name));
mixer.addEventListener("loop", (e) => console.log("looped:", e.action.getClip().name));
```

## AnimationAction

```javascript
const mixer = new THREE.AnimationMixer(mesh);
const clip = new THREE.AnimationClip("spin", 2, [
  new THREE.NumberKeyframeTrack(".rotation[y]", [0, 2], [0, Math.PI * 2]),
]);
const action = mixer.clipAction(clip);

action.play(); // schedule and start
action.reset(); // rewind to time 0 and cancel fades
action.stop(); // unschedule; properties no other scheduled action animates return to their original values
action.halt(0.3); // decelerate to a stop within 0.3 s

action.time = 0.5; // scrub within the clip
action.timeScale = -1; // negative plays backward
action.paused = true; // freeze in place

action.setLoop(THREE.LoopOnce, 1); // stop (or hold) after one pass
action.setLoop(THREE.LoopRepeat, Infinity); // the default
action.setLoop(THREE.LoopPingPong, 4); // alternate direction each pass
action.clampWhenFinished = true; // hold the final pose instead of resetting
```

### Fades and Crossfades

```javascript
const mixer = new THREE.AnimationMixer(mesh);
const idleClip = new THREE.AnimationClip("idle", 2, [
  new THREE.NumberKeyframeTrack(".position[y]", [0, 1, 2], [0, 0.05, 0]),
]);
const walkClip = new THREE.AnimationClip("walk", 1, [
  new THREE.NumberKeyframeTrack(".rotation[y]", [0, 1], [0, Math.PI]),
]);

const idle = mixer.clipAction(idleClip).play();
const walk = mixer.clipAction(walkClip).play();

// Hand over from idle to walk over 0.4 s; true as the third argument also warps
// both time scales so their cycle lengths match during the fade
idle.crossFadeTo(walk, 0.4, false);

// The same hand-over from the one-sided calls (use one form, not both)
walk.reset().fadeIn(0.4).play(); // ramp weight up from 0
idle.fadeOut(0.4); // ramp weight down to 0
```

## Playing glTF Animations

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const gltf = await new GLTFLoader().loadAsync("model.glb");
const model = gltf.scene;
scene.add(model);

// GLTFLoader's default material (MeshStandardMaterial) is lit: without lights the model renders black
const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(3, 5, 2);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x444444, 1.5));

// gltf.animations is an array of parsed AnimationClips
const clips = gltf.animations;

const mixer = new THREE.AnimationMixer(model);
const walkClip = THREE.AnimationClip.findByName(clips, "Walk");
if (walkClip) mixer.clipAction(walkClip).play();

// A second character: clone the skinned model, give the copy its own mixer
const model2 = SkeletonUtils.clone(model);
model2.position.x = 1.5; // beside the first one
scene.add(model2);
const mixer2 = new THREE.AnimationMixer(model2);
if (walkClip) mixer2.clipAction(walkClip).play(); // same clip, two mixers

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  const delta = timer.getDelta();
  mixer.update(delta);
  mixer2.update(delta);
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Skeletal Animation

```javascript
// mesh is the loaded model root; find its first SkinnedMesh
const skinnedMesh = mesh.getObjectByProperty("type", "SkinnedMesh");
const skeleton = skinnedMesh.skeleton;
console.log(skeleton.bones.map((b) => b.name));

// Grab a bone to pose or attach to
const handBone = skeleton.bones.find((b) => b.name === "RightHand");

// Visualize the rig
const helper = new THREE.SkeletonHelper(mesh);
scene.add(helper);
```

Procedural bone control runs **after** `mixer.update()`, or a clip that animates the same bone overwrites it:

```javascript
const skinnedMesh = mesh.getObjectByProperty("type", "SkinnedMesh");
const headBone = skinnedMesh.skeleton.bones.find((b) => b.name === "Head");
const mixer = new THREE.AnimationMixer(mesh); // play the model's clips on it (see Playing glTF Animations)

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  mixer.update(timer.getDelta()); // clips write their bone pose here

  // Override afterwards: update() overwrites every property the playing clips animate
  if (headBone) headBone.rotation.y = Math.sin(timer.getElapsed()) * 0.3;
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

Bones are Object3Ds, so attaching means parenting:

```javascript
const skinnedMesh = mesh.getObjectByProperty("type", "SkinnedMesh");
const handBone = skinnedMesh.skeleton.bones.find((b) => b.name === "RightHand");

const weapon = new THREE.Mesh(
  new THREE.BoxGeometry(0.08, 0.08, 0.5),
  new THREE.MeshNormalMaterial(),
);
if (handBone) {
  handBone.add(weapon); // weapon now follows the hand's bone transform
  weapon.position.set(0, 0, 0.25); // offset within the bone's space
  weapon.rotation.set(0, Math.PI / 2, 0);
}
```

## Morph Targets

```javascript
// mesh carries morph targets (loaded, or added by hand below)
const names = Object.keys(mesh.morphTargetDictionary); // name -> index
console.log(names, mesh.morphTargetInfluences); // one weight per target

// Blend a target by index or by name; 0 = base shape, 1 = full target
mesh.morphTargetInfluences[0] = 0.5;
const smileIndex = mesh.morphTargetDictionary["Mouth_Smile"];
if (smileIndex !== undefined) mesh.morphTargetInfluences[smileIndex] = 1;
```

Adding a morph target by hand:

```javascript
const geometry = mesh.geometry;

// A morph target is a full-size position (or normal) attribute beside the base one
const raised = geometry.attributes.position.array.slice(); // same vertex count
for (let i = 1; i < raised.length; i += 3) {
  raised[i] += 0.5; // y is every third component: lift every vertex
}

const target = new THREE.Float32BufferAttribute(raised, 3);
target.name = "raised"; // named targets stay addressable from tracks

geometry.morphAttributes.position = [target];
mesh.updateMorphTargets(); // rebuilds morphTargetInfluences and the dictionary
```

Driving morph targets:

```javascript
// Procedural, inside the render loop after timer.update():
// mesh.morphTargetInfluences[0] = (Math.sin(timer.getElapsed() * 2) + 1) / 2;

// Keyframed: a Number track drives one target by name — never a String track
const track = new THREE.NumberKeyframeTrack(
  ".morphTargetInfluences[raised]",
  [0, 0.5, 1],
  [0, 1, 0],
);
const clip = new THREE.AnimationClip("pulse", 1, [track]);
const mixer = new THREE.AnimationMixer(mesh);
mixer.clipAction(clip).play();
```

## Blending Clips by Weight

```javascript
const mixer = new THREE.AnimationMixer(mesh);
const idleClip = new THREE.AnimationClip("idle", 2, [
  new THREE.NumberKeyframeTrack(".position[y]", [0, 1, 2], [0, 0.05, 0]),
]);
const runClip = new THREE.AnimationClip("run", 1, [
  new THREE.NumberKeyframeTrack(".position[y]", [0, 0.5, 1], [0, 0.3, 0]),
]);

const idle = mixer.clipAction(idleClip).play();
const run = mixer.clipAction(runClip).play();

// Both sample every frame; the weights mix their poses
function setBlend(speed) {
  const t = Math.min(speed / 5, 1); // 0 = idle, 5 and up = full run
  idle.setEffectiveWeight(1 - t);
  run.setEffectiveWeight(t);
}

setBlend(2.5); // half idle, half run
```

### Additive Animation

```javascript
const mixer = new THREE.AnimationMixer(mesh);
const walkClip = new THREE.AnimationClip("walk", 1, [
  new THREE.NumberKeyframeTrack(".rotation[y]", [0, 1], [0, Math.PI]),
]);
const breathClip = new THREE.AnimationClip("breath", 2, [
  new THREE.NumberKeyframeTrack(".position[y]", [0, 1, 2], [0, 0.02, 0]),
]);

// Re-bases every track on its frame 0 and marks the clip additive
THREE.AnimationUtils.makeClipAdditive(breathClip);

mixer.clipAction(walkClip).play(); // normal base clip
mixer.clipAction(breathClip).play(); // adds its offsets on top of the base pose
```

## Clip Utilities

```javascript
const clip = new THREE.AnimationClip("walk", 2, [
  new THREE.NumberKeyframeTrack(".rotation[y]", [0, 1, 2], [0, Math.PI, Math.PI * 2]),
]);

const walk = THREE.AnimationClip.findByName([clip], "walk"); // name lookup
// Keeps the existing keyframes in frames [0, 31) at 30 fps (times 0 and 1); it never resamples
const cut = THREE.AnimationUtils.subclip(clip, "half", 0, 31, 30);
const copy = clip.clone(); // independent deep copy

clip.optimize(); // drops duplicate-time keys, and keys equal to both neighbours on any non-smooth track (Bezier tangents are not compacted)
clip.resetDuration(); // recomputes duration from the longest track — it does not seek
console.log(clip.validate()); // false for a fractional values-to-times ratio, an empty track, unsorted times or NaN
```

## Procedural Motion

```javascript
const timer = new THREE.Timer();
timer.connect(document);

// Exponential approach; a higher lambda snaps faster. Frame-rate independent.
// The same shape drives positions, rotations or any scalar each frame.
let scale = 0;

function animate() {
  timer.update();
  scale = THREE.MathUtils.damp(scale, 1, 4, timer.getDelta());
  mesh.scale.setScalar(0.5 + scale * 0.5);
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Performance

- The same AnimationClip can feed any number of mixers — share one clip set across characters instead of copying it.
- Call `mixer.update()` only for characters that are visible or about to be; an un-updated mixer simply leaves the last pose in place.
- A SkinnedMesh computes its bounding sphere once, the first time culling or raycasting needs it, so a clip that moves it far from that pose can get it culled at the screen edge: after `mixer.update()` call `mixer.getRoot().updateMatrixWorld()` then `skinnedMesh.computeBoundingSphere()` (a CPU pass over every vertex) or set `skinnedMesh.frustumCulled = false`.

```javascript
// Free a mixer and its cached bindings when its model leaves for good
const mixer = new THREE.AnimationMixer(mesh);
mixer.stopAllAction();
mixer.uncacheRoot(mesh); // releases the property bindings created for the model
```

## WebGPU and TSL

Everything above — mixer, actions, bone transforms, morph weights — is computed on the CPU and drives a model under WebGPURenderer unchanged. TSL adds animation on the GPU: `time` (elapsed seconds) and `deltaTime` (seconds since the previous frame) are float uniforms the renderer refreshes every frame, and the oscillators `oscSine`, `oscSquare`, `oscTriangle` and `oscSawtooth` return 0-to-1 waves with a one-second period from a time node, defaulting to `time` (`oscSine(time.mul(0.5))` takes two seconds). Per-vertex motion needs a node material and a subdivided mesh.

```javascript
import * as THREE from "three/webgpu";
import { positionLocal, time, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111418); // the WebGPU canvas is transparent by default

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 3);
camera.lookAt(0, 0, 0);

// Ripple a subdivided plane along its normal (local z) on the GPU; time drives it, no per-frame JS
const material = new THREE.MeshBasicNodeMaterial({ wireframe: true });
material.positionNode = positionLocal.add(
  vec3(0, 0, positionLocal.x.mul(2).sin().mul(time.mul(2).cos()).mul(0.2)),
);
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(3, 3, 32, 32), material);
mesh.rotation.x = -Math.PI / 2;
scene.add(mesh);

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

On a skinned mesh, build displacement on `positionLocal`: skinning, morph targets, batching and instancing are applied to it before `positionNode` runs, so the mixer-driven pose survives. `positionGeometry` is the raw rest-pose attribute — never the base for displacement on an animated mesh.

```javascript
import * as THREE from "three/webgpu";
import { color, mix, oscSine, positionLocal, vec3 } from "three/tsl";

// The pulse rides on top of the skinned pose: positionLocal already carries it
const material = new THREE.MeshBasicNodeMaterial();
material.positionNode = positionLocal.add(vec3(0, oscSine().mul(0.05), 0));

// Oscillators animate any node slot, not just positions
material.colorNode = mix(color(0x202030), color(0xffd24a), oscSine());

const skinnedMesh = scene.getObjectByProperty("type", "SkinnedMesh"); // a loaded animated model
if (skinnedMesh) skinnedMesh.material = material;
```

## Common Mistakes

- `const clock = new THREE.Clock();` → `const timer = new THREE.Timer(); timer.connect(document);`: the Clock class is deprecated in r183 and Timer is its replacement.
- `const elapsed = clock.getElapsedTime(); const delta = clock.getDelta();` → `timer.update(); const delta = timer.getDelta(); const elapsed = timer.getElapsed();`: getElapsedTime() calls getDelta() internally, so the following delta read is about zero and the mixer stalls.
- `requestAnimationFrame(animate);` → `renderer.setAnimationLoop(animate);`: both renderers' docs advise it over a manual loop, and it hands frames to the WebXR session when one starts.
- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";` → `import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";`: the examples/jsm specifier does not resolve under the standard import map.
- `new THREE.NumberKeyframeTrack(".material.opacity", [0, 1, 2], [1, 0])` → `new THREE.NumberKeyframeTrack(".material.opacity", [0, 1], [1, 0])`: values must hold times.length × valueSize entries; with a fractional ratio play() never returns, and any other mismatch samples wrong values.
- `new THREE.StringKeyframeTrack(".morphTargetInfluences[smile]", [0, 1], ["0", "1"])` → `new THREE.NumberKeyframeTrack(".morphTargetInfluences[smile]", [0, 1], [0, 1])`: morph influences are numbers, and a string track writes "0"/"1" strings into the array.
- `headBone.rotation.y = Math.sin(t) * 0.3; mixer.update(delta);` → `mixer.update(delta); headBone.rotation.y = Math.sin(t) * 0.3;`: update() writes the clip's pose over any value set before it.
- `clip.resetDuration()` → `action.reset()`: resetDuration() only recomputes the clip's duration from its longest track, while reset() rewinds the action to time 0 and re-enables it after a one-shot finished.
- `material.positionNode = positionGeometry.add(offset)` → `material.positionNode = positionLocal.add(offset)`: positionGeometry is the raw attribute, so building on it drops skinning, morph targets, batching and instancing.
- `const uTime = uniform(0); uTime.value = timer.getElapsed(); const wave = uTime.mul(2).sin();` → `const wave = time.mul(2).sin();`: the renderer already advances the TSL time node every frame, so a Timer-fed uniform is redundant per-frame JavaScript.
- `const model2 = model.clone();` → `const model2 = SkeletonUtils.clone(model);`: cloning a skinned model shares its skeleton between the copies; each copy needs its own.
- `import { clone } from "three/addons/utils/SkeletonUtils.js";` → `import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";`: the namespace keeps the call as SkeletonUtils.clone(model), which cannot be mistaken for the skeleton-sharing model.clone().
- `new Promise((resolve, reject) => loader.load(url, resolve, undefined, reject))` → `await loader.loadAsync(url)`: loaders already expose the promise form, and loadAsync takes the progress callback as its second argument.

## See Also

- `threejs-fundamentals` — render loop, Timer, renderer and scene setup
- `threejs-loaders` — loading glTF models and their animation clips
- `threejs-shaders` — GPU vertex displacement with TSL or ShaderMaterial
