# Lesson 05 — Sound Design

> The biggest lesson in the curriculum. By the end you'll understand how the YM2612 mental model maps directly onto modern FM synthesis, have a working FM synth running in Web Audio inside your Electron app, and have shipped your first VST3 plugin via JUCE in C++.

**Estimated time**: A weekend, minimum. Realistically a week or two if you're new to plugin development.
**Prerequisites**: Lessons 01–02. The YM2612 internals knowledge you already have. Basic C++ comfort (for the JUCE chapters).

This lesson has more chapters than the others. They build on each other but you can stop after any of them — every chapter ends with something runnable.

---

## Learning objectives

1. Map every YM2612 concept (operators, algorithms, envelopes, LFO, detune) onto its modern DSP equivalent.
2. Build a polyphonic FM synth in Web Audio that runs in your Electron app's React UI.
3. Install JUCE, set up a CMake-based plugin project, and build a working sine-wave VST3.
4. Extend the JUCE plugin into a 4-operator polyphonic FM synth with multiple algorithms.
5. Add parameter management via `AudioProcessorValueTreeState`.
6. Understand real-time-safe C++ — what you can and can't do in `processBlock`.
7. Connect a JUCE audio engine to an Electron UI for the hybrid pattern.

---

## Chapter 1 — Your YM2612 mental model is the modern mental model

If you've programmed for the YM2612, you already know:

- An **operator** is a sine wave generator with its own envelope.
- An **algorithm** is a topology — which operators modulate which.
- Modulation depth controls timbre.
- Detuning operators by small amounts makes patches "fatter."
- An LFO modulates pitch and/or amplitude over time.

Every modern soft-synth — Operator, FM8, Dexed, the FM section of Massive X — does the same thing with better resolution and friendlier UIs. Same math.

Direct mapping:

| YM2612 / SGDK | JUCE / Web Audio |
| --- | --- |
| Operator | `juce::dsp::Oscillator<float>` or a hand-written sine |
| Algorithm | A graph of oscillators connected in `processBlock` |
| Detune (DT1/DT2) | Small frequency offset per operator |
| Multiplier (MUL) | Integer multiplier on operator frequency |
| Total Level (TL) | Per-operator output gain |
| AR / D1R / D2R / RR / D1L | `juce::ADSR` |
| LFO | A sub-audio oscillator modulating other parameters |
| Feedback | Operator output feeds back into its own phase input |
| DAC mode (Channel 6) | Sample playback (`juce::AudioFormatReader`) |

If you internalize this table, the JUCE API in chapter 3 will feel like seeing the same place from a different angle.

---

## Chapter 2 — Web Audio FM synth (the quick win)

Before you spend a weekend on JUCE, prove the concept in JS. A working FM voice in Web Audio is ~30 lines. You can wire it into your R3F scene immediately.

```ts
// src/audio/FMVoice.ts
export class FMVoice {
  private ctx: AudioContext;
  private carrier: OscillatorNode;
  private modulator: OscillatorNode;
  private modGain: GainNode;
  private env: GainNode;
  private out: GainNode;
  private started = false;

  constructor(ctx: AudioContext, destination: AudioNode) {
    this.ctx = ctx;
    this.carrier   = ctx.createOscillator();
    this.modulator = ctx.createOscillator();
    this.modGain   = ctx.createGain();
    this.env       = ctx.createGain();
    this.out       = ctx.createGain();

    // FM topology: modulator -> modGain -> carrier.frequency
    //              carrier -> env -> out -> destination
    this.modulator.connect(this.modGain);
    this.modGain.connect(this.carrier.frequency);
    this.carrier.connect(this.env);
    this.env.connect(this.out);
    this.out.connect(destination);

    this.env.gain.value = 0;
    this.out.gain.value = 0.3;
  }

  noteOn(freq: number, modRatio = 2, modIndex = 200) {
    const now = this.ctx.currentTime;
    this.carrier.frequency.value = freq;
    this.modulator.frequency.value = freq * modRatio;
    this.modGain.gain.value = modIndex;

    this.env.gain.cancelScheduledValues(now);
    this.env.gain.setValueAtTime(0, now);
    this.env.gain.linearRampToValueAtTime(1, now + 0.005);
    this.env.gain.linearRampToValueAtTime(0.6, now + 0.205);

    if (!this.started) {
      this.carrier.start();
      this.modulator.start();
      this.started = true;
    }
  }

  noteOff(releaseSec = 0.3) {
    const now = this.ctx.currentTime;
    this.env.gain.cancelScheduledValues(now);
    this.env.gain.setValueAtTime(this.env.gain.value, now);
    this.env.gain.linearRampToValueAtTime(0, now + releaseSec);
  }
}
```

That's one voice. Wrap a `Map<midi, FMVoice>` around it and you have polyphony. Hook keyboard events to `noteOn` / `noteOff` and you're playing FM. See [`../../SOUND-DESIGN-JUCE-BRIDGE.md#chapter-2--web-audio-quick-wins-inside-electron`](../../SOUND-DESIGN-JUCE-BRIDGE.md) for the polyphonic wrapper.

---

## Chapter 3 — Installing JUCE

```bash
git clone https://github.com/juce-framework/JUCE.git ~/dev/JUCE
```

Needs:
- CMake 3.22+
- A C++17 compiler
  - **Windows**: Visual Studio 2022 Community (free) — make sure to install the "Desktop development with C++" workload
  - **macOS**: Xcode + Command Line Tools
  - **Linux**: gcc/clang + `libasound2-dev`, `libxinerama-dev`, `libxcursor-dev`, `libfreetype6-dev`, etc.

JUCE is MIT-licensed for non-commercial / education use. You only need a paid license to ship paid plugins without crediting JUCE.

---

## Chapter 4 — Your first VST3

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.22)
project(BarrySynth VERSION 0.1.0)

add_subdirectory("$ENV{HOME}/dev/JUCE" JUCE)

juce_add_plugin(BarrySynth
    COMPANY_NAME "Barry"
    PLUGIN_MANUFACTURER_CODE Barr
    PLUGIN_CODE Bsy1
    FORMATS VST3 AU Standalone
    PRODUCT_NAME "BarrySynth"
    IS_SYNTH TRUE
    NEEDS_MIDI_INPUT TRUE
)

target_sources(BarrySynth PRIVATE
    Source/PluginProcessor.cpp
    Source/PluginEditor.cpp
)
target_compile_features(BarrySynth PRIVATE cxx_std_17)
target_link_libraries(BarrySynth PRIVATE
    juce::juce_audio_utils
    juce::juce_dsp
    juce::juce_audio_plugin_client
)
```

`Source/PluginProcessor.h`:

```cpp
#pragma once
#include <juce_audio_processors/juce_audio_processors.h>

class BarrySynthProcessor : public juce::AudioProcessor {
public:
    BarrySynthProcessor();
    void prepareToPlay(double sr, int) override { sampleRate = sr; env.setSampleRate(sr); }
    void releaseResources() override {}
    void processBlock(juce::AudioBuffer<float>&, juce::MidiBuffer&) override;

    juce::AudioProcessorEditor* createEditor() override { return new juce::GenericAudioProcessorEditor(*this); }
    bool hasEditor() const override { return true; }
    const juce::String getName() const override { return "BarrySynth"; }
    bool acceptsMidi() const override { return true; }
    bool producesMidi() const override { return false; }
    bool isMidiEffect() const override { return false; }
    double getTailLengthSeconds() const override { return 1.0; }
    int getNumPrograms() override { return 1; }
    int getCurrentProgram() override { return 0; }
    void setCurrentProgram(int) override {}
    const juce::String getProgramName(int) override { return {}; }
    void changeProgramName(int, const juce::String&) override {}
    void getStateInformation(juce::MemoryBlock&) override {}
    void setStateInformation(const void*, int) override {}

private:
    double sampleRate = 44100.0;
    double phase = 0.0;
    double phaseInc = 0.0;
    bool   noteIsOn = false;
    juce::ADSR env;
};
```

`Source/PluginProcessor.cpp`:

```cpp
#include "PluginProcessor.h"

BarrySynthProcessor::BarrySynthProcessor()
    : AudioProcessor(BusesProperties().withOutput("Output", juce::AudioChannelSet::stereo(), true)) {
    juce::ADSR::Parameters p { 0.01f, 0.2f, 0.6f, 0.4f };
    env.setParameters(p);
}

void BarrySynthProcessor::processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midi) {
    buffer.clear();
    for (const auto m : midi) {
        const auto msg = m.getMessage();
        if (msg.isNoteOn()) {
            const double freq = juce::MidiMessage::getMidiNoteInHertz(msg.getNoteNumber());
            phaseInc = (freq / sampleRate) * juce::MathConstants<double>::twoPi;
            env.noteOn();
            noteIsOn = true;
        } else if (msg.isNoteOff()) {
            env.noteOff();
            noteIsOn = false;
        }
    }

    auto* L = buffer.getWritePointer(0);
    auto* R = buffer.getWritePointer(1);
    for (int n = 0; n < buffer.getNumSamples(); ++n) {
        const float s = std::sin(phase) * 0.3f * env.getNextSample();
        L[n] = R[n] = s;
        phase += phaseInc;
        if (phase > juce::MathConstants<double>::twoPi) phase -= juce::MathConstants<double>::twoPi;
    }
}

juce::AudioProcessor* JUCE_CALLTYPE createPluginFilter() {
    return new BarrySynthProcessor();
}
```

Build:

```bash
cmake -B build -S .
cmake --build build --config Release
```

The `.vst3` lands in `build/BarrySynth_artefacts/`. Copy it to your DAW's plugin folder (paths in [`../../SOUND-DESIGN-JUCE-BRIDGE.md#chapter-3--juce-installation-and-first-plugin`](../../SOUND-DESIGN-JUCE-BRIDGE.md)), rescan plugins, load on a MIDI track, play. Sine wave when you press a key — your first VST shipped.

---

## Chapter 5 — From sine to 4-operator FM synth

Extend chapter 4 into actual FM synthesis. The full code (FMOperator, FMVoice, PolyFMSynth classes implementing YM2612-style algorithms) is in [`../../SOUND-DESIGN-JUCE-BRIDGE.md#chapter-4--a-genesis-style-fm-synth-in-juce`](../../SOUND-DESIGN-JUCE-BRIDGE.md). About 150 lines of C++.

The structure:

- **`FMOperator`** — one sine + envelope + output level. Has `renderNextSample(modInput)` which produces one sample given the modulation input from another operator.
- **`FMVoice`** — owns 4 `FMOperator`s. Has a `setAlgorithm(...)` that picks the topology (parallel, serial, paired, etc.) and a `renderNextSample()` that runs the operators in the algorithm's order.
- **`PolyFMSynth`** — owns 8 `FMVoice`s. Handles voice allocation, voice stealing, MIDI dispatching.

In `BarrySynthProcessor::processBlock`, replace the sine wave loop with `synth.renderNextBlock(buffer)`. That's the entire integration.

---

## Chapter 6 — Parameters and presets

Manual setting of operator ratios via hardcoded values gets old fast. JUCE's `AudioProcessorValueTreeState` solves three problems at once:

1. Parameters that show up in the DAW's automation lane.
2. Preset save/load.
3. Generic UI generation (free knob-per-parameter editor).

```cpp
juce::AudioProcessorValueTreeState parameters {
    *this, nullptr, "BarrySynth",
    {
        std::make_unique<juce::AudioParameterFloat>("op2Ratio", "Op2 Ratio",
            juce::NormalisableRange<float>(0.5f, 16.0f, 0.5f), 2.0f),
        std::make_unique<juce::AudioParameterFloat>("modIndex", "Mod Index",
            juce::NormalisableRange<float>(0.0f, 1.0f, 0.001f), 0.5f),
        std::make_unique<juce::AudioParameterFloat>("attack", "Attack",
            juce::NormalisableRange<float>(0.001f, 5.0f, 0.001f, 0.3f), 0.01f),
        // ...
        std::make_unique<juce::AudioParameterChoice>("algorithm", "Algorithm",
            juce::StringArray{ "Alg0", "Alg4", "Alg7" }, 1),
    }
};
```

Read parameters once per block (not per sample) into local variables for performance:

```cpp
void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midi) {
    const float modIndex = *parameters.getRawParameterValue("modIndex");
    const int   alg      = (int) *parameters.getRawParameterValue("algorithm");
    synth.setAlgorithm(static_cast<FMVoice::Algorithm>(alg));
    synth.setModIndex(modIndex);
    // ...
}
```

Switch the editor from `GenericAudioProcessorEditor` to `juce::GenericAudioProcessorEditor(*this)` and you get auto-generated UI for every parameter. Free knobs.

---

## Chapter 7 — Real-time-safe C++ (the SGDK skill that pays off)

The audio thread has hard real-time deadlines: one `processBlock` call must complete in less time than the buffer represents at the current sample rate, or you get an audible glitch (xrun, pop). For a 256-sample buffer at 48kHz, that's 5.3 milliseconds.

You've been working under tighter constraints than this on the Genesis. The mental model transfers directly.

### Forbidden in `processBlock`

| Don't | Why |
| --- | --- |
| `new` / `delete` / `malloc` / `free` | Allocator can take milliseconds; you have microseconds |
| `std::mutex` (blocking) | Priority inversion deadlocks audio thread |
| `throw` exceptions | Stack unwinding is allocation-y |
| `std::vector::push_back` (if it reallocates) | Reallocation under the hood |
| `printf` / `Logger` | I/O has unbounded latency |
| File I/O | Same |

### Allowed

- Read `std::atomic` parameters
- Use lock-free FIFOs (`juce::AbstractFifo`)
- Pre-allocated `std::array`, `juce::AudioBuffer`
- Math (sin, cos, std::pow)
- SIMD (`juce::dsp::SIMDRegister<float>`)

The full guide with the `juce::AbstractFifo` pattern for GUI-to-audio-thread communication is in [`../../SOUND-DESIGN-JUCE-BRIDGE.md#chapter-9--real-time-safe-c-the-part-to-be-careful-about`](../../SOUND-DESIGN-JUCE-BRIDGE.md).

---

## Chapter 8 — The hybrid pattern (Electron UI + JUCE audio)

Three approaches, increasing in commitment, detailed in [`../../SOUND-DESIGN-JUCE-BRIDGE.md#chapter-7--the-electron--juce-hybrid-pattern`](../../SOUND-DESIGN-JUCE-BRIDGE.md):

1. **Local socket bridge** — Electron and JUCE run as separate processes, communicate over localhost TCP. Easiest to debug, cleanest separation. Recommended starting point.
2. **JUCE plugin with WebView UI** (JUCE 8+) — single `.vst3` whose UI is your React/Three.js app embedded via `juce::WebBrowserComponent`. Loads in any DAW.
3. **JUCE as a Node native addon** — compile JUCE into a `.node` module Electron loads. Single process, low IPC overhead, gnarly to set up.

For learning, do (1). For shipping a plugin, eventually do (2). (3) is mostly an option to know about, not one to choose.

---

## Hands-on tasks

These build incrementally. Each is a useful stopping point on its own.

- [ ] **T5.1** Build the Web Audio `FMVoice` from Chapter 2. Trigger `noteOn(440)` from a button click. Hear FM synthesis come out of your Electron app.
- [ ] **T5.2** Make a `PolyFMSynth` wrapper. Hook keyboard events (A/W/S/E/D/F/T/G/Y/H...) so you can play melodies. Stretch: add an on-screen keyboard.
- [ ] **T5.3** Install JUCE + CMake. Build the chapter 4 sine-wave plugin. Load it in a DAW (free options: Reaper, Cakewalk, Bitwig demo). Play notes.
- [ ] **T5.4** Replace the sine in the plugin with the chapter 5 4-operator FM voice (monophonic — one note at a time is fine). Pick one algorithm (Alg4 — parallel pairs is a good first pick). Hear FM coming from a real VST.
- [ ] **T5.5** Add `AudioProcessorValueTreeState` from chapter 6. Use the generic editor. Tweak parameters in the DAW.
- [ ] **T5.6** Add polyphony (the `PolyFMSynth` from `SOUND-DESIGN-JUCE-BRIDGE.md` chapter 4). Confirm chords work.
- [ ] **T5.7** Save a preset that sounds like a Genesis pad. Email the `.vst3` and the preset file to a friend.
- [ ] **T5.8** *(Big stretch)* The hybrid pattern: get the JUCE plugin (or its standalone build) to send audio levels over a localhost socket to your Electron app, and drive an R3F visualizer with that data.

---

## Quiz

1. In your Web Audio FM synth, you set `modGain.gain.value = 200` and the result is a buzzy mess. What does that 200 represent and how would you scale it more sensibly?
2. What is "voice stealing" in a polyphonic synth, and why is it necessary?
3. JUCE's `processBlock` is called with a 256-sample buffer at 48kHz. What's your time budget per call? What's the practical implication?
4. You added `std::cout << "note on"` to `processBlock` to debug. The audio now crackles intermittently. Why?
5. Your DAW's "automation lane" for your synth is blank — none of your parameters show up. What did you forget?
6. You want to add a `vector<float>` buffer for delay-line storage. Where should you allocate it, and where should it live?
7. Why does `*parameters.getRawParameterValue("modIndex")` return an `std::atomic<float>*`-ish thing instead of just a `float`?
8. The 4-operator FM topology where Op1 modulates Op2 modulates Op3 modulates Op4 (and only Op4 goes to output) is which classic algorithm number on the YM2612?

<details>
<summary>Show answers</summary>

1. `modGain.gain.value` is added directly to `carrier.frequency` (since the connection target is a `.frequency` AudioParam). So a value of 200 means "the modulator is producing peak deviations of ±200 Hz on the carrier's frequency." Whether that's musical depends on the carrier frequency: 200 Hz deviation on a 440 Hz carrier is huge (FM index ~0.45 at 880Hz modulator — buzzy). On a 4kHz carrier it's gentle. Scale `modIndex` proportionally to carrier frequency, or treat it as an "FM index" where `modIndex × modulatorFrequency` becomes the deviation.
2. When you receive a note-on event but all voices are already busy, you have to free one — pick the oldest, quietest, or least-recently-used voice and replace it with the new note. Without voice stealing, hitting more keys than you have voices for either drops new notes (frustrating to play) or fails silently. Real synths universally do this.
3. ~5.3 ms (256 / 48000). In practice you want to use under 50% of that to leave headroom for the rest of the audio graph. That means no allocations, no locks, no I/O — same constraints as Genesis HBlank handlers, just looser.
4. `std::cout` does I/O, which can block for an unbounded time (especially if the terminal is scrolling, or if stdout is piped). The audio thread missing its deadline is exactly a buffer underrun, which sounds like a crackle. Use `juce::Logger::writeToLog` from the message thread only, or a lock-free queue from audio thread to a logger thread.
5. The parameters need to be declared via `AudioProcessorValueTreeState` (or as `AudioParameterFloat`s passed to the processor constructor). Just having member variables isn't enough — the host has no way to know they exist. Wiring them into the `AudioProcessorValueTreeState` registers them with the DAW.
6. Allocate it in `prepareToPlay(sampleRate, blockSize)` — that's called when the host knows the buffer size and isn't on the audio thread. Store it as a member variable (`std::vector<float> delayLine` in the class). Resize there. In `processBlock`, only read/write into it; never call `.resize()` or `.push_back()` on a vector that might reallocate.
7. Audio parameters are read by the audio thread but written by the GUI thread (or by the host during automation). The atomic ensures that read-after-write doesn't tear or produce a half-updated value. JUCE returns a pointer to the atomic so you can `load()` cheaply in `processBlock`.
8. Algorithm 0 (serial chain — Op1 → Op2 → Op3 → Op4 → out). It's the most "tubular" / DX7-bell-like topology because every modulator is itself being modulated.

</details>

---

## Reference

- The deep dive: [`../../SOUND-DESIGN-JUCE-BRIDGE.md`](../../SOUND-DESIGN-JUCE-BRIDGE.md) — 10 chapters with the full `FMOperator` / `FMVoice` / `PolyFMSynth` source, parameter management, MIDI handling, the hybrid integration patterns, and the real-time-safety deep dive.
- JUCE docs: https://docs.juce.com/master/
- The "Audio Programmer" community / YouTube channel is excellent for JUCE specifically.
- Dexed (free DX7 emulator written in JUCE-adjacent C++) — great reference: https://github.com/asb2m10/dexed

---

## Next

→ [Lesson 06 — Capstone (Hybrid Projects)](../06-capstone-hybrid/README.md)
