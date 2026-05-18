# Lesson 04 — Genesis ROM Integration

> By the end of this lesson, you'll be loading a legally-owned Genesis ROM from disk through Electron's native file dialog, parsing its header in TypeScript (a direct port of structs you know from SGDK), decoding tile data into Three.js textures, and rendering ROMs as 3D cartridges in a clickable gallery.

**Estimated time**: 4–6 hours.
**Prerequisites**: Lessons 01–02. A legally-owned Genesis ROM (`.bin`, `.md`, `.gen`, or `.smd`). Your SGDK / 68k familiarity is the biggest unlock here.

---

## Legal reminder

Dumping ROMs from cartridges you own, for personal use: fine. Distributing them: not fine. Everything below assumes your own dumps. Keep `public/roms/` in `.gitignore`. The detailed reasoning is in [`../../GENESIS-EMU-GUIDE.md`](../../GENESIS-EMU-GUIDE.md).

---

## Learning objectives

1. Pick the right emulator integration path (EmulatorJS / wasm-genplus / roll-your-own) for what you're trying to build.
2. Read a Genesis ROM header in TypeScript using the same offsets you'd use in C.
3. Detect and de-interleave `.smd`-format ROMs.
4. Decode 4bpp tile data into RGBA buffers and load them as Three.js textures.
5. Build a 3D ROM gallery that previews each ROM's contents and launches an emulator on click.

---

## Chapter 1 — Why this works (Wasm + Emscripten)

Modern Genesis emulators in the browser aren't reimplementations — they're the **same C code** from `genesis_plus_gx` or similar, compiled to WebAssembly via Emscripten. SDL output is swapped for WebGL/Canvas, SDL audio for Web Audio, file I/O for JS-provided buffers. The 68k core, the VDP renderer, the YM2612 simulation — all your familiar territory.

For you, this is great news: **the emulator source is readable**. With Wasm built with `-g`, Chromium can step through C source in DevTools. Setting a breakpoint inside `cpu_68k.c`'s instruction dispatch loop is a perfectly normal Wednesday-night activity once you have this stack working.

---

## Chapter 2 — Three integration paths

| Path | Time to first ROM | Programmable access | Right for... |
| --- | --- | --- | --- |
| **EmulatorJS** | 15 min | Almost none — it's an iframe-ish wrapper | "I want to play ROMs in my app" |
| **wasm-genplus** | A few hours | Read/write VRAM, CRAM, RAM; hook audio | What you actually want |
| **Roll your own Wasm port** | A weekend+ | Total | Custom debuggers, instrumentation, weird ideas |

For this lesson we'll focus on the underlying mechanics: header parsing and tile extraction. Both work without any emulator — they're pure JS operating on the ROM bytes. Once you have these, hooking up `wasm-genplus` to actually run the ROM is incremental.

---

## Chapter 3 — Loading a ROM through Electron

You'll need two halves: a main-process IPC handler that opens a file dialog and reads bytes, and a preload bridge that exposes it to React.

**`electron/main.ts`** (extend the file from Lesson 01):

```ts
import { ipcMain, dialog, app } from 'electron';
import fs from 'node:fs/promises';
import path from 'node:path';

ipcMain.handle('rom:open', async () => {
  const result = await dialog.showOpenDialog({
    title: 'Select a Genesis ROM you legally own',
    filters: [{ name: 'Genesis ROM', extensions: ['bin', 'md', 'gen', 'smd'] }],
    properties: ['openFile'],
  });
  if (result.canceled || result.filePaths.length === 0) return null;
  const filepath = result.filePaths[0];
  const buf = await fs.readFile(filepath);
  return {
    name: path.basename(filepath),
    bytes: buf,    // Sent over IPC; arrives as Uint8Array in renderer
  };
});
```

**`electron/preload.ts`**:

```ts
contextBridge.exposeInMainWorld('api', {
  // ... existing entries
  openRom: () => ipcRenderer.invoke('rom:open'),
});
```

**React code**:

```ts
async function pickRom() {
  const rom = await window.api.openRom();
  if (!rom) return;
  const bytes = new Uint8Array(rom.bytes);
  console.log(`Loaded ${rom.name}, ${bytes.length} bytes`);
}
```

That's the whole pipeline. Click button → native dialog → file goes from disk through main into the renderer as a `Uint8Array` you can do anything with.

---

## Chapter 4 — Parsing the header (porting SGDK structs to TS)

The Genesis ROM header lives at offset `0x100` and is 256 bytes long. If you've ever poked at a ROM in SGDK, this is familiar territory.

```ts
// src/genesis/header.ts
export interface GenesisHeader {
  systemType: string;       // "SEGA GENESIS    " etc.
  copyright: string;
  titleDomestic: string;
  titleOverseas: string;
  serial: string;
  romChecksum: number;
  region: string;
}

export function parseGenesisHeader(rom: Uint8Array): GenesisHeader {
  if (rom.length < 0x200) throw new Error('ROM too small');
  const td = new TextDecoder('ascii');
  const slice = (a: number, b: number) =>
    td.decode(rom.subarray(a, b)).replace(/\0/g, '').trim();

  return {
    systemType:    slice(0x100, 0x110),
    copyright:     slice(0x110, 0x120),
    titleDomestic: slice(0x120, 0x150),
    titleOverseas: slice(0x150, 0x180),
    serial:        slice(0x180, 0x18E),
    romChecksum:   (rom[0x18E] << 8) | rom[0x18F],
    region:        slice(0x1F0, 0x1F3),
  };
}

export function verifyChecksum(rom: Uint8Array) {
  const stored = (rom[0x18E] << 8) | rom[0x18F];
  let sum = 0;
  for (let i = 0x200; i < rom.length - 1; i += 2) {
    sum = (sum + ((rom[i] << 8) | rom[i + 1])) & 0xFFFF;
  }
  return { stored, computed: sum, valid: sum === stored };
}
```

This is a near-direct port of code from any SGDK header utility. The big-endian byte assembly `(rom[a] << 8) | rom[a+1]` is the same in C; the `subarray` + `TextDecoder` is what JavaScript gives you instead of `memcpy` + `strncpy`.

### Handling `.smd` (interleaved)

Some ROM dumps are in Super Magic Drive format — 16KB blocks where odd bytes come first, then even bytes. You need to de-interleave before parsing the header.

The full version is in [`../../GENESIS-EMU-GUIDE.md#handling-smd-format-roms-the-interleaved-kind`](../../GENESIS-EMU-GUIDE.md). The detection heuristic: look for `"SEGA"` at `0x100` in the raw view. If not there, the file is probably `.smd` and needs de-interleaving (the first 512 bytes are an SMD-specific header to skip).

---

## Chapter 5 — Decoding tiles

Genesis tiles are 8×8 pixels, 4 bits per pixel, 32 bytes per tile, with two pixels packed per byte.

```ts
// src/genesis/tiles.ts
export function decodeTile(bytes: Uint8Array, offset: number): Uint8Array {
  const tile = new Uint8Array(64);     // 8x8 palette indices
  for (let row = 0; row < 8; row++) {
    for (let col = 0; col < 4; col++) {
      const b = bytes[offset + row * 4 + col];
      tile[row * 8 + col * 2]     = (b >> 4) & 0x0F;
      tile[row * 8 + col * 2 + 1] = b & 0x0F;
    }
  }
  return tile;
}
```

That's the bit-packing logic you've written in C for ROM hacking tools. Same exact loop.

### Genesis colors — 9-bit BGR in a 16-bit word

CRAM stores each color as `-- -- -- B B B -- G G G -- R R R --`. To get an RGB888 triple:

```ts
export function decodeColor(word: number): [number, number, number] {
  const r = ((word >> 1)  & 0x07) * 36;
  const g = ((word >> 5)  & 0x07) * 36;
  const b = ((word >> 9)  & 0x07) * 36;
  return [r, g, b];
}
```

The `* 36` expands a 3-bit value to fit in an 8-bit channel (since `7 * 36 = 252`, close to 255).

### From tile bytes to Three.js texture

```ts
import * as THREE from 'three';

export function tileToTexture(
  bytes: Uint8Array,
  offset: number,
  palette: Array<[number, number, number]>,
): THREE.DataTexture {
  const tile = decodeTile(bytes, offset);
  const rgba = new Uint8Array(64 * 4);
  for (let i = 0; i < 64; i++) {
    const idx = tile[i];
    const [r, g, b] = palette[idx] ?? [0, 0, 0];
    rgba[i * 4]     = r;
    rgba[i * 4 + 1] = g;
    rgba[i * 4 + 2] = b;
    rgba[i * 4 + 3] = idx === 0 ? 0 : 255;  // Palette index 0 is transparent
  }
  const t = new THREE.DataTexture(rgba, 8, 8, THREE.RGBAFormat);
  t.magFilter = THREE.NearestFilter;
  t.minFilter = THREE.NearestFilter;
  t.needsUpdate = true;
  return t;
}
```

You can now paint any single tile from any ROM onto a 3D plane. For tile banks (extracting an entire sprite sheet), the longer version in [`../../GENESIS-EMU-GUIDE.md#extracting-tile-and-sprite-data-into-threejs-textures`](../../GENESIS-EMU-GUIDE.md) shows the loop.

---

## Chapter 6 — The 3D ROM gallery (mini-project)

The whole point. Drop a folder of ROMs into the app, see a 3D room of cartridges, click one to play.

Skeleton:

```tsx
function RomGallery({ onPick }: { onPick: (rom: { bytes: Uint8Array; title: string }) => void }) {
  const [entries, setEntries] = useState<RomEntry[]>([]);

  useEffect(() => {
    (async () => {
      const files = await window.api.listRoms();
      const palette: Array<[number, number, number]> =
        Array.from({ length: 16 }, (_, i) => [i * 16, i * 16, i * 16]);
      const out: RomEntry[] = [];
      for (const f of files) {
        const raw = await window.api.readRom(f.path);
        const bytes = detectAndNormalize(new Uint8Array(raw));
        const header = parseGenesisHeader(bytes);
        const thumb = extractTileBank(bytes, 0x4000, 64, palette);
        out.push({ bytes, title: header.titleOverseas || header.titleDomestic, thumb });
      }
      setEntries(out);
    })();
  }, []);

  return (
    <group>
      {entries.map((rom, i) => {
        const angle = (i / entries.length) * Math.PI * 2;
        const x = Math.cos(angle) * 5;
        const z = Math.sin(angle) * 5;
        return (
          <group key={i} position={[x, 0, z]} rotation={[0, -angle + Math.PI / 2, 0]} onClick={() => onPick(rom)}>
            <mesh>
              <boxGeometry args={[1.5, 2, 0.3]} />
              <meshStandardMaterial color="#222" />
            </mesh>
            <mesh position={[0, 0.3, 0.16]}>
              <planeGeometry args={[1.3, 0.8]} />
              <meshBasicMaterial map={rom.thumb} toneMapped={false} />
            </mesh>
            <Text position={[0, -0.7, 0.16]} fontSize={0.15} maxWidth={1.3} color="white" anchorX="center">
              {rom.title}
            </Text>
          </group>
        );
      })}
    </group>
  );
}
```

Add `<OrbitControls />` and you can walk through your cartridge library. Add an emulator component that activates when `onPick` fires and you've got a working ROM player with a 3D launcher.

---

## Hands-on tasks

- [ ] **T4.1** Wire up the `rom:open` IPC handler and the preload bridge. Click a button → native dialog opens.
- [ ] **T4.2** Once a ROM is loaded, call `parseGenesisHeader` and `console.log` the result. Confirm titles, region, serial are correct for your test ROM.
- [ ] **T4.3** Run `verifyChecksum` against the ROM. Note: many homebrew ROMs and modded ROMs intentionally don't match. The check is just a curiosity here.
- [ ] **T4.4** Decode a single tile at any offset (try `0x10000` as a guess) and render it as a 3D plane. It probably won't look right with the default greyscale palette — that's expected. The point is the pipeline works.
- [ ] **T4.5** Build a tile-bank renderer that lays out 64 tiles in an 8×8 grid as one texture. Display it in the scene.
- [ ] **T4.6** Build the ROM gallery: a function in main that scans `public/roms/` and returns metadata, plus the React component that renders cartridges in a circle.
- [ ] **T4.7** *(Stretch)* Integrate `wasm-genplus` or EmulatorJS so that clicking a cartridge in the gallery actually plays the ROM. The emulator can render to a canvas which you mount as a `CanvasTexture` on a 3D plane.
- [ ] **T4.8** *(Big stretch)* Add an SGDK build watcher: an IPC handler that uses `fs.watch` on your SGDK project's `out/` directory, reloads the emulator whenever a new `.bin` is built.

---

## Quiz

1. Why does the renderer process not have direct `fs.readFile` access for opening ROMs? Walk through the route the bytes take from disk to your React component.
2. The Genesis ROM checksum at `0x18E` is a 16-bit word. Your TypeScript reads `(rom[0x18E] << 8) | rom[0x18F]`. What does this assume about byte order, and why is that correct?
3. A user drops a `.smd` ROM into the app and the header parser returns garbage. What's the problem and the high-level fix?
4. Why is palette index 0 typically rendered with alpha = 0?
5. You decode a tile and the colors look totally wrong. What's the most likely fix without changing the decoder?
6. Why is `magFilter = THREE.NearestFilter` important for retro pixel art textures?
7. For a 1MB ROM, where would you start looking for tile data? (Hint: it varies per game — what's a reasonable approach?)

<details>
<summary>Show answers</summary>

1. The renderer is sandboxed for security — it can't touch the filesystem directly. The flow: React calls `window.api.openRom()` (a preload-exposed function) → preload calls `ipcRenderer.invoke('rom:open')` → main process's `ipcMain.handle('rom:open', ...)` runs `dialog.showOpenDialog` and `fs.readFile` (which only main can do) → returns a buffer → IPC delivers it to the renderer as a `Uint8Array`.
2. Big-endian. The 68000 is big-endian, so all multi-byte values in a Genesis ROM are stored most-significant-byte first. The expression `(rom[a] << 8) | rom[a+1]` matches that layout.
3. The ROM is in Super Magic Drive interleaved format — bytes are scrambled in 16KB blocks. Solution: detect by checking for `"SEGA"` at offset `0x100` in the raw bytes. If not present, run a de-interleaver before parsing.
4. The Genesis VDP treats palette index 0 as transparent for sprites and the second-plane background. Following that convention in your decoder keeps the visual semantics correct when you composite tiles.
5. The palette is wrong. The decoder produces palette indices 0–15; you map those indices to RGB via a palette table. If you used a default greyscale palette, the colors will be all wrong. The fix is to extract the actual palette from CRAM (or from a known offset in the ROM if you're working static).
6. Three.js's default filtering is linear interpolation, which blurs adjacent pixels. For an 8×8 tile scaled up onto a screen-sized plane, that blur turns chunky pixel art into mush. `NearestFilter` keeps each source texel as a hard square, preserving the pixel-art look.
7. The reasonable approach is to scan: look at the bytes in 32-byte chunks, treat them as candidate tiles, render them as a grid, and visually scrub through the ROM at different offsets. Most games keep tile banks in contiguous chunks somewhere in the first half of the ROM. Communities like Sonic Retro have documented exact offsets for famous games — for unknown ROMs, scanning is how you learn. (And if you want to be fancy, write a quick UI slider that lets you scrub through the ROM in real-time, rendering whatever offset is selected.)

</details>

---

## Reference

- Deep dive: [`../../GENESIS-EMU-GUIDE.md`](../../GENESIS-EMU-GUIDE.md) — has the full SMD de-interleaver, the multi-tile bank renderer, the wasm-genplus integration sketch, the SGDK build watcher, and the audio-reactive visualizer pattern.
- Main README Genesis section: [`../../README.md#lesson-4--loading-sega-genesis-roms-in-the-desktop-app`](../../README.md)
- Sonic Retro disassemblies (for known-good tile offsets): https://github.com/sonicretro
- wasm-genplus: https://github.com/h1romas4/wasm-genplus

---

## Next

→ [Lesson 05 — Sound Design](../05-sound-design/README.md)
