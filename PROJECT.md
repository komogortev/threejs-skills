# PROJECT.md — threejs-skills

## Identity

- **Module:** `threejs-skills` (standalone — not an @base package, not a pwa-shell fork)
- **Role:** Public repository of Cursor AI agent skills for Three.js development. Auto-triggers reference guides when an agent task matches a skill description. Symlinked into `~/.cursor/skills/` to activate in any Cursor workspace.
- **Fork of:** N/A — original repo
- **Extracts to:** N/A — skills live here permanently and symlink to Cursor's skills directory

## North Star

A concise, always-current collection of Three.js skills that make any AI agent immediately productive with the @base ecosystem's animation, loading, interaction, and Mixamo FBX patterns — without re-explaining the pipeline each session.

## Current Milestone

**Stable v1** — Four skills published and symlinked. Skills reflect the production pipeline as of the `@base/player-three` swimming + five-tier landing checkpoint.

## V1 Scope

**In scope:**
- `threejs-animation` — AnimationClip, AnimationMixer, blending, skeletal animation, morph targets
- `threejs-loaders` — GLTF/GLB, FBX, textures, HDR, async loading, error handling
- `threejs-interaction` — Raycasting, Orbit/FPS/PointerLock controls, TransformControls, keyboard input
- `threejs-fbx-mixamo` — Full Mixamo FBX pipeline: naming convention, Vite URL resolution, clip accumulation, regex resolution, overlay system, landing tiers, water blending

**Out of scope for v1:**
- Skills for non-Three.js topics (these live in `~/.cursor/skills/` system-wide)
- Vue-specific Three.js integration (covered by pwa-shell project rules)
- Automated skill testing or CI pipeline

## Stack (beyond base fork)

- Pure Markdown — no build step
- Cursor skill format: YAML front-matter (`name`, `description`) + Markdown body
- Symlinked into `C:\Users\bip\.cursor\skills\` on Windows (junction points via PowerShell)

## Architectural Decisions

<!-- Append-only. Date each entry. Never remove old decisions. -->

- **2026-03-28** — `threejs-fbx-mixamo` is an original skill (not adapted from third-party). Documents the exact production pipeline in `@base/player-three`. Must be kept in sync as the pipeline evolves.
- **2026-03-28** — `threejs-animation`, `threejs-loaders`, `threejs-interaction` adapted from CloudAI-X/threejs-skills with attribution. Trimmed for conciseness; full credit retained in README.
- **2026-03-28** — Skills are updated when the production pipeline changes, not on a schedule. The `threejs-fbx-mixamo` skill is the canary — if the pipeline diverges, it breaks first.
