---
name: threejs-react-three-fiber
description: "Building three.js scenes from React with React Three Fiber and drei: renderer setup, the scene graph as JSX, frame-loop, state and loading hooks, pointer events, and adaptive performance, on WebGL or with WebGPU and TSL. Use when writing React components that render three.js."
---

# Three.js React Three Fiber

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

Targets `@react-three/fiber` 9.8 with React 19 and `@react-three/drei` 10.7. The Canvas creates and owns a WebGLRenderer, a scene and a camera; three.js objects become JSX elements. The Canvas fills its parent element, so give the parent a height. The component fragments below render inside a Canvas like this one; lit materials assume its lights.

```jsx
import { useRef } from "react";
import { Canvas, useFrame } from "@react-three/fiber";
import { OrbitControls } from "@react-three/drei";

function Box() {
  const ref = useRef();
  useFrame((_, delta) => {
    ref.current.rotation.y += delta;
  });
  return (
    <mesh ref={ref} position={[0, 0.5, 0]} castShadow>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="#d9915b" />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas shadows="percentage" camera={{ position: [3, 2, 4], fov: 50 }}>
      <color attach="background" args={["#111827"]} />
      <ambientLight intensity={0.4} />
      <directionalLight position={[3, 4, 2]} intensity={2} castShadow />
      <Box />
      <mesh rotation-x={-Math.PI / 2} receiveShadow>
        <planeGeometry args={[10, 10]} />
        <meshStandardMaterial color="#20293a" />
      </mesh>
      <OrbitControls makeDefault />
    </Canvas>
  );
}
```

## Canvas Props

```jsx
export function App() {
  return (
    <Canvas
      shadows="variance"
      dpr={[1, 2]}
      frameloop="demand"
      flat
      camera={{ position: [0, 2, 6], fov: 45 }}
      gl={{ powerPreference: "high-performance" }}
      onCreated={(state) => state.gl.setClearColor("#111827")}
    >
      {/* scene contents */}
    </Canvas>
  );
}
```

- `camera`: props for the default camera, or a camera instance. Without it you get a `PerspectiveCamera` with fov 75, near 0.1, far 1000, at z 5 looking at the origin; fiber updates aspect (or left/right/top/bottom with `orthographic`) and the projection matrix on every resize. `camera={{ manual: true }}` puts projection entirely in your hands.
- `dpr`: a number, or `[min, max]` clamped against `window.devicePixelRatio`; the range follows display changes.
- `shadows`: `true` enables shadow maps, a string (`'basic'`, `'percentage'`, `'variance'`) picks the filter type, an object merges into the renderer's `shadowMap` properties. `true`, `'soft'` and the default `false` all set PCFSoftShadowMap, which three swaps for PCFShadowMap with a console warning (WebGLRenderer when shadows render, WebGPURenderer on every Canvas); `'percentage'` is the same soft filter without the warning.
- `flat` sets NoToneMapping; without it fiber applies ACESFilmicToneMapping — unlike plain three.js, tone mapping is on by default. `linear` switches the output to LinearSRGBColorSpace; the default is sRGB. `legacy` disables three's ColorManagement.
- `gl`: props for the default `new WebGLRenderer({ canvas, antialias: true, alpha: true, powerPreference: "high-performance" })`, a renderer instance, or a factory function (the WebGPU path below).
- `frameloop`: `'always'` renders every frame, `'demand'` renders only after React changes or `invalidate()`, `'never'` hands the loop over (call the module-level `advance(timestamp)` export with a time in seconds; `useFrame`'s delta is the difference between successive timestamps, so milliseconds make it 1000 times too large).
- `events` replaces the event manager (the default maps DOM pointer events to raycasts); `eventSource` (an element pointer events attach to, default the Canvas wrapper) and `eventPrefix` (`'offset'` by default; `'page'`, `'client'`, `'layer'`, `'screen'` when the page scrolls or transforms the canvas) tune pointer mapping. `onCreated` runs once the renderer exists; `onPointerMissed` fires on clicks that hit nothing.

## The Scene Graph as JSX

Every export of the three namespace is an element: lowercase tag, capitalized class (`<mesh>` → `THREE.Mesh`, `<meshStandardMaterial>` → its material). Props apply to the instance; shorthand values work anywhere three has a `.set()`: `position={[1, 2, 3]}`, `position={new THREE.Vector3(1, 2, 3)}`, `position={5}` (setScalar), `color="#d9915b"`. Dashed props pierce (`rotation-x`, `material-color`, `geometry-attributes-position`); `attach="material-0"` attaches into an array index.

`args` is the constructor argument list; changing it rebuilds the object from scratch. Geometries and materials attach to their parent mesh automatically — no `attach` needed.

```jsx
function Torus() {
  return (
    <mesh position={[0, 1, 0]}>
      <torusKnotGeometry args={[0.7, 0.25, 128, 24]} />
      <meshStandardMaterial
        color="#42699c"
        roughness={0.2}
        metalness={0.8}
        onUpdate={(self) => console.log(self.type)}
      />
    </mesh>
  );
}
```

Mount an object you already hold with `<primitive object={obj} />`; one object per primitive (fiber keeps its per-object state on the object), and clone for copies. Fiber never disposes primitives — their lifetime is yours. `dispose={null}` on any element opts it out of unmount disposal. Register classes outside the three namespace with `extend`:

```jsx
import { extend } from "@react-three/fiber";
import { Water } from "three/addons/objects/Water.js";

extend({ Water });
// then render <water args={[geometry]} /> anywhere in the Canvas
```

In TypeScript the tags `line`, `audio`, `source` and `path` collide with DOM types; write `threeLine`, `threeAudio`, `threeSource`, `threePath` (fiber strips the prefix and resolves the three class).

## Hooks

`useFrame(callback, renderPriority?)` subscribes to the render loop; it runs before the scene renders:

```jsx
import { useRef } from "react";
import { useFrame } from "@react-three/fiber";

function Spinner({ radius = 2 }) {
  const ref = useRef();
  useFrame((state, delta) => {
    ref.current.rotation.y += delta;
    ref.current.position.x = Math.sin(state.clock.elapsedTime) * radius;
  });
  return (
    <mesh ref={ref}>
      <sphereGeometry args={[0.4, 32, 16]} />
      <meshBasicMaterial color="#e8b4a0" />
    </mesh>
  );
}
```

Subscribers with a positive `renderPriority` take rendering into their own hands — the Canvas stops rendering, and your callback must call `state.gl.render(state.scene, state.camera)` itself. At default priority, calling `gl.render` yourself double-renders.

`useThree(selector?)` reads Canvas state reactively; with no selector it returns the whole state (`gl`, `camera`, `scene`, `raycaster`, `pointer`, `size`, `viewport`, `invalidate`, `setDpr`, `setFrameloop`, `controls`, `performance`):

```jsx
import { useThree } from "@react-three/fiber";

const camera = useThree((state) => state.camera);
const invalidate = useThree((state) => state.invalidate);
```

Inside `useFrame` use the `state` argument instead — it is the same state without a re-render. `state.clock` is a `THREE.Clock` that fiber 9.8 creates, so every Canvas logs three's Clock deprecation warning; it does not come from your code.

`useLoader(ClassOrInstance, url, extensions?, onProgress?)` suspends until any three loader resolves and caches by loader and URL; its caller must sit under a Suspense boundary. The extensions callback runs on the one loader instance fiber keeps per class, so a setting it makes (a path, a decoder) applies to every later `useLoader` call with that class. `useGraph(object)` builds named lookup maps (`{ nodes, materials, meshes }`) from an object tree:

```jsx
import { useGraph, useLoader } from "@react-three/fiber";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

function Model({ url }) {
  const gltf = useLoader(GLTFLoader, url, (loader) => loader.setPath("assets/"));
  const { nodes } = useGraph(gltf.scene);
  return <primitive object={gltf.scene} />;
}
```

`useLoader.preload(Class, url)` warms the cache outside React; `useLoader.clear(Class, url)` drops an entry.

## Pointer Events

Objects with handlers are raycast on each pointer event (pointer moves test only objects with move, over, out, enter or leave handlers); handlers fire nearest first, bubbling from the hit object up its parents. The event carries the intersection plus helpers: `object`, `eventObject` (the object that registered the handler), `point`, `distance`, `face`, `instanceId`, `intersections`, `ray`, `camera`, `unprojectedPoint`, `delta` (pixels travelled since pointerdown, for click-like events), `nativeEvent`. Handlers: `onClick`, `onDoubleClick`, `onContextMenu`, `onWheel`, `onPointerUp`, `onPointerDown`, `onPointerMove`, `onPointerOver`, `onPointerOut`, `onPointerEnter`, `onPointerLeave`, `onPointerMissed` (`onPointerCancel` and `onLostPointerCapture` are typed but fiber never calls them on objects).

```jsx
import { useState } from "react";

function Pickable() {
  const [hovered, setHovered] = useState(false);
  return (
    <mesh
      onClick={(event) => {
        event.stopPropagation(); // underlying objects along the ray no longer receive it
        console.log(event.object.name, event.distance, event.point, event.instanceId);
      }}
      onPointerOver={(event) => {
        event.stopPropagation();
        setHovered(true);
      }}
      onPointerOut={() => setHovered(false)}
      onPointerDown={(event) => {
        event.target.setPointerCapture(event.pointerId); // this pointer's events reach the mesh off its surface until release
      }}
      onPointerUp={(event) => {
        event.target.releasePointerCapture(event.pointerId);
      }}
      onPointerMissed={() => console.log("clicked elsewhere")}
    >
      <sphereGeometry args={[0.75, 32, 16]} />
      <meshStandardMaterial color={hovered ? "#e8b4a0" : "#42699c"} />
    </mesh>
  );
}
```

Pointer events work under any renderer: the default raycaster tests CPU geometry in `intersectObject(object, true)`. Pass `raycaster={{ params: { Line: { threshold: 0.1 } } }}` to tune it.

## Loading Assets

Loading hooks suspend; put the boundary above them and the fallback shows while they load.

```jsx
import { Suspense } from "react";
import { Canvas } from "@react-three/fiber";
import { useGLTF, useProgress, Environment } from "@react-three/drei";

function Model({ url }) {
  const { scene, animations, nodes, materials } = useGLTF(url);
  return <primitive object={scene} />;
}

function Hud() {
  const { active, progress, errors } = useProgress(); // tracks three's DefaultLoadingManager
  return <div>{active ? `${progress.toFixed(0)}%` : errors.length > 0 ? "failed" : "done"}</div>;
}

export function App() {
  return (
    <>
      <Hud />
      <Canvas camera={{ position: [3, 2, 4], fov: 50 }}>
        <Suspense fallback={null}>
          <Environment preset="city" />
          <Model url="model.glb" />
        </Suspense>
      </Canvas>
    </>
  );
}
```

- `useGLTF(url)` loads through the GLTFLoader drei ships with (three-stdlib), with Draco and meshopt enabled by default; the Draco decoder comes from a pinned gstatic URL (`useGLTF.setDecoderPath` retargets it, `useGLTF(url, false)` disables it). `useGLTF.preload(url)` warms the cache. The result carries `scene`, `animations`, `cameras` plus `nodes`/`materials`/`meshes` lookup maps. For several copies of a skinned model, clone it once per component with `useMemo(() => SkeletonUtils.clone(scene), [scene])` (`import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js"`) and mount each clone in its own primitive.
- `useTexture(input)` loads one URL, an array (`const [a, b] = useTexture([x, y])`) or a keyed object; fiber sets sRGB color space on LDR (RGBA8) color maps assigned through props — `map`, `emissiveMap`, `envMap` among them — so color maps work without manual `colorSpace`, and scalar maps keep NoColorSpace.
- `<Preload all />` compiles every material once, when it mounts (with `all`, hidden objects' too), and renders the scene into a cube target to upload textures, trading a one-time stall for no first-view hitches; put it inside the Suspense boundary after the assets, or it runs before they load.
- `useProgress` is a plain store, so the Hud above reads it outside the Canvas in the DOM.

## drei Helpers

drei ships many more helpers than these; only this subset is verified here, and every helper imports from `@react-three/drei`.

Controls: `OrbitControls` constructs the controls class drei bundles, connects it to the event source (the Canvas wrapper by default), updates it every frame before user callbacks, calls `invalidate()` on change (so it works with `frameloop="demand"`) and turns damping on by default. `makeDefault` stores it in `state.controls` (Bounds and other helpers pick it up); `regress` drops the adaptive-performance value while dragging. Props pass straight through to the controls instance:

```jsx
import { OrbitControls } from "@react-three/drei";

<OrbitControls
  makeDefault
  enableDamping
  dampingFactor={0.1}
  target={[0, 1, 0]}
  minDistance={2}
  maxDistance={12}
  maxPolarAngle={Math.PI / 2}
  autoRotate
  autoRotateSpeed={0.5}
/>
```

`CameraControls` wraps the camera-controls library the same way (ref methods like `fitToBox` come from that library, and it updates with the frame delta).

Scene dressing:

```jsx
import {
  ContactShadows, Environment, Lightformer, Text, Html, Center, Bounds,
} from "@react-three/drei";

export function Dressing({ overlay }) { // overlay: a ref to a DOM element laid over the Canvas
  return (
    <group>
      <Environment resolution={256}>
        <Lightformer form="rect" intensity={2} position={[0, 4, 3]} scale={[8, 2, 1]} />
      </Environment>
      <ContactShadows position={[0, -1, 0]} opacity={0.6} scale={12} blur={2.4} far={4} />
      <Bounds fit clip observe margin={1.5}>
        <mesh>
          <sphereGeometry args={[1, 32, 16]} />
          <meshStandardMaterial metalness={0.9} roughness={0.2} />
        </mesh>
        <Center top position={[2.5, -1, 0]}>
          <mesh>
            <boxGeometry args={[1, 0.6, 1]} />
            <meshStandardMaterial />
          </mesh>
        </Center>
      </Bounds>
      <Text position={[0.75, 1.1, 0]} fontSize={0.4} anchorX="center" anchorY="bottom" color="#e8b4a0" outlineWidth={0.02}>
        three.js
      </Text>
      <Html portal={overlay} position={[2.5, 0.2, 0]} center distanceFactor={8} occlude="raycast">
        <div>dom overlay anchored to the scene</div>
      </Html>
    </group>
  );
}
```

- `ContactShadows` renders the scene from below into a depth target and blurs it — a soft contact shadow plane under objects near the floor, with no shadow-map setup; set `frames={1}` to bake it once for still scenes.
- `Lightformer` is a card of light — a mesh with a basic material, `form` `circle`, `ring`, `rect` or `box` — meant to sit inside an `<Environment>` children scene that bakes into reflections.
- `Text` renders SDF text (troika) with layout props `maxWidth`, `textAlign`, `lineHeight`, `letterSpacing`, `characters`; it suspends while the font loads.
- `Html` projects a DOM node into the scene; `transform` renders it as a 3D-rotated overlay instead of a screen-facing one. `occlude` (`true` or `"raycast"`) hides it while geometry blocks the ray to its anchor, so never anchor it inside a mesh; `occlude="blending"` hides it by depth through a ShaderMaterial (WebGL only). With fiber 9.8, drei 10.7 and React 19.2, an `Html` that mounts in the same commit as a default (WebGLRenderer) Canvas comes up empty, and so does one that remounts with its Canvas; pass `portal` a ref to a DOM element laid over the canvas, which keeps its container fixed, as the Dressing does: `<div ref={overlay} style={{ position: "absolute", inset: 0, pointerEvents: "none" }} />` after the Canvas, inside a `position: relative` wrapper.
- `Center` recenters children on their bounding box (`top`/`bottom`/`left`/`right`/`front`/`back` align to an edge, `precise={false}` measures per-object bounds instead of every vertex, `onCentered` reports the box). It measures vertex positions, so it misplaces drei `Text`, whose geometry is a unit quad; place Text with `anchorX`, `anchorY` and `position` instead.
- `Bounds` fits and frames the camera around its children (`fit`, `clip` adjusts near/far, `observe` refits on resize); `useBounds()` returns its imperative API (`refresh`, `reset`, `moveTo`, `lookAt`, `fit`, `clip`).

## Performance

Render on demand: nothing changes, nothing renders.

```jsx
import { useEffect, useRef } from "react";
import { useThree } from "@react-three/fiber";

function OnDemand() {
  const ref = useRef();
  const invalidate = useThree((state) => state.invalidate);
  // under <Canvas frameloop="demand"> imperative changes are invisible to React — request a frame for them
  useEffect(() => {
    const id = setInterval(() => {
      ref.current.rotation.y += 0.5;
      invalidate();
    }, 100);
    return () => clearInterval(id);
  }, [invalidate]);
  return (
    <mesh ref={ref}>
      <boxGeometry args={[1, 1, 1]} />
      <meshNormalMaterial />
    </mesh>
  );
}
```

React changes (props, state) invalidate automatically. Never call `setState` in `useFrame` — mutate objects through refs, as the Quick Start does; a store or `flushSync` covers the rare case the DOM must follow per-frame values.

Many copies of one shape share one draw call through `Instances`; each `Instance` is a declarative transform (and optional `color`) that composes into the instanced matrix every frame:

```jsx
import { Instance, Instances } from "@react-three/drei";

function Crowd({ count = 200 }) {
  return (
    <Instances limit={count} castShadow>
      <capsuleGeometry args={[0.25, 0.8, 4, 12]} />
      <meshStandardMaterial />
      {Array.from({ length: count }, (_, i) => (
        <Instance
          key={i}
          position={[(i % 15) - 7, 0.65, Math.floor(i / 15) - 7]}
          color={`hsl(${(i * 37) % 360}, 60%, 55%)`}
        />
      ))}
    </Instances>
  );
}
```

`Detailed` picks one of its children by camera distance (three's `LOD`), for cheap far-field geometry: child i shows from `distances[i]` on (the first entry is 0), and `hysteresis` is a fraction of that distance that delays the switch back to finer detail:

```jsx
import { Detailed } from "@react-three/drei";

<Detailed distances={[0, 15, 40]} hysteresis={0.1}>
  <mesh><sphereGeometry args={[1, 64, 32]} /><meshStandardMaterial /></mesh>
  <mesh><sphereGeometry args={[1, 16, 8]} /><meshStandardMaterial /></mesh>
  <mesh><sphereGeometry args={[1, 8, 4]} /><meshStandardMaterial /></mesh>
</Detailed>
```

Adaptive resolution: controls with `regress` drop `performance.current` to `min` (default 0.5) while dragging and restore it after 200 ms; `AdaptiveDpr` sets the pixel ratio to that value times the first ratio the Canvas resolved, so every drag ends at that first ratio. `PerformanceMonitor` samples real frame rates and reports through callbacks — it does not touch the pixel ratio itself. Drive the Canvas `dpr` prop from its callbacks through React state, because the Canvas re-applies that prop whenever it re-renders, which undoes a `setDpr` call made from inside it; keep `AdaptiveDpr` out of that Canvas, or the next drag restores the first ratio over a lowered `dpr`:

```jsx
import { useState } from "react";
import { Canvas } from "@react-three/fiber";
import { OrbitControls, AdaptiveDpr, PerformanceMonitor } from "@react-three/drei";

// lower the resolution while the controls move
export function DragTunedApp() {
  return (
    <Canvas dpr={[1, 2]}>
      <OrbitControls regress />
      <AdaptiveDpr pixelated />
      {/* scene contents */}
    </Canvas>
  );
}

// let the measured frame rate pick the resolution through the dpr prop
export function MonitoredApp() {
  const [dpr, setDpr] = useState([1, 2]);
  return (
    <Canvas dpr={dpr}>
      <OrbitControls />
      <PerformanceMonitor
        onDecline={() => setDpr(1)}
        onIncline={() => setDpr([1, 2])}
        flipflops={3}
        onFallback={() => setDpr(1)}
      />
      {/* scene contents */}
    </Canvas>
  );
}
```

Beyond fiber-specific tools: cap `dpr`, prefer KTX2 textures, merge or instance geometry, dispose what you unmount — see `threejs-performance`.

## Disposal

Fiber disposes what it creates when it unmounts (scheduled for idle time): the object behind every JSX element, geometries, materials and textures declared as JSX children included, and the renderer it built from `gl` props or a factory. Objects passed in through props (`geometry={...}`, `map={...}`) and primitives are never disposed, and `dispose={null}` exempts an element and its children. A renderer instance you passed in is not disposed, but fiber forces its context loss on unmount, so it cannot be reused. Assets loaded through `useLoader`, `useGLTF` and `useTexture` stay cached for remounts; the hook's `clear` only drops the cache entry, so dispose the geometries, materials and textures you no longer need yourself. The module-level `dispose(object)` export calls `dispose()` on the object and on each of its own properties that has one (a mesh's geometry and material); it does not visit children, so traverse a tree yourself.

## WebGPU and TSL

The `gl` prop takes an async factory: construct the WebGPURenderer with the props fiber hands you, await its initialization, return it — fiber waits for the promise inside the Canvas and renders the children once it resolves (a Suspense fallback around the Canvas does not show meanwhile). Because the Canvas seeds its element catalogue from the plain three namespace (which has no node materials), merge the `three/webgpu` namespace in with `extend` before using node-material elements. TSL nodes from `three/tsl` pass through props the same way, but a material compiles its nodes once: build nodes once (module scope or `useMemo`) and animate through `uniform()` values, because a new node passed on a later render is assigned without a recompile until you set `material.needsUpdate = true`:

```jsx
import * as THREE from "three/webgpu";
import { color } from "three/tsl";
import { Canvas, extend, useFrame } from "@react-three/fiber";
import { useRef } from "react";

extend(THREE); // node materials and WebGPU classes join the element catalogue

function Knot() {
  const ref = useRef();
  useFrame((_, delta) => {
    ref.current.rotation.y += delta;
  });
  return (
    <mesh ref={ref}>
      <torusKnotGeometry args={[0.8, 0.3, 128, 24]} />
      <meshBasicNodeMaterial colorNode={color(0x42699c)} />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas
      camera={{ position: [3, 2, 4], fov: 50 }}
      gl={async (props) => {
        const renderer = new THREE.WebGPURenderer(props);
        await renderer.init();
        return renderer;
      }}
    >
      <color attach="background" args={["#111827"]} />
      <Knot />
    </Canvas>
  );
}
```

Fiber applies its color defaults to a factory's renderer too: ACESFilmicToneMapping (NoToneMapping with `flat`) and sRGB output. WebGPURenderer tone-maps the whole frame, background included, so the same `<color attach="background">` renders darker than under WebGLRenderer unless the Canvas is `flat`.

Checked by rendering under WebGPURenderer on its WebGPU backend and with `forceWebGL: true` (fiber 9.8.2, drei 10.7.9). Working on both backends: the scene graph, hooks, events (the raycaster tests CPU geometry, so `positionNode` displacement is invisible to it), `useGLTF` (loaded materials convert to node materials), `useTexture`, `Preload`, `Center`, `Bounds`, `Instances`, `Detailed`, `OrbitControls`, `CameraControls`, `Html` without `occlude="blending"`, and `Environment` with `preset`, `map` or `.hdr` files; `useProgress`, `AdaptiveDpr` and `PerformanceMonitor` call no renderer API (read in their source).

Not WebGPU-safe: `Text` draws one untextured quad instead of glyphs (troika injects its SDF shader through `onBeforeCompile`), `ContactShadows` (an `onBeforeCompile` depth material and blur `ShaderMaterial`s, which three reports as not compatible), `Html` with `occlude="blending"`, `Environment` with `ground` or with `.jpg`/`.webp` gainmap `files` (both draw through ShaderMaterials; read in the source), and `Environment` with children, which bakes through a cube camera that gives reflections on the WebGPU backend but leaves them black on the WebGL 2 fallback, with no error. TSL material slots (`colorNode`, `positionNode`) behave exactly as in `threejs-shaders` — write `Fn` code, not GLSL.

## Common Mistakes

- `requestAnimationFrame(tick)` → `useFrame((state, delta) => { ... })`: the Canvas already runs the frame loop; a separate loop races it and bypasses its effects.
- `state.clock.getElapsedTime()` → `state.clock.elapsedTime`: getElapsedTime() calls getDelta() internally, which shortens the delta fiber measures for the next frame.
- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"` → `import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"`: `three/addons/` resolves both under bundlers and under the import maps three.js documents, while the `examples/jsm` path fails under those import maps.
- `useFrame(() => { ref.current.rotation.y += 0.01; })` → `useFrame((_, delta) => { ref.current.rotation.y += delta; })`: the second argument is the frame delta; a fixed increment runs at a different speed on every refresh rate.
- `useFrame((_, delta) => setRotation((r) => r + delta))` → `useFrame((_, delta) => { ref.current.rotation.y += delta; })`: state set in the frame loop re-renders React each frame; mutate objects through refs.
- `useFrame((state, delta) => { ref.current.rotation.y += delta; }, 1)` → `useFrame((_, delta) => { ref.current.rotation.y += delta; })`: a positive render priority makes the Canvas stop rendering, and the scene goes black unless your callback calls `state.gl.render(state.scene, state.camera)` itself.
- `<mesh position={{ x: 1, y: 0, z: 0 }}>` → `<mesh position={[1, 0, 0]}>`: `position` is a read-only property holding a Vector3, so a plain object throws "Cannot assign to read only property" and the Canvas fails, while arrays, numbers and Vector3s are written into the existing Vector3 (`fromArray`, `setScalar`, `copy`).
- `useEffect(() => { new GLTFLoader().load(url, (gltf) => setScene(gltf.scene)); }, [])` → `const gltf = useGLTF(url)`: useGLTF suspends, caches the parsed model by URL for remounts and preloading, and wires Draco and meshopt decoding, while the effect version refetches and reparses on every mount and has no Draco decoder.
- `useGLTF(url).scene.clone()` → `useMemo(() => SkeletonUtils.clone(scene), [scene])`: Object3D.clone() shares one skeleton, so copies stay bound to the original's bones, while SkeletonUtils rebinds each copy and useMemo keeps one clone per component instead of one per render.
- `scene.add(gltf.scene)` → `<primitive object={gltf.scene} />`: an imperatively added object takes no fiber event props and stays in the scene after the component unmounts, while a primitive takes event props and is added and removed with the component (its disposal stays yours).
- `<water args={[geometry]} />` → `extend({ Water }); <water args={[geometry]} />`: elements resolve against the three namespace only; addon classes must be registered first.
- `<line>` → `<threeLine>`: the bare tag collides with the SVG line element in TypeScript; fiber strips the leading `three` and builds THREE.Line.
- `<Canvas shadows>` → `<Canvas shadows="percentage">`: the boolean selects PCFSoftShadowMap, deprecated in r182 — three falls back to PCF with a console warning; the string picks the type with no warning.
- `<directionalLight intensity={2} />` → `<directionalLight castShadow intensity={2} />`: a mesh's castShadow only draws shadows when a light also casts and the Canvas shadows flag is set.
- `ref.current.position.x += 1` → `ref.current.position.x += 1; invalidate();`: under frameloop="demand" a bare mutation never repaints, so call `invalidate` from `useThree((state) => state.invalidate)` after imperative changes, while React prop changes invalidate themselves.
- `gl={new THREE.WebGPURenderer()}` → `gl={async (props) => { const renderer = new THREE.WebGPURenderer(props); await renderer.init(); return renderer; }}`: a renderer built without fiber's canvas draws into a canvas of its own that is never shown, and its render() throws until init() resolves, while the factory gets the Canvas's canvas and fiber renders only after it resolves.
- `new OrbitControls(camera, renderer.domElement)` → `<OrbitControls makeDefault />`: the drei component wires per-frame updates, invalidation and disposal; the raw class stalls with damping on and never invalidates a demand loop.
- `<Environment><Lightformer /></Environment>` → `<Environment files="env.hdr" />`: under WebGPURenderer the children form bakes through a cube camera whose reflections come out black on the WebGL 2 fallback backend with no error, while `.hdr` files and presets only assign a texture and work on both backends.

## See Also

- `threejs-fundamentals` - the renderer, scene and camera concepts the Canvas manages for you
- `threejs-loaders` - loader configuration, compression and disposal in depth
- `threejs-interaction` - raycasting and controls beyond fiber's pointer events
- `threejs-shaders` - TSL node materials and slots used inside fiber elements
- `threejs-performance` - profiling and optimization beyond fiber's adaptive tools
- `threejs-asset-pipeline` - gltfjsx components, and compressing models before useGLTF loads them
