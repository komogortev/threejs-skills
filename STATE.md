# STATE.md — threejs-skills

## Status

_Last updated: 2026-03-28_

**What's working:** All four skills symlinked and active in Cursor. `threejs-fbx-mixamo` reflects the production pipeline including water blending, five-tier landing severity, and the `SwimmableVolume` descriptor pattern. Skills auto-trigger via description matching in Cursor.

**What's broken / incomplete:** `threejs-fbx-mixamo` skill does not yet document the `ClimbVolume` descriptor or climbing controller pattern (design complete, implementation deferred). No `.cursor/rules/` directory exists in this repo — agents working on skill files have no project-specific guidance.

## Active Work

- Nothing — skills stable, reflecting the Session 9-10 checkpoint state.

## Blockers & Open Questions

- **[2026-03-28]** When climbing is implemented in `@base/player-three`, `threejs-fbx-mixamo` skill needs updating to cover `climb__*` FBX slot convention.

## Next Session

> When climbing is implemented in `threejs-engine-dev`, update `threejs-fbx-mixamo` skill: add the `climb__idle__attach`, `climb__move__up/down`, and `climb__exit__*` FBX slot documentation to the naming convention and overlay system sections.

## Decision Log

<!-- Append-only. One line per decision, newest first. -->

- **2026-03-28** — Skill content tracks production pipeline state, not a release schedule. `threejs-fbx-mixamo` updated to include water blending and five-tier landing tiers.

## Deferred

- **Climbing FBX slot documentation:** `threejs-fbx-mixamo` skill needs a climbing section. Deferred until climbing is implemented in `@base/player-three`.
- **`threejs-scene-builder` skill:** Possible future skill for `SceneDescriptor` authoring workflow, `SwimmableVolume`, editor export. Not yet written — deferred until scene authoring patterns stabilize.
- **CI for skill validation:** No automated check that skill YAML front-matter is valid. Low priority; manual review is sufficient.
