# Lesson 02 — React Three Fiber Primer

> By the end of this lesson, you'll understand how R3F maps Three.js onto React's component model, know when to render through React and when to bypass it, and have a small scene with multiple objects, custom materials, and post-processing running inside your Electron window.

**Estimated time**: 3–4 hours.
**Prerequisites**: Lesson 01 complete. Basic React (function components, hooks).

---

## Learning objectives

1. Recognize the equivalence between plain Three.js code and JSX-based R3F code.
2. Use `useFrame` correctly — and explain why you should *not* call `setState` inside it.
3. Pick the right material for the job (`meshBasicMaterial` vs. `meshStandardMaterial` vs. `meshPhysicalMaterial`).
4. Drop in `drei` helpers (`OrbitControls`, `Stars`, `Environment`, `Text`) without thinking about Three.js internals.
5. Add post-processing (bloom, chromatic aberration, scanlines) via `@react-three/postprocessing`.

---

## Chapter 1 — Three.js in 60 seconds

If you've never touched Three.js, the four concepts to internalize:

| Concept | What it is |
| --- | --- |
| **Scene** | The container for everything. Like the root `<group>` of an SVG. |
| **Camera** | The viewpoint. Has a position, rotation, and a projection (perspective or orthographic). |
| **Mesh** | A 3D object. Made of a geometry (shape) + material (appearance). |
| **Renderer** | The thing that takes a scene + camera and draws it to a canvas using WebGL. |

Plain Three.js setup (you'd write this every time without R3F):

```js
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);
camera.position.z = 5;

function animate() {
  requestAnimationFrame(animate);
  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;
  renderer.render(scene, camera);
}
animate();
```

That's the entire mental model. R3F is the same thing dressed in JSX.

---

## Chapter 2 — The R3F equivalent

```tsx
import { Canvas, useFrame } from '@react-three/fiber';
import { useRef } from 'react';
import * as THREE from 'three';

function Cube() {
  const ref = useRef<THREE.Mesh>(null!);
  useFrame(() => {
    ref.current.rotation.x += 0.01;
    ref.current.rotation.y += 0.01;
  });
  return (
    <mesh ref={ref}>
      <boxGeometry args={[1, 1, 1]} />
      <meshBasicMaterial color="green" />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas camera={{ position: [0, 0, 5] }}>
      <Cube />
    </Canvas>
  );
}
```

Map between the two:

| Plain Three.js | R3F |
| --- | --- |
| `new THREE.Scene()` | `<Canvas>` creates one for you |
| `new THREE.PerspectiveCamera(...)` | `camera={{ position: [...], fov: 75 }}` prop on `<Canvas>` |
| `new THREE.Mesh(geo, mat)` | `<mesh>` with `<boxGeometry>` + `<material>` children |
| `scene.add(mesh)` | Just render the JSX |
| `renderer.render(...)` in `requestAnimationFrame` | Implicit — `<Canvas>` runs the render loop |
| Mutating `cube.rotation.x` in the loop | `useFrame((state, delta) => { ref.current.rotation.x += delta })` |

The naming convention: every Three.js class `THREE.Foo` becomes a lowercase JSX element `<foo>`. R3F detects new types by reflecting over the `THREE` namespace, so you can use any geometry, material, light, helper, etc. by lowercasing its name. There's no R3F-specific list of supported elements.

---

## Chapter 3 — `useFrame` and the render loop

This is the big "don't shoot yourself in the foot" chapter. `useFrame` runs **every frame** — typically 60 times per second, sometimes 120 or 144 on high-refresh displays.

### Rule 1 — Don't `setState` inside `useFrame`

```tsx
// ❌ Don't do this
function Bad() {
  const [rot, setRot] = useState(0);
  useFrame((_, dt) => setRot(r => r + dt));
  return <mesh rotation={[0, rot, 0]}><boxGeometry /><meshBasicMaterial /></mesh>;
}
```

`setState` triggers a React re-render. 60 re-renders per second of your entire component tree is exactly the kind of waste you'd never do in C. Mutate the ref directly instead:

```tsx
// ✅ Do this
function Good() {
  const ref = useRef<THREE.Mesh>(null!);
  useFrame((_, dt) => { ref.current.rotation.y += dt; });
  return <mesh ref={ref}><boxGeometry /><meshBasicMaterial /></mesh>;
}
```

This is the same instinct as "don't allocate inside a vblank handler." Direct mutation of the GPU object is correct and cheap; round-tripping through React's diffing is wrong and expensive.

### Rule 2 — Use `delta` for frame-rate independence

`useFrame` gives you `(state, delta)` where `delta` is seconds since the last frame. If you do `rotation.y += 0.01`, the cube rotates faster on a 144Hz display than a 60Hz one. Use `rotation.y += dt * SPEED` instead.

### Rule 3 — `useFrame` runs even when the tab is hidden

…until the browser throttles it. For things like a music player or a network sync that should keep ticking, don't rely on `useFrame` alone — use `setInterval` or `requestIdleCallback`.

---

## Chapter 4 — Materials and lights

There are three materials you'll use 95% of the time.

### `meshBasicMaterial`

No lighting. The color is the color. Use this for:
- HUD elements
- Video textures (where the source video is already lit)
- Glowing objects (when paired with a bloom pass)

```tsx
<meshBasicMaterial color="#ff00ff" />
```

### `meshStandardMaterial`

Physically-based rendering (PBR). Responds to lights, reflects environment maps, has roughness and metalness. The default for most opaque objects.

```tsx
<meshStandardMaterial
  color="#888"
  metalness={0.6}
  roughness={0.3}
/>
```

### `meshPhysicalMaterial`

A superset of `meshStandardMaterial` with extra properties: clearcoat, transmission (refraction), iridescence, sheen. Slower. Use only when you need the effect.

### Lights to pair with `meshStandardMaterial`

- `<ambientLight intensity={0.3} />` — lights everything uniformly. Use sparingly; flat-looking.
- `<directionalLight position={[5, 5, 5]} intensity={1} />` — like the sun. Parallel rays, can cast shadows.
- `<pointLight position={[0, 2, 0]} intensity={1} />` — like a bare bulb. Omnidirectional.
- `<spotLight angle={Math.PI / 8} />` — cone of light. Like a flashlight.
- `<Environment preset="city" />` (from drei) — image-based lighting. Single biggest "looks pro" upgrade.

A scene without a `meshStandardMaterial`-compatible light source will render black.

---

## Chapter 5 — drei helpers

[`@react-three/drei`](https://github.com/pmndrs/drei) is a collection of ready-made R3F components. The ones you'll reach for constantly:

```tsx
import {
  OrbitControls,    // Drag to rotate camera. Essential for development.
  Stars,            // A nice space-background field of points.
  Environment,      // Image-based lighting from a preset or HDRI.
  Text,             // Render text as a 3D mesh.
  Html,             // Embed HTML inside the 3D scene. Useful for tooltips.
  useTexture,       // Hook to load a texture.
  useGLTF,          // Hook to load a .glb / .gltf model.
  PerspectiveCamera,// Programmable camera.
  Stats,            // FPS overlay. Drop in during development.
} from '@react-three/drei';

<Canvas>
  <Environment preset="sunset" />
  <Stars />
  <mesh>
    <torusKnotGeometry args={[1, 0.3, 128, 16]} />
    <meshStandardMaterial color="hotpink" metalness={0.8} roughness={0.1} />
  </mesh>
  <Text position={[0, 1.5, 0]} fontSize={0.3} color="white">
    Hello Barry
  </Text>
  <OrbitControls />
  <Stats />
</Canvas>
```

The mental model: drei is the "stdlib" for R3F. Anything you find yourself writing twice is probably already in drei.

---

## Chapter 6 — Post-processing

```bash
npm install @react-three/postprocessing postprocessing
```

```tsx
import { EffectComposer, Bloom, ChromaticAberration, Vignette } from '@react-three/postprocessing';

<Canvas>
  {/* ...your scene... */}
  <EffectComposer>
    <Bloom intensity={0.6} luminanceThreshold={0.4} />
    <ChromaticAberration offset={[0.001, 0.001]} />
    <Vignette eskil={false} offset={0.1} darkness={0.6} />
  </EffectComposer>
</Canvas>
```

What each does:

- **Bloom** — bright pixels bleed into their neighbors. Makes neon / emissive materials look luminous.
- **ChromaticAberration** — splits R/G/B slightly. Free "fancy lens" look at small offsets; CRT-like at large ones.
- **Vignette** — darkens the corners. Subtle, draws the eye to the center.
- **DepthOfField** — blurs anything not at the focus distance. Cinematic but expensive.
- **Noise** — adds film grain. Add a tiny amount (`opacity={0.04}`) to everything, immediately less "video-gamey."
- **Scanline** — horizontal lines. CRT vibes.

Post-processing is the single highest-ROI change you can make to an R3F scene. A subtle bloom + vignette + film grain takes a mediocre scene to "this looks like a real game."

---

## Hands-on tasks

- [ ] **T2.1** Add three different geometries to your scene (`<boxGeometry>`, `<sphereGeometry>`, `<torusGeometry>`) with three different colors. Position them so all three are visible.
- [ ] **T2.2** Make one of them rotate using `useFrame`. Use `delta` correctly so the speed is the same regardless of monitor refresh rate.
- [ ] **T2.3** Add a `<directionalLight>` with shadow casting (`castShadow`). Configure the material on a ground plane to receive shadows (`receiveShadow`). Confirm the shadow appears.
- [ ] **T2.4** Add `<OrbitControls>` so you can drag to rotate the camera, then `<Stats />` to see the FPS counter.
- [ ] **T2.5** Add an `<EffectComposer>` with at least Bloom + Vignette. Tweak `intensity` and `luminanceThreshold` until something glows nicely.
- [ ] **T2.6** *(Stretch)* Load a `.glb` model (find one free at sketchfab.com or use a primitive `.glb` from polyhaven.com). Render it with `useGLTF` and rotate it.
- [ ] **T2.7** *(Stretch)* Make an object's color change when you hover over it using `onPointerOver` / `onPointerOut`. Bonus: use a `useState` here — this is the correct case for it, because hover state isn't per-frame.

---

## Quiz

1. You have a state variable `position` that changes every frame inside `useFrame`. The app stutters and DevTools shows constant re-renders. What's the fix?
2. You added a `<meshStandardMaterial>` to a sphere and it appeared as a black silhouette. What's the most likely cause?
3. What's the difference between `useFrame((state, delta) => ...)` and `useEffect(() => { ... }, [])`?
4. When would you choose `meshBasicMaterial` over `meshStandardMaterial`?
5. Why does the JSX `<spotLight>` work even though there's no React component called `SpotLight` defined anywhere in your project?
6. Bloom looks great on a sphere but does nothing to a normal `<meshStandardMaterial>` cube. Why?

<details>
<summary>Show answers</summary>

1. Move the mutating logic out of state and into a `useRef`. Mutate `ref.current.position.x` directly inside `useFrame`. State should be used for things that need to trigger a re-render (e.g. a button label change), not for per-frame animation.
2. No lights in the scene. `meshStandardMaterial` requires at least one light source (ambient, directional, point, etc.) or an `<Environment>` for image-based lighting. With nothing to light it, the material reflects black.
3. `useFrame` runs every frame (60+ times per second) as part of the render loop. `useEffect` runs after React commits a render — typically once on mount and again when its dependencies change. Use `useFrame` for animation; use `useEffect` for setup, cleanup, and side effects tied to component lifecycle.
4. When you don't want lighting to affect the appearance: video textures (already lit by the source video), HUD elements, glow effects that get bloomed, or any case where you want the color value to be the final color.
5. R3F's JSX is reflective. The lowercase tag `<spotLight>` is converted at render time to `new THREE.SpotLight()`. Any class hanging off the `THREE` namespace works this way — no per-element React component required.
6. Bloom only affects pixels brighter than its `luminanceThreshold` (default ~0.9). A standard cube with `color="#888"` and no emissive isn't bright enough. Either: increase the light intensity, give the material an `emissive` color, or lower the threshold. For luminous-looking objects use `meshStandardMaterial color="red" emissive="red" emissiveIntensity={2}` or a `meshBasicMaterial` for guaranteed brightness.

</details>

---

## Reference

- Main R3F primer in [`../../README.md#lesson-2--react-three-fiber-primer`](../../README.md).
- R3F docs: https://docs.pmnd.rs/react-three-fiber
- drei docs: https://github.com/pmndrs/drei
- Three.js docs: https://threejs.org/docs/

---

## Next

→ [Lesson 03 — FMV in 3D](../03-fmv-in-3d/README.md)
