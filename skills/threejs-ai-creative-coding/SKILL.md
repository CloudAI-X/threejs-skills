---
name: threejs-ai-creative-coding
description: Plan, prototype, debug, and render cinematic, narrative, or generative Three.js browser scenes with AI assistance. Use for visual-story scenes, WebGL or GLSL, WebGPU, glass or prisms, Bezier light ribbons, 3D card fields, scroll-directed cameras, AI-generated GLB assets, or turning a creative brief into an interactive or exportable 3D experience.
---

# Three.js AI Creative Coding

Build a directed 3D system rather than a pile of visual effects. Use AI to accelerate exploration and implementation, but make composition, camera, material, motion, and performance decisions explicitly.

## Start with a visual contract

Write this compact specification before generating code or assets:

```text
Visual thesis:
Viewer path and final feeling:
Duration / aspect / interaction:
Acts: beginning -> disruption -> turn -> resolution:
Hero object and secondary objects:
Camera language:
Light, palette, and material language:
Motion rules and triggers:
Performance budget and device target:
Export target:
```

Assign each element a story role. For example, an emissive Bezier ribbon can be a trace of choice, a card field can be archival pressure, and a prism can split one identity into competing paths. Do not add an effect without a narrative or compositional job.

## Route implementation work to the right skill

Use this skill as the creative director and load the focused sibling skill before writing a subsystem. Keep the scene contract and implementation decisions consistent.

| Need | Load first | Creative decision to make |
| --- | --- | --- |
| Scene graph, camera, renderer, resize, color management | `threejs-fundamentals` | Define the camera language and coordinate system for the story. |
| Repeated cards, particles, ribbons, or custom meshes | `threejs-geometry` | Decide which forms are hero objects and which must be instanced. |
| Glass, PBR surfaces, transparency, or custom material behavior | `threejs-materials` | Choose physical plausibility versus a deliberate stylized illusion. |
| Key, rim, environment, fog, and shadows | `threejs-lighting` | Create hierarchy before adding glow or bloom. |
| Images, HDR environments, video, UVs, or render targets | `threejs-textures` | Decide what belongs in a texture versus geometry or a shader. |
| GLB/glTF, Draco, KTX2, and remote asset handling | `threejs-loaders` | Establish an asset budget and a graceful loading/failure state. |
| Mixers, morph targets, skeletal clips, and timing | `threejs-animation` | Express named narrative states instead of scattered animation flags. |
| Noise, fields, distortion, refraction-like effects, and ribbons | `threejs-shaders` | Specify coordinate space, uniforms, and the visual rule the shader serves. |
| Bloom, depth of field, screen effects, and custom passes | `threejs-postprocessing` | Preserve silhouette and contrast before screen-space treatment. |
| Raycasting, controls, pointer input, and selection | `threejs-interaction` | Decide whether interaction reveals, disturbs, or resolves the scene. |

## Build in reliable layers

1. Establish the renderer, scene, camera, resize handling, tone mapping, and neutral background.
2. Block composition with primitive geometry and a fixed camera path. Verify silhouettes before materials.
3. Add the hero object. For a prism, start with controlled lighting and a transmission-like material or artistic shader; do not promise cinematic dispersion from stock transparency.
4. Add a restrained light rig, fog, and only then post-processing. Tune exposure and contrast before bloom or chromatic aberration.
5. Add secondary systems such as a Bezier tube, card field, particles, or text planes. Give each a bounded update loop and deterministic seed when generative.
6. Define named states such as `arrival`, `compression`, `fracture`, and `return`. Drive them from one timeline, scroll progress value, or state machine.
7. Introduce AI-produced assets after a primitive version has established scale, lighting, and motion.
8. Add debug controls: pause, reset, act scrubber, camera helper, frame-time display, and a reduced-motion fallback where relevant.

## Direct AI precisely

Request a narrow, testable change with the current file, framework, Three.js revision, visual intent, constraints, and acceptance criteria:

```text
In the existing vanilla Three.js scene, add one tubular Bezier light ribbon.
It must move from z=-20 to z=4 in 8 seconds, retain 60 fps with 300 cards,
use no new dependencies, and expose color, width, and speed in CONFIG.
Do not rewrite unrelated code. Return a patch and explain the new parameters.
```

Do not request an entire cinematic scene from one vague sentence. Generate or modify one layer, run it, inspect the result, and then make the next change. Keep model or API credentials server-side or in ignored environment variables, never in browser code or committed prompts.

## Treat AI assets as untrusted inputs

- Prefer GLB/glTF with PBR maps when scene lighting must change. Check scale, origin, normals, texture color space, clips, triangle count, draw calls, and license before committing.
- Serve assets through HTTP; do not validate model loading only with `file://`.
- Set an asset budget early. Compress geometry and textures, avoid needless 4K maps, and test a constrained-device quality tier.
- If an image-to-3D result looks wrong, inspect normals, map assignment, environment lighting, and tone mapping before generating another model.

## Verify the artistic system

- Test resize, high-DPI rendering, tab restore, asset-load failure, and WebGL fallback messaging.
- Use delta time; never tie visual speed to frame count. Observe one complete narrative cycle at target resolution and population.
- Measure frame time and draw calls. Remove decorative effects before reducing the hero object's narrative legibility.
- Confirm camera direction, layer depth, and transitions against the visual contract.
- For a repeatable final video, use fixed timestep, fixed random seed, deterministic timeline, and frame-accurate capture.
- Deliver a runnable project, the visual contract, controls or run command, asset credits, and known performance limits.

## Suggested narrative patterns

- **Bezier light ribbon:** Animate a head/tail range along a stable curve instead of rebuilding geometry every frame; make its trajectory mean attraction, escape, collision, fragmentation, or return.
- **Glass prism:** Establish silhouette and reflections with controlled key and rim lights first. Separate physically plausible glass from stylized refraction, then choose materials, shaders, and compositing accordingly.
- **Parallax card field:** Keep semantic card data separate from mesh state. Map depth to several cues such as scale, opacity, velocity, and contrast, with a readable foreground/midground/background hierarchy.
- **Scroll direction:** Normalize scroll to `0..1`, map it to explicit acts, and interpolate camera targets and object state with easing. Do not bind raw scroll pixels to unrelated transforms.
