# Three.js Skills for Claude Code

Skill files that give Claude Code current, verified knowledge of three.js, so the code it writes runs on the release you actually install.

The skills target **three.js r186** (`three@0.186`). Every example outside the WebGPU and TSL section uses `WebGLRenderer`; that section, present in every skill, uses `WebGPURenderer`.

## Why

Claude Code knows three.js from its training data, which mixes many releases. A lot changed between r160 and r186: `RGBELoader` became `HDRLoader`, `Clock` gave way to `Timer`, `PostProcessing` became `RenderPipeline`, WebGL 1 support is gone, and the three.js docs now tell LLMs to use `WebGPURenderer` with TSL for custom shaders and materials. These skills give Claude Code the current API, working examples, and a Common Mistakes list of the outdated patterns models still reach for.

## Installation

With the [skills](https://skills.sh) CLI:

```bash
npx skills add cloudai-x/threejs-skills
```

Or manually. Claude Code loads skills from `~/.claude/skills/` (all your projects) or `.claude/skills/` (one project):

```bash
git clone https://github.com/CloudAI-X/threejs-skills.git

# all your projects
mkdir -p ~/.claude/skills && cp -r threejs-skills/skills/* ~/.claude/skills/

# or one project (run from the project root)
mkdir -p .claude/skills && cp -r threejs-skills/skills/* .claude/skills/
```

## Skills Included

| Skill                      | Covers                                                                          |
| -------------------------- | ------------------------------------------------------------------------------- |
| **threejs-fundamentals**   | Scene, cameras, renderer setup, animation loop, object hierarchy, transforms   |
| **threejs-geometry**       | Built-in shapes, BufferGeometry, merging, instancing, BatchedMesh, points, lines |
| **threejs-materials**      | Material types, PBR, blending, stencil, recompiles, node materials              |
| **threejs-lighting**       | Light types and physical units, shadows, HDR environments, light probes         |
| **threejs-textures**       | Loading, color spaces, UV channels, texture types, render targets, memory       |
| **threejs-animation**      | Keyframes, the animation mixer, skeletal and morph animation, blending          |
| **threejs-loaders**        | glTF with Draco, meshopt and KTX2, other formats, HDR, progress, disposal       |
| **threejs-shaders**        | ShaderMaterial and RawShaderMaterial, shader chunks, TSL and node materials     |
| **threejs-postprocessing** | EffectComposer, built-in renderer effects, the WebGPU RenderPipeline            |
| **threejs-interaction**    | Raycasting, camera controls, transform gizmos, drag and box selection           |

## How It Works

Claude Code loads a skill when your request matches its description. When you ask it to:

- create a 3D scene → `threejs-fundamentals`
- add lights and shadows → `threejs-lighting`
- load a glTF model → `threejs-loaders`
- write a custom shader or screen effect → `threejs-shaders` and `threejs-postprocessing`

## Skill File Structure

Every skill follows the same layout:

```markdown
---
name: threejs-<topic>
description: "What the skill covers. Use when ..."
---

# Three.js <Topic>

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start          a complete, runnable example
## <topic sections>     the API with working examples
## WebGPU and TSL       the same topic under WebGPURenderer
## Common Mistakes      outdated or wrong code → the r186 fix, with the reason
## See Also             related skills
```

## Verification

The r186 rewrite was checked against the published `three@0.186.1` package and the official Migration Guide:

- every import path and every `THREE.*` class used in an example exists in the package
- every JavaScript block parses as an ES module
- shader chunk names exist, and no removed or renamed API appears in example code
- every skill was reviewed against the three.js source, with rendering, timing and shader claims measured where possible

If you find something that is wrong on the current release, please open an issue with the skill name, the line and the three.js version you use.

## License

MIT, see [LICENSE](LICENSE).

## Acknowledgments

- [Three.js](https://threejs.org/) - the 3D library these skills document
