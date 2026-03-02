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

**Decision: JUCE UI now, Qt6 migration optional in Phase 4**

JUCE's `LookAndFeel` system is powerful enough to build a fully custom, modern-looking dark theme. With custom `Component::paint()` implementations, the UI can look as polished as any commercial DAW.

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
| CLAP | MIT | Full | Yes | Growing (15 DAWs) | **Primary** |
| AU | Apple | Limited | No | macOS only | macOS compat |
| LV2 | ISC | Good | Partial | Linux standard | Linux compat |

**Decision: CLAP as primary plugin format, VST3 for compatibility**

CLAP (Clever Audio Plugin) is an open standard with:
- MIT license (no SDK fees ever)
- True multi-threading support (critical for AI plugin chains)
- Full MIDI 2.0 + per-note automation (polyphonic expression)
- Adopted by Bitwig Studio, REAPER, and growing

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

| Model | License | SDR Quality | Speed | Selected |
|-------|---------|-------------|-------|---------|
| **Demucs v4** | MIT | 9.20 dB (SOTA) | Real-time GPU | **Primary** |
| Spleeter | MIT | 6.5 dB | 100x real-time | Fallback |
| Open-Unmix | MIT | 5.5–6.0 dB | Real-time | Lightweight option |

**Demucs v4 selected** — state-of-the-art quality, MIT licensed, actively maintained by Meta. Spleeter as a fast/light fallback for weaker hardware.

### 4.3 Auto-Mixing & Mastering

| Tool | License | Function | Selected |
|------|---------|----------|---------|
| **Matchering 2.0** | GPLv3 | Reference-based mastering | **Primary** |
| **Essentia (MTG)** | AGPL v3 | Audio analysis for mixing decisions | **Analysis layer** |
| Neural Amp Modeler | Open source | Guitar amp/pedal modeling | Plugin add-on |

Note: No truly open-source equivalent to iZotope Ozone exists yet. Matchering is the best available option for reference mastering.

### 4.4 Voice / Instrument Cloning

| Model | License | Quality | Training Data | Selected |
|-------|---------|---------|--------------|---------|
| **RVC v3** | MIT | Excellent | 5–10 min audio | **Primary** |
| **StyleTTS2** | MIT | ElevenLabs-level | 30+ min audio | High quality option |
| Kokoro-82M | Apache 2.0 | Near-ElevenLabs | Few seconds | Fast inference |
| Coqui XTTS-v2 | Non-commercial | Excellent | 6 seconds | **Rejected** — license |

**Avoid Coqui XTTS-v2**: Coqui AI shut down December 2025, license is non-commercial only.

---

## 5. Communication Layer — Decision Log

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

## 6. Quality Benchmarks — Reference Product Analysis

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

## 7. Custom Model Training — Research Summary

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

## 8. Monetization Research

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

## 9. Architecture Patterns from Industry

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
