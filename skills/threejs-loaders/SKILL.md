---
name: threejs-loaders
description: Three.js asset loading - GLTF, textures, images, models, async patterns. Use when loading 3D models, textures, HDR environments, or managing loading progress.
---

# Three.js Loaders

## Quick Start

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.load("model.glb", (gltf) => {
  scene.add(gltf.scene);
});

const textureLoader = new THREE.TextureLoader();
const texture = textureLoader.load("texture.jpg");
```

## LoadingManager

```javascript
const manager = new THREE.LoadingManager();
manager.onLoad = () => startGame();
manager.onProgress = (url, loaded, total) => {
  updateProgressBar((loaded / total) * 100);
};
manager.onError = (url) => console.error(`Error loading: ${url}`);

const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
```

## GLTF/GLB Loading

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.load("model.glb", (gltf) => {
  const model = gltf.scene;
  scene.add(model);

  // Animations
  if (gltf.animations.length > 0) {
    const mixer = new THREE.AnimationMixer(model);
    gltf.animations.forEach((clip) => mixer.clipAction(clip).play());
  }
});
```

### With Draco Compression

```javascript
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath("https://www.gstatic.com/draco/versioned/decoders/1.5.6/");

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);
gltfLoader.load("model.glb", (gltf) => scene.add(gltf.scene));
```

## FBX Loading

```javascript
import { FBXLoader } from "three/addons/loaders/FBXLoader.js";

const loader = new FBXLoader();
loader.load("model.fbx", (object) => {
  object.scale.setScalar(0.01); // Mixamo exports at 100x scale

  const mixer = new THREE.AnimationMixer(object);
  object.animations.forEach((clip) => mixer.clipAction(clip).play());

  scene.add(object);
});
```

## Texture Loading

```javascript
const loader = new THREE.TextureLoader();
loader.load("texture.jpg", (tex) => {
  tex.colorSpace = THREE.SRGBColorSpace; // For albedo/color maps
  tex.wrapS = THREE.RepeatWrapping;
  tex.wrapT = THREE.RepeatWrapping;
  tex.repeat.set(2, 2);
  tex.anisotropy = renderer.capabilities.getMaxAnisotropy();
  material.map = tex;
  material.needsUpdate = true;
});
```

### HDR Environment

```javascript
import { RGBELoader } from "three/addons/loaders/RGBELoader.js";

const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

new RGBELoader().load("environment.hdr", (texture) => {
  const envMap = pmremGenerator.fromEquirectangular(texture).texture;
  scene.environment = envMap;
  scene.background = envMap;
  texture.dispose();
  pmremGenerator.dispose();
});
```

## Async/Promise Patterns

```javascript
// Promisify any loader
function loadGLTF(url) {
  return new Promise((resolve, reject) => {
    new GLTFLoader().load(url, resolve, undefined, reject);
  });
}

// Load multiple in parallel
async function loadAssets() {
  const [model, envTexture] = await Promise.all([
    loadGLTF("model.glb"),
    loadRGBE("environment.hdr"),
  ]);
  scene.add(model.scene);
  scene.environment = envTexture;
}
```

### Parse from ArrayBuffer

```javascript
const response = await fetch("model.glb");
const buffer = await response.arrayBuffer();

new GLTFLoader().parse(buffer, "", (gltf) => {
  scene.add(gltf.scene);
});
```

## Caching

```javascript
THREE.Cache.enabled = true; // Enable built-in URL cache

// Manual cache
const modelCache = new Map();
async function loadCached(url) {
  if (modelCache.has(url)) return modelCache.get(url).clone();
  const gltf = await loadGLTF(url);
  modelCache.set(url, gltf.scene);
  return gltf.scene.clone();
}
```

## Error Handling

```javascript
async function loadWithRetry(url, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await loadGLTF(url);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise((r) => setTimeout(r, 1000 * (i + 1)));
    }
  }
}
```

## Performance Tips

1. Use Draco compression for large geometry
2. Use KTX2/Basis for textures
3. Enable `THREE.Cache.enabled = true`
4. Use `loader.setPath()` to avoid repeating base paths
5. Load progressively — show placeholder while loading

## See Also

- `threejs-fbx-mixamo` — Multi-FBX Mixamo animation loading pipeline
- `threejs-animation` — Playing loaded animations
