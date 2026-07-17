---
name: p5js-generative-animation
description: Create narrative, design-led, and exportable p5.js or Processing-inspired animations. Use for generative motion, animated typography, agent systems, Bézier artwork, p5.js WebGL, dynamic posters, short-form visual stories, and AI-assisted creative-coding workflows.
---

# P5.js Generative Animation

Build a directed visual system, not a collection of effects. p5.js is most powerful when its rules create the story: a current becomes social pressure, a typographic fracture becomes a sentence that cannot be said, and a field's recovery becomes a resolution.

## Quick Start: Narrative State Machine

Use a single configuration block and named acts. The simplest version is enough to turn a static idea into a visual story.

```javascript
const CFG = {
  seed: 42,
  palette: { background: "#11111a", quiet: "#A8C7FA", accent: "#FF8FA3" },
  duration: 12,
  acts: { arrival: 0, pressure: 0.25, fracture: 0.58, release: 0.82 },
};

function setup() {
  createCanvas(1080, 1350);
  randomSeed(CFG.seed);
  noiseSeed(CFG.seed);
}

function draw() {
  const t = (millis() / 1000) % CFG.duration;
  const progress = t / CFG.duration;
  background(CFG.palette.background);

  const state = progress < CFG.acts.pressure ? "arrival"
    : progress < CFG.acts.fracture ? "pressure"
    : progress < CFG.acts.release ? "fracture"
    : "release";

  drawScene(progress, state);
}

function drawScene(progress, state) {
  const intensity = state === "arrival" ? 0.25
    : state === "pressure" ? 0.55
    : state === "fracture" ? 1
    : 0.4;
  const radius = lerp(90, 420, easeOutCubic(intensity));
  noStroke();
  fill(CFG.palette.accent);
  circle(width / 2, height / 2, radius);
  fill(CFG.palette.quiet);
  textAlign(CENTER, CENTER);
  textSize(42);
  text(state, width / 2, height * 0.8);
}

function easeOutCubic(t) {
  return 1 - (1 - t) ** 3;
}
```

## Visual Contract

Before coding, define:

```text
Visual thesis:
Viewer's path and final feeling:
Duration / aspect / delivery:
Acts: arrival -> pressure -> turn -> resolution:
Entities, macro-system, and exceptional element:
Forces and their meaning:
Palette / typography / texture / negative space:
Sound or narration beats:
Performance and export target:
```

Give every visual element a compositional or narrative job. Remove decorative effects that do neither.

## Creation Flow

1. Write a beat sheet: each act gets a visual event, sound/voice beat, and measurable state change.
2. Block the composition with primitives. Confirm hierarchy, pace, and empty space before polishing.
3. Implement canvas and art direction; then one static hero form; repeated elements; motion; transitions; compositing; capture.
4. Record narration only after timing stabilizes. Edit deterministic captures, voice, music, and cuts into the final film.
5. Keep a still export and central parameters so the work can become a poster or interactive page.

## System Architecture

- Use direct 2D drawing for type, illustration, Bézier forms, and lightweight agent fields.
- Cache repeated expensive geometry in `p5.Graphics` during `setup()`; never rebuild the same complex path for each entity every frame.
- Use classes for independent agents with position, velocity, depth, phase, and lifecycle.
- Use `WEBGL` only when the story needs a camera, depth, or spatial composition; start with a small scene.
- Use shaders for field-based effects such as grain, distortion, feedback, or chromatic treatment when a CPU loop cannot meet the frame budget.
- Use DOM/CSS for accessible long-form text; use canvas typography for controlled cinematic compositing.

## Motion and Depth

Centralize every tunable value in `CONFIG`. Separate geometry from behavior. Add one motion rule at a time: drift, noise/sine sway, attraction, repulsion, spring, collision, or decay.

Map depth consistently to at least two of scale, speed, opacity, saturation, detail, or layer order. Render far layers first. Give major transitions named states and one progress timeline; avoid scattered boolean switches.

## AI Collaboration

Request narrow, testable changes. Include the current sketch mode, visual intent, existing architecture, constraints, and acceptance criteria.

```text
Add a Bézier light trail only to the fracture act. Keep it deterministic with
CFG.seed, behind foreground type, allocation-free in draw(), and expose width,
color, and speed in CONFIG. Do not rewrite the state machine. Return a patch.
```

Generate and verify the renderer, one visual system, force logic, transition, compositing, and export in sequence. Never put API keys in browser code or committed files.

## Performance and Export

- Set and reuse a seed while iterating; vary it only after behavior is approved.
- Test resize, high-DPI rendering, full state cycles, console warnings, layer order, and edge recycling.
- Use delta time for time-based motion; do not make speed frame-rate dependent.
- Profile allocations, texture memory, and frame rate at target entity counts. Cut decorative effects before legibility.
- Use browser recording for drafts. For final video, lock seed, dimensions, timestep, and timeline; use frame-accurate capture when repeatability matters.

## See Also

- `threejs-animation` for a 3D animation-loop perspective.
- `threejs-shaders` for GPU-based custom visual effects.
- `threejs-postprocessing` for screen-space compositing patterns.
