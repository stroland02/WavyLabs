# Wavy Labs — Technology Research & Decision Log

*Research conducted: March 1, 2026*

---

## 1. DAW Foundation — Decision Log

### Options Evaluated

| Project | Stars | Language | License | Type | Verdict |
|---------|-------|----------|---------|------|---------|
| LMMS | 9.5k | C++ / Qt | GPL-2+ | Full DAW | Rejected — VST2 only, 44.1kHz lock, consumer focus |
| Ardour | 4.8k | C++ / GTK | GPL-2+ | Full DAW | Rejected — ~1M LOC, aging GTK UI, hard to extend |
| Zrythm | — | C → C++20 | AGPL-3 | Full DAW | Rejected — v2 mid-rewrite, unstable to fork now |
| BespokeSynth | 4.5k | C++ | GPL-3 | Modular Synth | Rejected — not a DAW, no timeline/recording |
| openDAW | 1.3k | TypeScript | AGPL-3 | Web DAW | Rejected — self-described "not production ready" |
| **Tracktion Engine** | 1.4k | C++20/JUCE | GPLv3/Comm | **Library** | **SELECTED** |

### Why Tracktion Engine Won

Tracktion Engine is a library, not a complete DAW. You supply 100% of the UI and UX. This is actually the key advantage — Wavy Labs gets to design an AI-native interface from the ground up rather than retrofitting AI features into an existing DAW's assumptions.

- 115,000+ lines of production-tested audio code
- Clean `Engine → Edit → Track → Clip` API
- JUCE framework (same as UI layer — zero integration friction)
- Actively maintained (last commit: Feb 18, 2026)
- Dual GPLv3/Commercial license supports future SaaS model
- Tracktion Waveform Pro is built on it — real commercial validation
- LMN-3-DAW exists as a complete open-source reference implementation

---

## 2. UI Framework — Decision Log

### Options Evaluated

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| JUCE UI (native) | Same framework, zero friction, LMN-3-DAW reference | Dated default look, need custom LookAndFeel | **SELECTED** |
| Qt6 / QML | Modern GPU rendering, fluid animations | Two-framework complexity, separate build chain | Phase 4 option |
| Electron / Tauri | Modern web UI, huge ecosystem | 30ms+ audio latency, not suitable for pro DAW | Rejected |

**Decision: JUCE 8 UI now, Qt6 migration optional in Phase 4**

JUCE 8 (current release, latest 8.0.11) introduces several major improvements directly relevant to a DAW:

| JUCE 8 Feature | Relevance to Wavy Labs |
|----------------|------------------------|
| Direct2D renderer (Windows) | Blazing fast GPU-accelerated timeline and waveform rendering |
| MIDI 2.0 / UMP (8.0.11) | Native MIDI 2.0 device support — aligns with CLAP's MIDI 2.0 advantage |
| Windows Arm support | Runs natively on Surface Pro X / Snapdragon laptops |
| Improved text rendering | Sharper UI labels and parameter readouts at all DPI |

JUCE 8's `LookAndFeel` system is powerful enough to build a fully custom, modern-looking dark theme. With custom `Component::paint()` implementations, the UI can look as polished as any commercial DAW.

Zrythm v2 is migrating to JUCE + Qt6 — validates both choices but we avoid the complexity of two frameworks at launch.

### JUCE UI Built-in Components Available

| Component | Use in Wavy Labs |
|-----------|-----------------|
| `AudioThumbnail` | Waveform display in clips |
| `LookAndFeel_V4` | Global dark theme |
| `Viewport` + `Component` | Scrollable timeline |
| `ListBox` | Track list, plugin browser |
| `Slider` | Faders, knobs, parameters |
| `OpenGLContext` | GPU-accelerated rendering |

### Custom Components to Build

- Timeline / Arrangement View
- Clip component with waveform
- Piano Roll (MIDI editor grid)
- Mixer channel strip
- Level meter (real-time RMS/peak)
- Playhead
- AI Panel (all 4 AI feature views)

### Key Resources

- awesome-juce: `github.com/sudara/awesome-juce` — check before building any component
- AudioThumbnail tutorial: `juce.com/tutorials/tutorial_audio_thumbnail`
- LookAndFeel tutorial: `juce.com/tutorials/tutorial_look_and_feel_customisation`
- LMN-3-DAW: `github.com/FundamentalFrequency/LMN-3-DAW` — primary reference

---

## 3. Plugin Standard — Decision Log

| Standard | License | Multi-thread | MIDI 2.0 | Adoption | Verdict |
|----------|---------|-------------|----------|----------|---------|
| VST3 | Proprietary SDK (free) | Limited | No | Universal | Secondary |
| CLAP | MIT | Full | Yes | Growing — predicted to overtake VST3 | **Primary** |
| AU | Apple | Limited | No | macOS only | macOS compat |
| LV2 | ISC | Good | Partial | Linux standard | Linux compat |

**Decision: CLAP as primary plugin format, VST3 for compatibility**

CLAP (Clever Audio Plugin) is an open standard with:
- MIT license (no SDK fees ever)
- True multi-threading support (critical for AI plugin chains)
- Full MIDI 2.0 + per-note automation (polyphonic expression)
- Adopted by Bitwig Studio, REAPER, and growing
- KVR Audio industry analysis (2026): CLAP predicted to overtake VST3 as the primary standard within the next few years
- **JUCE 8 alignment**: JUCE 8's native MIDI 2.0 / UMP support complements CLAP's MIDI 2.0 capabilities — both frameworks now speak the same modern MIDI protocol

---

## 4. AI Models — Full Evaluation

### 4.1 Generative Music

| Model | License | Quality | Speed | Selected |
|-------|---------|---------|-------|---------|
| **DiffRhythm** | Apache 2.0 | Excellent | ~10s/5min song | **Primary** |
| AudioCraft/MusicGen | MIT code / CC-BY-NC weights | Excellent | 10–30s/30s | Secondary |
| Google Magenta | Apache 2.0 | Good | Real-time capable | Experimental |
| Bark (Suno) | MIT | Good | 5–10s/clip | Effects only |

**DiffRhythm selected as primary** — Apache 2.0 is fully permissive for commercial use. MusicGen weights have CC-BY-NC restriction limiting commercial deployment.

Architecture similarity to Suno: latent diffusion model, generates full songs with vocals from text prompts.

### 4.2 Stem Separation

| Model | Language | License | SDR Quality | Speed | Selected |
|-------|----------|---------|-------------|-------|---------|
| **demucs.cpp** | **C++17** | **MIT** | 9.20 dB (same weights) | Fast, CPU+GPU | **Tier 1 Primary** |
| Demucs v4 ONNX | C++ via ONNX | MIT | 9.20 dB | GPU fast | **Tier 1 Alternative** |
| Demucs v4 (Python) | Python | MIT | 9.20 dB (SOTA) | Real-time GPU | Tier 2 Fallback |
| Spleeter | Python | MIT | 6.5 dB | 100x real-time | Tier 2 Fast fallback |
| Open-Unmix | Python | MIT | 5.5–6.0 dB | Real-time | Lightweight option |

**demucs.cpp selected as primary** — identical model quality to Python Demucs v4 but runs in C++17 with GGML. Zero Python dependency. Ships inside the app binary. Demucs v4 ONNX (from Mixxx GSoC 2025) is a viable alternative via ONNX Runtime C++ API.

### 4.3 Auto-Mixing & Mastering

| Tool | License | Function | Selected |
|------|---------|----------|---------|
| **Matchering 2.0** | GPLv3 | Reference-based mastering | **Primary** |
| **Essentia (MTG)** | AGPL v3 | Audio analysis for mixing decisions | **Analysis layer** |
| Neural Amp Modeler | Open source | Guitar amp/pedal modeling | Plugin add-on |

Note: No truly open-source equivalent to iZotope Ozone exists yet. Matchering is the best available option for reference mastering.

### 4.4 Voice / Instrument Cloning

| Model | License | Quality | Training Required | Tier | Selected |
|-------|---------|---------|------------------|------|---------|
| **Kokoro-82M** | Apache 2.0 | Near-ElevenLabs | None (zero-shot) | **Tier 1 candidate** | **Primary TTS** |
| **RVC v3** | MIT | Excellent | 5–10 min audio | Tier 2 | **Primary cloning** |
| **StyleTTS2** | MIT | ElevenLabs-level | 30+ min audio | Tier 2 | High quality option |
| **Qwen3-TTS** | Open (Alibaba 2025) | Near SOTA | None (zero-shot) | Tier 2 | Zero-shot synthesis |
| **Chatterbox** | Open source | Good | None | Tier 2 | No-setup cloning |
| Coqui XTTS-v2 | Non-commercial | Excellent | 6 seconds | — | **Rejected** — license |

**Kokoro-82M elevated to Tier 1 candidate (2026 update):**
- Apache 2.0 license (fully permissive — can ship inside the app binary)
- Only 82M parameters — small enough to ship bundled with the app
- Runs on CPU with no GPU required
- ONNX-exportable: can be loaded via ONNX Runtime C++ API (same path as demucs.cpp)
- This means voice preview / TTS can work for every user with zero setup, same as stem separation
- Inworld AI ranked it as best open-source option for real-time applications (2026)

**Qwen3-TTS (2025, Alibaba):** Uses a discrete multi-codebook language model that treats audio generation like language modeling — goes directly from text to speech tokens. No fine-tuning required for voice matching. Strong alternative to Kokoro-82M for Tier 2.

**Chatterbox:** Community-validated open source voice cloning that works reliably. No commercial restrictions. Good for users who want quick voice matching without training a full RVC model.

**Avoid Coqui XTTS-v2**: Coqui AI shut down December 2025, license is non-commercial only.

---

## 5. AI Backend Language — Decision Log (Updated v1.1)

### Original Plan: Python-only backend
Python was chosen because all AI libraries (audiocraft, demucs, matchering, rvc) ship as Python packages. Fastest path to working AI features.

### Revised Plan: Tiered C++ / Python

Research discovered that Python is NOT required for every feature:

| Discovery | Implication |
|-----------|------------|
| **demucs.cpp** (MIT, github.com/sevagh/demucs.cpp) | C++17 port of Demucs v3+v4 using GGML. No Python. Stem separation works out of the box for every user. |
| **Demucs v4 → ONNX** (Mixxx GSoC 2025) | Demucs v4 converted to ONNX format. Runnable via ONNX Runtime C++ API. Confirmed working. |
| **ONNX Runtime C++ API** (onnxruntime.ai) | Full C++ API confirmed working in JUCE audio plugins (JUCE forum thread evidence). |
| **Neutone SDK** (arxiv Aug 2025) | Framework for deploying PyTorch models as CLAP/VST3 plugins — AI becomes a loadable plugin. |

### Final Decision: Three Tiers

```
Tier 1 — C++ (embedded, zero user setup)
  demucs.cpp + ONNX Runtime
  Covers: stem separation, small real-time models
  Ships: inside the app binary

Tier 2 — Python sidecar (installs on first Tier 2 feature use)
  DiffRhythm, MusicGen, Matchering, RVC v3, Qwen3-TTS, Chatterbox, Ollama
  Covers: music generation, mastering, voice cloning, zero-shot TTS
  Ships: auto-downloaded, runs as localhost gRPC server

Tier 3 — Cloud (optional SaaS)
  Covers: heavy models, users without GPU
  Ships: managed cloud endpoints
```

**Key UX benefit of Tier 1 (updated 2026):** Two AI features now ship in the app with zero setup:
1. **Stem separation** via demucs.cpp — GGML C++17, works on every machine
2. **Voice preview / TTS** via Kokoro-82M — Apache 2.0, 82M params, ONNX-capable, CPU-only

This means users get two compelling AI features on first launch before ever touching Python.

**Python is still used** for everything that doesn't yet have a mature C++ equivalent (transformer-scale generation models, full voice cloning training). This will shift over time as ONNX exports mature.

## 7. Communication Layer — Decision Log

### gRPC vs REST

| Factor | gRPC | REST/JSON |
|--------|------|-----------|
| Payload size | 3–7x smaller | Baseline |
| Parse speed | 5–10x faster | Baseline |
| Streaming | Native (server-streaming) | Polling workaround |
| Typing | Strongly typed .proto | Loosely typed JSON |
| C++ support | Native client libs | Manual HTTP |

**gRPC selected** — The streaming support is essential for showing real-time progress of AI generation tasks (e.g., "Generating stems... 47%"). Protocol Buffers also enforce a typed contract between C++ and Python.

---

## 8. Quality Benchmarks — Reference Product Analysis

### Suno AI
- Architecture (confirmed): Transformer autoregressive model + diffusion upscaler + neural singing synthesizer
- Uses audio codec (EnCodec-style) to tokenize audio before generation
- Similar to GPT for text, but generates audio tokens
- **Open source equivalent**: DiffRhythm (Apache 2.0) matches this approach

### ElevenLabs
- Voice data collection → speaker embedding extraction → neural TTS conditioned on embedding
- Professional Clone requires 30+ min audio; Instant Clone works from ~1 min
- **Open source equivalents**: RVC v3 (instant clone quality), StyleTTS2 (professional quality)

### Songboy
- Could not be found in search results as of March 2026
- **Action required**: User to share link/details for accurate benchmarking

---

## 9. Custom Model Training — Research Summary

### Base Model for Fine-Tuning: MusicGen

Fine-tuning is achievable and well-documented:
- Guide: activeloop.ai — "Fine-Tuning MusicGen for Text-Based Music Generation"
- Repo: `github.com/ylacombe/musicgen-dreamboothing`
- MuseControlLite (ICML 2025): fully open-source controllable training framework

### Best Training Datasets

| Dataset | Size | License | Best for |
|---------|------|---------|----------|
| Free Music Archive | 106,574 tracks | CC | Genre diversity |
| MTG-Jamendo | 55,000 tracks | CC BY | Full songs |
| MusicNet | 330 recordings | CC0 | Classical |
| Slakh2100 | 2,100 multi-track | CC BY 4.0 | Stems/separation training |
| NSynth (Google) | 300,000 notes | CC BY 4.0 | Instrument modeling |

### Compute to Start

MusicGen-small fine-tune: 1x A100, 8–24 hours, ~$50 cloud cost. Highly achievable.

---

## 10. Monetization Research

### Successful Open Source Audio Monetization Models

| Product | Model | Outcome |
|---------|-------|---------|
| Audacity | Free + paid enterprise | 114M+ downloads |
| Blender | Free + studio fund + merch | Multi-million dollar foundation |
| BandLab | Free core + SaaS AI features | Millions of users |

### Recommended Model for Wavy Labs

1. **AGPL v3 core** — free, open, copyleft ensures improvements stay open
2. **SaaS tier** ($9–15/month) — cloud GPU, sync, advanced AI
3. **Model marketplace** — community AI models, revenue share
4. **Avoid**: forced telemetry, late-stage license changes (both destroyed Audacity trust in 2023)

---

## 11. Architecture Patterns from Industry

### iZotope Three-Stage Pattern (Replicate This)
1. User preference stage: genre, target, reference
2. ML analysis stage: classify audio characteristics
3. Intelligent DSP stage: apply processing with ML-informed parameters

### Accusonus "Smart Feed" Pattern
- Don't feed entire audio to AI model
- Extract only relevant features: rhythm, pitch, spectral data
- Reduces inference load 60%, faster processing

### BandLab Accessibility Pattern
- Core DAW free and always open
- Premium: cloud GPU access, advanced models
- Marketplace for community tools
