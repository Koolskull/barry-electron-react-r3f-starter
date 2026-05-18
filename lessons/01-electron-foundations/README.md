# Lesson 01 — Electron Foundations

> By the end of this lesson, you'll understand what Electron actually is, the main-vs-renderer split, why the preload script exists, and you'll have a desktop window running your own React code that you built to an installable EXE.

**Estimated time**: 2–3 hours.
**Prerequisites**: Node.js 18+ installed, a code editor, basic comfort with `npm` commands.

---

## Learning objectives

By the end of this lesson you should be able to:

1. Explain the main process vs. renderer process to someone else.
2. Identify which APIs (`fs`, `dialog`, `BrowserWindow`) live in which process.
3. Wire up a secure preload script that exposes a single Node API to React.
4. Build a `.exe` / `.dmg` / `.AppImage` of a working desktop app.
5. Use Chromium DevTools to debug both the renderer and (via VS Code) the main process.

---

## Chapter 1 — What Electron actually is

Electron is **two things glued together**:

1. **Chromium** — the browser engine. Same rendering / WebGL / Web Audio stack as Chrome.
2. **Node.js** — the runtime you'd use for a CLI tool or server.

When your Electron app starts, Node runs first (this is the "main process"). It creates a Chromium window (the "renderer process"). The two processes communicate over **IPC** (inter-process communication).

A useful mental model from your C background:

```
Genesis cartridge analogue            Electron analogue
─────────────────────────────         ─────────────────────────────
Boot ROM → 68k → loads game    →      Node runs main.ts → creates BrowserWindow
68k ↔ Z80 (via bus arbiter)    →      main ↔ renderer (via IPC + preload)
VDP draws sprites              →      Chromium draws DOM + WebGL
YM2612 generates audio         →      Web Audio API generates audio
```

The split exists for the same reason the Z80 was kept separate from the 68k: process isolation makes the system more robust. A renderer crash (a memory leak in your React app, a runaway shader) doesn't take down the whole app.

### What Electron isn't

- It's not "a way to make web apps native." Your app *is* a web app; Electron just gives it OS-level capabilities and a window.
- It's not magic for performance. A poorly-written Electron app is still a poorly-written app. Hardware acceleration is on by default for WebGL, but JavaScript is JavaScript.
- It's not small. An empty Electron app is ~150MB because it ships Chromium. There's no way around this. If size matters more than capability, look at Tauri (Rust + system webview) — but you lose the consistent rendering Chromium gives you.

---

## Chapter 2 — Main vs. Renderer

The single most important concept in Electron. Once this clicks, most of the API documentation makes sense in retrospect.

### Main process

- **One per app.** Started by `electron .` or by the packaged binary.
- Has the full Node.js standard library: `fs`, `path`, `os`, `child_process`, `net`.
- Has Electron's "main-only" modules: `app`, `BrowserWindow`, `dialog`, `Menu`, `Tray`, `ipcMain`.
- Cannot touch the DOM. There's no `document` here.

### Renderer process

- **One per window** (your app can have many windows).
- Is a Chromium tab. Has `document`, `window`, `navigator`, WebGL, Web Audio, etc.
- Does NOT have Node access by default (and you should keep it that way).
- Talks to main via `window.electronAPI.something()` if you've set up a preload script.

### Why no Node access in the renderer

Imagine your React app loads a YouTube video as an FMV asset. That video's player is JavaScript running in your renderer. If your renderer had `fs` access, that JavaScript could read `~/.ssh/id_rsa`. The same is true if your app loads any URL with content you didn't write yourself.

This is exactly the "Z80 wants the bus" problem on the Genesis — uncontrolled access to shared resources is a recipe for chaos. The solution in both cases is a controlled bridge.

---

## Chapter 3 — The preload script (the bridge)

The preload script is the *only* file that runs in the renderer's context **with Node access intact** before any web content loads. It's the place where you decide exactly which Node-ish capabilities to expose to your React code.

```ts
// electron/preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('api', {
  openRomDialog: () => ipcRenderer.invoke('rom:open'),
  saveScreenshot: (dataUrl: string) => ipcRenderer.invoke('screenshot:save', dataUrl),
  // Add more carefully. Each line here is API surface you can never take back.
});
```

In your React code:

```ts
const result = await window.api.openRomDialog();
```

What's happening:

1. React calls `window.api.openRomDialog()`. This is just a function; it doesn't have file access.
2. The preload-exposed function `ipcRenderer.invoke('rom:open')` sends a message to the main process and returns a `Promise`.
3. The main process's `ipcMain.handle('rom:open', ...)` handler runs. **This is where the actual `fs` call happens.** Main process has full Node access.
4. The handler returns a value, which becomes the resolved value of the renderer's `Promise`.

### Three security invariants to remember

| Setting | Default | Keep it? |
| --- | --- | --- |
| `contextIsolation` | `true` | YES. This is what keeps your preload's `require` out of the renderer's hands. |
| `nodeIntegration` | `false` | YES (keep false). Setting to true is "expose all of Node to the renderer" — basically gives any script in the page filesystem access. |
| `sandbox` | `false` (but worth turning ON) | Consider enabling. Restricts preload to a narrower API surface. |

If a tutorial tells you to "just turn on `nodeIntegration` for now," that tutorial is wrong. Use IPC. It takes 5 extra minutes and prevents an entire class of vulnerabilities.

---

## Chapter 4 — Your first window

Open `electron/main.ts` in this repo (or create it if working from scratch):

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

Three things people miss the first time:

1. **`isDev` from `app.isPackaged`** — *do not* use `NODE_ENV`. Vite controls that variable for the renderer, and it's not set in the main process the way you'd expect.
2. **The macOS `activate` handler** — Mac apps stay running with no windows. Without this handler, clicking your dock icon does nothing.
3. **`win-all-closed` skipping quit on darwin** — same reason. Mac apps quit via `Cmd+Q`, not by closing the last window.

### Running it

```bash
npm install
npm run dev
```

A window should open with the React app inside it. Hot reload works in the renderer — edit `src/App.tsx`, see the change immediately. For changes to `electron/main.ts`, you'll need to restart `npm run dev`.

---

## Chapter 5 — Building to a real EXE

```bash
# Build for your current platform
npm run make
```

Output lands in `out/make/`. On Windows you get a `Setup.exe` (Squirrel-based installer). On macOS, a `.dmg`. On Linux, a `.AppImage` (or `.deb` / `.rpm` if you configure Forge for them).

For cross-platform builds, Forge accepts `--platform` and `--arch`:

```bash
npm run make -- --platform=win32 --arch=x64
```

But realistically: build for Windows on Windows, Mac on Mac, Linux on Linux (or in CI). Cross-compiling Electron apps works in theory and breaks in practice.

### Things that will bite you the first time you ship

- **Code signing.** An unsigned `.exe` triggers SmartScreen ("Windows protected your PC"). Friends will think the app is broken. For real distribution, get a code-signing certificate. For sharing with one friend, it's fine — they click "More info → Run anyway."
- **macOS Gatekeeper.** Stricter than Windows. Unsigned `.app` files won't open at all by default; users need to right-click → Open the first time, or you need an Apple Developer ID.
- **Auto-update.** Squirrel handles this if you point Forge at a static file host. Don't worry about it until you've shipped v1.

---

## Hands-on tasks

Check each off in this file as you complete it (you can commit progress to git).

- [ ] **T1.1** Clone the starter and run `npm install` + `npm run dev`. Confirm a window opens.
- [ ] **T1.2** Change the window's `width` and `backgroundColor` in `electron/main.ts`. Confirm it persists across restarts.
- [ ] **T1.3** Add a new IPC handler `app:version` in `electron/main.ts` that returns `app.getVersion()`. Expose it via `preload.ts`. Display the version somewhere in your React UI.
- [ ] **T1.4** Run `npm run make`. Find the built executable in `out/make/`. Run it without `npm run dev` going. Confirm it works standalone.
- [ ] **T1.5** Open DevTools (`Ctrl+Shift+I` / `Cmd+Opt+I`) in the running app. Find the `<canvas>` element in the Elements panel. Switch to the Console and run `THREE.REVISION`. See what version of Three.js the scene is using.
- [ ] **T1.6** *(Stretch)* Add a "second window" feature — an IPC call from React that asks main to open a second `BrowserWindow`. Test that closing one doesn't close the other.

---

## Quiz

Try to answer without scrolling. Then check.

1. You wrote `window.fs.readFileSync('/etc/passwd')` in a React component. It throws `ReferenceError: window.fs is undefined`. Why?
2. You set `nodeIntegration: true` and `contextIsolation: false` "because the tutorial said so." Your app loads a YouTube embed. What's the security risk?
3. `ipcRenderer.invoke` vs `ipcRenderer.send` — what's the practical difference?
4. You closed all the windows on macOS and were surprised the app didn't quit. Was that a bug or correct behavior, and what code controls it?
5. After running `npm run make`, you tried to email the resulting `.exe` to a friend. Gmail blocked it. What's the cleanest workaround that's not "rename it to .txt"?
6. Why is `app.isPackaged` preferred over `process.env.NODE_ENV === 'production'` for detecting prod vs. dev?

<details>
<summary>Show answers</summary>

1. The renderer doesn't have Node access — `fs` isn't available on `window`. To read files, you'd add an IPC handler in main, expose it through `preload.ts` via `contextBridge`, and call `window.api.readFile(...)` instead.
2. With `nodeIntegration: true` and `contextIsolation: false`, any JavaScript loaded into the renderer — including third-party iframes, ad networks, or compromised dependencies — can call `require('fs')` and read arbitrary files. The YouTube embed alone probably won't, but you've handed the whole filesystem to anything that runs in your renderer.
3. `invoke` is request-response: it returns a `Promise` resolved by the main process's `handle` callback. `send` is fire-and-forget — no return value, and the main process listens with `ipcMain.on(...)`. Use `invoke` for anything where you want a result.
4. Correct behavior. On macOS, apps typically stay in the dock even with no windows. The `app.on('window-all-closed', ...)` handler in `main.ts` explicitly checks `process.platform !== 'darwin'` and only quits on other platforms.
5. Upload the file to a service like GitHub Releases, Dropbox, or a transfer service and share the link. Gmail blocks executable attachments by extension. Long-term, you'll want code-signing + an installer that can pass virus scanners; short-term, a link works fine.
6. `NODE_ENV` is set by tooling (Vite, webpack, esbuild) and is reliable in the renderer but unreliable in the main process — depending on how you invoke `electron`, it may not be set at all. `app.isPackaged` is a runtime check that's true exactly when the app is running from an actual built artifact, regardless of build tooling.

</details>

---

## Reference

- Full main process / preload examples: [`../../README.md`](../../README.md) sections "How Electron actually works" and "Lesson 1".
- Official docs: https://www.electronjs.org/docs/latest/
- Forge packaging: https://www.electronforge.io/

---

## Next

→ [Lesson 02 — React Three Fiber Primer](../02-r3f-primer/README.md)
