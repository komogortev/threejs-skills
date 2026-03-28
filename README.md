# threejs-skills

Agent skills for Three.js development — auto-triggered reference guides for Cursor AI and Claude.

## Skills

| Skill | Description | Origin |
|---|---|---|
| [`threejs-animation`](skills/threejs-animation/SKILL.md) | AnimationClip, AnimationMixer, AnimationAction, blending, skeletal animation, morph targets, procedural motion | Adapted from [CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills) |
| [`threejs-loaders`](skills/threejs-loaders/SKILL.md) | GLTF/GLB, FBX, textures, HDR, async loading, caching, error handling | Adapted from [CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills) |
| [`threejs-interaction`](skills/threejs-interaction/SKILL.md) | Raycasting, camera controls (Orbit/FPS/Pointer Lock), TransformControls, keyboard input | Adapted from [CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills) |
| [`threejs-fbx-mixamo`](skills/threejs-fbx-mixamo/SKILL.md) | Mixamo FBX multi-clip pipeline, clip name resolution, AnimationMixer overlay system | Original |

## Credits

`threejs-animation`, `threejs-loaders`, and `threejs-interaction` are adapted from
[CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills) — trimmed and
restructured for conciseness. The original repo has no explicit license; these adaptations
are included with attribution and all credit for the original content belongs to the
CloudAI-X contributors.

`threejs-fbx-mixamo` is an original skill documenting the production Mixamo FBX animation
pipeline used in the `@base/player-three` package.

## What Are Agent Skills?

Skills are Markdown files (`.SKILL.md`) that teach an AI agent a specialized workflow.
When the description matches the user's request, the agent reads and follows the skill automatically.

Cursor reads skills from `~/.cursor/skills/<skill-name>/SKILL.md`.

## Installation

### Option A — Clone and symlink (recommended)

```bash
git clone https://github.com/<you>/threejs-skills.git
```

Then copy or symlink individual skills into `~/.cursor/skills/`:

```powershell
# Windows (PowerShell)
$repo = "C:\path\to\threejs-skills"
$dest = "$env:USERPROFILE\.cursor\skills"

New-Item -ItemType Junction -Path "$dest\threejs-animation"   -Target "$repo\skills\threejs-animation"
New-Item -ItemType Junction -Path "$dest\threejs-loaders"     -Target "$repo\skills\threejs-loaders"
New-Item -ItemType Junction -Path "$dest\threejs-interaction" -Target "$repo\skills\threejs-interaction"
New-Item -ItemType Junction -Path "$dest\threejs-fbx-mixamo"  -Target "$repo\skills\threejs-fbx-mixamo"
```

```bash
# macOS / Linux
repo="/path/to/threejs-skills"
dest="$HOME/.cursor/skills"

ln -s "$repo/skills/threejs-animation"   "$dest/threejs-animation"
ln -s "$repo/skills/threejs-loaders"     "$dest/threejs-loaders"
ln -s "$repo/skills/threejs-interaction" "$dest/threejs-interaction"
ln -s "$repo/skills/threejs-fbx-mixamo"  "$dest/threejs-fbx-mixamo"
```

### Option B — Copy files directly

Copy any `skills/<name>/SKILL.md` into `~/.cursor/skills/<name>/SKILL.md`.

## `threejs-fbx-mixamo` — Custom Skill

The flagship skill in this repo. Documents the production FBX animation pipeline used in
the `@base/player-three` package:

- **File naming:** `category__subcategory__action.fbx` convention
- **Vite glob URL resolution** for FBX assets in monorepo packages
- **Two-phase loading:** character FBX (mesh + skeleton) + N animation FBXes (clips only)
- **Clip accumulation** onto `root.userData['gltfAnimations']`
- **Hips position stripping** (`stripMixamoHipsPositionTracks`) to prevent root drift
- **Clip deduplication** by normalized name
- **Regex-based clip resolution** — maps Mixamo internal names to semantic slots
- **Mixer root selection** — SkinnedMesh vs rig group depending on track format
- **Pre-registration pattern** — all actions created at construction, weight 0
- **Weight blending** — locomotion layers (stand/crouch × idle/walk/run/strafe)
- **Overlay system** — jump/land/hazard/recovery one-shots with burst timers and locoSuppress
- **Landing tier system** — none/soft/medium/hard/critical/fatal with fallback chain
- **Water blending** — tread ↔ swim forward via smoothed blend scalar
- **Retargeting** — `SkeletonUtils.retargetClip` vs lightweight track name remap

## Skill Format

Each skill follows the standard format:

```markdown
---
name: skill-name
description: Trigger text — what the agent matches against. Be specific about when to use.
---

# Skill Title

...content...
```

## Stack Compatibility

- Three.js r150+
- TypeScript
- Vite (for FBX URL glob resolution)
- `@base/player-three` package (for the Mixamo pipeline skill)

## License

MIT
