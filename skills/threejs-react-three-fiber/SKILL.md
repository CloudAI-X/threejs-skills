---
name: threejs-react-three-fiber
description: React Three Fiber (@react-three/fiber) and drei patterns - useFrame, on-demand rendering, avoiding re-renders, LOD, preloading, Suspense, cleanup on unmount. Use when building Three.js scenes inside React, editing files with <Canvas>, useFrame, useThree, useGLTF, or debugging R3F performance and re-render problems.
---

# React Three Fiber

R3F renders a Three.js scene graph with React's reconciler. The single most
important rule: **React's render cycle and the animation loop are different
clocks.** Almost every R3F performance problem comes from mixing them.

## Quick Start

```jsx
import { Canvas, useFrame } from "@react-three/fiber";
import { OrbitControls } from "@react-three/drei";
import { useRef } from "react";

function Box() {
  const ref = useRef();
  useFrame((state, delta) => {
    ref.current.rotation.y += delta * 0.5;
  });
  return (
    <mesh ref={ref}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="orange" />
    </mesh>
  );
}

export default function Scene() {
  return (
    <Canvas camera={{ position: [0, 0, 5], fov: 75 }}>
      <ambientLight intensity={0.5} />
      <directionalLight position={[5, 5, 5]} />
      <Box />
      <OrbitControls />
    </Canvas>
  );
}
```

## The Core Rule: Mutate, Don't setState

Calling `setState` at 60fps triggers 60 React reconciliations per second. Mutate
the object directly through a ref instead — R3F reads the live scene graph.

```jsx
// Bad: a React re-render every frame
const [rotation, setRotation] = useState(0);
useFrame(() => setRotation((r) => r + 0.01));

// Good: direct mutation, zero re-renders
const ref = useRef();
useFrame(() => {
  ref.current.rotation.x += 0.01;
});
```

This applies to anything animated: position, scale, material color, uniform
values, camera. Reserve `setState` for things that genuinely change the
*structure* of the scene.

## Never Allocate Inside useFrame

`useFrame` runs 60+ times a second. Every object created there becomes garbage,
and GC pauses show up as frame hitches.

```jsx
// Bad: a new Vector3 every frame
useFrame(() => {
  mesh.current.position.copy(new THREE.Vector3(1, 2, 3));
});

// Good: allocate once, reuse forever
const target = useMemo(() => new THREE.Vector3(1, 2, 3), []);
useFrame(() => {
  mesh.current.position.copy(target);
});
```

The same applies to `new THREE.Color()`, `new THREE.Matrix4()`, `new THREE.Quaternion()`,
array literals, and object literals passed to helpers.

## Use delta for Frame-Rate Independence

A hard-coded increment runs 2x faster on a 120Hz display than on a 60Hz one.

```jsx
useFrame((state, delta) => {
  // Bad: speed depends on refresh rate
  mesh.current.rotation.x += 0.1;

  // Good: same speed everywhere
  mesh.current.rotation.x += delta * SPEED;
});
```

`state` also carries `clock`, `camera`, `scene`, `gl`, `pointer`, `size`, and
`invalidate` — read them from there rather than calling `useThree` in the loop.

## On-Demand Rendering

If nothing is animating, do not render every frame. `frameloop="demand"` renders
only when React commits a change or something calls `invalidate()`. On a static
product viewer this takes GPU usage from continuous to near zero.

```jsx
<Canvas frameloop="demand">
  <Scene />
</Canvas>
```

Then trigger frames manually when state changes outside React's knowledge:

```jsx
import { useThree } from "@react-three/fiber";

function Controls() {
  const invalidate = useThree((state) => state.invalidate);

  // e.g. after a tween tick, a drag, or a texture finishing load
  const onChange = () => invalidate();
}
```

Gotcha: with `frameloop="demand"`, anything that mutates the scene inside
`useFrame` still runs — but a mutation from an event handler, an
`IntersectionObserver`, or an async callback will **not** be visible until
something invalidates. Controls like `<OrbitControls />` call `invalidate()`
themselves.

Other modes: `frameloop="always"` (default) and `frameloop="never"` (you drive
`advance()` yourself).

## Avoiding Re-Renders

### Selector form of useThree

```jsx
// Bad: re-renders whenever ANY piece of R3F state changes
const { camera, gl, scene } = useThree();

// Good: subscribes to one slice
const camera = useThree((state) => state.camera);
```

### React.memo on expensive subtrees

```jsx
const ExpensiveModel = React.memo(function ExpensiveModel({ url }) {
  const { scene } = useGLTF(url);
  return <primitive object={scene} />;
});
```

Memoization only helps if props are stable — pass primitives or memoized
objects, not fresh array/object literals (`position={[0, 0, 0]}` allocates a new
array on every parent render).

### Toggle visibility instead of unmounting

Unmounting disposes geometry and materials; remounting reuploads buffers and
recompiles shaders — a visible stall.

```jsx
// Bad: full teardown and rebuild on every toggle
{showModel && <Model />}

// Good: keeps GPU resources warm
<Model visible={showModel} />
```

Use conditional mounting when the object is genuinely gone for a long time (you
*want* the VRAM back); use `visible` for anything that toggles frequently.

## Loading

### Suspense

R3F's loader hooks suspend, so wrap them:

```jsx
import { Suspense } from "react";
import { useProgress, Html } from "@react-three/drei";

function Loader() {
  const { progress } = useProgress();
  return <Html center>{progress.toFixed(0)}% loaded</Html>;
}

<Canvas>
  <Suspense fallback={<Loader />}>
    <Model />
  </Suspense>
</Canvas>;
```

### Preload before the component mounts

```jsx
import { useGLTF, useTexture } from "@react-three/drei";

// Module scope — starts fetching immediately, before render
useGLTF.preload("/model.glb");
useTexture.preload("/albedo.ktx2");

function Model() {
  const { scene } = useGLTF("/model.glb"); // already warm
  return <primitive object={scene} />;
}
```

`<Preload all />` from drei additionally forces shader compilation up front,
removing the first-frame compile stall.

### Compressed assets

```jsx
// drei's useGLTF wires DRACOLoader when you pass `true`
const { scene } = useGLTF("/model.glb", true);

// KTX2 needs the renderer for transcoder support detection
const gl = useThree((state) => state.gl);
```

See `threejs-asset-pipeline` for producing the compressed assets.

## Level of Detail

drei's `<Detailed />` wraps `THREE.LOD` — children map to the `distances` array
in order.

```jsx
import { Detailed } from "@react-three/drei";

<Detailed distances={[0, 20, 50]}>
  <HighDetail />
  <MediumDetail />
  <LowDetail />
</Detailed>;
```

LOD commonly buys 30-40% frame time back in large scenes.

## Cleanup on Unmount

R3F automatically disposes objects it created from JSX. It does **not** dispose
resources you created imperatively, or objects reached via `<primitive>`.

```jsx
useEffect(() => {
  return () => {
    geometry.dispose();
    material.dispose();
    texture.dispose();
    renderTarget.dispose();
  };
}, [geometry, material, texture, renderTarget]);
```

Opt out of automatic disposal when you're caching an object across mounts:

```jsx
<primitive object={cachedScene} dispose={null} />
```

See `threejs-performance` for what a full dispose must cover (textures hanging
off the material, ImageBitmap sources, render targets).

## Monitoring

```jsx
import { Perf } from "r3f-perf";

<Canvas>
  <Perf position="top-left" />
  <Scene />
</Canvas>;
```

`r3f-perf` shows FPS, GPU time, draw calls, triangles, geometry/texture counts,
and per-component cost. Watch the memory counters: they should plateau, not
climb. drei's `<Stats />` is a lighter alternative.

## Adaptive Quality

drei ships components that degrade the scene when the frame budget slips:

```jsx
import { AdaptiveDpr, AdaptiveEvents, PerformanceMonitor } from "@react-three/drei";

<Canvas dpr={[1, 2]}>
  <PerformanceMonitor onDecline={() => setEffectsEnabled(false)} />
  <AdaptiveDpr pixelated />
  <AdaptiveEvents />
</Canvas>;
```

`dpr={[1, 2]}` clamps device pixel ratio — the single highest-leverage setting
on mobile, where a DPR of 3 means rendering 9x the pixels.

## Post-Processing

Use `@react-three/postprocessing` (a wrapper over `pmndrs/postprocessing`),
which merges compatible effects into a single pass:

```jsx
import { EffectComposer, Bloom, Vignette, SMAA } from "@react-three/postprocessing";

<EffectComposer multisampling={0}>
  <Bloom luminanceThreshold={0.9} intensity={0.8} />
  <Vignette darkness={0.6} />
  <SMAA />
</EffectComposer>;
```

`multisampling={0}` disables MSAA — cheaper, and redundant once SMAA runs.
See `threejs-postprocessing` for the underlying pipeline rules.

## Common Mistakes

1. **`setState` in `useFrame`** — the #1 cause of R3F jank.
2. **Inline array props** (`position={[x, y, z]}`) inside memoized components —
   defeats memoization; memoize the array or accept the re-render.
3. **`useThree()` destructured** in a hot component — subscribe with a selector.
4. **Forgetting `invalidate()`** under `frameloop="demand"` — the scene silently
   stops updating.
5. **Creating materials in render** — a new material per render means a new
   shader program compile. Hoist to module scope or `useMemo`.
6. **Conditional mounting for frequent toggles** — use `visible` instead.

## See Also

- `threejs-performance` - Draw call budgets, disposal, profiling
- `threejs-asset-pipeline` - Producing compressed GLB/KTX2 assets
- `threejs-postprocessing` - Effect pipeline ordering and cost
- `threejs-fundamentals` - The underlying Three.js objects R3F creates
