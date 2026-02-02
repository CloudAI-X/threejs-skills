# Three.js Skills for Claude Code and AdaL

A curated collection of Three.js skill files that provide Claude Code with foundational knowledge for creating 3D elements and interactive experiences.

## Purpose

When working with Three.js, Claude Code starts with general programming knowledge but lacks specific Three.js API details, best practices, and common patterns. These skill files bridge that gap by providing:

- Accurate API references and constructor signatures
- Working code examples for common use cases
- Performance optimization tips
- Integration patterns between different Three.js systems

## Installation

Clone this repository into your project or copy the `.claude/skills` directory:

```bash
git clone https://github.com/pinkforest/threejs-playground.git
```

Or add as a submodule:

```bash
git submodule add https://github.com/pinkforest/threejs-playground.git
```

## Skills Included

| Skill                      | Description                                                             |
| -------------------------- | ----------------------------------------------------------------------- |
| **threejs-fundamentals**   | Scene setup, cameras, renderer, Object3D hierarchy, coordinate systems  |
| **threejs-geometry**       | Built-in shapes, BufferGeometry, custom geometry, instancing            |
| **threejs-materials**      | PBR materials, basic/phong/standard materials, shader materials         |
| **threejs-lighting**       | Light types, shadows, environment lighting, light helpers               |
| **threejs-textures**       | Texture types, UV mapping, environment maps, render targets             |
| **threejs-animation**      | Keyframe animation, skeletal animation, morph targets, animation mixing |
| **threejs-loaders**        | GLTF/GLB loading, texture loading, async patterns, caching              |
| **threejs-shaders**        | GLSL basics, ShaderMaterial, uniforms, custom effects                   |
| **threejs-postprocessing** | EffectComposer, bloom, DOF, screen effects, custom passes               |
| **threejs-interaction**    | Raycasting, camera controls, mouse/touch input, object selection        |

## How It Works

Claude Code automatically loads skill files from the `.claude/skills` directory when they match the context of your request. When you ask Claude Code to:

- Create a 3D scene → `threejs-fundamentals` is loaded
- Add lighting and shadows → `threejs-lighting` is loaded
- Load a GLTF model → `threejs-loaders` is loaded
- Create custom visual effects → `threejs-shaders` and `threejs-postprocessing` are loaded

## Usage Examples

### Basic Scene Setup

Ask Claude Code:

> "Create a basic Three.js scene with a rotating cube"

Claude Code will use `threejs-fundamentals` to generate accurate boilerplate with proper renderer setup, animation loop, and resize handling.

### Loading 3D Models

Ask Claude Code:

> "Load a GLTF model with Draco compression and play its animations"

Claude Code will use `threejs-loaders` and `threejs-animation` to generate code with proper loader configuration, animation mixer setup, and error handling.

### Custom Shaders

Ask Claude Code:

> "Create a custom shader material with a fresnel effect"

Claude Code will use `threejs-shaders` to generate working GLSL code with proper uniform declarations and coordinate space handling.

## Skill File Structure

Each skill file follows a consistent format:

```markdown
---
name: skill-name
description: When this skill should be activated
---

# Skill Title

## Quick Start

[Minimal working example]

## Core Concepts

[Detailed API documentation with examples]

## Common Patterns

[Real-world usage patterns]

## Performance Tips

[Optimization guidance]

## See Also

[Related skills]
```

## Verification

These skills have been audited against the official Three.js documentation (r160+) for:

- Correct class names and constructor signatures
- Valid property names and method signatures
- Accurate import paths (`three/addons/` format)
- Working code examples
- Current best practices

## Contributing

Found an error or want to add coverage for additional Three.js features?

1. Fork the repository
2. Edit or create skill files in `.claude/skills/`
3. Verify against [Three.js documentation](https://threejs.org/docs/)
4. Submit a pull request

### Skill File Guidelines

- Use accurate, tested code examples
- Include both simple and advanced patterns
- Document performance implications
- Cross-reference related skills
- Keep examples concise but complete

## License

MIT License - Feel free to use, modify, and distribute.

## Acknowledgments

- [Three.js](https://threejs.org/) - The 3D library these skills document
