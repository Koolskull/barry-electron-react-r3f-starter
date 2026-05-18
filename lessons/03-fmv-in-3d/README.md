# Lesson 03 — FMV in 3D

> By the end of this lesson, you'll have your existing FMV game running as a 3D texture in an R3F scene, with a branching state machine for navigation, clickable hotspots on the video plane, and a CRT/VHS shader pass over the top.

**Estimated time**: 4–6 hours (faster if you're porting an existing FMV project).
**Prerequisites**: Lessons 01 and 02. An FMV-ish `.mp4` file to play with (a few seconds of any video will do).

---

## Learning objectives

1. Render a video as a Three.js `VideoTexture` rather than a DOM `<video>` overlay, and explain why that matters.
2. Avoid the three classic pitfalls: tainted textures (CORS), tone-mapped color shifts, and decoder leaks on unmount.
3. Build a branching FMV state machine in React, with conditional choices and optional flag-gated branches.
4. Add clickable 3D hotspots tied to specific moments in the video timeline.
5. Apply post-processing (scanlines + chromatic aberration + a custom VHS shader) for retro feel.

---

## Chapter 1 — Why a `VideoTexture`, not a `<video>` tag

Two ways to put video in an Electron window:

1. **DOM `<video>` overlay** — absolute-positioned, sits on top of (or under) the canvas.
2. **`THREE.VideoTexture`** — the video becomes a texture you map onto a 3D plane (or any mesh).

You want option 2 for an actual FMV game, because:

- **Composability** — the video lives in the same 3D space as your HUD, characters, particle effects, lighting. Want a 3D arrow pointing at someone in the frame? Trivial.
- **Shaders apply** — CRT bend, scanlines, VHS distortion all process the video frame just like any other geometry. With a DOM overlay you'd need CSS filters, which are weaker and don't compose with WebGL.
- **Click handling** — raycasting against the video plane gives you the hit point in the texture's UV space, which you can map back to a region of the video. DOM overlays force you to do manual coordinate math.
- **Reactive lighting** — if a scene has a flickering candle, the FMV plane responds to it. Adds a layer of atmospheric integration impossible with overlays.

The tradeoff: audio is decoupled from the visual. The video element still plays its audio via Web Audio. This is **a feature, not a bug** — you can route FMV audio through effects, EQ, or merge it with a synth (see Lesson 05).

---

## Chapter 2 — The production-quality FMVScreen component

The full version (with imperative API for seek/pause/play) lives in [`../../FMV-GUIDE.md`](../../FMV-GUIDE.md). Here's the essence, annotated so you understand each line:

```tsx
import { useMemo, useEffect } from 'react';
import * as THREE from 'three';

interface FMVScreenProps {
  src: string;
  width?: number;
  height?: number;
  loop?: boolean;
  onEnded?: () => void;
}

export function FMVScreen({
  src, width = 16, height = 9, loop = false, onEnded,
}: FMVScreenProps) {
  // useMemo prevents recreating the <video> on every render
  const videoEl = useMemo(() => {
    const v = document.createElement('video');
    v.src = src;
    v.crossOrigin = 'anonymous';   // Required or texture is "tainted" by CORS
    v.loop = loop;
    v.playsInline = true;
    v.preload = 'auto';
    return v;
  }, [src, loop]);

  const texture = useMemo(() => {
    const t = new THREE.VideoTexture(videoEl);
    t.colorSpace = THREE.SRGBColorSpace;    // Avoids weird gamma shifts
    t.minFilter = THREE.LinearFilter;
    t.magFilter = THREE.LinearFilter;
    t.generateMipmaps = false;              // Mipmaps don't make sense for live video
    return t;
  }, [videoEl]);

  useEffect(() => {
    const handleEnded = () => onEnded?.();
    videoEl.addEventListener('ended', handleEnded);
    videoEl.play().catch(() => { /* autoplay policy — handle via UI */ });
    return () => {
      videoEl.removeEventListener('ended', handleEnded);
      videoEl.pause();
      videoEl.src = '';
      videoEl.load();
      texture.dispose();   // Critical: video decoders leak otherwise
    };
  }, [videoEl, texture, onEnded]);

  return (
    <mesh>
      <planeGeometry args={[width, height]} />
      <meshBasicMaterial map={texture} toneMapped={false} />
    </mesh>
  );
}
```

### The three classic pitfalls

| Pitfall | Symptom | Fix |
| --- | --- | --- |
| **No `crossOrigin`** | WebGL refuses to sample the texture; sphere is black | `videoEl.crossOrigin = 'anonymous'` + serve videos from same origin or with CORS headers |
| **Tone mapping kicks in** | Video looks washed out or wrong colors | `colorSpace = THREE.SRGBColorSpace` + material `toneMapped={false}` |
| **No texture.dispose()** | App leaks decoders; eventually Chromium fails to allocate | Cleanup in `useEffect` return — `pause()`, clear src, `.load()`, `texture.dispose()` |

---

## Chapter 3 — Branching state machine

Classic FMV game loop: show a clip, present choices, jump to another clip. This is a state machine.

```tsx
type SceneId = 'title' | 'hallway' | 'kitchen' | 'gameover';

const SCENES: Record<SceneId, { src: string; choices?: { label: string; next: SceneId }[] }> = {
  title:    { src: '/video/title.mp4',    choices: [{ label: 'Start', next: 'hallway' }] },
  hallway:  { src: '/video/hallway.mp4',  choices: [
    { label: 'Go to kitchen', next: 'kitchen' },
    { label: 'Stay',          next: 'gameover' },
  ]},
  kitchen:  { src: '/video/kitchen.mp4',  choices: [{ label: 'Restart', next: 'title' }] },
  gameover: { src: '/video/end.mp4' },
};

function FMVGame() {
  const [scene, setScene] = useState<SceneId>('title');
  const [showChoices, setShowChoices] = useState(false);
  const current = SCENES[scene];

  return (
    <>
      <Canvas><FMVScreen src={current.src} loop={false} onEnded={() => setShowChoices(true)} /></Canvas>
      {showChoices && current.choices?.map(c => (
        <button key={c.next} onClick={() => { setShowChoices(false); setScene(c.next); }}>
          {c.label}
        </button>
      ))}
    </>
  );
}
```

That's a complete FMV game in ~30 lines. Add flags (`hasKey`, `metTheNeighbor`) to gate choices conditionally and you've got a real adventure structure.

The deeper version in [`../../FMV-GUIDE.md#branching-state-machines-the-fmv-game-core`](../../FMV-GUIDE.md) adds:
- `showIf` predicates for conditional choices
- `effect` callbacks for setting flags
- Timed choices (Telltale style)
- Visit counters

---

## Chapter 4 — Hotspots on the video plane

```tsx
function Hotspot({ position, size, onClick, label }: {
  position: [number, number, number];
  size: [number, number];
  onClick: () => void;
  label: string;
}) {
  const [hovered, setHovered] = useState(false);
  return (
    <mesh
      position={position}
      onClick={(e) => { e.stopPropagation(); onClick(); }}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <planeGeometry args={size} />
      <meshBasicMaterial color="white" transparent opacity={hovered ? 0.2 : 0.0} />
    </mesh>
  );
}
```

Place hotspots in front of (smaller Z) your `FMVScreen`. They're invisible by default and become semi-transparent on hover. Clicking advances state.

For hotspots that appear only during specific time ranges, gate them on the FMV's `currentTime`:

```tsx
const [time, setTime] = useState(0);

<FMVScreen src={src} onTimeUpdate={(t) => setTime(t)} />
{time >= 5 && time <= 8 && (
  <Hotspot
    position={[2, 0, 0.1]}
    size={[1, 1]}
    onClick={() => goto('kitchen')}
    label="Walk to door"
  />
)}
```

---

## Chapter 5 — CRT / VHS post-processing

Wrap your canvas in `<EffectComposer>`:

```tsx
import { EffectComposer, ChromaticAberration, Scanline, Noise } from '@react-three/postprocessing';
import { BlendFunction } from 'postprocessing';

<Canvas>
  <FMVScreen src={src} />
  <EffectComposer>
    <Scanline density={1.5} blendFunction={BlendFunction.OVERLAY} />
    <ChromaticAberration offset={[0.002, 0.002]} />
    <Noise opacity={0.08} blendFunction={BlendFunction.OVERLAY} />
  </EffectComposer>
</Canvas>
```

For a more authentic VHS — tracking lines, color bleed, frame jitter — use the custom shader in [`../../FMV-GUIDE.md#crt--vhs--scanline-shaders`](../../FMV-GUIDE.md). Worth pasting in once just to see how clean a GLSL effect is to wire up in R3F.

---

## Chapter 6 — Loading videos from disk in Electron

For bundling vs. external load patterns, see [`../../FMV-GUIDE.md#loading-fmv-clips-from-disk-via-electron`](../../FMV-GUIDE.md).

Short version:

- **Small projects** — drop `.mp4` files in `public/video/` and reference them as `/video/foo.mp4`. Bundled into the EXE.
- **Big projects (lots of FMV)** — keep videos in a `videos/` folder next to the EXE. Use Electron IPC to enumerate them. Load via `file://` URLs.

ffmpeg recipe for FMV-ready encoding:

```bash
ffmpeg -i source.mov \
  -c:v libx264 -preset slow -crf 18 \
  -profile:v high -pix_fmt yuv420p \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  out.mp4
```

The `-movflags +faststart` is non-negotiable for streaming/seeking behavior.

---

## Hands-on tasks

- [ ] **T3.1** Drop an `.mp4` into `public/video/` and render it using `FMVScreen` from this lesson. Confirm it plays inside a 3D scene with `OrbitControls`.
- [ ] **T3.2** Wrap your FMV in `<EffectComposer>` with at least Scanline + Noise. Tweak `density` and `opacity` until you like the look.
- [ ] **T3.3** Build a 3-scene branching FMV (title → choice → ending). Use the `SCENES` pattern from chapter 3.
- [ ] **T3.4** Add a clickable hotspot that appears at a specific time range in the video and triggers a scene change.
- [ ] **T3.5** Add `onTimeUpdate` to the `FMVScreen` and display the current playback time in an overlay (use drei's `<Html>` for the overlay).
- [ ] **T3.6** *(Stretch)* Implement the custom VHS shader from `FMV-GUIDE.md`. Add a UI toggle to enable/disable it.
- [ ] **T3.7** *(Stretch — if you have an existing FMV project)* Port your existing FMV game over to use `FMVScreen` and `SCENES`. The rest of your code shouldn't need to change much.

---

## Quiz

1. You added a video and got a black plane. Console shows a CORS-related warning. What's the fix?
2. The video plays but the colors look washed out compared to the source `.mp4`. What two settings are likely involved?
3. Your FMV game has 50 scenes. After clicking through them, the app gets sluggish and the Memory tab shows growing detached video elements. What did you forget?
4. You want a hotspot to be clickable only between t=8s and t=12s in the video. What's the cleanest pattern?
5. Why does `videoEl.play()` sometimes throw an `NotAllowedError` even though autoplay seems harmless?
6. The Scanline post-processing effect makes your HUD text (rendered via drei's `<Text>`) also get scanlined, which you don't want. How do you exclude UI from the post-processing pipeline?

<details>
<summary>Show answers</summary>

1. The video element needs `crossOrigin = 'anonymous'` and the video has to be served with appropriate CORS headers (or hosted on the same origin). When loading from `public/`, both conditions are usually met automatically — the issue is most often forgetting `crossOrigin` on the element itself, which makes WebGL mark the texture as tainted and refuse to sample it.
2. (a) `texture.colorSpace = THREE.SRGBColorSpace` to set the correct decoding curve. (b) `<meshBasicMaterial toneMapped={false}>` so the renderer's filmic tone mapping doesn't get applied to already-graded video content.
3. The cleanup in `useEffect`'s return. Specifically: `videoEl.pause()`, clear `videoEl.src`, call `videoEl.load()` to flush the decoder, and `texture.dispose()`. Without these, each scene leaves a hanging video element + decoder. The Memory tab's "detached video" count is the diagnostic.
4. Wire the FMV's `onTimeUpdate` to a `useState`, then conditionally render the `<Hotspot>` based on `time >= 8 && time <= 12`. Don't try to do it with CSS visibility — the click handlers need to be inactive too, not just hidden.
5. Browser autoplay policy. Audio playback (and any video that has audio) requires a user gesture to start. The first `.play()` call before the user has clicked anywhere will reject. The fix is to start playback from a click handler, or to mute the video (`muted = true`) for the initial autoplay and unmute later.
6. Render the HUD outside the `<Canvas>` (as plain HTML overlay) — post-processing only affects what's inside the canvas. Alternatively, put HUD elements in a second `<Canvas>` with no post-processing, layered on top via CSS positioning. For overlaid 3D HUD specifically (e.g. an indicator floating in front of the scene), drei's `<Hud>` component manages this for you.

</details>

---

## Reference

- Deep dive: [`../../FMV-GUIDE.md`](../../FMV-GUIDE.md) — has the imperative-API version of FMVScreen with seek/pause, the full state machine with flags, the custom VHS shader code.
- Main README FMV section: [`../../README.md#lesson-3--fmv-video-as-a-3d-texture`](../../README.md)

---

## Next

→ [Lesson 04 — Genesis ROM Integration](../04-genesis-roms/README.md)
