# Genesis Emulator Integration Guide

Companion to the main `README.md`. This file goes deep on running Sega Genesis ROMs inside the Electron app and using your SGDK knowledge to do things no off-the-shelf emulator does.

---

## Legal note (read this once)

Dumping ROMs from cartridges you legally own, for personal use, is fine. Distributing those ROMs — or downloading dumps you didn't make yourself — isn't. Everything in this guide assumes you're working with your own cartridge dumps. Keep `public/roms/` in `.gitignore`. Don't ship ROMs in your built EXE.

If you want to publicly share a built game, use **homebrew ROMs** — there's a huge SGDK homebrew community on itch.io and the SGDK forums, and most of those devs are happy to have their work showcased if you credit them.

---

## What this guide actually covers

1. [Why this works at all — Wasm + Emscripten](#why-this-works-at-all--wasm--emscripten)
2. [Three ways to integrate an emulator](#three-ways-to-integrate-an-emulator)
3. [Path A: EmulatorJS (zero-config)](#path-a-emulatorjs-zero-config)
4. [Path B: wasm-genplus (the one I'd actually pick)](#path-b-wasm-genplus-the-one-id-actually-pick)
5. [Path C: Building your own Wasm port of Genesis Plus GX](#path-c-building-your-own-wasm-port-of-genesis-plus-gx)
6. [Parsing the ROM header (your first JS port of Genesis structs)](#parsing-the-rom-header-your-first-js-port-of-genesis-structs)
7. [Extracting tile and sprite data into Three.js textures](#extracting-tile-and-sprite-data-into-threejs-textures)
8. [3D ROM gallery (concrete project)](#3d-rom-gallery-concrete-project)
9. [Bridging emulator audio into R3F visualizers](#bridging-emulator-audio-into-r3f-visualizers)
10. [Save states, screenshots, native file dialogs](#save-states-screenshots-native-file-dialogs)
11. [SGDK homebrew workflow inside this app](#sgdk-homebrew-workflow-inside-this-app)

---

## Why this works at all — Wasm + Emscripten

The whole "run Genesis in the browser" thing relies on **Emscripten**, which is a toolchain that compiles C/C++ into WebAssembly. The emulator C code — the same code that runs `genesis_plus_gx` on a PC — is recompiled, the SDL graphics calls are replaced with WebGL/Canvas calls, the SDL audio calls become Web Audio calls, and the resulting `.wasm` binary runs in any browser (or Electron).

For you, this is a gift. You're already comfortable reading the C source. So instead of "emulator that you treat as a black box," you have "emulator whose `cpu_68k.c` you can crack open and step through." The Wasm doesn't hide the C source if you build it with `-g`.

Performance is genuinely good — close to native on a modern machine. Genesis emulation is light enough that even the slow paths are fine; you've got 7.67 MHz of 68000 to simulate on a CPU that's a thousand times faster.

---

## Three ways to integrate an emulator

| Approach | Time to first ROM running | Flexibility | Best for |
| --- | --- | --- | --- |
| **EmulatorJS** | 15 minutes | Low | Quick proof of concept; "emulator as a feature" |
| **wasm-genplus** | A few hours | High | The sweet spot for what you want to do |
| **Roll your own** | A weekend (if you know Emscripten) or weeks (if you don't) | Total | Custom debugger / visualizer projects |

For Barry specifically, I'd go straight to **wasm-genplus** — it gives you JS bindings into the emulator's memory and state, which is what makes the cool projects possible.

---

## Path A: EmulatorJS (zero-config)

EmulatorJS is a drop-in web emulator front-end. It supports a bunch of systems, Genesis included. Installation is "include some scripts, point it at a ROM URL":

```tsx
// src/components/EmulatorJSPlayer.tsx
import { useEffect, useRef } from 'react';

interface Props {
  romUrl: string;
}

declare global {
  interface Window {
    EJS_player: string;
    EJS_core: string;
    EJS_gameUrl: string;
    EJS_pathtodata: string;
  }
}

export function EmulatorJSPlayer({ romUrl }: Props) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current) return;
    containerRef.current.id = 'game';

    window.EJS_player = '#game';
    window.EJS_core = 'segaMD';            // Mega Drive / Genesis
    window.EJS_gameUrl = romUrl;
    window.EJS_pathtodata = 'https://cdn.emulatorjs.org/stable/data/';

    const script = document.createElement('script');
    script.src = 'https://cdn.emulatorjs.org/stable/data/loader.js';
    document.body.appendChild(script);

    return () => { script.remove(); };
  }, [romUrl]);

  return <div ref={containerRef} style={{ width: '100%', height: '100%' }} />;
}
```

That's it. Genesis games run. The downside is that you have basically no programmatic access — it's an `<iframe>`-style abstraction. No way to read VRAM, hook audio, or get at the running state. Fine for "I just want to play Sonic in my desktop app" — not enough for "I want to extract sprite tables and put them in a 3D scene."

For development, you'll want to download EmulatorJS and ship it locally rather than relying on the CDN — Electron apps that depend on a CDN feel broken when offline.

---

## Path B: wasm-genplus (the one I'd actually pick)

`wasm-genplus` (https://github.com/h1romas4/wasm-genplus) is Genesis Plus GX compiled to Wasm with JS bindings. You get:

- `genplus.loadRom(bytes)` — load a ROM
- `genplus.frame()` — advance one frame
- Access to the framebuffer (`Uint8Array` of RGB pixels)
- Access to the audio buffer (`Float32Array` of samples)
- Read/write access to 68k RAM, VRAM, CRAM, VSRAM (this is the part that matters)

The integration boils down to: each animation frame, call `genplus.frame()`, blit the framebuffer to a canvas, push the audio buffer into Web Audio. With R3F, you can use that canvas as a `CanvasTexture` and put it on a 3D plane — same way you did with video.

### Sketch of the integration

```tsx
// src/components/GenesisEmulator.tsx
import { useEffect, useRef, useState } from 'react';
import * as THREE from 'three';
// import init, { GenesisEmu } from 'wasm-genplus'; // shape depends on the build

interface Props {
  romBytes: Uint8Array;
}

const FRAME_WIDTH = 320;
const FRAME_HEIGHT = 240;

export function GenesisEmulator({ romBytes }: Props) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [emu, setEmu] = useState<any>(null);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      // const wasm = await init();
      // const instance = new GenesisEmu();
      // instance.loadRom(romBytes);
      // if (!cancelled) setEmu(instance);
    })();
    return () => { cancelled = true; };
  }, [romBytes]);

  useEffect(() => {
    if (!emu || !canvasRef.current) return;
    const ctx = canvasRef.current.getContext('2d')!;
    const imageData = ctx.createImageData(FRAME_WIDTH, FRAME_HEIGHT);

    let rafId = 0;
    const loop = () => {
      emu.frame();
      const fb = emu.getFramebuffer();  // Uint8Array length 320*240*4 RGBA
      imageData.data.set(fb);
      ctx.putImageData(imageData, 0, 0);

      // Audio: emu.getAudioSamples() returns interleaved L/R float32
      // Push into a Web Audio AudioBufferSourceNode or ScriptProcessorNode/AudioWorklet
      rafId = requestAnimationFrame(loop);
    };
    rafId = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(rafId);
  }, [emu]);

  return (
    <canvas
      ref={canvasRef}
      width={FRAME_WIDTH}
      height={FRAME_HEIGHT}
      style={{ imageRendering: 'pixelated', width: 640, height: 480 }}
    />
  );
}
```

To use this as a 3D texture in R3F:

```tsx
function EmulatorScreen({ romBytes }: { romBytes: Uint8Array }) {
  // [hidden] canvas drives the emulator; texture mirrors it
  const canvasRef = useRef<HTMLCanvasElement>(document.createElement('canvas'));
  canvasRef.current.width = 320;
  canvasRef.current.height = 240;

  const texture = useMemo(() => {
    const t = new THREE.CanvasTexture(canvasRef.current);
    t.magFilter = THREE.NearestFilter;
    t.minFilter = THREE.NearestFilter;
    return t;
  }, []);

  // ... run the emu loop in a useEffect, then in useFrame:
  useFrame(() => { texture.needsUpdate = true; });

  return (
    <mesh>
      <planeGeometry args={[4, 3]} />
      <meshBasicMaterial map={texture} toneMapped={false} />
    </mesh>
  );
}
```

Now Sonic is running on a 3D plane in a scene you can rotate, light, post-process — same composability as the FMV stuff in `FMV-GUIDE.md`.

---

## Path C: Building your own Wasm port of Genesis Plus GX

You probably don't need to. But if you do — say, because you want to patch the VDP to emit a "sprite drawn at (x, y) using tile T" event stream that a 3D visualizer subscribes to — here's the rough shape:

1. Clone `genesis_plus_gx` source.
2. Install Emscripten.
3. Write a small `main.c` shim that:
   - Initializes the emulator
   - Exports functions via `EMSCRIPTEN_KEEPALIVE` for JS to call (`loadRom`, `frame`, `getRam`, etc.)
   - Replaces SDL graphics output with a memory-mapped framebuffer the JS side reads
4. Build with `emcc -O3 -s WASM=1 -s EXPORTED_FUNCTIONS="['_loadRom','_frame','_getRam']" -s ALLOW_MEMORY_GROWTH=1`
5. In Electron, load the resulting `.wasm` + `.js` glue, call your exports.

This is a real project — at least a weekend of work, more if you've never used Emscripten. The payoff is total control: you can instrument any part of the emulator. Want to log every YM2612 register write to a file? Add three lines of C in `sound/ym2612.c` and rebuild.

Wait until you have a specific reason to do this. Until then, `wasm-genplus` covers 95% of what you'd want.

---

## Parsing the ROM header (your first JS port of Genesis structs)

Every Genesis ROM has a fixed-layout header at offset `0x100`. You've parsed this struct in C a hundred times. Here it is in TypeScript:

```ts
// src/genesis/header.ts
export interface GenesisHeader {
  systemType: string;       // "SEGA GENESIS", "SEGA MEGA DRIVE", etc.
  copyright: string;        // e.g. "(C)SEGA    1991.NOV"
  titleDomestic: string;    // Japanese title (Shift-JIS, but ASCII works for most)
  titleOverseas: string;    // US/EU title
  serial: string;           // "GM XXXXXX-XX"
  romChecksum: number;      // expected checksum over $200..end
  ioSupport: string;        // "J", "6", "JK6", etc — controller types
  romStart: number;
  romEnd: number;
  ramStart: number;
  ramEnd: number;
  sramInfo: Uint8Array;     // 12 bytes describing battery-backed SRAM
  modemSupport: string;
  reserved: string;
  region: string;           // "J", "U", "E", "JUE", etc
}

export function parseGenesisHeader(rom: Uint8Array): GenesisHeader {
  if (rom.length < 0x200) throw new Error('ROM too small to have a valid header');

  const td = new TextDecoder('ascii');
  const slice = (start: number, end: number) =>
    td.decode(rom.subarray(start, end)).replace(/\0/g, '').trim();

  return {
    systemType:    slice(0x100, 0x110),
    copyright:     slice(0x110, 0x120),
    titleDomestic: slice(0x120, 0x150),
    titleOverseas: slice(0x150, 0x180),
    serial:        slice(0x180, 0x18E),
    romChecksum:   (rom[0x18E] << 8) | rom[0x18F],
    ioSupport:     slice(0x190, 0x1A0),
    romStart:     (rom[0x1A0] << 24) | (rom[0x1A1] << 16) | (rom[0x1A2] << 8) | rom[0x1A3],
    romEnd:       (rom[0x1A4] << 24) | (rom[0x1A5] << 16) | (rom[0x1A6] << 8) | rom[0x1A7],
    ramStart:     (rom[0x1A8] << 24) | (rom[0x1A9] << 16) | (rom[0x1AA] << 8) | rom[0x1AB],
    ramEnd:       (rom[0x1AC] << 24) | (rom[0x1AD] << 16) | (rom[0x1AE] << 8) | rom[0x1AF],
    sramInfo:      rom.subarray(0x1B0, 0x1BC),
    modemSupport:  slice(0x1BC, 0x1C8),
    reserved:      slice(0x1C8, 0x1F0),
    region:        slice(0x1F0, 0x1F3),
  };
}

export function verifyChecksum(rom: Uint8Array): { stored: number; computed: number; valid: boolean } {
  const stored = (rom[0x18E] << 8) | rom[0x18F];
  let sum = 0;
  for (let i = 0x200; i < rom.length - 1; i += 2) {
    sum = (sum + ((rom[i] << 8) | rom[i + 1])) & 0xFFFF;
  }
  return { stored, computed: sum, valid: sum === stored };
}
```

Twenty-line port of code you've written before. Now you can build the gallery component.

### Handling SMD-format ROMs (the interleaved kind)

If a ROM file is in `.smd` format (Super Magic Drive interleaved), bytes are arranged in 16KB blocks with odd bytes in the first half and even bytes in the second. Deinterleave before parsing the header:

```ts
export function deinterleaveSMD(smd: Uint8Array): Uint8Array {
  // First 512 bytes are SMD header — skip
  if (smd.length < 512) throw new Error('Not a valid SMD ROM');
  const body = smd.subarray(512);
  const out = new Uint8Array(body.length);
  const blockSize = 16384;
  for (let block = 0; block < body.length; block += blockSize) {
    const halfBlock = blockSize / 2;
    for (let i = 0; i < halfBlock && block + halfBlock + i < body.length; i++) {
      out[block + i * 2 + 1] = body[block + i];
      out[block + i * 2]     = body[block + halfBlock + i];
    }
  }
  return out;
}

export function detectAndNormalize(rom: Uint8Array): Uint8Array {
  // ".bin" / ".gen" are raw. ".smd" is interleaved.
  // Heuristic: check for "SEGA" at 0x100 in the raw view
  const td = new TextDecoder('ascii');
  if (td.decode(rom.subarray(0x100, 0x104)) === 'SEGA') return rom;
  return deinterleaveSMD(rom);
}
```

---

## Extracting tile and sprite data into Three.js textures

This is the move that justifies all the setup. Genesis tiles are 8×8 pixels, 4 bits per pixel, packed as 32 bytes per tile. ROM tile data layout varies wildly per game (it's in whatever location the dev put it), but for many games it sits in a contiguous block.

### Tile decoder

```ts
// src/genesis/tiles.ts
/**
 * Decode a Genesis 8x8 4bpp tile (32 bytes) into a 64-element palette index array.
 */
export function decodeTile(bytes: Uint8Array, offset: number): Uint8Array {
  const tile = new Uint8Array(64);
  for (let row = 0; row < 8; row++) {
    for (let col = 0; col < 4; col++) {
      const byte = bytes[offset + row * 4 + col];
      tile[row * 8 + col * 2]     = (byte >> 4) & 0x0F;
      tile[row * 8 + col * 2 + 1] = byte & 0x0F;
    }
  }
  return tile;
}

/**
 * Decode a Genesis CRAM color (9-bit BGR, but stored in a 16-bit word).
 * Format: -- -- -- B B B -- G G G -- R R R --
 */
export function decodeColor(word: number): [number, number, number] {
  const r = ((word >> 1)  & 0x07) * 36;  // expand 3-bit to 8-bit
  const g = ((word >> 5)  & 0x07) * 36;
  const b = ((word >> 9)  & 0x07) * 36;
  return [r, g, b];
}

/**
 * Render a single 8x8 tile into an RGBA buffer using a 16-entry palette.
 */
export function tileToRGBA(tile: Uint8Array, palette: Array<[number, number, number]>): Uint8Array {
  const rgba = new Uint8Array(64 * 4);
  for (let i = 0; i < 64; i++) {
    const idx = tile[i];
    const [r, g, b] = palette[idx] ?? [0, 0, 0];
    rgba[i * 4]     = r;
    rgba[i * 4 + 1] = g;
    rgba[i * 4 + 2] = b;
    rgba[i * 4 + 3] = idx === 0 ? 0 : 255; // index 0 is usually transparent
  }
  return rgba;
}
```

### From RGBA buffer to Three.js texture

```ts
import * as THREE from 'three';

export function rgbaToTexture(rgba: Uint8Array, w = 8, h = 8): THREE.DataTexture {
  const t = new THREE.DataTexture(rgba, w, h, THREE.RGBAFormat);
  t.magFilter = THREE.NearestFilter;
  t.minFilter = THREE.NearestFilter;
  t.needsUpdate = true;
  return t;
}
```

### Putting it together — extract a tile bank, render as a sprite sheet

```ts
export function extractTileBank(
  rom: Uint8Array,
  tileBankOffset: number,
  tileCount: number,
  palette: Array<[number, number, number]>,
): THREE.DataTexture {
  // Build an N-tile-wide sprite sheet
  const COLS = 16;
  const ROWS = Math.ceil(tileCount / COLS);
  const sheetW = COLS * 8;
  const sheetH = ROWS * 8;
  const sheet = new Uint8Array(sheetW * sheetH * 4);

  for (let t = 0; t < tileCount; t++) {
    const tile = decodeTile(rom, tileBankOffset + t * 32);
    const tx = (t % COLS) * 8;
    const ty = Math.floor(t / COLS) * 8;
    for (let y = 0; y < 8; y++) {
      for (let x = 0; x < 8; x++) {
        const idx = tile[y * 8 + x];
        const [r, g, b] = palette[idx] ?? [0, 0, 0];
        const off = ((ty + y) * sheetW + (tx + x)) * 4;
        sheet[off]     = r;
        sheet[off + 1] = g;
        sheet[off + 2] = b;
        sheet[off + 3] = idx === 0 ? 0 : 255;
      }
    }
  }

  return rgbaToTexture(sheet, sheetW, sheetH);
}
```

For a known game (say, Sonic 1), the tile data starts around `0x4D900` and runs for thousands of tiles. Plug those numbers in, get a sprite sheet, slap it on a 3D plane. You're now staring at every graphic in a Genesis game inside a 3D scene.

Finding the *right* offset for an unknown ROM is its own adventure — usually involves checking the disassembly (Sonic Retro and other communities have them well-documented) or scanning for plausible tile-shaped data.

---

## 3D ROM gallery (concrete project)

Concept: drop a folder of ROMs into the app, get a 3D room with floating cartridges, each cartridge labeled with its title and rendered with a thumbnail extracted from the ROM itself.

### Component skeleton

```tsx
// src/components/RomGallery.tsx
import { useEffect, useState } from 'react';
import { Text } from '@react-three/drei';
import * as THREE from 'three';
import { parseGenesisHeader, detectAndNormalize } from '../genesis/header';
import { extractTileBank } from '../genesis/tiles';

interface RomEntry {
  filename: string;
  title: string;
  region: string;
  thumbnail: THREE.DataTexture;
  bytes: Uint8Array;
}

export function RomGallery({ onPick }: { onPick: (rom: RomEntry) => void }) {
  const [roms, setRoms] = useState<RomEntry[]>([]);

  useEffect(() => {
    (async () => {
      const files = await window.api.listRoms();  // ipcRenderer.invoke('rom:list')
      const entries: RomEntry[] = [];

      // Default Genesis-ish palette; in production extract from CRAM at runtime
      const defaultPalette: Array<[number, number, number]> =
        Array.from({ length: 16 }, (_, i) => [i * 16, i * 16, i * 16]);

      for (const file of files) {
        const raw = await window.api.readRom(file.path);
        const bytes = detectAndNormalize(new Uint8Array(raw));
        const header = parseGenesisHeader(bytes);
        // Pick a tile bank that probably has logo data — usually somewhere in first 256KB
        const thumb = extractTileBank(bytes, 0x4000, 64, defaultPalette);
        entries.push({
          filename: file.name,
          title: header.titleOverseas || header.titleDomestic,
          region: header.region,
          thumbnail: thumb,
          bytes,
        });
      }
      setRoms(entries);
    })();
  }, []);

  return (
    <group>
      {roms.map((rom, i) => {
        const angle = (i / roms.length) * Math.PI * 2;
        const radius = 5;
        return (
          <group
            key={rom.filename}
            position={[Math.cos(angle) * radius, 0, Math.sin(angle) * radius]}
            rotation={[0, -angle + Math.PI / 2, 0]}
            onClick={() => onPick(rom)}
          >
            {/* Cartridge body */}
            <mesh>
              <boxGeometry args={[1.5, 2, 0.3]} />
              <meshStandardMaterial color="#222" />
            </mesh>
            {/* Label */}
            <mesh position={[0, 0.3, 0.16]}>
              <planeGeometry args={[1.3, 0.8]} />
              <meshBasicMaterial map={rom.thumbnail} toneMapped={false} />
            </mesh>
            {/* Title */}
            <Text
              position={[0, -0.7, 0.16]}
              fontSize={0.15}
              maxWidth={1.3}
              color="white"
              anchorX="center"
              anchorY="middle"
            >
              {rom.title}
            </Text>
          </group>
        );
      })}
    </group>
  );
}
```

Wire that up inside a `<Canvas>` with an `<OrbitControls>` and you've got a navigable 3D room of your own cartridges. Click one → `onPick` fires → swap the main scene to the `GenesisEmulator` component running that ROM.

This kind of project would take a real team a month in a traditional game engine. You can knock out a working prototype in an afternoon because everything's just React components.

---

## Bridging emulator audio into R3F visualizers

The emulator gives you a `Float32Array` of audio samples per frame. Push it through an `AnalyserNode` and you've got real-time FFT data you can drive 3D objects with:

```tsx
import { useRef, useEffect, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

function AudioReactiveBox({ analyser }: { analyser: AnalyserNode }) {
  const meshRef = useRef<THREE.Mesh>(null!);
  const data = useMemo(() => new Uint8Array(analyser.frequencyBinCount), [analyser]);

  useFrame(() => {
    analyser.getByteFrequencyData(data);
    // Average low frequencies (bass)
    let bass = 0;
    for (let i = 0; i < 16; i++) bass += data[i];
    bass /= 16;
    const scale = 1 + (bass / 255) * 0.8;
    meshRef.current.scale.setScalar(scale);
  });

  return (
    <mesh ref={meshRef}>
      <sphereGeometry args={[1, 32, 32]} />
      <meshStandardMaterial color="#f0f" emissive="#80f" emissiveIntensity={0.5} />
    </mesh>
  );
}
```

You feed the emulator's audio output into the `AnalyserNode` upstream — same pattern as wiring `MediaElementSourceNode` from a video. Now: load Sonic, the bass thumps in the music, a glowing 3D sphere pulses next to the screen. That's *your homebrew demo aesthetic* but on modern hardware with shaders.

For visualizing the YM2612 channels individually (not just the mixed output), you need an emulator with per-channel audio taps — wasm-genplus exposes some of this; the rolled-your-own approach gives total access.

---

## Save states, screenshots, native file dialogs

These are the "feels like a real app" features. All live in the main process.

### Screenshot

```ts
// electron/main.ts
ipcMain.handle('screenshot:save', async (_event, dataUrl: string) => {
  const result = await dialog.showSaveDialog({
    defaultPath: `screenshot-${Date.now()}.png`,
    filters: [{ name: 'PNG', extensions: ['png'] }],
  });
  if (result.canceled || !result.filePath) return false;
  const base64 = dataUrl.replace(/^data:image\/png;base64,/, '');
  await fs.writeFile(result.filePath, Buffer.from(base64, 'base64'));
  return true;
});
```

In the renderer:

```tsx
function captureScreenshot() {
  const canvas = document.querySelector('canvas')!;  // R3F's canvas
  const dataUrl = canvas.toDataURL('image/png');
  window.api.saveScreenshot(dataUrl);
}
```

### Save states

```ts
ipcMain.handle('savestate:write', async (_event, romName: string, slot: number, state: Uint8Array) => {
  const dir = path.join(app.getPath('userData'), 'savestates', romName);
  await fs.mkdir(dir, { recursive: true });
  await fs.writeFile(path.join(dir, `slot-${slot}.bin`), state);
});

ipcMain.handle('savestate:read', async (_event, romName: string, slot: number) => {
  const file = path.join(app.getPath('userData'), 'savestates', romName, `slot-${slot}.bin`);
  try {
    return await fs.readFile(file);
  } catch {
    return null;
  }
});
```

The emulator's `getState` / `setState` calls give you the bytes; wrapping them in IPC is trivial. `app.getPath('userData')` is the OS-correct place to store this (`%APPDATA%` on Windows, `~/Library/Application Support/` on Mac).

---

## SGDK homebrew workflow inside this app

The full circle moment: build a Genesis ROM with SGDK on your machine, then load it directly into this Electron app to test. Two ways:

### Option 1: Watch the build output folder
Set up an `fs.watch` in the main process on your SGDK project's `out/` folder. Whenever the `.bin` updates (i.e. you ran `make`), auto-reload the emulator. You're now using an Electron app you wrote as your Genesis dev environment.

```ts
import { watch } from 'node:fs';

function watchSGDKOutput(projectDir: string, onChange: (romPath: string) => void) {
  const out = path.join(projectDir, 'out');
  watch(out, (event, filename) => {
    if (filename?.endsWith('.bin')) onChange(path.join(out, filename));
  });
}
```

### Option 2: Run SGDK's `make` from inside Electron

```ts
import { spawn } from 'node:child_process';

ipcMain.handle('sgdk:build', async (_event, projectDir: string) => {
  return new Promise((resolve, reject) => {
    const proc = spawn('make', [], { cwd: projectDir, shell: true });
    let stdout = '', stderr = '';
    proc.stdout.on('data', (d) => stdout += d.toString());
    proc.stderr.on('data', (d) => stderr += d.toString());
    proc.on('close', (code) => {
      if (code === 0) resolve({ ok: true, stdout });
      else reject({ ok: false, code, stderr });
    });
  });
});
```

A "Build & Run" button in your React UI that triggers SGDK's makefile and reloads the emulator. You've just built your own integrated dev environment for Genesis homebrew. That'd be a real flex on the SGDK Discord.

---

## Recap

You probably noticed the pattern by now: at each stage, the part that involves *understanding the Genesis* is the part you already know, and the part that involves *making it show up on screen in a desktop app* is the part this repo handles. That's the whole pitch.

Start with Path B (wasm-genplus), get one ROM running on a 3D plane, then layer on the cool stuff: header parsing → tile extraction → ROM gallery → SGDK build integration. Each step is small. The aggregate is something nobody else has built.
