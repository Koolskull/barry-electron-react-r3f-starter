# Progress Tracker

Master checklist for the curriculum. Each task is a checkbox you tick when complete — commit your progress to git so you can see how far you've come.

The per-lesson task lists in each `lessons/NN-.../README.md` are the source of truth; this file mirrors them as a single dashboard. Keep both updated as you go (or just check off here and skim the lesson READMEs for context when you need it).

---

## How to use this tracker

1. Pick a lesson. Start with [01](lessons/01-electron-foundations/README.md) unless you have a reason not to.
2. Read the chapters at your own pace.
3. Do the hands-on tasks. Tick them off here AND in the lesson's README.
4. Take the quiz. Try to answer before peeking. Tick off the quiz once you've genuinely tried each question.
5. Move to the next lesson.

You can do this out of order if you want — the lessons are not strictly sequential, though they build on each other. Sound design (Lesson 05) requires the least from earlier lessons; Genesis (Lesson 04) and FMV (Lesson 03) require Lessons 01–02.

---

## Lesson 01 — Electron Foundations

**Status**: not started | in progress | complete (delete two)

- [ ] **T1.1** Clone the starter and run `npm install` + `npm run dev`. Confirm a window opens.
- [ ] **T1.2** Change the window's `width` and `backgroundColor`. Confirm it persists.
- [ ] **T1.3** Add `app:version` IPC handler. Expose via preload. Display in React.
- [ ] **T1.4** Run `npm run make`. Run the built executable standalone.
- [ ] **T1.5** Open DevTools and inspect the scene. Run `THREE.REVISION` in the console.
- [ ] **T1.6** *(Stretch)* Add a "second window" feature via IPC.
- [ ] **Quiz** — 6 questions on processes, security, IPC, packaging.

---

## Lesson 02 — React Three Fiber Primer

**Status**: not started

- [ ] **T2.1** Three geometries with three colors, all visible.
- [ ] **T2.2** Rotate one of them with `useFrame` using `delta` for frame-rate independence.
- [ ] **T2.3** Directional light with shadow casting. Ground plane receiving shadows.
- [ ] **T2.4** `<OrbitControls />` + `<Stats />`.
- [ ] **T2.5** `<EffectComposer>` with at least Bloom + Vignette.
- [ ] **T2.6** *(Stretch)* Load a `.glb` model.
- [ ] **T2.7** *(Stretch)* Hover-to-change-color using `useState` (correct case for state).
- [ ] **Quiz** — 6 questions on useFrame, materials, JSX-to-Three mapping, post-fx.

---

## Lesson 03 — FMV in 3D

**Status**: not started

- [ ] **T3.1** Render an `.mp4` as `VideoTexture` on a 3D plane.
- [ ] **T3.2** Wrap in `<EffectComposer>` with Scanline + Noise.
- [ ] **T3.3** 3-scene branching FMV using `SCENES` pattern.
- [ ] **T3.4** Time-gated clickable hotspot.
- [ ] **T3.5** Overlay current playback time via drei `<Html>`.
- [ ] **T3.6** *(Stretch)* Custom VHS shader from the FMV guide.
- [ ] **T3.7** *(Stretch — if applicable)* Port existing FMV project into the starter.
- [ ] **Quiz** — 6 questions on texture vs DOM, CORS, decoder leaks, autoplay policy.

---

## Lesson 04 — Genesis ROM Integration

**Status**: not started

- [ ] **T4.1** `rom:open` IPC handler + preload bridge. Native dialog works.
- [ ] **T4.2** `parseGenesisHeader` returns correct titles/serial/region.
- [ ] **T4.3** Run `verifyChecksum`. Note the result.
- [ ] **T4.4** Decode a single tile and render it as a 3D plane.
- [ ] **T4.5** Tile-bank renderer (8×8 grid of tiles as one texture).
- [ ] **T4.6** ROM gallery — circle of 3D cartridges with extracted previews.
- [ ] **T4.7** *(Stretch)* Integrate wasm-genplus / EmulatorJS for actual playback.
- [ ] **T4.8** *(Big stretch)* SGDK build watcher with hot reload.
- [ ] **Quiz** — 7 questions on the IPC route, byte order, .smd, palette format, tile filtering.

---

## Lesson 05 — Sound Design

**Status**: not started

- [ ] **T5.1** Web Audio `FMVoice`. Trigger from a button.
- [ ] **T5.2** Polyphonic `PolyFMSynth` driven by keyboard.
- [ ] **T5.3** Install JUCE + CMake. Build the chapter 4 sine plugin. Load in DAW.
- [ ] **T5.4** Replace sine with 4-operator FM voice (monophonic).
- [ ] **T5.5** Add `AudioProcessorValueTreeState` + generic editor.
- [ ] **T5.6** Polyphony in the JUCE plugin.
- [ ] **T5.7** Save a Genesis-style preset. Email the `.vst3` to a friend.
- [ ] **T5.8** *(Big stretch)* Hybrid pattern: JUCE → socket → Electron R3F visualizer.
- [ ] **Quiz** — 8 questions on FM math, voice stealing, RT safety, parameter management.

---

## Lesson 06 — Capstone (Hybrid Project)

**Status**: not started

- [ ] **T6.1** Pick a project. Write a paragraph in `CAPSTONE.md` describing the minimum.
- [ ] **T6.2** Build a 50-line proof-of-concept for the riskiest piece.
- [ ] **T6.3** First "something on screen" milestone.
- [ ] **T6.4** Minimum shippable built.
- [ ] **T6.5** Run `npm run make` (or JUCE build). Real artifact exists.
- [ ] **T6.6** Send to at least one other person. Watch them use it.
- [ ] **T6.7** Fix the top three confusing things. Re-ship.
- [ ] **T6.8** *(Recommended)* Write a one-page postmortem.

---

## Overall progress

Fill in as you go. Aspirational: hit every box. Realistic: get through Lessons 01–05, pick one capstone, ship it. Either way, you'll have built more than most people do in a year of "learning web tech."

| Lesson | Tasks done | Quiz done | Notes |
| --- | --- | --- | --- |
| 01 Electron | 0/6 | ☐ | |
| 02 R3F | 0/7 | ☐ | |
| 03 FMV | 0/7 | ☐ | |
| 04 Genesis | 0/8 | ☐ | |
| 05 Sound | 0/8 | ☐ | |
| 06 Capstone | 0/8 | n/a | |
