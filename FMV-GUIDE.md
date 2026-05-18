# FMV Guide — Full Motion Video in React Three Fiber + Electron

Companion to the main `README.md`. This file goes deep on the FMV-specific stuff.

---

## Congrats again, Barry

Building an FMV game in modern 3D is one of those projects that sounds simple until you start doing it. The hard parts (asset pipeline, timing, branching scripting, the *feel* of late-night-90s CD-ROM weirdness) aren't solved by frameworks — they're solved by taste and persistence. You're working on something that doesn't really have a modern template, which is exactly why it'll stand out.

The lessons below are the practical bits that took FMV devs years to figure out in the era of QuickTime 3 and `.cpk` codecs. You get to skip all of it.

---

## Table of contents

1. [Why video-as-texture (not `<video>` overlay)](#why-video-as-texture-not-video-overlay)
2. [The full FMV component, annotated](#the-full-fmv-component-annotated)
3. [Picking a codec for desktop FMV](#picking-a-codec-for-desktop-fmv)
4. [Branching state machines (the FMV game core)](#branching-state-machines-the-fmv-game-core)
5. [Hotspots — clickable regions on video](#hotspots--clickable-regions-on-video)
6. [Syncing 3D objects to the video timeline](#syncing-3d-objects-to-the-video-timeline)
7. [CRT / VHS / scanline shaders](#crt--vhs--scanline-shaders)
8. [Loading FMV clips from disk via Electron](#loading-fmv-clips-from-disk-via-electron)
9. [Authoring workflow — getting clips production-ready](#authoring-workflow--getting-clips-production-ready)

---

## Why video-as-texture (not `<video>` overlay)

There are two ways to put video in your React app:

1. **HTML overlay** — drop a `<video>` element absolutely-positioned on top of the canvas.
2. **Three.js `VideoTexture`** — sample the video as a texture and map it onto a 3D plane (or arbitrary mesh).

The first is *easier* but the second is *correct* for your game, because:

- You can rotate, scale, perspective-warp, and post-process the video with the rest of your 3D scene.
- Shaders apply uniformly — your CRT bend affects the video and the HUD identically.
- Click coordinates from the 3D camera's raycaster are already in the same space as the video, so hotspot detection becomes a `raycaster.intersectObject(videoMesh)` call instead of manual DOM math.
- Lighting from the scene falls on the video. Want a flickering candle next to the screen affecting how the video looks? `meshStandardMaterial` instead of `meshBasicMaterial` and you're done.

The tradeoff is that audio is decoupled from the visual — the video element renders to texture, but its audio still comes out through the WebAudio graph. That's actually a feature, not a bug, because it means you can route the FMV audio through effects (reverb, EQ, the synth from Lesson 5) before it hits the speakers.

---

## The full FMV component, annotated

Here's the production-quality version. The README has a shorter version; this is the one to actually use.

```tsx
// src/components/FMVScreen.tsx
import { useRef, useEffect, useMemo, useState, forwardRef, useImperativeHandle } from 'react';
import * as THREE from 'three';

export interface FMVScreenHandle {
  play: () => Promise<void>;
  pause: () => void;
  seek: (seconds: number) => void;
  getCurrentTime: () => number;
  getDuration: () => number;
}

export interface FMVScreenProps {
  src: string;
  width?: number;
  height?: number;
  loop?: boolean;
  muted?: boolean;
  autoplay?: boolean;
  /** Called when video ends. Use to drive branching state machine. */
  onEnded?: () => void;
  /** Called on every animation frame with current playback time in seconds. */
  onTimeUpdate?: (timeSec: number) => void;
  /** WebAudio destination so you can wire FMV audio into your synth chain. */
  audioDestination?: AudioNode;
}

export const FMVScreen = forwardRef<FMVScreenHandle, FMVScreenProps>(({
  src,
  width = 16,
  height = 9,
  loop = false,
  muted = false,
  autoplay = true,
  onEnded,
  onTimeUpdate,
  audioDestination,
}, ref) => {
  const [ready, setReady] = useState(false);

  const videoEl = useMemo(() => {
    const v = document.createElement('video');
    v.src = src;
    v.crossOrigin = 'anonymous';
    v.loop = loop;
    v.muted = muted;
    v.playsInline = true;
    v.preload = 'auto';
    return v;
  }, [src, loop, muted]);

  const texture = useMemo(() => {
    const t = new THREE.VideoTexture(videoEl);
    t.colorSpace = THREE.SRGBColorSpace;
    t.minFilter = THREE.LinearFilter;
    t.magFilter = THREE.LinearFilter;
    t.generateMipmaps = false;
    return t;
  }, [videoEl]);

  // Wire video element into Web Audio graph (optional, for processing)
  useEffect(() => {
    if (!audioDestination) return;
    const ctx = audioDestination.context;
    const source = (ctx as AudioContext).createMediaElementSource(videoEl);
    source.connect(audioDestination);
    return () => source.disconnect();
  }, [videoEl, audioDestination]);

  useEffect(() => {
    const onLoaded = () => setReady(true);
    const onTime = () => onTimeUpdate?.(videoEl.currentTime);
    const onEndedInner = () => onEnded?.();

    videoEl.addEventListener('loadeddata', onLoaded);
    videoEl.addEventListener('timeupdate', onTime);
    videoEl.addEventListener('ended', onEndedInner);

    if (autoplay) {
      videoEl.play().catch(() => {
        // Autoplay blocked — needs user gesture. App-level UI should handle.
      });
    }

    return () => {
      videoEl.removeEventListener('loadeddata', onLoaded);
      videoEl.removeEventListener('timeupdate', onTime);
      videoEl.removeEventListener('ended', onEndedInner);
      videoEl.pause();
      videoEl.src = '';
      videoEl.load();
      texture.dispose();
    };
  }, [videoEl, texture, autoplay, onTimeUpdate, onEnded]);

  useImperativeHandle(ref, () => ({
    play: () => videoEl.play(),
    pause: () => videoEl.pause(),
    seek: (s) => { videoEl.currentTime = s; },
    getCurrentTime: () => videoEl.currentTime,
    getDuration: () => videoEl.duration,
  }), [videoEl]);

  if (!ready) {
    // Render a black plane while video buffers — same dimensions
    return (
      <mesh>
        <planeGeometry args={[width, height]} />
        <meshBasicMaterial color="#000" />
      </mesh>
    );
  }

  return (
    <mesh>
      <planeGeometry args={[width, height]} />
      <meshBasicMaterial map={texture} toneMapped={false} />
    </mesh>
  );
});
FMVScreen.displayName = 'FMVScreen';
```

The `forwardRef` + `useImperativeHandle` pattern is what lets you call `fmvRef.current?.seek(12.5)` from a parent component — useful for scrubbing, save states, "rewind to last choice" gameplay mechanics.

---

## Picking a codec for desktop FMV

Electron is Chromium. Chromium plays:

- **H.264** (the universal one). Works everywhere. Use it unless you have a reason not to.
- **VP9**. Better compression than H.264, royalty-free. Plays in Chromium without issue.
- **AV1**. Even better compression. Decoder is slower; for FMV-heavy games on older laptops, may stutter.
- **Theora / Ogg**. Don't.

For the actual `.mp4` files in your `public/video/` folder, here's what experienced FMV/cutscene devs use as defaults:

```bash
# Use ffmpeg. This is the recipe.
ffmpeg -i source.mov \
  -c:v libx264 -preset slow -crf 18 \
  -profile:v high -pix_fmt yuv420p \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  out.mp4
```

The `-movflags +faststart` flag matters specifically for streaming/seeking — it puts the MOOV atom at the start of the file so Chromium can start decoding without downloading the whole thing first. If your FMV ever feels like it takes too long to "start," this is usually why.

For pixel-art / low-res FMV (where you want big chunky pixels), use `-vf "scale=320:240:flags=neighbor"` to downsample with nearest-neighbor and then upscale via texture filtering in Three.js (set `minFilter = THREE.NearestFilter`).

---

## Branching state machines (the FMV game core)

FMV games are mostly state machines. The cleanest pattern in React:

```tsx
// src/game/scenes.ts
export type SceneId =
  | 'title'
  | 'street-day'
  | 'apartment-day'
  | 'apartment-night'
  | 'rooftop'
  | 'gameover-bad'
  | 'gameover-good';

export interface Choice {
  label: string;
  next: SceneId;
  /** Optional condition. If false, choice is hidden. Use for inventory/flag gating. */
  showIf?: (flags: GameFlags) => boolean;
  /** Mutate flags when this choice is taken. */
  effect?: (flags: GameFlags) => void;
}

export interface Scene {
  src: string;
  /** When the video ends, what happens? Default: hold last frame, show choices. */
  onEnd?: 'choices' | { goto: SceneId };
  choices?: Choice[];
  /** Optional timed choices (Telltale-style). When set, choices appear at this time. */
  choiceShowAt?: number;
  /** If set, an unanswered choice this many seconds after choiceShowAt picks the default. */
  choiceTimeout?: { afterSec: number; defaultIdx: number };
}

export interface GameFlags {
  hasKey: boolean;
  trustedNeighbor: boolean;
  visitCount: Record<SceneId, number>;
}

export const SCENES: Record<SceneId, Scene> = {
  'title': {
    src: '/video/title.mp4',
    onEnd: 'choices',
    choices: [{ label: 'Start', next: 'street-day' }],
  },
  'street-day': {
    src: '/video/street-day.mp4',
    onEnd: 'choices',
    choices: [
      { label: 'Knock on the door', next: 'apartment-day' },
      { label: 'Climb the fire escape', next: 'rooftop', showIf: (f) => f.hasKey },
    ],
  },
  // ... and so on
  'gameover-bad': { src: '/video/end-bad.mp4' },
  'gameover-good': { src: '/video/end-good.mp4' },
};
```

And the runtime:

```tsx
// src/game/FMVGame.tsx
import { useState, useRef, useCallback } from 'react';
import { Canvas } from '@react-three/fiber';
import { FMVScreen, FMVScreenHandle } from '../components/FMVScreen';
import { SCENES, SceneId, GameFlags, Choice } from './scenes';

const INITIAL_FLAGS: GameFlags = {
  hasKey: false,
  trustedNeighbor: false,
  visitCount: {} as Record<SceneId, number>,
};

export function FMVGame() {
  const [sceneId, setSceneId] = useState<SceneId>('title');
  const [flags, setFlags] = useState<GameFlags>(INITIAL_FLAGS);
  const [showChoices, setShowChoices] = useState(false);
  const fmvRef = useRef<FMVScreenHandle>(null);

  const scene = SCENES[sceneId];

  const goto = useCallback((next: SceneId, effect?: Choice['effect']) => {
    setFlags(f => {
      const newFlags = { ...f, visitCount: { ...f.visitCount, [next]: (f.visitCount[next] ?? 0) + 1 } };
      effect?.(newFlags);
      return newFlags;
    });
    setShowChoices(false);
    setSceneId(next);
  }, []);

  const onEnded = useCallback(() => {
    if (scene.onEnd === 'choices' || scene.onEnd === undefined) {
      setShowChoices(true);
    } else if (scene.onEnd && typeof scene.onEnd === 'object') {
      goto(scene.onEnd.goto);
    }
  }, [scene, goto]);

  const visibleChoices = scene.choices?.filter(c => !c.showIf || c.showIf(flags)) ?? [];

  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <Canvas camera={{ position: [0, 0, 6] }}>
        <FMVScreen
          ref={fmvRef}
          src={scene.src}
          autoplay
          onEnded={onEnded}
        />
      </Canvas>
      {showChoices && (
        <div style={{
          position: 'absolute', bottom: '10%', left: '50%', transform: 'translateX(-50%)',
          display: 'flex', gap: 16,
        }}>
          {visibleChoices.map((c, i) => (
            <button
              key={i}
              onClick={() => goto(c.next, c.effect)}
              style={{ padding: '12px 24px', fontSize: 18 }}
            >
              {c.label}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

That's a complete FMV game engine in ~80 lines of TypeScript. Add scenes to `SCENES`, add `.mp4` files to `public/video/`, ship.

---

## Hotspots — clickable regions on video

The 90s FMV move: invisible polygons over parts of the video frame that the player can click. Implemented in R3F via raycasting against transparent meshes:

```tsx
import { useState } from 'react';
import { ThreeEvent } from '@react-three/fiber';

function Hotspot({
  position,
  size,
  label,
  onClick,
}: {
  position: [number, number, number];
  size: [number, number];
  label: string;
  onClick: () => void;
}) {
  const [hovered, setHovered] = useState(false);
  return (
    <mesh
      position={position}
      onClick={(e: ThreeEvent<MouseEvent>) => { e.stopPropagation(); onClick(); }}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <planeGeometry args={size} />
      <meshBasicMaterial
        color={hovered ? '#fff' : '#fff'}
        transparent
        opacity={hovered ? 0.2 : 0.0}
      />
    </mesh>
  );
}
```

Place these in front of (smaller Z) your `FMVScreen` mesh at the screen-space coordinates of, say, a door in the frame. The cursor will change when hovering, and clicking advances state.

For hotspots that appear/disappear over time (e.g. a character is only clickable for 3 seconds while they're on-screen), gate the render on `currentTime` from the FMV's `onTimeUpdate` callback.

---

## Syncing 3D objects to the video timeline

This is where R3F earns its keep over plain `<video>`. Suppose you want a 3D arrow to appear next to a character when they enter the frame at 0:08, and follow them until they leave at 0:12:

```tsx
function GuidedArrow({ time }: { time: number }) {
  // Tracking data — in production, generate this from video annotations
  const TRACK: Array<{ t: number; pos: [number, number, number] }> = [
    { t: 8.0,  pos: [-3, 1, 0] },
    { t: 9.0,  pos: [-2, 1, 0] },
    { t: 10.0, pos: [-1, 1, 0] },
    { t: 11.0, pos: [ 0, 1, 0] },
    { t: 12.0, pos: [ 1, 1, 0] },
  ];

  if (time < TRACK[0].t || time > TRACK[TRACK.length - 1].t) return null;

  // Linear interpolation between adjacent keyframes
  let i = TRACK.findIndex(k => k.t > time) - 1;
  if (i < 0) i = 0;
  const a = TRACK[i], b = TRACK[i + 1];
  const u = (time - a.t) / (b.t - a.t);
  const pos: [number, number, number] = [
    a.pos[0] + (b.pos[0] - a.pos[0]) * u,
    a.pos[1] + (b.pos[1] - a.pos[1]) * u,
    a.pos[2] + (b.pos[2] - a.pos[2]) * u,
  ];

  return (
    <mesh position={pos} rotation={[0, 0, Math.PI / 2]}>
      <coneGeometry args={[0.2, 0.6, 8]} />
      <meshStandardMaterial color="#ff3" emissive="#ff3" emissiveIntensity={2} />
    </mesh>
  );
}
```

In your FMV game component, hold a `currentTime` state, update it from the FMV's `onTimeUpdate` callback (throttle to 30Hz if you want), pass it to `<GuidedArrow time={currentTime} />`. Now the 3D arrow is locked to the video timeline.

You can author the `TRACK` arrays by hand for short clips, or build a quick in-Electron annotation tool: play the video, click "set keyframe" at the current time, store the 3D position. Five minutes of work, saves hours of guessing.

---

## CRT / VHS / scanline shaders

The single biggest visual upgrade for FMV is post-processing. R3F integrates with `@react-three/postprocessing` (which wraps `postprocessing`):

```bash
npm install @react-three/postprocessing postprocessing
```

```tsx
import { EffectComposer, Noise, ChromaticAberration, Scanline } from '@react-three/postprocessing';
import { BlendFunction } from 'postprocessing';

<Canvas>
  <FMVScreen src="/video/scene1.mp4" />
  <EffectComposer>
    <Scanline density={1.5} blendFunction={BlendFunction.OVERLAY} />
    <ChromaticAberration offset={[0.002, 0.002]} />
    <Noise opacity={0.08} blendFunction={BlendFunction.OVERLAY} />
  </EffectComposer>
</Canvas>
```

For a *custom* VHS effect (tracking lines, wobble, color bleeding), write your own shader pass:

```tsx
import { Effect } from 'postprocessing';
import { Uniform } from 'three';

const fragmentShader = /* glsl */ `
  uniform float uTime;

  void mainImage(const in vec4 inputColor, const in vec2 uv, out vec4 outputColor) {
    // Vertical jitter — picks a row, displaces it horizontally
    float jitter = step(0.995, sin(uv.y * 100.0 + uTime * 5.0));
    vec2 sampleUV = uv;
    sampleUV.x += jitter * 0.02;

    // Color bleed — sample R and B with slight horizontal offset
    float r = texture2D(inputBuffer, sampleUV + vec2( 0.003, 0)).r;
    float g = texture2D(inputBuffer, sampleUV).g;
    float b = texture2D(inputBuffer, sampleUV + vec2(-0.003, 0)).b;

    // Scanline darkening — every other row slightly darker
    float scan = 0.85 + 0.15 * sin(uv.y * 800.0);

    outputColor = vec4(vec3(r, g, b) * scan, inputColor.a);
  }
`;

class VHSEffect extends Effect {
  constructor() {
    super('VHSEffect', fragmentShader, {
      uniforms: new Map([['uTime', new Uniform(0)]]),
    });
  }
  update(_renderer: unknown, _inputBuffer: unknown, deltaTime: number) {
    const u = this.uniforms.get('uTime')!;
    u.value += deltaTime;
  }
}
```

Then wrap it for R3F:

```tsx
import { forwardRef, useMemo } from 'react';

const VHS = forwardRef((_props, ref) => {
  const effect = useMemo(() => new VHSEffect(), []);
  return <primitive ref={ref} object={effect} dispose={null} />;
});

// Usage:
// <EffectComposer><VHS /></EffectComposer>
```

GLSL is C-ish enough that this should read naturally. The big mental shift from SGDK shaders (which... there weren't really shaders on Genesis) is that this code runs per-pixel, in parallel, on the GPU.

---

## Loading FMV clips from disk via Electron

For a game shipped as an EXE, you have two options:

### Option A — Bundle videos as static assets

Put `.mp4` files in `public/video/`. They'll be bundled into the final `.asar` / app resources. Reference them by URL: `/video/intro.mp4`.

Pros: simple, works everywhere.
Cons: ASAR has a size limit in some packaging configs; large video bundles inflate the installer.

### Option B — Load from external `videos/` folder next to the EXE

Better for big games. In main process:

```ts
ipcMain.handle('video:list', async () => {
  const videosDir = path.join(path.dirname(app.getPath('exe')), 'videos');
  const files = await fs.readdir(videosDir);
  return files.filter(f => f.endsWith('.mp4')).map(f => ({
    name: f,
    url: `file://${path.join(videosDir, f)}`,
  }));
});
```

In renderer:

```tsx
const videos = await window.api.listVideos();
// videos = [{ name: 'intro.mp4', url: 'file:///.../videos/intro.mp4' }, ...]
```

Pass the `url` into `<FMVScreen src={...} />` directly — Chromium reads `file://` URLs natively. Just make sure your main process has `webPreferences.webSecurity` set such that this is allowed (in production builds it usually is for local files).

---

## Authoring workflow — getting clips production-ready

The boring-but-essential stuff:

1. **Standardize on a resolution and framerate.** 1280×720 @ 30fps is a good default — looks fine on modern displays, compresses well, no judder. Don't mix framerates between clips; the seams are visible.
2. **Re-encode all clips with the same ffmpeg settings.** Inconsistent codecs cause Chromium to do warm-up work between scenes, which manifests as a half-second freeze when transitioning.
3. **Audio at 48kHz, stereo, AAC 192kbps.** Loudness-normalized to -14 LUFS (use `ffmpeg-normalize` or Adobe Audition).
4. **Pre-render the last frame of each clip as a still image.** When the video ends, swap the video texture for the still on the same plane — avoids the brief blank frame between clips.
5. **Keep a manifest.** `public/video/manifest.json` listing every clip with metadata (duration, title, hotspot tracks, etc.). Easier to keep your `SCENES` map in sync this way.

---

## What to build first

If your existing FMV game is more or less working in React + Three.js, here's the highest-leverage order to migrate it into this Electron starter:

1. Copy your video files into `public/video/`.
2. Replace whatever you're using to render video with the `FMVScreen` component from this guide.
3. Wrap your scene graph in a `<Canvas>` and (if it isn't already) move all your video into 3D space — even if just as a flat plane.
4. Add `EffectComposer` with one or two of the post-process effects above. Even a subtle scanline pass elevates the whole thing instantly.
5. `npm run dev` and play through. Once it feels right, `npm run make` and you've got a desktop EXE.

You're not starting from scratch — you're upgrading what you've got. That's a much shorter trip than it feels like.
