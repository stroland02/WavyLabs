# Wavy Labs — Technology Research & Decision Log

**Version:** 2.0  
**Research conducted:** March 1–2, 2026  
**Strategic decision:** Open-source first — every core feature runs locally with zero cloud dependency.

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

Tracktion Engine is a library, not a complete DAW. You supply 100% of the UI and UX. This is the key advantage — Wavy Labs designs an AI-native interface from the ground up rather than retrofitting AI features into an existing DAW's assumptions.

- 115,000+ lines of production-tested audio code
- Clean `Engine → Edit → Track → Clip` API
- JUCE framework (same as UI layer — zero integration friction)
- Actively maintained (last commit: Feb 18, 2026)
- Dual GPLv3/Commercial license supports future SaaS model
- Tracktion Waveform Pro is built on it — real commercial validation
- LMN-3-DAW exists as a complete open-source reference implementation

---

## 2. UI Framework — Decision Log

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| JUCE UI (native) | Same framework, zero friction, LMN-3-DAW reference | Dated default look, need custom LookAndFeel | **SELECTED** |
| Qt6 / QML | Modern GPU rendering, fluid animations | Two-framework complexity, separate build chain | Phase 4 option |
| Electron / Tauri | Modern web UI, huge ecosystem | 30ms+ audio latency, not suitable for pro DAW | Rejected |

**Decision: JUCE 8 UI now, Qt6 migration optional in Phase 4**

JUCE 8 (current release, latest 8.0.11) key features for Wavy Labs:

| JUCE 8 Feature | Relevance |
|----------------|-----------|
| Direct2D renderer (Windows) | GPU-accelerated timeline and waveform rendering |
| MIDI 2.0 / UMP (8.0.11) | Native MIDI 2.0 device support — aligns with CLAP |
| Windows Arm support | Runs natively on Surface Pro X / Snapdragon laptops |
| Improved text rendering | Sharper UI labels at all DPI |

**Custom components to build:** Timeline/Arrangement View, Clip component, Piano Roll, Mixer channel strip, Level meter, Playhead, AI Panel, **Code View** (new — Code-to-Music editor tab).

---

## 3. Plugin Standard — Decision Log

| Standard | License | Multi-thread | MIDI 2.0 | Verdict |
|----------|---------|-------------|----------|---------|
| CLAP | MIT | Full | Yes | **Primary** |
| VST3 | Proprietary SDK (free) | Limited | No | Secondary |
| AU | Apple | Limited | No | macOS compat |
| LV2 | ISC | Good | Partial | Linux compat |

**Decision: CLAP primary, VST3 for compatibility.** CLAP is MIT licensed, supports true multi-threading and full MIDI 2.0, adopted by Bitwig and REAPER. JUCE 8's native MIDI 2.0/UMP support further aligns with CLAP.

---

## 4. AI Models — Full Evaluation (Updated v2.0)

### 4.1 Music Generation

| Model | License | Quality | Speed | VRAM | Status |
|-------|---------|---------|-------|------|--------|
| **ACE-Step 1.5** | **MIT** | SongEval 8.09 (above Suno v5) | <2s A100, <10s RTX 3090 | <4 GB | **PRIMARY — replaces DiffRhythm** |
| **YuE** | **Apache 2.0** | Matches some proprietary systems | Moderate | 8 GB (quantized) | **VOCAL GENERATION** |
| **Bark (Suno)** | **MIT** | Good for SFX/ambient | 5–10s/clip | 8–12 GB | **SFX/AMBIENT — NEW** |
| DiffRhythm | Apache 2.0 | Excellent | ~10s/5min song | Moderate | Demoted to fallback |
| MusicGen | MIT code / CC-BY-NC weights | Excellent | 10–30s/30s | High | Rejected — NC weights |
| HeartMuLa 3B | Apache 2.0 | Good lyrics control | Moderate | Moderate | Tracking only |
| MagentaRT | Apache 2.0 (+terms) | Good instrumental | Real-time | Moderate | Phase 3+ jam mode |

**Why ACE-Step 1.5 replaced DiffRhythm:**
- Outperforms most commercial music models on SongEval benchmarks (8.09, above Suno v5)
- Hybrid LM planner + Diffusion Transformer architecture
- Built-in audio repainting, cover generation, continuation, vocal-to-BGM
- <4 GB VRAM — runs on consumer hardware
- MIT license with training on licensed/royalty-free data
- LoRA fine-tuning from just a few reference songs
- 50+ languages for lyrics/prompts

**Why Bark was added:**
- Suno's only open-source model — MIT licensed, 38.9k GitHub stars
- Fills the gap between TTS and music generation: laughter, sighs, ambient, simple musical elements
- GPT-style three-stage transformer (Text → Semantic → Coarse acoustic → Fine acoustic)
- 100+ speaker presets, paralinguistic tags (`[laughs]`, `[sighs]`, `[music]`)
- Uniquely valuable for ad-lib generation and ambient soundscapes in DAW context

### 4.2 Stem Separation

| Model | Language | License | SDR | Status |
|-------|----------|---------|-----|--------|
| **demucs.cpp** | **C++17** | **MIT** | 9.20 dB | **Tier 1 PRIMARY — no change** |
| Demucs v4 ONNX | C++ via ONNX | MIT | 9.20 dB | Tier 1 alternative |
| Demucs v4 (Python) | Python | MIT | 9.20 dB | Tier 2 fallback |

No open-source model has dethroned Demucs as of March 2026. MDX-Net/MDX'23 can outperform on specific stems (e.g., vocals) — consider bundling UVR5's MDX-Net models as an advanced user option. Dual-path Mamba networks show promise in benchmarks but lack production-ready implementations.

### 4.3 Voice / TTS

| Model | License | Params | Quality | Training Required | Status |
|-------|---------|--------|---------|------------------|--------|
| **NeuTTS Air** | **Apache 2.0** | 748M | Excellent | None (3-sec clone) | **Tier 1 PRIMARY — replaces Kokoro-82M** |
| **Chatterbox Turbo** | **MIT** | 350M | 63.75% preferred over ElevenLabs | None | **Tier 2 zero-shot** |
| **Chatterbox Multilingual** | **MIT** | 550M | Good | None | **Tier 2 multilingual (23 langs)** |
| **RVC v2** | **MIT** | — | Excellent | 5–10 min audio | **Tier 2 trained cloning** |
| **Qwen3-TTS VoiceDesign** | Open | — | Near SOTA | None | **Tier 2 voice-from-description** |
| Kokoro-82M | Apache 2.0 | 82M | Good | None | Demoted — NeuTTS Air is better |
| RVC v3 | MIT | — | Unknown | Unknown | **NOT RELEASED** as of March 2026 |
| OpenAudio S1 | CC-BY-NC-SA-4.0 | 4B | #1 TTS-Arena2 | None | Rejected — non-commercial weights |
| StyleTTS2 | MIT | — | ElevenLabs-level | 30+ min audio | Available but complex |
| Coqui XTTS-v2 | Non-commercial | — | Excellent | 6 seconds | Rejected — license |

**Why NeuTTS Air replaced Kokoro-82M:**
- GGUF format runs via llama.cpp — same infrastructure as demucs.cpp (unified Tier 1)
- Q4 quantization brings practical size under 1 GB despite 748M params
- Real-time on CPU, runs on Raspberry Pi
- Built-in Perth watermarking
- Apache 2.0 license
- 3-second voice cloning from reference audio

**Why RVC v2 not v3:**
RVC project README still says "Please look forward to the pretrained base model of RVCv3." As of March 2026, v3 weights have NOT been released. The WebUI repo (34.6k stars) is maintained but all releases are v2-based.

### 4.4 Audio-to-MIDI (NEW)

| Model | License | Developer | Status |
|-------|---------|-----------|--------|
| **basic-pitch** | **Apache 2.0** | Spotify | **SELECTED** |
| omnizart | — | — | Alternative for polyphonic |

basic-pitch enables: generate audio with ACE-Step → export as MIDI → edit in piano roll → re-render with different instruments. Replicates Udio's audio-to-MIDI feature using fully open-source tooling.

### 4.5 Code-to-Music Pipeline (NEW — Novel Differentiator)

No existing DAW or AI music platform offers this. An LLM generates executable Python code from natural language prompts, where the code programmatically synthesizes music.

| Component | License | Role |
|-----------|---------|------|
| Llama 4 (via Ollama) | Open | Code generation LLM |
| midiutil | MIT | MIDI file generation |
| FluidSynth | LGPL | MIDI → audio via SoundFonts |
| pyo | LGPL | Python-native audio DSP |

**Advantages over neural audio generation:**
1. Deterministic & reproducible (same code = same output)
2. Fully editable (code IS the arrangement)
3. Parametric control (every parameter is an explicit variable)
4. Exportable as MIDI/notation
5. Version control friendly (arrangements become diffable text)
6. Composable (chain code blocks for complex arrangements)

**Training plan:** LoRA fine-tune Llama 4 Scout on 10k–50k (prompt, code) pairs from Sonic Pi tutorials, SuperCollider docs, MIDI datasets converted to midiutil code, music theory formalized as code, and Algorave community code.

### 4.6 Mastering

| Tool | License | Function | Status |
|------|---------|----------|--------|
| **Matchering 2.0** | GPLv3 | Reference-based mastering | **PRIMARY — no change** |
| **Essentia** | AGPL v3 | Audio analysis | **Analysis layer** — evaluate SaaS implications |

---

## 5. AI Backend Architecture — Decision Log (v2.0)

### Three-Tier Architecture

```
Tier 1 — C++ (embedded, zero user setup)
  demucs.cpp + ONNX Runtime + NeuTTS Air (GGUF via llama.cpp)
  Covers: stem separation, noise reduction, TTS/voice preview
  Ships: inside the app binary
  Key: demucs.cpp and NeuTTS Air share GGML/llama.cpp infrastructure

Tier 2 — Python sidecar (auto-installs on first Tier 2 feature use)
  ACE-Step 1.5, YuE, Bark, Llama 4, Chatterbox Turbo,
  RVC v2, basic-pitch, Matchering, Code-to-Music pipeline
  Covers: generation, cloning, conversion, mastering, code-to-music
  Ships: auto-downloaded, runs as localhost gRPC server

Tier 3 — Optional cloud (BYO API key, zero dependency)
  Covers: users without local GPU, optional premium APIs
  Ships: user provides own API keys if desired
```

**UX benefit of Tier 1:** Three AI features ship in the app with zero setup:
1. Stem separation via demucs.cpp
2. Voice preview/TTS via NeuTTS Air (GGUF Q4)
3. Noise reduction via ONNX Runtime

Users get compelling AI features on first launch before ever touching Python.

### gRPC for Tier 2 Communication

gRPC selected over REST for: 3–7x smaller payloads, 5–10x faster parsing, native streaming (progress updates), strongly typed .proto contracts between C++ and Python.

---

## 6. Competitive Analysis — Strategic Decisions (v2.0)

### Suno

- Architecture: Hybrid transformer + diffusion, EnCodec-style tokenization. v5 ELO 1,293.
- No public developer API as of March 2026. ~100M users, $2.45B valuation.
- Only open-source contribution: **Bark** (MIT) — text-to-audio, not music generation.
- **Decision:** Do NOT depend on Suno API. Integrate Bark (MIT) for SFX/ambient. Use ACE-Step 1.5 as primary music generation (outperforms Suno v5 on benchmarks).

### ElevenLabs

- No core models open-sourced. SDKs only (elevenlabs-python/js/swift, all MIT).
- Full API: Eleven v3 TTS (70+ languages), Flash v2.5 (~75ms), Eleven Music, Stem Separation, Sound Effects, Scribe v2, Voice Cloning, Voice Design.
- CEO stated models will be "commoditized" over time.
- **Decision:** Do NOT depend on ElevenLabs. Open-source alternatives match or beat quality: Chatterbox preferred 63.75% over ElevenLabs in blind tests. Offer as optional Tier 3 BYO-API-key connector only.

### Udio

- Architecture: Latent Diffusion Transformer, 48kHz stereo, 10-min songs.
- UMG settlement (Oct 2025) — first AI music platform under full major label licensing.
- Python SDK exists BUT walled garden — NOT integrable for third-party apps.
- **Decision:** Do NOT integrate Udio. Replicate features with open source: ACE-Step repainting for inpainting, Demucs for stems, basic-pitch for audio-to-MIDI, Chatterbox for voice cloning.

### Why Open-Source Wins

| Dimension | Proprietary (Suno/Udio/ElevenLabs) | Wavy Labs (Open Source) |
|-----------|-------------------------------------|------------------------|
| Runs offline | ❌ | ✅ |
| Output editable | ❌ Opaque audio | ✅ Code + MIDI + stems |
| Vendor lock-in | ❌ Pricing changes, rate limits, shutdowns | ✅ Zero dependency |
| Privacy | ❌ Audio uploaded to cloud | ✅ Never leaves device |
| Cost per generation | ❌ Per-track/per-character fees | ✅ Free after hardware |
| Customizable | ❌ No fine-tuning | ✅ LoRA from your songs |
| Code-to-Music | ❌ Not offered anywhere | ✅ Unique differentiator |

---

## 7. License Audit — Full Stack (v2.0)

| Component | License | Commercial Safe? | Notes |
|-----------|---------|-----------------|-------|
| ACE-Step 1.5 | MIT | ✅ | Trained on licensed/royalty-free data |
| YuE | Apache 2.0 | ✅ | |
| Bark (Suno) | MIT | ✅ | Code + weights |
| Chatterbox (all) | MIT | ✅ | |
| NeuTTS Air | Apache 2.0 | ✅ | |
| Qwen3-TTS | Open source | ✅ | Check specific variant |
| RVC v2 | MIT | ✅ | |
| Demucs / demucs.cpp | MIT | ✅ | |
| basic-pitch | Apache 2.0 | ✅ | |
| midiutil | MIT | ✅ | |
| MagentaRT | Apache 2.0 (+terms) | ✅ | Review bespoke terms |
| Matchering | GPLv3 | ✅ | Copyleft — distribute source |
| Essentia | AGPLv3 | ⚠️ | SaaS implications — evaluate |
| pyo | LGPL | ⚠️ | Dynamic link only |
| FluidSynth | LGPL | ⚠️ | Dynamic link only |
| OpenAudio S1 | CC-BY-NC-SA-4.0 | ❌ | Non-commercial weights — not used |
| MusicGen weights | CC-BY-NC-4.0 | ❌ | Non-commercial — not used |

---

## 8. Monetization — Decision Log

### Model: Open Core + Optional Premium

1. **AGPL v3 core** — free, open, copyleft ensures forks stay open
2. **Optional BYO-API-key cloud** — users plug in their own API keys if they want cloud features
3. **Community model marketplace** — users share LoRAs and custom style models (revenue share)
4. **Professional support contracts** — studio/enterprise tier
5. **Avoid:** forced telemetry, late-stage license changes, mandatory cloud dependency

### Reference: Successful Open Source Audio Models

| Product | Model | Outcome |
|---------|-------|---------|
| Audacity | Free + enterprise | 114M+ downloads |
| Blender | Free + studio fund | Multi-million dollar foundation |
| BandLab | Free core + SaaS AI | Millions of users |

---

## 9. Reference Repositories (Updated v2.0)

| Project | URL | Role |
|---------|-----|------|
| Tracktion Engine | github.com/Tracktion/tracktion_engine | Core audio engine |
| JUCE | github.com/juce-framework/JUCE | UI + audio framework |
| LMN-3-DAW | github.com/FundamentalFrequency/LMN-3-DAW | DAW structure reference |
| awesome-juce | github.com/sudara/awesome-juce | JUCE component library list |
| **ACE-Step 1.5** | **github.com/ace-step/ACE-Step-1.5** | **Tier 2: Music generation (MIT)** |
| **YuE** | **github.com/multimodal-art-projection/YuE** | **Tier 2: Vocal song generation** |
| **Bark** | **github.com/suno-ai/bark** | **Tier 2: SFX/ambient (MIT)** |
| **demucs.cpp** | **github.com/sevagh/demucs.cpp** | **Tier 1: Stem separation (MIT)** |
| **NeuTTS Air** | **github.com/neuphonic/neutts** | **Tier 1: On-device TTS (Apache 2.0)** |
| **Chatterbox** | **github.com/resemble-ai/chatterbox** | **Tier 2: Voice cloning (MIT)** |
| **basic-pitch** | **github.com/spotify/basic-pitch** | **Tier 2: Audio-to-MIDI (Apache 2.0)** |
| **RVC v2** | **github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI** | **Tier 2: Trained voice cloning (MIT)** |
| ONNX Runtime | onnxruntime.ai | Tier 1: C++ model inference |
| Matchering | github.com/sergree/matchering | Tier 2: AI mastering |
| Essentia | github.com/MTG/essentia | Tier 2: Audio analysis |
| Qwen3-TTS | huggingface.co/Qwen/Qwen3-TTS | Tier 2: Zero-shot TTS |
| MagentaRT | github.com/magenta/magenta-realtime | Phase 3+: Real-time jam mode |
| Demucs ONNX (Mixxx) | mixxx.org/news/2025-10-27-gsoc2025-demucs-to-onnx | Demucs v4 ONNX guide |

---

## 10. Detailed Research Documents

Full research analysis is available in the `research/` directory:

- **[research/open_source_models.md](research/open_source_models.md)** — Comprehensive analysis of open-source models across music generation, stem separation, voice/TTS, and real-time audio. Conducted March 2, 2026.
- **[research/commercial_platforms.md](research/commercial_platforms.md)** — Analysis of Suno, ElevenLabs, and Udio technology stacks, API capabilities, open-source contributions, and Code-to-Music pipeline design. Conducted March 2, 2026.
