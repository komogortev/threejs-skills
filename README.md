# threejs-skills

Four agent skills for Three.js development: Markdown files that an AI coding agent such as Claude Code loads
when your request matches a skill's description. Three are condensed references for common Three.js work. The
fourth, `threejs-fbx-mixamo`, documents a Mixamo FBX character-animation pipeline written for the
[`@base/player-three`](https://github.com/komogortev/vue-three-base-packages) package.

## Skills

| Skill | Covers | Origin |
|---|---|---|
| [`threejs-animation`](skills/threejs-animation/SKILL.md) | AnimationClip, AnimationMixer, AnimationAction, blending, skeletal animation, morph targets, procedural motion | Adapted from [CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills) |
| [`threejs-loaders`](skills/threejs-loaders/SKILL.md) | GLTF/GLB, FBX, textures, HDR, async loading, caching, error handling | Adapted from CloudAI-X/threejs-skills |
| [`threejs-interaction`](skills/threejs-interaction/SKILL.md) | Raycasting, camera controls (Orbit, first-person, pointer lock), TransformControls, keyboard input | Adapted from CloudAI-X/threejs-skills |
| [`threejs-fbx-mixamo`](skills/threejs-fbx-mixamo/SKILL.md) | Mixamo FBX multi-clip pipeline, clip name resolution, AnimationMixer weight blending, one-shot overlays | Original |

### `threejs-fbx-mixamo`

The original skill. It describes how a character controller loads one base character FBX plus separate
animation FBX files, merges the clips, strips hips position tracks to prevent root drift, resolves Mixamo clip
names to semantic slots with regular expressions, and blends locomotion layers with jump, landing and water
overlays. It assumes Vite (for glob-based FBX URL resolution) and the helpers in `@base/player-three`.

## Install

Claude Code reads skills from `~/.claude/skills/<skill-name>/SKILL.md`. Copy or link the folders you want:

```bash
git clone https://github.com/komogortev/threejs-skills.git
cp -r threejs-skills/skills/threejs-fbx-mixamo ~/.claude/skills/
```

On Windows PowerShell, a junction keeps the skill in sync with the clone:

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\threejs-fbx-mixamo" -Target "C:\path\to\threejs-skills\skills\threejs-fbx-mixamo"
```

Each skill is one `SKILL.md` with `name` and `description` frontmatter; the description is what the agent
matches against.

## Credits

`threejs-animation`, `threejs-loaders` and `threejs-interaction` are adapted, trimmed and restructured, from
[CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills). All credit for the original content
belongs to the CloudAI-X contributors.

## License

The repository's own work, including `threejs-fbx-mixamo`, is [MIT](./LICENSE) licensed. The three adapted
skills are derived from a repository that publishes no licence, so the MIT terms do not extend to the
upstream content in them.
