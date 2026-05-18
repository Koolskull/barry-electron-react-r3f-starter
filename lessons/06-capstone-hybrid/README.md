# Lesson 06 — Capstone: Hybrid Project

> The final lesson. You pick one of four hybrid projects (or invent your own) and ship it. Each combines at least three of the skills from earlier lessons. This is where the curriculum stops being "learn things" and starts being "make a thing."

**Estimated time**: A week to a month, depending on scope.
**Prerequisites**: Lessons 01–05, or honestly: parts of 01–05 plus willingness to look things up.

---

## How this lesson works differently

The previous lessons were structured: chapters, then tasks, then a quiz. This one is open-ended. You pick a project, define your own "minimum shippable" version, and the only milestone that matters is **a `.exe` you've sent to someone who isn't you.**

The four projects below are starting points. Don't feel obligated to pick one — they're examples of "what stitching the lessons together can produce." Inventing your own is at least as good.

---

## Project A — "Genesis B-Roll"

**The pitch**: an FMV game where the in-world TVs, monitors, and arcade cabinets are actually running your legally-dumped Genesis ROMs in real time. Camera moves through a 3D environment; the videos on the walls keep playing; the arcade in the corner has Sonic running on it.

**Skills it combines**: Lessons 01 (Electron), 02 (R3F), 03 (FMV in 3D), 04 (Genesis ROM integration).

**Minimum shippable**:
- A 3D scene with one FMV plane and one Genesis emulator on a 3D plane.
- Both running simultaneously, both visible at once, no glitching.
- Built to an EXE.

**Stretch goals**:
- Multiple emulators running different ROMs simultaneously (CPU permitting).
- ROM gallery as one of the 3D objects — click it to swap which ROM is loaded.
- FMV plays through, then triggers the emulator to start.
- Camera moves on a timed path (use drei's `useScroll` or `CameraControls`).

**The hard part**: keeping both the video decoder and the Wasm emulator running smoothly at 60fps. Test on the slowest machine you can find before declaring it done.

---

## Project B — "Live-Scored Cutscenes"

**The pitch**: your FMV plays. A JUCE-based FM synth (running as a sidecar process) is responsible for the music. Markers in the video metadata send "note on" messages that drive the synth in sync with the video. The synth audio is captured back and fed into a 3D visualizer behind the FMV.

**Skills it combines**: Lessons 01, 02, 03 (FMV), 05 (Sound design — both Web Audio and JUCE).

**Minimum shippable**:
- FMV plays in the Electron app.
- Specific times in the video trigger specific MIDI notes via Web Audio FM synth (no JUCE yet, just Web Audio for v1).
- The audio drives an R3F visualizer.

**Stretch goals**:
- Replace Web Audio with a standalone JUCE process talking over localhost socket.
- Markers stored in a separate `.json` file ("at t=3.4s, play C4 on patch X") authored manually.
- A tiny in-app marker editor: play video, hit a key to drop a marker at the current time.

**The hard part**: timing. Web Audio latency is unpredictable. Don't rely on it for tight sync — instead, schedule notes ahead of the video's current time using `AudioContext.currentTime`.

---

## Project C — "Cartridge Museum"

**The pitch**: drop a folder full of `.bin` files into the app. It parses each header, extracts the first interesting tile bank, and renders each ROM as a 3D cartridge model in a navigable space. Walk through the room, look at each cartridge, click one to play it. The 3D models are generated *from the ROM data itself*.

**Skills it combines**: Lessons 01, 02, 04 (heavily), some optional 03 for ambient video on the walls.

**Minimum shippable**:
- ROM gallery component (you built this in Lesson 04's tasks).
- Click a cartridge → emulator activates and plays it.
- Walk-around camera (drei's `PointerLockControls` is a one-liner first-person camera).

**Stretch goals**:
- Each cartridge spins slowly, with a parallax shadow.
- Hover over a cartridge → tooltip with title, year, serial.
- A back-of-the-cartridge view showing the ROM hex dumped in a stylized way.
- Custom palette extraction (find the actual CRAM/palette data per ROM instead of using a default).

**The hard part**: tile offset discovery. For unknown ROMs, finding the offset where the title screen tile data lives is its own mini-project. A "scrub through the ROM with a slider" debugging UI is genuinely useful here.

---

## Project D — "VST with a Three.js UI"

**The pitch**: a JUCE VST3 plugin whose UI is a full React + Three.js scene, hosted via JUCE 8's `WebBrowserComponent`. Plug it into a DAW. The UI animates in response to the audio (envelopes, FFT spectrum, LFO position). The audio is the FM synth from Lesson 05. Distribute as a single `.vst3` file.

**Skills it combines**: Lessons 01 partially (the React/build half), 02 (R3F), 05 (deepest).

**Minimum shippable**:
- JUCE plugin with `WebBrowserComponent` editor.
- The web UI shows at least one knob and one R3F-animated element (rotating cube, oscilloscope).
- Knob changes propagate to JUCE; audio responds.

**Stretch goals**:
- Real-time spectrum analyzer in R3F driven by FFT data sent from JUCE.
- Envelope visualization: a 3D shape morphs based on the current ADSR position of each operator.
- Preset thumbnails as 3D objects.

**The hard part**: JUCE WebView ↔ C++ bridge. JUCE 8's API is decent but new enough that examples are sparse. Plan to read the JUCE forum and source code.

---

## A few "invent your own" prompts

- A music player that loads your Genesis ROMs and plays only their soundtracks (extract the music driver, run it in Wasm, ignore graphics).
- An SGDK companion app: open your SGDK project, see a 3D preview of tilemaps as you edit them, hot-reload the ROM on every `make`.
- A homebrew demo viewer — display Genesis demos as 3D rotating screens against a synthwave background.
- A Genesis tile editor as an Electron app, with a 3D preview of the tile in context (placed on a 3D model of a sprite from your game).
- An interactive FMV portfolio site that you build as an Electron app and also export to a static website (Vite makes both possible from the same source).

---

## Hands-on tasks

These are intentionally vague — fill them in for your specific project.

- [ ] **T6.1** Pick a project (A, B, C, D, or your own). Write a paragraph describing the minimum shippable version. Commit that paragraph to git as `CAPSTONE.md` in your repo.
- [ ] **T6.2** Identify the riskiest unknown technical piece. Build a 50-line proof-of-concept that just demonstrates *that one piece* in isolation. (Example: for Project A, can both an FMV and emulator render simultaneously without one starving the other?)
- [ ] **T6.3** Get a "first something on screen" milestone working. Doesn't have to be good. Has to be visible.
- [ ] **T6.4** Build the minimum shippable. Decide ruthlessly what's out of scope.
- [ ] **T6.5** Run `npm run make` (or the JUCE build). Get an actual artifact.
- [ ] **T6.6** Send the artifact to at least one other person. Watch them try to use it. Note what confused them.
- [ ] **T6.7** Fix the top three confusing things. Re-ship.
- [ ] **T6.8** *(Optional but recommended)* Write a one-page postmortem. What worked, what didn't, what you'd do differently. This is the single most useful artifact a project produces, and the one most people skip.

---

## Quiz

There's no quiz for this lesson. Or rather: the project is the quiz, and your friends are the graders.

But here are three questions worth thinking through before you start:

1. What's the *one* thing about this project that, if it doesn't work, kills the whole idea? Build that first.
2. If you only have 20 hours to spend, what's the version that fits in 20 hours? Build that version, not the "real" version.
3. Who will be the first person to use this besides you? What will their first 10 seconds look like? Optimize for those 10 seconds first.

---

## You're done with the curriculum

If you've genuinely made it through Lessons 01–06 with the hands-on tasks done, you can:

- Ship desktop apps with native features
- Build 3D web/desktop scenes with R3F
- Render video as a first-class 3D citizen (not an afterthought)
- Integrate Genesis ROMs into modern UIs
- Build FM synthesizers in Web Audio AND in JUCE
- Ship VST3 / AU plugins
- Bridge web frontends to C++ audio backends

That's a genuinely unusual skill stack. There aren't many people who can do all of these. The intersection — Genesis-aware desktop apps with custom synth backends and 3D UIs — is yours.

What you build with it is the interesting question. The curriculum is the rope ladder; the climb is up to you.

---

## Where the rope ladder ends

Things worth learning *after* this curriculum, in rough order of usefulness:

- **GLSL shaders** in earnest. R3F lets you write custom shaders for materials and post-fx. A weekend on the Book of Shaders (https://thebookofshaders.com) pays off forever.
- **WebAssembly authoring** — once you've integrated other people's Wasm, writing your own (e.g. a port of an SGDK utility, or a custom audio DSP module) is the natural next step. Rust + `wasm-bindgen` is the friendly path; C + Emscripten is the SGDK-adjacent path.
- **Three.js's animation system** — `AnimationMixer`, GLTF animations, FK/IK rigs. Worth knowing if you ever want characters that move.
- **DAW automation and OSC** — for the audio side, controlling your synth from a hardware MIDI controller or another DAW over OSC opens up live-performance possibilities.
- **More serious DSP** — Will Pirkle's "Designing Audio Effect Plugins in C++" is the closest thing to a textbook for the JUCE ecosystem.

---

## Final reference

- Master README hub: [`../../README.md`](../../README.md)
- All three deep guides: [`../../FMV-GUIDE.md`](../../FMV-GUIDE.md), [`../../GENESIS-EMU-GUIDE.md`](../../GENESIS-EMU-GUIDE.md), [`../../SOUND-DESIGN-JUCE-BRIDGE.md`](../../SOUND-DESIGN-JUCE-BRIDGE.md)
- Progress tracker: [`../../PROGRESS.md`](../../PROGRESS.md)

Good luck, Barry. Send me the EXE when you're done.
