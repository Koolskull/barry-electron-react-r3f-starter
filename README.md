# Barry's Electron + React + R3F Desktop Starter

A learning-focused starter repo built specifically for **Barry** — a developer with a strong C / SGDK (Sega Genesis Development Kit) background who's moving into modern web tech and wants to ship his React + Three.js / React Three Fiber projects (including his FMV game) as real desktop applications.

This README is the main lesson hub. The companion guides (`FMV-GUIDE.md`, `GENESIS-EMU-GUIDE.md`, `SOUND-DESIGN-JUCE-BRIDGE.md`) go deeper on specific topics, but everything important is explained here too.

---

## A note before we start (because you're skeptical of AI — fair)

This isn't trying to replace what you already know. Your SGDK and C background is genuinely useful here — most of what makes modern desktop app development hard is stuff you already understand: memory layout, real-time constraints, hardware buffers, the difference between a frontend (renderer) and a backend (main process). The tools in this repo (Electron, React, Three.js, JUCE) are just *new shells around concepts you've already wrestled with on bare metal*.

If something in here sounds like marketing fluff, ignore it. The code examples are what matter. Type them out, run them, break them, fix them. That's how this stuff actually goes from "AI suggestion" to "skill you own."

---

## Big congrats on the FMV game

Building a working **Full Motion Video game in Three.js / React Three Fiber** is no small thing. FMV as a genre (Night Trap, Phantasmagoria, the Sega CD library) has a very specific feel and the developers who pulled it off in the 90s were doing custom video codecs, branching state machines, and synchronized scripting on hardware that barely had enough RAM to hold a single frame. Bringing that genre into modern 3D, with interactive overlays and shaders, is a real project. The rest of this repo is built around helping you ship it as a polished desktop app.

---

## Table of contents

1. [Quick start](#quick-start)
2. [How Electron actually works (for C devs)](#how-electron-actually-works-for-c-devs)
3. [Project structure](#project-structure)
4. [Lesson 1 — Wrapping any React app as a desktop EXE](#lesson-1--wrapping-any-react-app-as-a-desktop-exe)
5. [Lesson 2 — React Three Fiber primer](#lesson-2--react-three-fiber-primer)
6. [Lesson 3 — FMV: video as a 3D texture](#lesson-3--fmv-video-as-a-3d-texture)
7. [Lesson 4 — Loading Sega Genesis ROMs in the desktop app](#lesson-4--loading-sega-genesis-roms-in-the-desktop-app)
8. [Lesson 5 — Sound design: from YM2612 to JUCE, VST, AU](#lesson-5--sound-design-from-ym2612-to-juce-vst-au)
9. [Lesson 6 — Hybrid projects (FMV + Genesis + synth)](#lesson-6--hybrid-projects-fmv--genesis--synth)
10. [Building and shipping a real EXE](#building-and-shipping-a-real-exe)
11. [Debugging tips that map to your SGDK workflow](#debugging-tips-that-map-to-your-sgdk-workflow)
12. [Where to go from here](#where-to-go-from-here)

---

## Quick start

```bash
git clone https://github.com/Koolskull/barry-electron-react-r3f-starter.git
cd barry-electron-react-r3f-starter
npm install
npm run dev
```

If `npm run dev` opens a desktop window with a 3D scene in it, you're done with setup. From here, every lesson below is something you can paste into a file and run.

To produce a real installable `.exe` / `.dmg` / AppImage:

```bash
npm run make
```

The output lands in `out/`.

> **Heads up:** This starter uses `electron-vite` + Electron Forge under the hood. If `npm install` complains about a missing template, run `npx create-electron-app@latest barry-app --template=vite-typescript` in a sibling folder, then copy `src/`, `electron/`, and `vite.config.ts` over. The lessons in this README work identically either way.

---

## How Electron actually works (for C devs)

Mental model — this is the one piece of jargon to get straight, everything else follows from it:

| C / SGDK concept | Electron equivalent | What it actually is |
| --- | --- | --- |
| `main()` in your C program | **Main process** (`electron/main.ts`) | A Node.js process that owns the app lifecycle, creates windows, talks to the OS (filesystem, dialogs, menus). One per app. |
| Hardware registers / VDP | **Renderer process** (your React code) | A Chromium browser window. Has access to the DOM, WebGL, Web Audio. Multiple allowed — one per window. |
| DMA / safe memory bridge | **Preload script** (`electron/preload.ts`) | A privileged script that runs *before* your renderer. The only safe way to expose Node APIs (filesystem, IPC) to your React code without breaking sandboxing. |
| Inline asm / `dmaCopy()` | **IPC (`ipcMain` / `ipcRenderer`)** | Message passing between main and renderer. Asynchronous by default. Use `invoke`/`handle` for request-response. |

If you've ever debugged a Genesis game where the 68k tried to touch Z80 RAM while the Z80 still had the bus — yeah, the renderer-touching-Node-APIs-directly thing is the same class of bug. `contextIsolation: true` + a preload script is Electron's "give the bus back properly."

---

## Project structure

```
barry-electron-react-r3f-starter/
├── electron/
│   ├── main.ts             # Main process — creates the window, handles OS stuff
│   └── preload.ts          # Safe bridge between Node and React
├── src/
│   ├── main.tsx            # React entry point
│   ├── App.tsx             # Top-level component (drop your FMV scene here)
│   ├── components/
│   │   ├── FMVScreen.tsx
│   │   ├── EmulatorPlayer.tsx
│   │   └── Genesis3DVisualizer.tsx
│   └── styles.css
├── public/
│   └── roms/               # Your legally-owned Genesis ROMs (gitignored)
├── FMV-GUIDE.md
├── GENESIS-EMU-GUIDE.md
├── SOUND-DESIGN-JUCE-BRIDGE.md
├── vite.config.ts
├── package.json
└── README.md               # You're reading it
```

The `public/roms/` directory should be in `.gitignore`. ROMs you own are legal to dump for personal use; redistributing them isn't. Treat that directory like you'd treat `.env`.

---

## Lesson 1 — Wrapping any React app as a desktop EXE

The minimum viable Electron app is two files: a main process and a renderer. Here's what the starter wires up.

### `electron/main.ts`

```ts
import { app, BrowserWindow } from 'electron';
import path from 'node:path';

const isDev = !app.isPackaged;

function createWindow() {
  const win = new BrowserWindow({
    width: 1280,
    height: 800,
    backgroundColor: '#000000',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      // Enable for R3F: hardware accel is on by default but worth knowing the flag
      // additionalArguments: ['--enable-gpu-rasterization']
    },
  });

  if (isDev) {
    win.loadURL('http://localhost:5173');
    win.webContents.openDevTools({ mode: 'detach' });
  } else {
    win.loadFile(path.join(__dirname, '../renderer/index.html'));
  }
}

app.whenReady().then(createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});

app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) createWindow();
});
```

A few things worth noting:

- `contextIsolation: true` and `nodeIntegration: false` are the secure defaults. Don't turn them off "to make things easier" — you'll be exposing your renderer (which runs arbitrary web content, including any video URLs you load) directly to the OS filesystem.
- `app.isPackaged` is the most reliable way to detect dev vs. production. Don't use `NODE_ENV` — Vite owns that variable for the renderer.
- The macOS `activate` handler is there because Macs let you close all windows without quitting the app. If you skip it, your dock icon will look weird.

### `electron/preload.ts`

```ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('api', {
  openRomDialog: () => ipcRenderer.invoke('rom:open'),
  readRom: (filepath: string) => ipcRenderer.invoke('rom:read', filepath),
  saveScreenshot: (dataUrl: string) => ipcRenderer.invoke('screenshot:save', dataUrl),
});
```

In React, you'll then call `window.api.openRomDialog()` — and that's the *only* surface area through which the renderer can touch Node APIs. Clean separation.

### `src/App.tsx` (minimum viable R3F scene)

```tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Stars } from '@react-three/drei';
import { useRef } from 'react';
import * as THREE from 'three';
import { useFrame } from '@react-three/fiber';

function SpinningBox() {
  const mesh = useRef<THREE.Mesh>(null!);
  useFrame((_, dt) => {
    mesh.current.rotation.x += dt * 0.5;
    mesh.current.rotation.y += dt * 0.8;
  });
  return (
    <mesh ref={mesh}>
      <boxGeometry args={[2, 2, 2]} />
      <meshStandardMaterial color="hotpink" />
    </mesh>
  );
}

export default function App() {
  return (
    <div style={{ width: '100vw', height: '100vh', background: '#000' }}>
      <Canvas camera={{ position: [0, 0, 6] }}>
        <ambientLight intensity={0.4} />
        <pointLight position={[5, 5, 5]} intensity={1.0} />
        <SpinningBox />
        <Stars radius={50} depth={50} count={2000} factor={4} fade />
        <OrbitControls />
      </Canvas>
    </div>
  );
}
```

Run `npm run dev`. You should get a desktop window with a pink cube spinning over a starfield, draggable with the mouse. That's the foundation — everything else in this repo is just adding stuff on top of this.

---

## Lesson 2 — React Three Fiber primer

R3F is a thin React wrapper over Three.js. The mental shift is small:

| Plain Three.js | R3F |
| --- | --- |
| `new THREE.Mesh(geo, mat)` | `<mesh><boxGeometry/><meshStandardMaterial/></mesh>` |
| `scene.add(mesh)` | Just render the JSX |
| `renderer.render(scene, camera)` in a `requestAnimationFrame` | Implicit — `<Canvas>` runs the loop |
| `requestAnimationFrame` callback | `useFrame((state, delta) => { ... })` |
| `THREE.Clock` | First arg to `useFrame` has `state.clock` |

The biggest gotcha for someone coming from imperative code: **don't mutate state inside `useFrame` and expect React to re-render**. `useFrame` runs every frame outside React's render cycle — mutate `ref.current.position.x` directly. Don't `setState` per frame; you'll thrash.

Same rule applies in C: you wouldn't trigger a vblank interrupt to do palette work that the raster's about to overwrite. Stay in the render loop for render-loop work.

### A slightly more interesting scene

```tsx
import { Canvas, useFrame } from '@react-three/fiber';
import { Environment, OrbitControls } from '@react-three/drei';
import { useRef } from 'react';
import * as THREE from 'three';

function Asteroid({ seed }: { seed: number }) {
  const ref = useRef<THREE.Mesh>(null!);
  const offset = seed * 1.7;
  useFrame((state) => {
    const t = state.clock.elapsedTime + offset;
    ref.current.position.x = Math.cos(t * 0.3) * (3 + seed * 0.4);
    ref.current.position.z = Math.sin(t * 0.3) * (3 + seed * 0.4);
    ref.current.rotation.y += 0.01;
  });
  return (
    <mesh ref={ref}>
      <icosahedronGeometry args={[0.3, 0]} />
      <meshStandardMaterial color="#888" flatShading />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas camera={{ position: [0, 4, 8] }}>
      <Environment preset="night" />
      <ambientLight intensity={0.2} />
      <directionalLight position={[5, 8, 5]} intensity={1} castShadow />
      {Array.from({ length: 12 }, (_, i) => <Asteroid key={i} seed={i} />)}
      <OrbitControls />
    </Canvas>
  );
}
```

This is the kind of scene you can extend with your FMV footage as a centerpiece, with audio-reactive lighting, etc.

---

## Lesson 3 — FMV: video as a 3D texture

This is the lesson for your FMV game. The trick: an HTML `<video>` element can be used as a Three.js texture directly. Every frame, Three.js samples whatever the video is currently showing.

### A clean FMV component

```tsx
import { useRef, useEffect, useMemo } from 'react';
import * as THREE from 'three';
import { useFrame, useThree } from '@react-three/fiber';

interface FMVScreenProps {
  src: string;
  width?: number;
  height?: number;
  loop?: boolean;
  onEnded?: () => void;
}

export function FMVScreen({
  src,
  width = 16,
  height = 9,
  loop = true,
  onEnded,
}: FMVScreenProps) {
  const videoEl = useMemo(() => {
    const v = document.createElement('video');
    v.src = src;
    v.crossOrigin = 'anonymous';
    v.loop = loop;
    v.muted = false;
    v.playsInline = true;
    return v;
  }, [src, loop]);

  const texture = useMemo(() => {
    const t = new THREE.VideoTexture(videoEl);
    t.colorSpace = THREE.SRGBColorSpace;
    t.minFilter = THREE.LinearFilter;
    t.magFilter = THREE.LinearFilter;
    return t;
  }, [videoEl]);

  useEffect(() => {
    const handleEnded = () => onEnded?.();
    videoEl.addEventListener('ended', handleEnded);
    // Autoplay policies require user gesture — call play() from a click handler
    // in your real app. For dev, .play() will throw silently which is fine.
    videoEl.play().catch(() => {});
    return () => {
      videoEl.removeEventListener('ended', handleEnded);
      videoEl.pause();
      videoEl.src = '';
      texture.dispose();
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

### Why each piece matters

- **`useMemo` for the video element** — without it, every React re-render makes a new `<video>` and your audio stutters / playback restarts.
- **`crossOrigin = 'anonymous'`** — without it, the texture will be tainted and WebGL won't sample it. Bites everyone the first time.
- **`colorSpace = SRGBColorSpace`** + **`toneMapped={false}`** — keeps the colors looking like the source video instead of getting Three.js's filmic tone mapping applied. FMV games live and die on grading; don't let R3F's defaults wreck yours.
- **Cleanup in `useEffect` return** — videos hold serious resources. The `.pause()` + clear `src` + `texture.dispose()` pattern is the difference between a clean app and one that leaks decoders.

### Branching FMV state machines

The classic FMV mechanic — show a clip, present choices, jump to another clip based on input — maps cleanly to React state:

```tsx
type SceneId = 'intro' | 'hallway' | 'kitchen' | 'gameover';

const SCENES: Record<SceneId, { src: string; choices?: { label: string; next: SceneId }[] }> = {
  intro:    { src: '/video/intro.mp4',    choices: [{ label: 'Open the door', next: 'hallway' }] },
  hallway:  { src: '/video/hallway.mp4',  choices: [
    { label: 'Go left',  next: 'kitchen' },
    { label: 'Go back',  next: 'intro' },
  ]},
  kitchen:  { src: '/video/kitchen.mp4',  choices: [{ label: 'Eat the sandwich', next: 'gameover' }] },
  gameover: { src: '/video/gameover.mp4' },
};

function FMVGame() {
  const [scene, setScene] = useState<SceneId>('intro');
  const current = SCENES[scene];
  return (
    <>
      <Canvas>
        <FMVScreen src={current.src} loop={false} onEnded={() => {/* hold last frame */}} />
      </Canvas>
      {current.choices && (
        <div className="choices">
          {current.choices.map(c =>
            <button key={c.next} onClick={() => setScene(c.next)}>{c.label}</button>
          )}
        </div>
      )}
    </>
  );
}
```

This is the same state machine you'd write in C with an `enum` and a `switch`, just with JSX rendering hung off it.

### CRT / VHS shader feel

For the authentic FMV-on-CRT vibe, wrap your `<Canvas>` in `<EffectComposer>` (from `@react-three/postprocessing`) and chain a scanline / chromatic aberration / noise pass. The `FMV-GUIDE.md` has the full shader code for a custom VHS distortion effect.

---

## Lesson 4 — Loading Sega Genesis ROMs in the desktop app

Now we get into your wheelhouse. The whole point of bridging your SGDK knowledge to web is realizing **the emulator everyone uses on the web (Genesis Plus GX) is the same C codebase you'd find in standalone emulators** — it's just been compiled to WebAssembly via Emscripten.

### The conceptual flow

1. User clicks "Load ROM" in your Electron app.
2. Electron's main process opens a native file dialog via `dialog.showOpenDialog`.
3. Selected ROM gets read off disk into a `Uint8Array` via Node's `fs`.
4. That byte array is passed (via IPC) to the renderer.
5. The renderer feeds it into the emulator (which is a Wasm module).
6. The emulator runs the 68k + Z80 + VDP + YM2612 simulation in a tight loop, drawing frames to a `<canvas>`.

Each of these stages is a place where your C knowledge pays off — especially step 5/6, where being able to actually *read* the emulator source is a superpower.

### Picking an emulator core

| Option | Pro | Con |
| --- | --- | --- |
| **EmulatorJS** (drop-in iframe) | Zero setup, works in 5 minutes | Black box, hard to extend |
| **wasm-genplus** (Genesis Plus GX compiled to Wasm) | You can call into the emulator from JS; access to memory, VRAM, audio buffers | Have to build/host the Wasm yourself |
| **Roll your own with Emscripten** | Total control; you can patch the C source | Significant project; not recommended as a starting point |

Start with EmulatorJS or wasm-genplus depending on whether you want to *play* ROMs or *peek at their internals* (you do — that's the fun part).

### Picking a ROM via Electron's native dialog

In `electron/main.ts`:

```ts
import { ipcMain, dialog } from 'electron';
import fs from 'node:fs/promises';

ipcMain.handle('rom:open', async () => {
  const result = await dialog.showOpenDialog({
    title: 'Select a Genesis ROM you legally own',
    filters: [{ name: 'Genesis ROM', extensions: ['bin', 'md', 'gen', 'smd'] }],
    properties: ['openFile'],
  });
  if (result.canceled || result.filePaths.length === 0) return null;
  const buf = await fs.readFile(result.filePaths[0]);
  return { name: result.filePaths[0].split(/[\\/]/).pop(), bytes: buf };
});
```

In React:

```tsx
async function loadRom() {
  const rom = await window.api.openRomDialog();
  if (!rom) return;
  // rom.bytes is a Buffer (becomes Uint8Array in the renderer)
  emulator.loadRom(new Uint8Array(rom.bytes));
}
```

### Reading ROM internals (the part where your SGDK background shines)

Every Genesis ROM has a header at `0x100` with metadata you already know intimately:

```ts
function parseHeader(rom: Uint8Array) {
  const td = new TextDecoder('ascii');
  return {
    systemType:    td.decode(rom.subarray(0x100, 0x110)).trim(),   // "SEGA GENESIS    "
    copyright:     td.decode(rom.subarray(0x110, 0x120)).trim(),
    titleDomestic: td.decode(rom.subarray(0x120, 0x150)).trim(),
    titleOverseas: td.decode(rom.subarray(0x150, 0x180)).trim(),
    serial:        td.decode(rom.subarray(0x180, 0x18E)).trim(),
    checksum:      (rom[0x18E] << 8) | rom[0x18F],
    region:        td.decode(rom.subarray(0x1F0, 0x1F3)).trim(),
  };
}
```

That's a ~20-line JS port of a struct you've probably parsed a hundred times in C. Once you have this, you can build a "ROM gallery" component in R3F that shows each ROM as a 3D cartridge model with the box art and metadata on the side. The `GENESIS-EMU-GUIDE.md` walks through extracting tile data (`0x???` onwards depending on game) into Three.js textures.

---

## Lesson 5 — Sound design: from YM2612 to JUCE, VST, AU

This is the section you said you really want to dig into. We'll go in three parts:

1. What you already know from the YM2612 and how it maps to modern DSP.
2. Web Audio as a quick win (everything in JS, runs in your existing Electron app).
3. JUCE as the serious option (C++ — your home language — and what real plugins are written in).

### 5.1 — Your YM2612 knowledge translates directly

The YM2612 in the Genesis is a **6-channel 4-operator FM synth** with a specific topology of 8 algorithms. Modern soft-synths and JUCE plugins are doing exactly the same math — they just have a friendlier API and aren't latency-locked to a 7.67 MHz clock.

| YM2612 concept | Modern equivalent | Notes |
| --- | --- | --- |
| Operator (carrier/modulator) | Oscillator in JUCE/Web Audio | Sine waves multiplied/summed in the right topology |
| Algorithm (the 8 routings) | A graph of oscillators | You can draw arbitrary graphs in JUCE; the YM2612's 8 are just the routings Yamaha hardwired |
| Total Level (TL) | Output gain per operator | Same idea, log-scaled |
| ADSR / SSG-EG envelopes | `juce::ADSR` or Web Audio `GainNode` automation | The envelope math is identical |
| Detune (DT1/DT2) | Per-oscillator frequency offset | Crucial for FM character |
| LFO | LFO node modulating pitch/amplitude | Same |
| Z80 PCM channel (DAC mode) | Sample playback in any modern API | Replace 8-bit raw with a 24-bit WAV and feel reborn |

Once you internalize the mapping, recreating "Genesis sound" in JUCE is a one-week project, not a one-year one.

### 5.2 — Quick win: Web Audio FM synth inside your Electron app

You can do real FM synthesis without leaving the renderer. This is fast to prototype and lets you wire it into your R3F scene for visual feedback.

```ts
// src/audio/fmVoice.ts
export class FMVoice {
  private ctx: AudioContext;
  private carrier: OscillatorNode;
  private modulator: OscillatorNode;
  private modGain: GainNode;
  private out: GainNode;
  private env: GainNode;

  constructor(ctx: AudioContext, destination: AudioNode) {
    this.ctx = ctx;
    this.carrier = ctx.createOscillator();
    this.modulator = ctx.createOscillator();
    this.modGain = ctx.createGain();
    this.out = ctx.createGain();
    this.env = ctx.createGain();

    // Topology: modulator -> modGain -> carrier.frequency
    //           carrier -> env -> out -> destination
    this.modulator.connect(this.modGain);
    this.modGain.connect(this.carrier.frequency);
    this.carrier.connect(this.env);
    this.env.connect(this.out);
    this.out.connect(destination);

    this.env.gain.value = 0;
    this.out.gain.value = 0.3;
  }

  noteOn(freq: number, ratio = 2, modIndex = 200) {
    const now = this.ctx.currentTime;
    this.carrier.frequency.value = freq;
    this.modulator.frequency.value = freq * ratio;
    this.modGain.gain.value = modIndex;

    // ADSR envelope (attack 5ms, decay 200ms, sustain 0.6, release set in noteOff)
    this.env.gain.cancelScheduledValues(now);
    this.env.gain.setValueAtTime(0, now);
    this.env.gain.linearRampToValueAtTime(1, now + 0.005);
    this.env.gain.linearRampToValueAtTime(0.6, now + 0.205);

    this.carrier.start();
    this.modulator.start();
  }

  noteOff(releaseSec = 0.3) {
    const now = this.ctx.currentTime;
    this.env.gain.cancelScheduledValues(now);
    this.env.gain.setValueAtTime(this.env.gain.value, now);
    this.env.gain.linearRampToValueAtTime(0, now + releaseSec);
    this.carrier.stop(now + releaseSec + 0.01);
    this.modulator.stop(now + releaseSec + 0.01);
  }
}
```

That's a single FM voice — one carrier, one modulator. Add more modulators and you've got the YM2612's operator topology in JS. Hook it up to keyboard input and you can play it from a `useEffect` listener.

### 5.3 — JUCE: the serious path

JUCE is a C++ framework. You're already fluent in C and you wrote audio code on the Genesis — JUCE is going to feel natural, not foreign. It is the framework most commercial VST/AU/AAX plugins are written in (Spitfire, Output, Arturia, FabFilter all use it for at least some products).

**What JUCE gives you that Web Audio can't:**

- Sub-millisecond latency (Web Audio is at the mercy of the browser's audio thread)
- Native VST3 / AU / AAX / LV2 plugin formats — your synth loads in Ableton, Logic, Reaper, Pro Tools
- Direct hardware access (ASIO on Windows, Core Audio on Mac, JACK on Linux)
- Real-time-safe primitives (`juce::AudioBuffer`, lock-free FIFOs)

**Minimum path to first JUCE plugin:**

1. Install JUCE: clone `github.com/juce-framework/JUCE`. It's MIT for non-commercial use; you only need a license if you ship paid products without crediting JUCE.
2. Use the Projucer (or CMake — recommended for any project you'll touch in 5 years) to scaffold a "Audio Plugin" project.
3. The generated `PluginProcessor.cpp` has a `processBlock` function. *This is your only hot path.* It runs once per audio buffer (usually 64–512 samples). Do **no** allocation in here, exactly like you'd do no allocation in a Genesis HBlank interrupt.

Tiny example — JUCE's "make a sine wave when you receive a MIDI note":

```cpp
// PluginProcessor.h
class MyPluginProcessor : public juce::AudioProcessor {
public:
    void prepareToPlay(double sampleRate, int blockSize) override;
    void processBlock(juce::AudioBuffer<float>&, juce::MidiBuffer&) override;
private:
    double currentSampleRate = 44100.0;
    double currentAngle = 0.0;
    double angleDelta = 0.0;
    bool noteOn = false;
};

// PluginProcessor.cpp
void MyPluginProcessor::prepareToPlay(double sampleRate, int) {
    currentSampleRate = sampleRate;
}

void MyPluginProcessor::processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midi) {
    buffer.clear();

    for (const auto meta : midi) {
        const auto msg = meta.getMessage();
        if (msg.isNoteOn()) {
            const double freq = juce::MidiMessage::getMidiNoteInHertz(msg.getNoteNumber());
            angleDelta = (freq / currentSampleRate) * 2.0 * juce::MathConstants<double>::pi;
            noteOn = true;
        } else if (msg.isNoteOff()) {
            noteOn = false;
        }
    }

    if (!noteOn) return;

    auto* left  = buffer.getWritePointer(0);
    auto* right = buffer.getWritePointer(1);
    for (int n = 0; n < buffer.getNumSamples(); ++n) {
        const float s = (float) std::sin(currentAngle) * 0.2f;
        left[n] = right[n] = s;
        currentAngle += angleDelta;
    }
}
```

Compile it (CMake → Visual Studio / Xcode), drop the resulting `.vst3` into your DAW's plugin folder, instantiate it on a MIDI track, play notes. That's your first VST shipped. Everything else (FM synth, samplers, effects) is just more elaborate `processBlock` logic.

The `SOUND-DESIGN-JUCE-BRIDGE.md` file in this repo has the multi-chapter deep dive: setting up CMake, building a Genesis-style FM synth, MIDI handling, AU vs VST3 packaging, and the hybrid Electron+JUCE pattern where your React UI talks to a C++ audio engine over a local socket or shared memory.

---

## Lesson 6 — Hybrid projects (FMV + Genesis + synth)

This is where the repo earns its existence. The fun stuff is the *intersection* of these three skills, not any one in isolation. Some realistic project ideas:

### Project A: "Genesis B-roll"
An FMV game where the in-world TVs and arcade cabinets are actually running your legally-dumped Genesis ROMs in real time, via the wasm-genplus core overlaid on a 3D plane. Your camera moves through a 3D environment, the videos on the walls keep playing, the arcade in the corner has Sonic running on it. Self-contained desktop EXE.

### Project B: "Live-scored cutscenes"
Your FMV video plays in R3F. A JUCE-based FM synth (running as a sidecar process, communicating with Electron over a local Unix socket) is responsible for the music. Markers in your video metadata send "note on" messages that drive the synth in sync. The synth audio is captured back and fed into the renderer for an FFT-driven 3D visualizer behind the FMV. You're now writing tools that score themselves.

### Project C: "Cartridge museum"
Drop a folder full of `.bin` files into the app. It parses each header, extracts the first VRAM tile bank, and renders each ROM as a 3D cartridge model in a navigable space — click one, the emulator loads and plays it. The 3D models are generated *from the ROM data itself*. This is exactly the kind of thing your SGDK knowledge unlocks.

### Project D: "VST that shows you Three.js"
JUCE 8 ships with a `WebBrowserComponent` that can host a full Chromium view. You can build a VST3 plugin whose UI is actually your React + Three.js code. The audio thread (C++) and the UI (JS) talk over JUCE's WebView IPC. Distribute the `.vst3` as a single file — DAW users get a plugin with a 3D animated UI that responds to the audio.

---

## Building and shipping a real EXE

Electron Forge handles this. From the project root:

```bash
# Windows .exe + installer
npm run make -- --platform=win32

# macOS .dmg (must be on a Mac, or use CI)
npm run make -- --platform=darwin

# Linux AppImage / deb / rpm
npm run make -- --platform=linux
```

Output lands in `out/make/`. The `.exe` is a portable build by default; for a real Windows installer (Squirrel.Windows or NSIS), edit `forge.config.ts`'s `makers` array — Forge's docs cover this.

### Things people miss the first time

- **Code signing**: an unsigned `.exe` will trip SmartScreen on Windows and Gatekeeper on macOS. For sharing with friends, that's fine; for shipping, you need a code-signing cert (~$80–$300/yr) and to wire it into Forge's config.
- **Bundle size**: a barebones Electron app is ~150MB. Most of that is the Chromium binary, and there's nothing you can do about it. This is the cost of bringing a browser engine with you.
- **Auto-updates**: Squirrel (built into Forge) handles this on Windows and Mac. Point it at a static file host (S3, R2, even GitHub Releases) and the app self-updates.

---

## Debugging tips that map to your SGDK workflow

| What you'd do on the Genesis | What you do in Electron |
| --- | --- |
| Open Gens KMod / BlastEm's debugger | `Ctrl+Shift+I` opens Chromium DevTools — full source-mapped debugger, breakpoints, profiler |
| Watch VRAM in real time | DevTools' "Memory" tab — heap snapshots, allocation timelines |
| Insert `KDebug_Alert("...")` | `console.log()` in the renderer, `console.log()` in main shows up in the terminal you launched from |
| Drop into single-step on an interrupt | DevTools' "Pause on exceptions" + `debugger;` statement in JS, or VS Code's Node debugger attached to main process |
| Check sprite limits / VDP overdraw | DevTools' "Performance" tab — flame graph of every frame, R3F-specific overlays available via the `r3f-perf` package |
| Read register values | DevTools' "Console" — `THREE.MeshStandardMaterial.prototype.uniformsNeedUpdate`, etc. inspectable live |

For the C-compiled-to-Wasm side (Genesis emulator), you can build the Wasm with `-g` and get source-mapped C debugging *in DevTools*. That's not a typo. Modern Chrome can step through C source code when running the Wasm. Once you've done that once it permanently changes how you feel about web tech.

---

## Where to go from here

A reasonable order of attack for Barry specifically:

1. **Get the starter running.** Pink cube on starfield. Five minutes.
2. **Drop your existing FMV React code into `src/`.** Use the `FMVScreen` component from Lesson 3 to render your video as a texture on a 3D plane rather than a raw `<video>` tag. Now your FMV is in a real 3D scene.
3. **Wrap it as an EXE.** `npm run make`. Send the `.exe` to a friend. This is the moment it stops feeling like "web stuff" and starts feeling like an actual app.
4. **Add the ROM loader from Lesson 4.** Even without a full emulator, parse the headers of your Genesis ROMs and render them as a "rom gallery" 3D component. Pure JS, no Wasm yet.
5. **Hook up a Web Audio FM synth (Lesson 5.2)** to play notes when you click on a 3D object in the scene. Now you've got interactive synthesis.
6. **Pick up JUCE (Lesson 5.3)** when the Web Audio version starts feeling limiting — usually around the time you want polyphony, presets, and to load your synth in a DAW.

You're already further along than you think. The C/SGDK background is a real edge, not a "but I'm new to web" handicap. The web stuff is the easy part — what you actually know is the hard part.

---

## Companion guides

- **`FMV-GUIDE.md`** — Deeper on video-as-texture, branching state machines, CRT shaders, syncing 3D objects to video timelines, native fullscreen handling.
- **`GENESIS-EMU-GUIDE.md`** — Deeper on emulator integration, tile/sprite extraction from ROM data, building a Genesis-aware 3D debugger.
- **`SOUND-DESIGN-JUCE-BRIDGE.md`** — Multi-chapter walkthrough from YM2612 → JUCE → VST3/AU → Electron-JUCE hybrid plugins with R3F UIs.

---

## License

MIT. Do whatever you want with it. Especially: build something weird with it.
