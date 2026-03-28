---
name: threejs-animation
description: Three.js animation - keyframe animation, skeletal animation, morph targets, animation mixing. Use when animating objects, playing GLTF animations, creating procedural motion, or blending animations.
---

# Three.js Animation

## Quick Start

```javascript
import * as THREE from "three";

// Simple procedural animation
const clock = new THREE.Clock();

function animate() {
  const delta = clock.getDelta();
  const elapsed = clock.getElapsedTime();

  mesh.rotation.y += delta;
  mesh.position.y = Math.sin(elapsed) * 0.5;

  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

## Animation System Overview

Three.js animation system has three main components:

1. **AnimationClip** - Container for keyframe data
2. **AnimationMixer** - Plays animations on a root object
3. **AnimationAction** - Controls playback of a clip

## AnimationClip

Stores keyframe animation data.

```javascript
// Create animation clip
const times = [0, 1, 2]; // Keyframe times (seconds)
const values = [0, 1, 0]; // Values at each keyframe

const track = new THREE.NumberKeyframeTrack(
  ".position[y]", // Property path
  times,
  values,
);

const clip = new THREE.AnimationClip("bounce", 2, [track]);
```

### KeyframeTrack Types

```javascript
// Number track (single value)
new THREE.NumberKeyframeTrack(".opacity", times, [1, 0]);

// Vector track (position, scale)
new THREE.VectorKeyframeTrack(".position", times, [
  0, 0, 0, // t=0
  1, 2, 0, // t=1
  0, 0, 0, // t=2
]);

// Quaternion track (rotation)
const q1 = new THREE.Quaternion().setFromEuler(new THREE.Euler(0, 0, 0));
const q2 = new THREE.Quaternion().setFromEuler(new THREE.Euler(0, Math.PI, 0));
new THREE.QuaternionKeyframeTrack(
  ".quaternion",
  [0, 1],
  [q1.x, q1.y, q1.z, q1.w, q2.x, q2.y, q2.z, q2.w],
);

// Color track
new THREE.ColorKeyframeTrack(".material.color", times, [
  1, 0, 0, // red
  0, 1, 0, // green
  0, 0, 1, // blue
]);
```

### Interpolation Modes

```javascript
track.setInterpolation(THREE.InterpolateLinear);   // Default
track.setInterpolation(THREE.InterpolateSmooth);   // Cubic spline
track.setInterpolation(THREE.InterpolateDiscrete); // Step function
```

## AnimationMixer

```javascript
const mixer = new THREE.AnimationMixer(model);
const action = mixer.clipAction(clip);
action.play();

// Update in animation loop
function animate() {
  const delta = clock.getDelta();
  mixer.update(delta); // Required!
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
```

### Mixer Events

```javascript
mixer.addEventListener("finished", (e) => {
  console.log("Animation finished:", e.action.getClip().name);
});
mixer.addEventListener("loop", (e) => {
  console.log("Animation looped:", e.action.getClip().name);
});
```

## AnimationAction

```javascript
const action = mixer.clipAction(clip);

// Playback
action.play();
action.stop();
action.reset();

// Loop modes
action.loop = THREE.LoopRepeat;   // Default: loop forever
action.loop = THREE.LoopOnce;     // Play once and stop
action.loop = THREE.LoopPingPong; // Alternate forward/backward
action.repetitions = 3;

// Clamping
action.clampWhenFinished = true; // Hold last frame when done

// Weight (for blending)
action.setEffectiveWeight(1); // 0-1

// Speed
action.timeScale = 1; // Negative = reverse

// Blending
action.blendMode = THREE.NormalAnimationBlendMode;
action.blendMode = THREE.AdditiveAnimationBlendMode;
```

### Fade In/Out

```javascript
action.reset().fadeIn(0.5).play();
action.fadeOut(0.5);

// Crossfade
action1.crossFadeTo(action2, 0.5, true);
action2.play();
```

## Loading GLTF Animations

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.load("model.glb", (gltf) => {
  const mixer = new THREE.AnimationMixer(gltf.scene);
  const clips = gltf.animations;

  // Play by name
  const walkClip = THREE.AnimationClip.findByName(clips, "Walk");
  if (walkClip) mixer.clipAction(walkClip).play();
});
```

## Skeletal Animation

```javascript
// Access skeleton
const skinnedMesh = model.getObjectByProperty("type", "SkinnedMesh");
const skeleton = skinnedMesh.skeleton;

// Programmatic bone control
const headBone = skeleton.bones.find((b) => b.name === "Head");
if (headBone) headBone.rotation.y = Math.PI / 4;

// Attach objects to bones
const weapon = new THREE.Mesh(weaponGeometry, weaponMaterial);
const handBone = skeleton.bones.find((b) => b.name === "RightHand");
if (handBone) handBone.add(weapon);

// Skeleton helper (debug)
scene.add(new THREE.SkeletonHelper(model));
```

## Animation Blending

All actions play simultaneously at weight 0; weights sum to ~1.

```javascript
const idleAction = mixer.clipAction(idleClip);
const walkAction = mixer.clipAction(walkClip);
const runAction = mixer.clipAction(runClip);

idleAction.play(); walkAction.play(); runAction.play();

function updateAnimations(speed) {
  if (speed < 0.1) {
    idleAction.setEffectiveWeight(1);
    walkAction.setEffectiveWeight(0);
    runAction.setEffectiveWeight(0);
  } else if (speed < 5) {
    const t = speed / 5;
    idleAction.setEffectiveWeight(1 - t);
    walkAction.setEffectiveWeight(t);
    runAction.setEffectiveWeight(0);
  } else {
    const t = Math.min((speed - 5) / 5, 1);
    idleAction.setEffectiveWeight(0);
    walkAction.setEffectiveWeight(1 - t);
    runAction.setEffectiveWeight(t);
  }
}
```

### Additive Blending

```javascript
// Convert to additive
THREE.AnimationUtils.makeClipAdditive(additiveClip);

const additiveAction = mixer.clipAction(additiveClip);
additiveAction.blendMode = THREE.AdditiveAnimationBlendMode;
additiveAction.play();
```

## Animation Utilities

```javascript
const clip = THREE.AnimationClip.findByName(clips, "Walk");
const subclip = THREE.AnimationUtils.subclip(clip, "subclip", 0, 30, 30);
clip.optimize(); // Remove redundant keyframes
```

## Morph Targets

```javascript
// Set by index
mesh.morphTargetInfluences[0] = 0.5;

// Set by name
const smileIndex = mesh.morphTargetDictionary["smile"];
mesh.morphTargetInfluences[smileIndex] = 1;

// Animate
const track = new THREE.NumberKeyframeTrack(
  ".morphTargetInfluences[smile]",
  [0, 0.5, 1],
  [0, 1, 0],
);
```

## Procedural Patterns

### Spring Physics

```javascript
class Spring {
  constructor(stiffness = 100, damping = 10) {
    this.stiffness = stiffness;
    this.damping = damping;
    this.position = 0;
    this.velocity = 0;
    this.target = 0;
  }
  update(dt) {
    const force = -this.stiffness * (this.position - this.target);
    const dampingForce = -this.damping * this.velocity;
    this.velocity += (force + dampingForce) * dt;
    this.position += this.velocity * dt;
    return this.position;
  }
}
```

## Performance Tips

1. Share clips across multiple mixers — `AnimationClip` is data only
2. Call `clip.optimize()` to remove redundant keyframes
3. Pause `mixer.update()` for off-screen characters
4. Use LOD for animation rigs on distant characters

## See Also

- `threejs-fbx-mixamo` — Mixamo FBX multi-clip loading and overlay pipeline
- `threejs-loaders` — Loading animated GLTF/FBX models
