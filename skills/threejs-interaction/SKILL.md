---
name: threejs-interaction
description: Three.js interaction - raycasting, controls, mouse/touch input, object selection. Use when handling user input, implementing click detection, adding camera controls, or creating interactive 3D experiences.
attribution: Adapted from CloudAI-X/threejs-skills (https://github.com/CloudAI-X/threejs-skills). All credit for original content to CloudAI-X contributors.
---

# Three.js Interaction

## Quick Start

```javascript
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

function onClick(event) {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(scene.children);
  if (intersects.length > 0) console.log("Clicked:", intersects[0].object);
}

window.addEventListener("click", onClick);
```

## Raycaster

```javascript
const raycaster = new THREE.Raycaster();

// From camera (mouse picking)
raycaster.setFromCamera(mousePosition, camera);

// From arbitrary origin
raycaster.set(origin, direction); // direction must be normalized

// Intersections
const intersects = raycaster.intersectObjects(objects, recursive);
// intersects[0]: { distance, point, face, faceIndex, object, uv, normal }
```

### Correct Mouse Coordinates

```javascript
// Normalize to [-1, 1] NDC
function updateMouse(event, canvas) {
  const rect = canvas.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
}
```

### Raycaster Options

```javascript
raycaster.near = 0;
raycaster.far = 100;
raycaster.params.Line.threshold = 0.1;
raycaster.params.Points.threshold = 0.1;
raycaster.layers.set(1); // Only intersect layer 1
```

### Hover Effects (Throttled)

```javascript
let lastRaycast = 0;
let hoveredObject = null;

function onMouseMove(event) {
  if (Date.now() - lastRaycast < 50) return; // 20fps max
  lastRaycast = Date.now();

  updateMouse(event, renderer.domElement);
  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(hoverableObjects);

  if (hoveredObject) {
    hoveredObject.material.emissive.set(0x000000);
    document.body.style.cursor = "default";
  }

  if (intersects.length > 0) {
    hoveredObject = intersects[0].object;
    hoveredObject.material.emissive.set(0x444444);
    document.body.style.cursor = "pointer";
  } else {
    hoveredObject = null;
  }
}
```

## Camera Controls

### OrbitControls

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.minPolarAngle = 0;
controls.maxPolarAngle = Math.PI / 2;
controls.minDistance = 2;
controls.maxDistance = 50;
controls.target.set(0, 1, 0);
controls.autoRotate = false;

// Must call in animation loop when damping is enabled
function animate() {
  controls.update();
  renderer.render(scene, camera);
}
```

### PointerLockControls (FPS)

```javascript
import { PointerLockControls } from "three/addons/controls/PointerLockControls.js";

const controls = new PointerLockControls(camera, document.body);
document.addEventListener("click", () => controls.lock());

controls.addEventListener("lock", () => console.log("Pointer locked"));
controls.addEventListener("unlock", () => console.log("Pointer unlocked"));
```

### TransformControls (Editor Gizmo)

```javascript
import { TransformControls } from "three/addons/controls/TransformControls.js";

const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls);
transformControls.attach(selectedMesh);
transformControls.setMode("translate"); // 'translate', 'rotate', 'scale'

transformControls.addEventListener("dragging-changed", (e) => {
  orbitControls.enabled = !e.value; // Disable orbit while dragging
});
```

## Keyboard Input

```javascript
const keys = new Set();
document.addEventListener("keydown", (e) => keys.add(e.code));
document.addEventListener("keyup", (e) => keys.delete(e.code));

function update() {
  const speed = 0.1;
  if (keys.has("KeyW")) player.position.z -= speed;
  if (keys.has("KeyS")) player.position.z += speed;
  if (keys.has("KeyA")) player.position.x -= speed;
  if (keys.has("KeyD")) player.position.x += speed;
  if (keys.has("Space")) player.position.y += speed;
}
```

## Coordinate Conversion

```javascript
// World → Screen
function worldToScreen(position, camera) {
  const v = position.clone().project(camera);
  return {
    x: ((v.x + 1) / 2) * window.innerWidth,
    y: (-(v.y - 1) / 2) * window.innerHeight,
  };
}

// Ray-Plane intersection (e.g., place object on ground)
const groundPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
const intersection = new THREE.Vector3();
raycaster.ray.intersectPlane(groundPlane, intersection);
```

## Performance Tips

1. Throttle `mousemove` raycasts (50ms minimum interval)
2. Use layers to filter raycast targets — `raycaster.layers.set(N)`
3. Use simplified invisible collision meshes for raycasting complex models
4. Disable controls when not needed: `controls.enabled = false`

## See Also

- `threejs-fundamentals` — Camera and scene setup
- `threejs-animation` — Animating interactions
