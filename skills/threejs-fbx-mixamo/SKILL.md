---
name: threejs-fbx-mixamo
description: Mixamo FBX animation pipeline for Three.js character controllers — loading a base character FBX, accumulating separate Mixamo animation FBX clips, clip name resolution, AnimationMixer weight blending, and overlay (one-shot) system. Use when working with Mixamo character animations, FBX clip loading, the @base/player-three package, CharacterAnimationRig, locomotion blending, or jump/land/water overlay animations.
---

# Three.js Mixamo FBX Animation Pipeline

This skill documents the exact FBX→Mixamo→AnimationMixer→overlay pipeline used in the
`@base/player-three` package. It is the canonical reference for all character animation work.

## Architecture Overview

```
Character FBX          Animation FBX × N
(mesh + skeleton,      (clips only, no mesh)
 no animations)              │
       │               FBXLoader × N
       │                     │
  FBXLoader            object.animations[]
       │                     │
  root object ──────── accumulate all clips
       │
  root.userData['gltfAnimations'] = deduped + sanitized clips
       │
  CharacterAnimationRig(root)
       │
  AnimationMixer (mixer root = SkinnedMesh OR rig group)
       │
  ALL actions pre-registered at weight 0 (loop AND one-shots)
       │
  tick() → weight blending (loco layers) + overlay burst logic
```

## File Naming Convention

```
assets/fbx/<category>/<category>__<subcategory>__<action>.fbx

Examples:
  locomotion/locomotion__idle__stand.fbx
  locomotion/locomotion__walk__forward.fbx
  locomotion/locomotion__run__slow.fbx
  air/air__jump__rise.fbx
  air/air__land__soft.fbx
  water/water__swim__forward.fbx
  hazard/hazard__wall__stumble.fbx
  transitions/transition__idle_stand__walk_forward.fbx
```

The `__` separator is purely a file naming convention. The internal Mixamo clip name
inside each FBX (e.g. `"Jumping Up"`, `"Walking"`) is unrelated to the file name.

## Vite URL Resolution

```typescript
// Vite resolves at transform time → content-hashed URLs in production
const _fbxAssetUrls = (import.meta as any).glob('../assets/**/*.fbx', {
  query: '?url',
  import: 'default',
  eager: true,
}) as Record<string, string>

export const MIXAMO_FBX_CLIP_URLS: string[] = FBX_ASSET_PATHS
  .map((relativePath) => _fbxAssetUrls[`../assets/${relativePath}`])
  .filter((url): url is string => typeof url === 'string' && url.length > 0)
```

**Key:** This is Vite-only. Consuming apps must use Vite. The `dist/` output is
not usable outside a Vite build pipeline.

## Loading Phase

### 1 — Load the character FBX (mesh + skeleton)

```typescript
import { FBXLoader } from 'three/addons/loaders/FBXLoader.js'

const loader = new FBXLoader()
const characterRoot = await new Promise<THREE.Group>((resolve, reject) => {
  loader.load(characterUrl, resolve, undefined, reject)
})

// Mixamo exports at 1/100 scale — normalize to 1.0 before anything else
characterRoot.scale.setScalar(1)         // already normalized in @base
// Or if raw Mixamo export: characterRoot.scale.setScalar(0.01)

// Character FBX has an empty animations[] — clips come from separate files
console.log(characterRoot.animations) // []
```

### 2 — Load each animation FBX and accumulate clips

```typescript
const allClips: THREE.AnimationClip[] = []

for (const url of MIXAMO_FBX_CLIP_URLS) {
  const animObject = await new Promise<THREE.Group>((resolve, reject) => {
    loader.load(url, resolve, undefined, reject)
  })
  // Clips are on animObject.animations — DO NOT add animObject to the scene
  allClips.push(...animObject.animations)
}
```

**Important:** Animation FBX files have a skeleton but typically no SkinnedMesh
(or a dummy one). Never add them to the scene. Extract `.animations` and discard
the object.

### 3 — Sanitize and store

```typescript
import { sanitizeMixamoClips } from '@base/player-three'

// Strip root Hips position tracks (prevents mocap drift / root motion circles).
// Translation comes from PlayerController physics, not animation data.
const sanitized = sanitizeMixamoClips(allClips)

// Store on userData['gltfAnimations'] — the conventional field CharacterAnimationRig reads.
// Using this field means the rig works identically for GLTF and FBX sources.
characterRoot.userData['gltfAnimations'] = sanitized

// Now construct the rig
const rig = new CharacterAnimationRig(characterRoot, { debugClipResolution: true })
```

## Clip Deduplication

`CharacterAnimationRig` dedupes clips before resolving:

```typescript
function normalizeClipName(name: string): string {
  return name.toLowerCase().replace(/\s*\(\d+\)\s*$/, '').trim()
}

function dedupeClipsByName(clips: THREE.AnimationClip[]): THREE.AnimationClip[] {
  const seen = new Set<string>()
  return clips.filter((clip) => {
    const key = normalizeClipName(clip.name)
    if (seen.has(key)) return false
    seen.add(key)
    return true
  })
}
```

Mixamo sometimes exports the same clip twice with suffix ` (2)`. Always dedupe before
passing to clip resolution functions.

## Clip Name Resolution

Mixamo internal clip names are **independent of FBX file names**.
A file named `locomotion__walk__forward.fbx` contains a clip called `"Walking"`.

### Normalize for matching

```typescript
export function normalizeClipLabelForMatch(name: string): string {
  return name
    .toLowerCase()
    .replace(/^[^|]+\|/, '')   // strip namespace: "mixamorig|Walking" → "walking"
    .replace(/__/g, ' ')
    .replace(/[._]/g, ' ')
    .replace(/\s+/g, ' ')
    .trim()
}
```

### Pattern matching

```typescript
export function pickClipByPatterns(
  clips: readonly THREE.AnimationClip[],
  patterns: readonly RegExp[],
): THREE.AnimationClip | undefined {
  for (const re of patterns) {
    const hit = clips.find(
      (c) => re.test(c.name) || re.test(normalizeClipLabelForMatch(c.name))
    )
    if (hit) return hit
  }
  return undefined
}
```

Always order patterns **most-specific → least-specific**. The first match wins.

### Common Mixamo internal clip names

| FBX file pattern | Typical Mixamo clip name |
|---|---|
| `locomotion__idle__stand` | `"Idle"`, `"Neutral Idle"` |
| `locomotion__walk__forward` | `"Walking"`, `"Walking (1)"` |
| `locomotion__walk__backward` | `"Walking Backwards"` |
| `locomotion__walk__strafe_left` | `"Left Strafe"` |
| `locomotion__run__forward` | `"Running"` |
| `locomotion__crouch__idle` | `"Crouching Idle"`, `"Male Crouch Pose"` |
| `locomotion__crouch__walk_forward` | `"Crouched Walking"` |
| `air__jump__rise` | `"Jumping Up"` |
| `air__jump__fall` | `"Jumping Down"`, `"Falling Idle"` |
| `air__land__soft` | `"Landing"`, `"Soft Landing"` |
| `air__land__medium` | `"Hard Landing Medium"` |
| `water__tread__idle` | `"Floating"`, `"Water Tread Idle"` |
| `water__swim__forward` | `"Swimming"`, `"Swimming Forward"` |

## Mixer Root Selection

```typescript
function clipsUseSkinnedMeshMixerRoot(clips: readonly THREE.AnimationClip[]): boolean {
  for (const clip of clips) {
    for (const t of clip.tracks) {
      if (t.name.includes('.bones[')) return true  // retargeted clip
    }
  }
  return false
}

// In CharacterAnimationRig constructor:
const mixerRoot = clipsUseSkinnedMeshMixerRoot(clips) ? skinned : rigRoot
this.mixer = new THREE.AnimationMixer(mixerRoot)
```

- **Remap-only clips** — tracks use bone names directly (e.g. `mixamorigHips.quaternion`) → mixer root = rig group
- **Retargeted clips** — tracks use `.bones[name]` (from `SkeletonUtils.retargetClip`) → mixer root = `SkinnedMesh`

## Action Pre-registration Pattern

ALL actions are created at construction time and `.play()` is called with weight 0.
No lazy action creation. No crossfade API.

```typescript
const playLoop = (clip: THREE.AnimationClip | undefined, w: number): THREE.AnimationAction | null => {
  if (!clip || !this.mixer) return null
  const a = this.mixer.clipAction(clip)
  a.setLoop(THREE.LoopRepeat, Infinity)
  a.clampWhenFinished = false
  a.play()
  a.setEffectiveWeight(w)
  return a
}

const playOnce = (clip: THREE.AnimationClip | undefined, w: number): THREE.AnimationAction | null => {
  if (!clip || !this.mixer) return null
  const a = this.mixer.clipAction(clip)
  a.setLoop(THREE.LoopOnce, 1)
  a.clampWhenFinished = true   // hold last frame when done
  a.play()
  a.setEffectiveWeight(w)
  return a
}

// Stand locomotion — idle starts at 1, everything else at 0
this.stand = {
  idle:    playLoop(idleStand, 1),
  walkFwd: playLoop(walkFwdStand, 0),
  runFwd:  playLoop(runFwdStand, 0),
  walkBack: playLoop(walkBackStand, 0),
  strafeL: playLoop(strafeLStand, 0),
  strafeR: playLoop(strafeRStand, 0),
}

// One-shot overlays — all at weight 0 until triggered
this.jumpRise  = playOnce(jumpClip, 0)
this.landSoft  = playOnce(landSoftClip, 0)
this.landMedium = playOnce(landMediumClip, 0)
// ... etc
```

## Weight Blending (Locomotion Layers)

Each tick, weights are computed from the player velocity vector and applied directly.
No `crossFadeTo()`. No `fadeIn()/fadeOut()`.

```typescript
// Internal blend scalars updated each tick:
// moveBlend:   0=idle pose, 1=full directional locomotion
// crouchBlend: 0=stand layer, 1=crouch layer
// runGate:     0=walk, 1=run (only in stand.walkFwd / stand.runFwd)

// Directional weights from velocity projection:
const speed = vel.length()
vel.normalize()
const fwd = _worldFwd.dot(vel)   // [-1..1] forward component
const right = _worldRight.dot(vel)

// Decompose into cardinal weights
const wF = Math.max(0, fwd)
const wB = Math.max(0, -fwd)
const wR = Math.max(0, right)
const wL = Math.max(0, -right)

// Cross-blend stand vs crouch:
const standLoco = 1 - crouchBlend
const crouchLoco = crouchBlend
applyLocoWeights(this.stand,  idleW * standLoco, wF, wB, wL, wR, moveBlend * standLoco, runGate, locoSuppress)
applyLocoWeights(this.crouch, idleW * crouchLoco, wF, wB, wL, wR, moveBlend * crouchLoco, 0, locoSuppress)
```

## Overlay (One-Shot) System

Overlays temporarily suppress the locomotion layer using a burst timer.

```typescript
// On jump trigger:
jumpRise.reset().play()
// burst timer counts down; during burst: jumpRise weight = 1, locoSuppress = 0

// Burst decay (each tick):
if (this.landBurstSeconds > 0) {
  this.landBurstSeconds -= delta
  if (this.landBurstSeconds <= 0) {
    // burst expired — fade action back to 0
    this.stopLandActions()
  }
}

// locoSuppress applied in applyLocoWeights():
const m = moveLayer * locoSuppress  // locoSuppress = 0 suppresses all loco weights
```

### Landing Tier System

```typescript
export type LandImpactTier = 'none' | 'soft' | 'medium' | 'hard' | 'critical' | 'fatal'

// Tier boundaries (fall distance in metres):
//  none     < 0.38 m + < 0.14 s air
//  soft     0.38 m – 1.45 m
//  medium   1.45 m – 3.65 m
//  hard     3.65 m – 8.0 m
//  critical 8.0 m – 20.0 m
//  fatal    > 20.0 m

export function computeLandImpactTier(
  fallDistanceMeters: number | undefined,
  airTimeSeconds: number | undefined,
): LandImpactTier { /* ... */ }

// Tier plays the highest-available action, falling back to lower tiers:
// fatal → critical → hard → medium → soft
// "Slot only" FBX files (empty animation) activate when a real clip is supplied.
```

### Water Blending

```typescript
// waterSwimBlend: 0 = tread (upright float), 1 = swim forward
// Smoothed each tick toward target based on horizontal velocity
const targetSwimBlend = horizontalSpeed > swimThreshold ? 1 : 0
this.waterSwimBlend = THREE.MathUtils.damp(this.waterSwimBlend, targetSwimBlend, 8, delta)

this.waterTread?.setEffectiveWeight((1 - this.waterSwimBlend) * waterWeight)
this.waterSwimFwd?.setEffectiveWeight(this.waterSwimBlend * waterWeight)
```

## Retargeting (Cross-Skeleton)

Use when the animation FBX skeleton has different bone name prefixes from the character.

```typescript
import { retargetMixamoClipsToCharacter, primarySkinnedMeshForRig } from '@base/player-three'

const targetSkinned = primarySkinnedMeshForRig(characterRoot)
if (targetSkinned) {
  const retargeted = retargetMixamoClipsToCharacter(
    targetSkinned,
    sourceAnimScene,  // the loaded animation FBX Group
    sourceClips,
  )
  // retargeted clips use .bones[name] tracks → require SkinnedMesh mixer root
  allClips.push(...retargeted)
}
```

**When to retarget vs remap:**
- Same Mixamo skeleton, different namespace prefix (`mixamorig:` vs `mixamorigHips`) → `remapClipTracksToTargetSkeleton()` (cheap)
- Different base skeleton → `retargetMixamoClipsToCharacter()` via `SkeletonUtils.retargetClip` (expensive, bakes)

## Hips Position Stripping

```typescript
// Mixamo FBX clips animate root Hips.position (mocap drift / root motion).
// Strip it so translation is owned entirely by PlayerController physics.

export function stripMixamoHipsPositionTracks(clip: THREE.AnimationClip): THREE.AnimationClip {
  const tracks = clip.tracks.filter((t) => !isRootHipsPositionTrack(t.name))
  if (tracks.length === clip.tracks.length) return clip
  return new THREE.AnimationClip(clip.name, clip.duration, tracks)
}

// Call via:
const sanitized = sanitizeMixamoClips(allClips) // maps stripMixamoHipsPositionTracks over array
```

This handles both raw FBX track names (`mixamorigHips.position`) and retargeted track names
(`.bones[mixamorigHips].position`).

## Debug Clip Resolution

```typescript
const rig = new CharacterAnimationRig(characterRoot, {
  debugClipResolution: true,   // logs resolved clip names at construction
  debugAnimationTriggers: true, // logs land tier + metrics on each landing
})
```

The console output shows which Mixamo clip name was matched to each slot.
If a slot is `undefined`, the pattern list in `locomotionClipAssignments.ts` or
`animationOverlayAssignments.ts` needs a new entry.

## Adding a New Animation Slot

1. Add the FBX file to `assets/fbx/<category>/`
2. Add the path to `FBX_ASSET_PATHS` in `mixamoFbxClipUrls.ts`
3. Add the semantic slot type to `AnimationOverlaySlot` in `animationOverlayAssignments.ts`
4. Add `pickClipByPatterns()` call in the relevant `resolve*()` function
5. Add `playOnce()` or `playLoop()` call in `CharacterAnimationRig` constructor
6. Add burst timer fields and weight logic in `tick()` or the relevant update method
7. Export the new type from `index.ts` if needed

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Clip never matches | Enable `debugClipResolution: true` — check the actual Mixamo internal name, add a regex pattern |
| Root drift / moonwalking | Call `sanitizeMixamoClips()` to strip hips position tracks |
| Wrong mixer root → no animation | Check whether clips use `.bones[...]` — retargeted clips need `SkinnedMesh` root |
| All slots `undefined` at construction | Ensure `root.userData['gltfAnimations']` is set BEFORE `new CharacterAnimationRig(root)` |
| Duplicate clips, wrong one wins | Deduplication is by normalized name; rename the FBX if two clips normalize to the same string |
| Scale mismatch (giant/tiny character) | Normalize scale to `1.0` before constructing the rig; Mixamo raw exports are at `0.01` |
| "Slot only" FBX does nothing | Replace the placeholder FBX with a real Mixamo clip download |
| Animation FBX object added to scene | Never `scene.add(animObject)` — extract `.animations` and discard |

## Package API Reference

```typescript
// @base/player-three exports:

// Loading / sanitization
sanitizeMixamoClips(clips)           // strip hips position tracks
stripMixamoHipsPositionTracks(clip)  // single clip version
MIXAMO_FBX_CLIP_URLS                 // Vite-resolved URL array

// Clip resolution
pickClipByPatterns(clips, patterns)          // core matcher
normalizeClipLabelForMatch(name)             // normalize for regex
resolveCharacterLocomotionClips(clips)       // → CharacterLocomotionClipSet
resolveCharacterOverlayClips(clips)          // → CharacterOverlayClipSet
resolveWaterClips(clips)                     // → CharacterWaterClipSet
computeLandImpactTier(fallDist, airTime)     // → LandImpactTier

// Skeleton
primarySkinnedMeshForRig(root)                           // find primary SkinnedMesh
largestSkinnedMesh(scene)                                // largest by vertex count
pruneExtraSkinnedMeshes(root)                            // keep only primary
findMixamoHipBoneName(bones)                             // detect root hip bone
remapClipTracksToTargetSkeleton(clip, skeleton)          // cheap name remap
retargetMixamoClipsToCharacter(target, source, clips)    // full retarget

// Rig
CharacterAnimationRig     // main class — wraps mixer + all actions
PlayerController          // physics + state machine; drives rig via tick()
```

## See Also

- `threejs-animation` — AnimationMixer, AnimationAction, blending fundamentals
- `threejs-loaders` — FBXLoader, async loading patterns
- `docs/threejs-engine-dev/player-state-transition-animation-map.md` — full slot/state map
- `SHARED/packages/player-three/src/` — source of truth for all patterns above
