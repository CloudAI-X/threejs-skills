---
name: threejs-interaction
description: "Picking with raycasting, camera controls, transform gizmos, drag, hover and box selection for three.js, identical under WebGL and WebGPU. Use when handling mouse, touch or keyboard input, detecting clicks on 3D objects, adding camera controls, or making scene objects interactive."
---

# Three.js Interaction

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 1.5, 5);

const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x3a86ff }),
);
scene.add(mesh);
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 5, 4);
scene.add(light, new THREE.AmbientLight(0xffffff, 0.4));

const controls = new OrbitControls(camera, renderer.domElement); // connects itself
controls.enableDamping = true; // requires controls.update() every frame

const timer = new THREE.Timer();
timer.connect(document); // freezes the delta while the tab is hidden

const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

function setPointer(event) {
  // NDC coordinates relative to the canvas, so the page can scroll or embed it
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.set(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
  );
}

renderer.domElement.addEventListener("pointerdown", (event) => {
  setPointer(event);
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObject(mesh); // also tests children (recursive by default), as a loaded model needs
  if (hits.length > 0) console.log("clicked", hits[0].object);
});

function animate() {
  timer.update();
  controls.update(timer.getDelta());
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

Pointer events (`pointerdown`, `pointermove`, `pointerup`) cover mouse, touch and pen in one listener set, so no separate touch handlers are needed.

## Raycasting

Faces pointing away from the ray origin are not hit; set `material.side = THREE.DoubleSide` to catch both sides. A ray that grazes an edge shared by two faces can produce two coincident hits. Invisible meshes are still raycast — the Raycaster tests layers, not visibility — so an invisible simple proxy can stand in for a complex model.

### What an intersection contains

```javascript
const raycaster = new THREE.Raycaster();
const clickables = [mesh]; // the smallest list you can: the meshes, not scene.children
const hits = raycaster.intersectObjects(clickables, false); // sorted, closest first
// Each hit:
// {
//   distance: number,                          // from the ray origin, world units
//   point: THREE.Vector3,                      // hit point in world space
//   face: { a, b, c, normal, materialIndex },  // plain object; a, b, c are vertex indices; normal in object space
//   faceIndex: number,
//   object: THREE.Object3D,
//   uv: THREE.Vector2,                         // undefined when the geometry has no uv
//   uv1: THREE.Vector2,                        // second UV set, when the geometry has one
//   normal: THREE.Vector3,                     // interpolated vertex normal in object space, flipped to face the ray
//   barycoord: THREE.Vector3,                  // barycentric coordinates of the hit point
//   instanceId: number,                        // set on InstancedMesh hits
//   batchId: number,                           // set on BatchedMesh hits
//   distanceToRay: number                      // set on Points hits
// }
```

### Raycaster configuration

```javascript
const raycaster = new THREE.Raycaster();

// From any origin and direction (normalize the direction first)
raycaster.set(new THREE.Vector3(0, 2, 0), new THREE.Vector3(1, 0, 0).normalize());

// Range: hits closer than near or farther than far are discarded
raycaster.near = 0.1;
raycaster.far = 10;

// Precision for Points and Line, in world units (both default to 1)
raycaster.params.Points.threshold = 0.2;
raycaster.params.Line.threshold = 0.2;

// Restrict picking to layer 1. Objects join with enable; set would also remove
// them from layer 0, where the default camera looks, so use set only on a pick-only proxy.
mesh.layers.enable(1); // stays on layer 0 too, so the camera still renders the mesh
raycaster.layers.set(1); // tests layer 1 only; raycaster.layers.enable(1) would test layers 0 and 1
```

### Dragging on a plane

```javascript
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const groundPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0); // normal, constant
const hit = new THREE.Vector3();

renderer.domElement.addEventListener("pointermove", (event) => {
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.set(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
  );
  raycaster.setFromCamera(pointer, camera);
  // null when the ray is parallel to the plane or points away from it
  if (event.buttons !== 0 && raycaster.ray.intersectPlane(groundPlane, hit) !== null) {
    mesh.position.copy(hit); // while a button is held, drag the mesh along the ground
  }
});
```

## Camera Controls

Every controls class extends `Controls` and shares one base API: construct with the camera and the DOM element (DragControls takes its array of objects first), and the constructor wires its own listeners; `enabled = false` to ignore input, `disconnect()` / `connect(element)` to detach or move, `dispose()` on teardown. `update()` takes a delta in seconds for OrbitControls, FlyControls and FirstPersonControls — pass `timer.getDelta()`; call it without arguments for the others.

### OrbitControls

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const controls = new OrbitControls(camera, renderer.domElement);

controls.enableDamping = true; // requires update() every frame
controls.dampingFactor = 0.05;

controls.minDistance = 2;
controls.maxDistance = 50;
controls.minPolarAngle = 0.1;          // vertical limits, radians
controls.maxPolarAngle = Math.PI / 2;  // stop at the horizon
controls.minAzimuthAngle = -Math.PI / 4;
controls.maxAzimuthAngle = Math.PI / 4;

controls.target.set(0, 1, 0);  // the point orbited around
controls.zoomToCursor = true; // wheel zooms toward the pointer

controls.autoRotate = true;      // requires update() every frame; stops at an azimuth limit
controls.autoRotateSpeed = 2.0;

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  controls.update(timer.getDelta()); // the delta makes autoRotate frame-rate independent
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

OrbitControls fires `change`, `start` and `end` events. `MapControls` is an OrbitControls subclass tuned for map navigation:

```javascript
import { MapControls } from "three/addons/controls/MapControls.js";

// Left-drag pans, right-drag (or left-drag with ctrl/meta/shift) orbits,
// middle button or wheel zooms. screenSpacePanning is false, so panning
// stays parallel to the ground. Every OrbitControls option applies.
const controls = new MapControls(camera, renderer.domElement);
controls.enableDamping = true;
```

### FlyControls

```javascript
import { FlyControls } from "three/addons/controls/FlyControls.js";

const controls = new FlyControls(camera, renderer.domElement);
controls.movementSpeed = 10;
controls.rollSpeed = Math.PI / 24;
controls.dragToLook = true; // steer only while a mouse button is held

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  controls.update(timer.getDelta()); // movement scales by the delta
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

### FirstPersonControls

FirstPersonControls has a drag-based look model: press and drag to look (look speed grows with the drag distance and eases through `dampingFactor`, default 0.1, where 1 disables the easing), WASD or arrows to move, R and F for up and down, hold the left button to fly forward and the right button to fly backward. Touch works through pointer events: one finger moves forward, two fingers move backward.

```javascript
import { FirstPersonControls } from "three/addons/controls/FirstPersonControls.js";

const controls = new FirstPersonControls(camera, renderer.domElement);
controls.movementSpeed = 10;
controls.lookSpeed = 0.005;   // default
controls.dampingFactor = 0.1; // default; 1 disables the easing
controls.constrainVertical = true;
controls.verticalMin = Math.PI / 4;
controls.verticalMax = (Math.PI * 3) / 4;

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  controls.update(timer.getDelta()); // the delta is required; without it the position becomes NaN
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

### PointerLockControls

```javascript
import { PointerLockControls } from "three/addons/controls/PointerLockControls.js";

const controls = new PointerLockControls(camera, document.body);
controls.pointerSpeed = 1.0; // mouse sensitivity

controls.addEventListener("lock", () => console.log("locked"));
controls.addEventListener("unlock", () => console.log("unlocked"));
document.body.addEventListener("click", () => controls.lock()); // needs a user gesture

const velocity = new THREE.Vector3();
let moveForward = false; // let: the key handlers reassign these
let moveBackward = false;

document.addEventListener("keydown", (event) => {
  if (event.code === "KeyW") moveForward = true;
  if (event.code === "KeyS") moveBackward = true;
});
document.addEventListener("keyup", (event) => {
  if (event.code === "KeyW") moveForward = false;
  if (event.code === "KeyS") moveBackward = false;
});

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  const delta = timer.getDelta();
  if (controls.isLocked) {
    // ease toward the target speed, then move the camera forward parallel to the ground (xz-plane)
    velocity.z = THREE.MathUtils.damp(
      velocity.z,
      10 * (Number(moveForward) - Number(moveBackward)),
      10,
      delta,
    );
    controls.moveForward(velocity.z * delta);
    // controls.moveRight(distance) does the same along the camera's right axis
  }
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

The controls move the camera passed to the constructor (`controls.object`); it renders without being added to the scene, so `scene.add(controls.object)` is needed only when the camera has children.

### TrackballControls

```javascript
import { TrackballControls } from "three/addons/controls/TrackballControls.js";

const controls = new TrackballControls(camera, renderer.domElement);
controls.rotateSpeed = 2.0;
controls.zoomSpeed = 1.2;
controls.panSpeed = 0.8;
controls.staticMoving = true; // no inertia; otherwise dynamicDampingFactor (default 0.2)

function animate() {
  controls.update(); // no delta argument; call controls.handleResize() after the canvas resizes
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

## TransformControls

A gizmo for moving, rotating and scaling objects. The controls object is not an Object3D; the gizmo it renders must be added to the scene separately.

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { TransformControls } from "three/addons/controls/TransformControls.js";

const orbitControls = new OrbitControls(camera, renderer.domElement);

const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls.getHelper()); // the gizmo; adding the controls themselves only logs an error

transformControls.attach(mesh); // show the gizmo on this object
transformControls.setMode("translate"); // "translate" | "rotate" | "scale"
transformControls.setSpace("local");    // "local" | "world"; scale always acts locally
transformControls.setSize(0.8);
transformControls.setTranslationSnap(0.5);
transformControls.setRotationSnap(THREE.MathUtils.degToRad(15));
transformControls.setScaleSnap(0.25);

transformControls.addEventListener("dragging-changed", (event) => {
  orbitControls.enabled = !event.value; // pause orbiting while the gizmo is dragged
});
transformControls.addEventListener("objectChange", () => {
  console.log(mesh.position); // fires as the gizmo moves the object
});

window.addEventListener("keydown", (event) => {
  if (event.key === "g") transformControls.setMode("translate");
  if (event.key === "r") transformControls.setMode("rotate");
  if (event.key === "s") transformControls.setMode("scale");
  if (event.key === "Escape") transformControls.detach(); // hide the gizmo
});
```

## DragControls

Drags objects directly: left or middle drag moves the object in a plane parallel to the screen, right drag rotates it around the camera's up and right axes.

```javascript
import { DragControls } from "three/addons/controls/DragControls.js";

const dragControls = new DragControls([mesh], camera, renderer.domElement);
dragControls.rotateSpeed = 1; // default
dragControls.mouseButtons = { LEFT: THREE.MOUSE.PAN, MIDDLE: THREE.MOUSE.PAN, RIGHT: THREE.MOUSE.ROTATE }; // the default map
// transformGroup = true drags the outermost Group above the hit instead (pass objects = [group]); event.object is then that Group, which has no material, so use event.object.material?.emissive in the handlers

dragControls.addEventListener("dragstart", (event) => {
  event.object.material.emissive?.set(0x333333);
});
dragControls.addEventListener("drag", (event) => {
  event.object.position.y = 0; // clamp to the ground mid-drag
});
dragControls.addEventListener("dragend", (event) => {
  event.object.material.emissive?.set(0x000000);
});

// The built-in picker: dragControls.raycaster. The dragged objects: dragControls.objects.
```

## Selection

```javascript
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const selectable = [mesh];
let selected = null;

function setPointerFromEvent(event) {
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.set(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
  );
}

renderer.domElement.addEventListener("pointerdown", (event) => {
  setPointerFromEvent(event);
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObjects(selectable, false)[0];
  if (selected) selected.material.emissive.set(0x000000); // deselect the previous mesh
  selected = hit ? hit.object : null;
  if (selected) selected.material.emissive.set(0x444444);
});

renderer.domElement.addEventListener("pointermove", (event) => {
  setPointerFromEvent(event);
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObjects(selectable, false)[0];
  document.body.style.cursor = hit ? "pointer" : "default"; // hover feedback
});
```

For teardown, keep named handler references so you can `removeEventListener` them, and call `controls.dispose()` for every control and `timer.dispose()` for the timer.

### Box selection

`SelectionBox` selects the objects whose bounding-sphere center falls inside a screen-space rectangle; `SelectionHelper` draws the rectangle and tracks the drag.

```javascript
import { SelectionBox } from "three/addons/interactive/SelectionBox.js";
import { SelectionHelper } from "three/addons/interactive/SelectionHelper.js";

const selectionBox = new SelectionBox(camera, scene);
const helper = new SelectionHelper(renderer, "selectBox"); // a div with this class: give .selectBox position: fixed, a border and a background in CSS, or nothing shows

function ndc(event) {
  return new THREE.Vector3(
    (event.clientX / window.innerWidth) * 2 - 1,
    -(event.clientY / window.innerHeight) * 2 + 1,
    0.5,
  );
}

document.addEventListener("pointerdown", (event) => {
  selectionBox.startPoint.copy(ndc(event));
});

document.addEventListener("pointermove", (event) => {
  if (helper.isDown) selectionBox.endPoint.copy(ndc(event));
});

document.addEventListener("pointerup", (event) => {
  selectionBox.endPoint.copy(ndc(event));
  const selected = selectionBox.select(); // InstancedMesh and BatchedMesh hits go to selectionBox.instances and .batches
  console.log(selected);
});

// on teardown: helper.dispose() removes the helper's listeners and its rectangle element
```

## Screen and World Coordinates

```javascript
// World to screen: project, then map NDC (-1..1) to CSS pixels
const ndc = mesh.getWorldPosition(new THREE.Vector3()).project(camera);
const screenX = ((ndc.x + 1) / 2) * window.innerWidth;
const screenY = ((-ndc.y + 1) / 2) * window.innerHeight;
// ndc.z outside [-1, 1] means the point is outside the near-to-far range
```

```javascript
// Screen to world: unproject into a ray, then walk it to the plane z = 0
const worldPoint = new THREE.Vector3(0.1, 0.2, 0.5).unproject(camera);
const direction = worldPoint.sub(camera.position).normalize();
const distance = -camera.position.z / direction.z; // the ray misses the plane when direction.z is 0
const world = camera.position.clone().addScaledVector(direction, distance);
// The same result comes from a Raycaster built with setFromCamera and intersectPlane
```

## WebGPU and TSL
Controls and Raycaster are renderer-neutral: the picking and camera-control code above runs unchanged under WebGPURenderer. Raycaster tests CPU-side geometry, so it never sees `positionNode` displacement — pick against an undisplaced proxy mesh instead.
```javascript
import * as THREE from "three/webgpu";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
const renderer = new THREE.WebGPURenderer({ antialias: true }); await renderer.init(); // required before any other use
renderer.setPixelRatio(window.devicePixelRatio); renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene(); scene.background = new THREE.Color(0x111318); // WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100); camera.position.set(0, 1.5, 5);
scene.add(new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshBasicMaterial({ color: 0x3a86ff }))); // unlit: no light needed
const controls = new OrbitControls(camera, renderer.domElement);
const timer = new THREE.Timer(); timer.connect(document);
renderer.setAnimationLoop(function animate() {
  timer.update();
  controls.update(timer.getDelta());
  renderer.render(scene, camera);
});
```

## Common Mistakes

- `scene.add(transformControls)` → `scene.add(transformControls.getHelper())`: TransformControls extends Controls, an EventDispatcher rather than an Object3D, so this logs an error and the gizmo never appears.
- `new OrbitControls(camera)` → `new OrbitControls(camera, renderer.domElement)`: the constructor wires its listeners to the element; without one the controls never receive input.
- `controls.update(clock.getDelta())` → `timer.update(); controls.update(timer.getDelta())`: Clock is deprecated in r183; create a Timer, connect it, and update it first thing each frame.
- `import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"` → `import { OrbitControls } from "three/addons/controls/OrbitControls.js"`: the documented import maps point `three/addons/` at examples/jsm/ and define no `three/examples/` key, so this specifier does not resolve in the browser.
- `requestAnimationFrame(animate)` → `renderer.setAnimationLoop(animate)`: both renderers' docs advise it over a manual loop, and it hands frames to the WebXR session when one starts.
- `flyControls.update()` → `flyControls.update(timer.getDelta())`: FlyControls and FirstPersonControls scale movement by the delta, so without it the position becomes NaN.
- `orbitControls.update()` → `orbitControls.update(timer.getDelta())`: without a delta, autoRotate speed depends on the display refresh rate.
- `mesh.layers.set(1)` → `mesh.layers.enable(1)`: `set` replaces the mask, removing the mesh from layer 0, so the default camera stops rendering it.
- `hit.face instanceof THREE.Face3` → `hit.face.normal`: the intersection's face is a plain `{ a, b, c, normal, materialIndex }` object; Face3 does not exist.
- `const moveForward = false` → `let moveForward = false`: the keydown handler reassigns the flag, and assigning to a const throws a TypeError.
- `controls.getObject()` → `controls.object`: PointerLockControls.getObject() was removed in r168; the controls manage the camera passed to the constructor.
- `dragControls.getObjects()` → `dragControls.objects`: removed in r168; getRaycaster() became the raycaster property the same way.
- `dragControls.activate()` → `dragControls.connect(renderer.domElement)`: renamed from activate() in r168 (deactivate() became disconnect()), and connect() needs the element: without one it throws a TypeError, as the old names do.
- `firstPersonControls.activeLook = false` → `firstPersonControls.lookSpeed = 0`: FirstPersonControls has no activeLook property, so assigning it does nothing and drag look stays on.

## See Also

- `threejs-fundamentals` - scenes, cameras, renderer setup and the animation loop
- `threejs-animation` - animating objects in response to interaction
- `threejs-postprocessing` - outline and bloom effects for selection highlights
- `threejs-loaders` - loading the glTF models you make interactive
- `threejs-react-three-fiber` - pointer-event props and drei controls when the scene is written in React
