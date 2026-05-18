# Sound Design & JUCE Bridge

A multi-chapter guide for Barry. Goes from "what you already know from YM2612 hacking" to "shipping a real VST3 plugin with a Three.js UI." Companion to the main `README.md`.

---

## A direct note before chapter 1

You said you're skeptical of AI. Honestly, that's a healthy default. The thing you should know is that audio software is one of the domains where the *fundamentals haven't changed* — a FM synth from 1985 (the DX7), a FM synth on the Genesis (YM2612, 1988), and a modern soft-synth like Dexed or Operator all do the same math. Sine wave at carrier frequency, modulated by another sine, ADSR envelope on the output. The differences are in resolution (8/16-bit vs. 32-bit float), polyphony (the YM2612's 6 channels vs. 64+ today), and the UI you wrap around it.

What this means for you: you already understand the parts that take years to learn. The parts you don't know yet — JUCE's API, real-time-safe C++, plugin packaging — are weeks of work, not months. Don't undersell what you already have.

---

## Table of contents

1. [Chapter 1 — Your YM2612 mental model is the modern mental model](#chapter-1--your-ym2612-mental-model-is-the-modern-mental-model)
2. [Chapter 2 — Web Audio quick wins inside Electron](#chapter-2--web-audio-quick-wins-inside-electron)
3. [Chapter 3 — JUCE installation and first plugin](#chapter-3--juce-installation-and-first-plugin)
4. [Chapter 4 — A Genesis-style FM synth in JUCE](#chapter-4--a-genesis-style-fm-synth-in-juce)
5. [Chapter 5 — VST3 vs AU vs AAX vs LV2](#chapter-5--vst3-vs-au-vs-aax-vs-lv2)
6. [Chapter 6 — MIDI, presets, parameters](#chapter-6--midi-presets-parameters)
7. [Chapter 7 — The Electron + JUCE hybrid pattern](#chapter-7--the-electron--juce-hybrid-pattern)
8. [Chapter 8 — 3D audio visualizers with R3F](#chapter-8--3d-audio-visualizers-with-r3f)
9. [Chapter 9 — Real-time-safe C++ (the part to be careful about)](#chapter-9--real-time-safe-c-the-part-to-be-careful-about)
10. [Chapter 10 — Shipping](#chapter-10--shipping)

---

## Chapter 1 — Your YM2612 mental model is the modern mental model

The YM2612 in the Genesis has 6 FM channels, each with 4 operators and 8 algorithm topologies. An operator is a sine wave generator with its own envelope. An algorithm is a graph: which operators modulate which other operators, and which operators are summed into the final output.

### The mapping table you'll keep coming back to

| YM2612 / SGDK | JUCE / DSP equivalent | Notes |
| --- | --- | --- |
| Operator | `juce::dsp::Oscillator<float>` or a hand-written sine | Generates a sine wave at a frequency |
| Algorithm | An audio graph you build manually in `processBlock` | The YM2612's 8 algorithms are 8 specific graphs out of infinite possible |
| Detune (DT1/DT2) | Add a small offset to the operator's frequency | Detuning operators is what gives FM patches their "thickness" |
| Multiplier (MUL) | Multiply the operator frequency by an integer | YM2612's MUL field is 0–15 mapping to multipliers `0.5..15` |
| Total Level (TL) | Output gain per operator, in dB | YM2612 uses 0..127 where 0 is loudest |
| Rate Scaling (RS) | How fast envelope rates scale with note pitch | Higher notes decay faster — gives natural feel |
| AR / D1R / D2R / RR / D1L | ADSR envelope (Attack / Decay / Sustain / Release) | YM2612 splits decay into two phases (D1 to sustain level D1L, then D2 to silence) |
| LFO (low-frequency oscillator) | A second oscillator at sub-audio rate modulating pitch and/or amplitude | Same on every synth ever made |
| Feedback (operator 1) | Output of an operator wraps back into its own phase input | Gives operator 1 sawtooth-like character at high values |
| DAC mode (Channel 6) | A sample playback voice | Replace 8-bit samples with 24-bit and the world opens up |
| PSG (SN76489) | A 4-channel square wave + noise generator | Closer to NES sound. Modern equivalent: a simple square oscillator |

If you internalize this table, the JUCE API in chapter 4 will feel like seeing the same place from a different angle.

### A concrete example — YM2612 Algorithm 4 in pseudocode

YM2612 Algorithm 4 is a popular pad/lead topology:

```
Operator 1 -> Operator 2 -> output
Operator 3 -> Operator 4 -> output
(Operators 2 and 4 are both summed to output)
```

In C this would look something like:

```c
float channel_output(YM2612_Channel* ch, float t) {
    float op1 = sin(2.0f * PI * ch->op[0].freq * t)  * ch->op[0].env(t);
    float op2 = sin(2.0f * PI * ch->op[1].freq * t + op1) * ch->op[1].env(t);
    float op3 = sin(2.0f * PI * ch->op[2].freq * t)  * ch->op[2].env(t);
    float op4 = sin(2.0f * PI * ch->op[3].freq * t + op3) * ch->op[3].env(t);
    return op2 + op4;
}
```

JUCE version (chapter 4 has the full class structure):

```cpp
float renderSample(double t) {
    float op1 = std::sin(twoPi * op[0].freq * t) * op[0].envelope.getNextSample();
    float op2 = std::sin(twoPi * op[1].freq * t + op1) * op[1].envelope.getNextSample();
    float op3 = std::sin(twoPi * op[2].freq * t) * op[2].envelope.getNextSample();
    float op4 = std::sin(twoPi * op[3].freq * t + op3) * op[3].envelope.getNextSample();
    return op2 + op4;
}
```

That's it. That's the move. The C is C++. The Genesis envelope is `juce::ADSR`. The phase modulation logic is identical.

---

## Chapter 2 — Web Audio quick wins inside Electron

Before you go full JUCE, prove the concepts in JS. Web Audio is built into Chromium (which is what Electron is), and a working FM voice in Web Audio is 30 lines of code. Once you've got it, you can wire it into your R3F scene immediately.

### Single FM voice (recap from main README, extended)

```ts
// src/audio/FMVoice.ts
export interface FMVoiceParams {
  carrierFreq: number;
  modulatorRatio: number;   // e.g. 2.0 for 2:1, 0.5 for 1:2
  modulationIndex: number;  // depth of modulation in Hz
  attack: number;
  decay: number;
  sustain: number;
  release: number;
}

export class FMVoice {
  private ctx: AudioContext;
  private carrier: OscillatorNode;
  private modulator: OscillatorNode;
  private modGain: GainNode;
  private env: GainNode;
  private out: GainNode;
  private started = false;
  private params: FMVoiceParams;

  constructor(ctx: AudioContext, destination: AudioNode, params: FMVoiceParams) {
    this.ctx = ctx;
    this.params = params;
    this.carrier   = ctx.createOscillator();
    this.modulator = ctx.createOscillator();
    this.modGain   = ctx.createGain();
    this.env       = ctx.createGain();
    this.out       = ctx.createGain();
    this.out.gain.value = 0.3;
    this.env.gain.value = 0;

    this.modulator.connect(this.modGain);
    this.modGain.connect(this.carrier.frequency);
    this.carrier.connect(this.env);
    this.env.connect(this.out);
    this.out.connect(destination);
  }

  noteOn(freq: number) {
    const now = this.ctx.currentTime;
    this.carrier.frequency.value = freq;
    this.modulator.frequency.value = freq * this.params.modulatorRatio;
    this.modGain.gain.value = this.params.modulationIndex;

    this.env.gain.cancelScheduledValues(now);
    this.env.gain.setValueAtTime(0, now);
    this.env.gain.linearRampToValueAtTime(1, now + this.params.attack);
    this.env.gain.linearRampToValueAtTime(this.params.sustain, now + this.params.attack + this.params.decay);

    if (!this.started) {
      this.carrier.start();
      this.modulator.start();
      this.started = true;
    }
  }

  noteOff() {
    const now = this.ctx.currentTime;
    this.env.gain.cancelScheduledValues(now);
    this.env.gain.setValueAtTime(this.env.gain.value, now);
    this.env.gain.linearRampToValueAtTime(0, now + this.params.release);
  }

  dispose(after = 1.0) {
    setTimeout(() => {
      try { this.carrier.stop(); this.modulator.stop(); } catch {}
      this.out.disconnect();
    }, after * 1000);
  }
}
```

### Polyphonic synth on top of it

```ts
// src/audio/PolyFMSynth.ts
import { FMVoice, FMVoiceParams } from './FMVoice';

export class PolyFMSynth {
  private ctx: AudioContext;
  private master: GainNode;
  private voices = new Map<number, FMVoice>();  // midi note -> voice
  private params: FMVoiceParams;

  constructor(params: FMVoiceParams) {
    this.ctx = new AudioContext();
    this.master = this.ctx.createGain();
    this.master.gain.value = 0.5;
    this.master.connect(this.ctx.destination);
    this.params = params;
  }

  noteOn(midi: number, velocity = 1.0) {
    if (this.voices.has(midi)) return;
    const freq = 440 * Math.pow(2, (midi - 69) / 12);
    const v = new FMVoice(this.ctx, this.master, this.params);
    v.noteOn(freq);
    this.voices.set(midi, v);
  }

  noteOff(midi: number) {
    const v = this.voices.get(midi);
    if (!v) return;
    v.noteOff();
    v.dispose(this.params.release + 0.1);
    this.voices.delete(midi);
  }

  getAnalyser() {
    const a = this.ctx.createAnalyser();
    a.fftSize = 1024;
    this.master.connect(a);
    return a;
  }

  setParams(p: Partial<FMVoiceParams>) {
    this.params = { ...this.params, ...p };
  }
}
```

### Driving it from a keyboard

```tsx
// src/components/SynthKeyboard.tsx
import { useEffect, useRef } from 'react';
import { PolyFMSynth } from '../audio/PolyFMSynth';

const KEY_TO_MIDI: Record<string, number> = {
  'a': 60, 'w': 61, 's': 62, 'e': 63, 'd': 64, 'f': 65,
  't': 66, 'g': 67, 'y': 68, 'h': 69, 'u': 70, 'j': 71, 'k': 72,
};

export function SynthKeyboard() {
  const synth = useRef<PolyFMSynth | null>(null);

  useEffect(() => {
    synth.current = new PolyFMSynth({
      carrierFreq: 440, modulatorRatio: 2.0, modulationIndex: 200,
      attack: 0.005, decay: 0.2, sustain: 0.6, release: 0.4,
    });
    return () => { /* synth cleanup */ };
  }, []);

  useEffect(() => {
    const down = (e: KeyboardEvent) => {
      const midi = KEY_TO_MIDI[e.key];
      if (midi && !e.repeat) synth.current?.noteOn(midi);
    };
    const up = (e: KeyboardEvent) => {
      const midi = KEY_TO_MIDI[e.key];
      if (midi) synth.current?.noteOff(midi);
    };
    window.addEventListener('keydown', down);
    window.addEventListener('keyup', up);
    return () => {
      window.removeEventListener('keydown', down);
      window.removeEventListener('keyup', up);
    };
  }, []);

  return null;
}
```

Drop `<SynthKeyboard />` into your `App.tsx`, press A/W/S/E... and you're playing FM synthesis. From there, hook the `getAnalyser()` output into the audio visualizer from chapter 8.

This whole stack is "production-ready" for an in-app synth. Where it stops being enough: you can't load it into Ableton as a VST. For that, JUCE.

---

## Chapter 3 — JUCE installation and first plugin

JUCE is a C++ framework. It is the industry standard for plugin development. Almost every commercial VST you've used is JUCE under the hood.

### Install (CMake path — recommended)

```bash
# 1. Clone JUCE
git clone https://github.com/juce-framework/JUCE.git ~/dev/JUCE

# 2. Make sure you have CMake (3.22+) and a C++ compiler
#    Windows: Visual Studio 2022 Community + CMake
#    macOS:   Xcode + Command Line Tools
#    Linux:   gcc/clang + cmake + the ALSA/X11 dev packages
```

### Project skeleton

```cmake
# CMakeLists.txt
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

target_compile_definitions(BarrySynth PRIVATE
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_VST3_CAN_REPLACE_VST2=0
)
```

```bash
cmake -B build -S .
cmake --build build --config Release
```

Output: a `.vst3` (Windows/Mac/Linux) and a `.component` (Mac only, AU) in `build/BarrySynth_artefacts/`. Drop them into:

- Windows: `C:\Program Files\Common Files\VST3\`
- macOS VST3: `~/Library/Audio/Plug-Ins/VST3/`
- macOS AU: `~/Library/Audio/Plug-Ins/Components/`
- Linux: `~/.vst3/`

DAW will pick them up on next scan.

### The minimum viable plugin source

`Source/PluginProcessor.h`:

```cpp
#pragma once
#include <juce_audio_processors/juce_audio_processors.h>

class BarrySynthProcessor : public juce::AudioProcessor {
public:
    BarrySynthProcessor();
    ~BarrySynthProcessor() override = default;

    void prepareToPlay(double sampleRate, int blockSize) override;
    void releaseResources() override {}
    void processBlock(juce::AudioBuffer<float>&, juce::MidiBuffer&) override;

    juce::AudioProcessorEditor* createEditor() override;
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
    double currentSampleRate = 44100.0;
    double phase = 0.0;
    double phaseIncrement = 0.0;
    bool   noteOn = false;
    float  velocity = 0.0f;
    juce::ADSR envelope;
};
```

`Source/PluginProcessor.cpp`:

```cpp
#include "PluginProcessor.h"
#include "PluginEditor.h"

BarrySynthProcessor::BarrySynthProcessor()
    : AudioProcessor(BusesProperties().withOutput("Output", juce::AudioChannelSet::stereo(), true)) {
    juce::ADSR::Parameters p;
    p.attack = 0.01f; p.decay = 0.2f; p.sustain = 0.6f; p.release = 0.4f;
    envelope.setParameters(p);
}

void BarrySynthProcessor::prepareToPlay(double sampleRate, int) {
    currentSampleRate = sampleRate;
    envelope.setSampleRate(sampleRate);
}

void BarrySynthProcessor::processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midi) {
    buffer.clear();

    for (const auto meta : midi) {
        const auto msg = meta.getMessage();
        if (msg.isNoteOn()) {
            const double freq = juce::MidiMessage::getMidiNoteInHertz(msg.getNoteNumber());
            phaseIncrement = (freq / currentSampleRate) * juce::MathConstants<double>::twoPi;
            velocity = msg.getFloatVelocity();
            noteOn = true;
            envelope.noteOn();
        } else if (msg.isNoteOff()) {
            noteOn = false;
            envelope.noteOff();
        }
    }

    auto* left  = buffer.getWritePointer(0);
    auto* right = buffer.getWritePointer(1);

    for (int n = 0; n < buffer.getNumSamples(); ++n) {
        const float envValue = envelope.getNextSample();
        const float sample = (float) std::sin(phase) * 0.3f * velocity * envValue;
        left[n] = sample;
        right[n] = sample;
        phase += phaseIncrement;
        if (phase > juce::MathConstants<double>::twoPi) phase -= juce::MathConstants<double>::twoPi;
    }
}

juce::AudioProcessorEditor* BarrySynthProcessor::createEditor() {
    return new BarrySynthEditor(*this);
}

juce::AudioProcessor* JUCE_CALLTYPE createPluginFilter() {
    return new BarrySynthProcessor();
}
```

`Source/PluginEditor.h` + `.cpp`: a minimal editor with one knob and a label. JUCE's docs show this in detail; for first build, an empty editor (`return new juce::GenericAudioProcessorEditor(*this);`) is fine.

Build it, load it in Reaper or Ableton on a MIDI track, play notes, hear sine waves. That's your first VST shipped.

---

## Chapter 4 — A Genesis-style FM synth in JUCE

Now we extend chapter 3's plugin into actual FM synthesis. We'll build a 4-operator voice (matching YM2612 channel structure) and a polyphonic voice manager.

### The operator

```cpp
// Source/FMOperator.h
#pragma once
#include <juce_audio_processors/juce_audio_processors.h>

class FMOperator {
public:
    void prepareToPlay(double sampleRate) {
        this->sampleRate = sampleRate;
        envelope.setSampleRate(sampleRate);
    }

    void setEnvelope(float attack, float decay, float sustain, float release) {
        juce::ADSR::Parameters p { attack, decay, sustain, release };
        envelope.setParameters(p);
    }

    void noteOn() { envelope.noteOn(); phase = 0.0; }
    void noteOff() { envelope.noteOff(); }

    void setFrequency(double freq) {
        phaseIncrement = (freq / sampleRate) * juce::MathConstants<double>::twoPi;
    }

    void setOutputLevel(float level) { this->outputLevel = level; }

    /** modInput is the phase modulation in radians (from another operator's output) */
    float renderNextSample(float modInput) {
        const float envValue = envelope.getNextSample();
        const float sample = std::sin(phase + modInput) * outputLevel * envValue;
        phase += phaseIncrement;
        if (phase > juce::MathConstants<double>::twoPi) phase -= juce::MathConstants<double>::twoPi;
        return sample;
    }

    bool isActive() const { return envelope.isActive(); }

private:
    double sampleRate = 44100.0;
    double phase = 0.0;
    double phaseIncrement = 0.0;
    float outputLevel = 1.0f;
    juce::ADSR envelope;
};
```

### The 4-operator voice

```cpp
// Source/FMVoice.h
#pragma once
#include "FMOperator.h"

class FMVoice {
public:
    enum class Algorithm {
        // YM2612 algorithms
        Alg0,  // Op1 -> Op2 -> Op3 -> Op4 -> out (serial)
        Alg4,  // (Op1 -> Op2) + (Op3 -> Op4) -> out (parallel pairs)
        Alg7,  // Op1 + Op2 + Op3 + Op4 -> out (all parallel)
    };

    void prepareToPlay(double sampleRate) {
        for (auto& op : ops) op.prepareToPlay(sampleRate);
    }

    void setAlgorithm(Algorithm alg) { algorithm = alg; }

    void noteOn(double freq, float velocity) {
        currentFreq = freq;
        currentVelocity = velocity;
        // YM2612 multipliers per operator — make these parameters in production
        ops[0].setFrequency(freq * mult[0]);
        ops[1].setFrequency(freq * mult[1]);
        ops[2].setFrequency(freq * mult[2]);
        ops[3].setFrequency(freq * mult[3]);
        for (auto& op : ops) op.noteOn();
        active = true;
    }

    void noteOff() {
        for (auto& op : ops) op.noteOff();
    }

    float renderNextSample() {
        if (!active) return 0.0f;
        switch (algorithm) {
            case Algorithm::Alg0: {
                const float op1 = ops[0].renderNextSample(0.0f);
                const float op2 = ops[1].renderNextSample(op1);
                const float op3 = ops[2].renderNextSample(op2);
                const float op4 = ops[3].renderNextSample(op3);
                return op4 * currentVelocity;
            }
            case Algorithm::Alg4: {
                const float op1 = ops[0].renderNextSample(0.0f);
                const float op2 = ops[1].renderNextSample(op1);
                const float op3 = ops[2].renderNextSample(0.0f);
                const float op4 = ops[3].renderNextSample(op3);
                return (op2 + op4) * 0.5f * currentVelocity;
            }
            case Algorithm::Alg7: {
                float sum = 0.0f;
                for (auto& op : ops) sum += op.renderNextSample(0.0f);
                return sum * 0.25f * currentVelocity;
            }
        }
        return 0.0f;
    }

    bool isActive() const {
        for (const auto& op : ops) if (op.isActive()) return true;
        active = false;
        return false;
    }

    int getNoteNumber() const { return currentNote; }
    void setNoteNumber(int n) { currentNote = n; }

private:
    std::array<FMOperator, 4> ops;
    Algorithm algorithm = Algorithm::Alg4;
    float mult[4] = { 1.0f, 1.0f, 2.0f, 1.0f };
    double currentFreq = 440.0;
    float currentVelocity = 1.0f;
    int currentNote = 60;
    mutable bool active = false;
};
```

### The polyphonic synth

```cpp
// Source/PolyFMSynth.h
#pragma once
#include "FMVoice.h"

class PolyFMSynth {
public:
    static constexpr int kVoiceCount = 8;

    void prepareToPlay(double sampleRate) {
        for (auto& v : voices) v.prepareToPlay(sampleRate);
    }

    void handleMidiMessage(const juce::MidiMessage& m) {
        if (m.isNoteOn())  startNote(m.getNoteNumber(), m.getFloatVelocity());
        if (m.isNoteOff()) stopNote(m.getNoteNumber());
    }

    void renderNextBlock(juce::AudioBuffer<float>& buffer) {
        auto* left  = buffer.getWritePointer(0);
        auto* right = buffer.getWritePointer(1);
        for (int n = 0; n < buffer.getNumSamples(); ++n) {
            float s = 0.0f;
            for (auto& v : voices) if (v.isActive()) s += v.renderNextSample();
            const float out = juce::jlimit(-1.0f, 1.0f, s * 0.3f);
            left[n] = out;
            right[n] = out;
        }
    }

private:
    void startNote(int note, float velocity) {
        // Pick an inactive voice, or steal the oldest
        FMVoice* target = nullptr;
        for (auto& v : voices) if (!v.isActive()) { target = &v; break; }
        if (!target) target = &voices[0];  // voice-stealing strategy: simplest
        target->setNoteNumber(note);
        target->noteOn(juce::MidiMessage::getMidiNoteInHertz(note), velocity);
    }

    void stopNote(int note) {
        for (auto& v : voices) if (v.isActive() && v.getNoteNumber() == note) v.noteOff();
    }

    std::array<FMVoice, kVoiceCount> voices;
};
```

Now plug `PolyFMSynth` into `BarrySynthProcessor`'s `processBlock`:

```cpp
void BarrySynthProcessor::processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midi) {
    buffer.clear();
    for (const auto meta : midi) synth.handleMidiMessage(meta.getMessage());
    synth.renderNextBlock(buffer);
}
```

You now have an 8-voice FM polysynth, structurally modeled after a YM2612 channel. Add per-operator UI controls (knobs for each operator's ratio, level, ADSR, plus an algorithm selector), and you have a real synth.

---

## Chapter 5 — VST3 vs AU vs AAX vs LV2

| Format | Owner | Where it loads | Should you target it? |
| --- | --- | --- | --- |
| **VST3** | Steinberg | Most DAWs (Ableton, Reaper, FL Studio, Cubase, Studio One, Bitwig...) | Yes. Default. |
| **AU** | Apple | Logic, GarageBand, Mainstage (macOS only) | Yes if you ever ship for Mac. JUCE makes this almost free. |
| **AAX** | Avid | Pro Tools | Only if you're paid to. Requires a free-but-gated SDK from Avid. |
| **LV2** | Open source | Ardour, Carla, plus most Linux DAWs | Nice-to-have. JUCE supports it via a build flag in modern versions. |
| **Standalone** | Just an EXE | Itself | Yes. The standalone build of a JUCE plugin runs without a DAW — great for "load the synth, jam, done" workflows. |

The `FORMATS` line in `CMakeLists.txt` from chapter 3 — `FORMATS VST3 AU Standalone` — is the entire list of changes needed to ship to all three. JUCE handles the format-specific shimming.

For your use case (sharing with friends, learning, eventually maybe ship something), `VST3 + Standalone` is the right call. Add `AU` if you have a Mac.

---

## Chapter 6 — MIDI, presets, parameters

The two pieces of JUCE that take the longest to internalize are **`AudioProcessorValueTreeState`** (for parameter management) and **`MidiBuffer`** (for MIDI handling). Both are worth learning properly because they're how your synth interoperates with DAWs.

### Parameter management

```cpp
// In your processor's header
juce::AudioProcessorValueTreeState parameters;

// Constructor body
BarrySynthProcessor::BarrySynthProcessor()
    : AudioProcessor(BusesProperties().withOutput("Output", juce::AudioChannelSet::stereo(), true)),
      parameters(*this, nullptr, "BarrySynth",
        {
            std::make_unique<juce::AudioParameterFloat>("op2Ratio", "Op2 Ratio",
                juce::NormalisableRange<float>(0.5f, 16.0f, 0.5f), 2.0f),
            std::make_unique<juce::AudioParameterFloat>("modIndex", "Mod Index",
                juce::NormalisableRange<float>(0.0f, 1.0f, 0.001f), 0.5f),
            std::make_unique<juce::AudioParameterFloat>("attack", "Attack",
                juce::NormalisableRange<float>(0.001f, 5.0f, 0.001f, 0.3f), 0.01f),
            std::make_unique<juce::AudioParameterFloat>("decay", "Decay",
                juce::NormalisableRange<float>(0.001f, 5.0f, 0.001f, 0.3f), 0.2f),
            std::make_unique<juce::AudioParameterFloat>("sustain", "Sustain",
                juce::NormalisableRange<float>(0.0f, 1.0f, 0.001f), 0.6f),
            std::make_unique<juce::AudioParameterFloat>("release", "Release",
                juce::NormalisableRange<float>(0.001f, 5.0f, 0.001f, 0.3f), 0.4f),
            std::make_unique<juce::AudioParameterChoice>("algorithm", "Algorithm",
                juce::StringArray{ "Alg0", "Alg4", "Alg7" }, 1),
        })
{
    // ... existing setup
}
```

In `processBlock`, read parameters cheaply via `parameters.getRawParameterValue("op2Ratio")` (returns an `std::atomic<float>*` — load it once per block, not per sample, for performance).

The `AudioProcessorValueTreeState` gives you for free:
- DAW-side parameter automation
- Preset save/load (`parameters.copyState()` / `replaceState()`)
- Generic UI generation (`juce::GenericAudioProcessorEditor`)
- Thread-safe parameter access

It's worth spending an evening reading the JUCE tutorial on this class. Once you grok it, everything else gets easier.

### MIDI gotchas

- MIDI messages in `processBlock`'s buffer have **sample offsets** — they say "at sample 12 within this 256-sample block, note 60 turned on." For sample-accurate handling, render samples 0–11 with the current state, *then* apply the note-on, then render samples 12–255. Most synths don't bother and just apply all MIDI events at the start of the block; the audible difference is rarely worth the complexity.
- Pitch bend, modwheel, aftertouch are all just CC messages. Read them via `msg.isPitchWheel()`, `msg.isControllerOfType(1)`, etc.
- Velocity is `msg.getFloatVelocity()` (0..1). Don't use the raw 7-bit `msg.getVelocity()` directly.

---

## Chapter 7 — The Electron + JUCE hybrid pattern

The pattern where this all comes together: a JUCE *audio engine* talking to an Electron *UI*.

### Why bother?

- JUCE's UI toolkit is fine but not amazing. React + R3F + GLSL shaders runs circles around it for visuals.
- Web Audio is fine for simple synths but breaks down under serious load (heavy DSP, low latency requirements, AudioWorklet thread is a pain to manage).
- Plugin formats *require* C++ (no DAW loads `.js` files as plugins; they load `.vst3`/`.au`).

The hybrid: **JUCE owns the audio thread, Electron owns the UI thread, they talk over IPC.**

### Three integration approaches, increasing in commitment

#### 7.1 — Standalone JUCE + Electron UI talking over a local socket

The most pragmatic. Two separate processes:

1. JUCE app, running headless or with a tiny window, listening on a localhost TCP/UDP socket.
2. Electron app providing the UI.

They exchange JSON or a small binary protocol: "set param X to value Y", "audio level is currently Z", "render this preset," etc.

```cpp
// JUCE side — pseudocode using juce::StreamingSocket
void startControlServer(int port = 7777) {
    serverSocket.createListener(port);
    serverThread = std::make_unique<std::thread>([this, port] {
        while (running) {
            auto* client = serverSocket.waitForNextConnection();
            if (client) handleClient(client);
        }
    });
}
```

```ts
// Electron side
import net from 'node:net';
const client = net.createConnection({ port: 7777 }, () => {
  client.write(JSON.stringify({ cmd: 'setParam', name: 'modIndex', value: 0.8 }) + '\n');
});
client.on('data', (chunk) => {
  const msg = JSON.parse(chunk.toString());
  // e.g. msg = { type: 'rmsLevel', value: 0.4 }
  updateVisualizer(msg.value);
});
```

Pros: clean process separation, both sides can be debugged independently, JUCE can crash without taking the UI down.
Cons: two processes to install/manage, IPC latency means the UI is ~5ms behind the audio (fine for visualizers, bad for direct manipulation that has to feel snappy).

#### 7.2 — JUCE plugin with WebView UI (JUCE 8+)

JUCE 8 ships `juce::WebBrowserComponent` with full Chromium hosting. Your `PluginEditor` can be:

```cpp
class BarrySynthEditor : public juce::AudioProcessorEditor {
public:
    BarrySynthEditor(BarrySynthProcessor& p)
      : AudioProcessorEditor(p), processor(p),
        webView(juce::WebBrowserComponent::Options{}
            .withResourceProvider([](const juce::String& url) {
                // Serve files bundled with the plugin
                return std::nullopt;
            })
            .withNativeIntegrationEnabled()) {

        addAndMakeVisible(webView);
        webView.goToURL("http://localhost:5173");  // or bundled file:// URL
        setSize(800, 600);
    }

    void resized() override { webView.setBounds(getLocalBounds()); }

private:
    BarrySynthProcessor& processor;
    juce::WebBrowserComponent webView;
};
```

The plugin is *one file* (`.vst3`) that loads in a DAW, and its UI is your React + Three.js app. Parameter sync happens via JUCE's WebView native bindings.

Search "JUCE 3DVerb" for a real example of this — a reverb plugin with a Three.js reactive visualizer in the UI.

Pros: single artifact, loads in a DAW like any plugin, gets your slick UI.
Cons: locked to JUCE 8+, debugging the embedded web UI is fiddly, bundle size is bigger (Chromium embedded).

#### 7.3 — Electron app that loads JUCE as a native node addon

Compile JUCE code into a `.node` module (via `node-addon-api`) and load it from Electron. Audio runs in C++ on a thread, exposed to JS via N-API.

Pros: single process, low IPC overhead.
Cons: gnarly to set up, JUCE's audio device management doesn't love living inside someone else's event loop, you're locked to one packaging.

**Recommendation**: start with 7.1 (sockets). It's the easiest to debug and the most flexible. Move to 7.2 when you actually want to ship a plugin.

---

## Chapter 8 — 3D audio visualizers with R3F

Once you have audio coming from somewhere (Web Audio, JUCE-over-socket, FMV soundtrack), driving 3D visuals is fun.

### Pull-based: AnalyserNode + useFrame

```tsx
import { useRef, useEffect, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

export function SpectrumBars({ analyser }: { analyser: AnalyserNode }) {
  const groupRef = useRef<THREE.Group>(null!);
  const BAR_COUNT = 64;
  const data = useMemo(() => new Uint8Array(analyser.frequencyBinCount), [analyser]);

  useFrame(() => {
    analyser.getByteFrequencyData(data);
    const children = groupRef.current.children;
    for (let i = 0; i < BAR_COUNT; i++) {
      const bin = Math.floor(i * (data.length / BAR_COUNT));
      const value = data[bin] / 255;
      const bar = children[i] as THREE.Mesh;
      bar.scale.y = 0.1 + value * 4;
      bar.position.y = bar.scale.y / 2;
    }
  });

  return (
    <group ref={groupRef}>
      {Array.from({ length: BAR_COUNT }, (_, i) => (
        <mesh key={i} position={[(i - BAR_COUNT / 2) * 0.3, 0, 0]}>
          <boxGeometry args={[0.2, 1, 0.2]} />
          <meshStandardMaterial
            color={new THREE.Color().setHSL(i / BAR_COUNT, 1, 0.5)}
            emissive={new THREE.Color().setHSL(i / BAR_COUNT, 1, 0.5)}
            emissiveIntensity={0.5}
          />
        </mesh>
      ))}
    </group>
  );
}
```

### Push-based: audio thread sends levels over IPC

If your audio engine is JUCE-over-socket (chapter 7.1), the UI side just receives messages:

```ts
client.on('data', (chunk) => {
  const msgs = chunk.toString().split('\n').filter(Boolean).map(s => JSON.parse(s));
  for (const msg of msgs) {
    if (msg.type === 'spectrum') {
      // msg.bins = Float32Array-ish array of 64 values 0..1
      visualizerRef.current?.setSpectrum(msg.bins);
    }
  }
});
```

For the visualizer, expose an imperative `setSpectrum` via `useImperativeHandle` and store the array in a ref. Then in `useFrame`, read from the ref and update geometry. This avoids per-message React re-renders, which would kill performance at 30Hz+.

### Visualizers worth building

- **Channel-per-channel YM2612 view** — 6 spheres in a circle, each scaled by its channel's RMS, colored by which algorithm it's running. Looks like a Genesis composing for itself.
- **Algorithm topology viz** — render the 4 operators as connected nodes, with line thickness proportional to modulation depth. Switch algorithms and watch the graph rewire.
- **FFT terrain** — a `PlaneGeometry` whose vertex heights are driven by the FFT bins. Scrolls forward over time creating a Doppler-effect mountain range of the spectrum.
- **Live waveform tube** — a `TubeGeometry` whose path is the audio waveform extruded along Z. Wraps around itself like an oscilloscope spiral.

---

## Chapter 9 — Real-time-safe C++ (the part to be careful about)

This is the chapter where your SGDK background is *the* asset, not just nice to have. The audio thread in JUCE has hard real-time deadlines: if a single `processBlock` call takes longer than the buffer duration, you get an audible glitch (xrun, drop-out, pop). Same constraint you knew on the Genesis where missing a vblank means tearing.

### Rules for code that runs in `processBlock`

| Don't | Why |
| --- | --- |
| `new` / `delete` / `malloc` / `free` | Allocator can take milliseconds in pathological cases; you have microseconds |
| Use `std::mutex` (block on it) | Priority inversion deadlocks the audio thread; horrible bug to find |
| Throw exceptions | Stack unwinding is allocation-y and unpredictable |
| Call `std::vector::push_back` / `.resize()` | Reallocates internally |
| Call `printf` / `std::cout` / `juce::Logger` | I/O — unbounded latency |
| Take locks held by GUI thread | See mutex point |
| Use `std::map` / `std::unordered_map` (read OK, write — depends) | Many ops allocate; depends on the implementation |
| Do file I/O | Sync I/O blocks; even async via OS is unpredictable |

### Things you *can* do

- Read `std::atomic` parameters
- Use lock-free FIFOs (`juce::AbstractFifo`, `boost::lockfree::queue`)
- Pre-allocated `juce::AudioBuffer<float>` (reuse, don't recreate)
- `std::array` with compile-time sizes
- Simple math, sin/cos (modern compilers have these fast)
- SIMD intrinsics (JUCE has `juce::dsp::SIMDRegister<float>`)

### Communicating from GUI thread to audio thread

For setting a parameter:

```cpp
// GUI thread, in a slider's change callback:
parameters.getParameter("modIndex")->setValueNotifyingHost(newValue);
// Internally: this writes to an atomic, audio thread reads it next block.
```

For sending arbitrary data (e.g., "load this preset"):

```cpp
// Use juce::AbstractFifo for a lock-free queue
class PresetLoaderFifo {
    juce::AbstractFifo abstractFifo { 32 };
    std::array<Preset, 32> presets;

public:
    bool pushPreset(const Preset& p) {
        int start1, size1, start2, size2;
        abstractFifo.prepareToWrite(1, start1, size1, start2, size2);
        if (size1 + size2 == 0) return false;
        presets[start1] = p;  // No allocation if Preset is POD-ish
        abstractFifo.finishedWrite(size1 + size2);
        return true;
    }

    bool popPreset(Preset& out) {
        int start1, size1, start2, size2;
        abstractFifo.prepareToRead(1, start1, size1, start2, size2);
        if (size1 + size2 == 0) return false;
        out = presets[start1];
        abstractFifo.finishedRead(size1 + size2);
        return true;
    }
};
```

Audio thread polls `popPreset` once per block. Zero locks, zero allocation, single-producer single-consumer safe.

This is exactly the kind of constraint you've been working under for years on Genesis. Real-time audio is bare-metal coding with a `float` SDK on top.

---

## Chapter 10 — Shipping

### Standalone synth EXE

JUCE's `Standalone` target produces an executable that runs without a DAW. It opens an audio device, hosts the plugin, exposes parameters. Use it for:
- Live performance (load synth, plug in MIDI controller, play)
- Distribution to non-DAW-using friends ("here's a fun synth, double-click")
- Testing during development (no DAW restart needed between code changes)

### Plugin distribution

Until you sell things, just zip up the `.vst3` (and `.component` for Mac) and share. People install by copying to:
- Windows: `C:\Program Files\Common Files\VST3`
- macOS: `~/Library/Audio/Plug-Ins/VST3` (and `.component` to `Components/`)
- Linux: `~/.vst3`

For real distribution, you'll want:
- Code signing (~$80/yr for a basic cert, mandatory on macOS for non-experts to even open your installer)
- An installer (Inno Setup on Windows, `pkgbuild` on Mac)
- A landing page with screenshots and demo audio
- Optionally Gumroad or your own site for sales

### Combining with the Electron starter

The dream end state: a single download containing:
- `BarrySynth.exe` — the Electron app (UI + R3F visualizer + FMV game + ROM gallery)
- `BarrySynth-Audio.exe` — the JUCE standalone audio engine, launched as a child process by the Electron app
- A folder of presets and sample audio

User double-clicks `BarrySynth.exe`, both processes start, they connect over a localhost socket, your full 3D-FMV-with-FM-synth-soundtrack experience runs as one perceived application.

This is roughly how Native Instruments' Komplete Kontrol works internally, and how Output's Arcade plays nice with DAW automation. You'd be building in the same architecture as the pros, with your specific C/SGDK-via-Genesis flavor on top.

---

## Where to start tomorrow

If chapter 4 (the full FM synth) feels like a lot, here's the minimum path that gets you to "I shipped my first VST" in roughly a weekend:

1. **Friday evening**: Install JUCE + CMake. Build the chapter 3 hello-world plugin. Load it in Reaper or a free DAW. Play a sine wave.
2. **Saturday**: Replace the sine wave with the chapter 4 4-operator voice. One algorithm. One voice (monophonic — no polyphony yet). Hear FM synthesis come out of your code.
3. **Sunday**: Add `AudioProcessorValueTreeState` (chapter 6) and a generic editor. Now you can tweak the parameters in the DAW. Save a preset that sounds like a Genesis pad. Send it to a friend.

By Sunday night you've shipped a VST3 plugin. Polyphony, multiple algorithms, the React UI, all of that is incremental from here.

You've got the foundation already. JUCE is the SGDK of the audio plugin world — same flavor of "low-level enough to feel honest," with much friendlier tooling. Welcome.
