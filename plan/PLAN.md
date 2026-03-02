# Wavy Labs — Consolidated AI-Powered DAW Plan

**Version:** 2.0 (Consolidated)  
**Date:** March 2, 2026  
**Philosophy:** Open-source first. Every core feature runs locally. No cloud dependency required.

---

## 1. Vision

Wavy Labs is an AI-powered Digital Audio Workstation (DAW) that puts state-of-the-art music generation, voice synthesis, stem separation, and intelligent audio editing directly on the user's machine. Unlike Suno, Udio, and ElevenLabs — which lock capabilities behind cloud APIs and walled gardens — Wavy Labs runs entirely offline using open-source models with permissive licenses.

The defining differentiator is the **Code-to-Music pipeline**: an LLM generates executable Python code that programmatically creates music, giving users deterministic, editable, version-controllable compositions that no other DAW or AI music platform offers.

---

## 2. Architecture Overview

Wavy Labs uses a three-tier architecture organized by runtime environment and performance requirements:

```
┌──────────────────────────────────────────────────────────┐
│                    WAVY LABS DAW                          │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  TIER 1 — Embedded C++ (ships in app binary)       │  │
│  │  Ultra-low latency, always available, no deps      │  │
│  │                                                    │  │
│  │  • Stem separation ─── demucs.cpp (GGML)           │  │
│  │  • Noise reduction ─── ONNX Runtime                │  │
│  │  • TTS / voice ─────── NeuTTS Air (GGUF Q4)       │  │
│  └────────────────────────────────────────────────────┘  │
│                         │                                │
│                    IPC / gRPC                             │
│                         │                                │
│  ┌────────────────────────────────────────────────────┐  │
│  │  TIER 2 — Python Sidecar (background process)      │  │
│  │  GPU-accelerated, model-heavy, async tasks         │  │
│  │                                                    │  │
│  │  • Music generation ──── ACE-Step 1.5 (primary)    │  │
│  │  • Vocal song gen ────── YuE                       │  │
│  │  • Sound effects ─────── Bark (Suno, MIT)          │  │
│  │  • Code-to-Music ─────── Llama 4 + midiutil/pyo   │  │
│  │  • Voice cloning ─────── Chatterbox Turbo + RVC v2 │  │
│  │  • Audio repainting ──── ACE-Step 1.5              │  │
│  │  • Audio-to-MIDI ─────── basic-pitch (Spotify)     │  │
│  │  • Mastering ─────────── Matchering + Essentia     │  │
│  │  • LLM orchestration ─── Llama 4 via Ollama        │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  TIER 3 — Optional Cloud (BYO API key)             │  │
│  │  Zero dependency. Users can optionally connect      │  │
│  │  their own API keys for premium cloud features.     │  │
│  │                                                    │  │
│  │  • Heavy generation when user has no local GPU     │  │
│  │  • Optional ElevenLabs / future Suno API key       │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

The Tier 1 C++ layer and Tier 2 Python sidecar share a common infrastructure foundation: both use GGML-based model execution (demucs.cpp and NeuTTS Air via llama.cpp), simplifying the build system and deployment.

---

## 3. Model Stack — Complete Specification

### 3.1 Tier 1 — Embedded C++ (Ships in App)

These models are compiled into the application binary or loaded at startup. They must be fast, lightweight, and require no external dependencies.

#### Stem Separation: demucs.cpp (GGML)

| Property | Value |
|----------|-------|
| Model | Demucs htdemucs_ft |
| Format | GGML quantized weights |
| License | MIT |
| Quality | ~9.2 dB SDR — best open-source stem separator as of March 2026 |
| Stems | Vocals, drums, bass, other |
| Status | No open-source model has dethroned Demucs. MDX-Net/MDX'23 architectures can outperform on specific stems (e.g., MDX-Net for vocals) — consider bundling UVR5's MDX-Net models as an advanced user option. |

#### Noise Reduction: ONNX Runtime

| Property | Value |
|----------|-------|
| Format | ONNX optimized model |
| License | MIT (ONNX Runtime) |
| Notes | Lightweight spectral denoising, runs on CPU |

#### TTS / Voice Preview: NeuTTS Air

| Property | Value |
|----------|-------|
| Params | 748M (Qwen2 0.5B backbone) |
| Format | GGUF Q4/Q8 — runs via llama.cpp |
| License | Apache 2.0 |
| Cloning | 3 seconds of reference audio |
| Inference | Real-time on CPU, runs on Raspberry Pi |
| Languages | English (NeuTTS-Nano multilingual variants available) |
| Repo | `github.com/neuphonic/neutts` |
| Watermarking | Built-in Perth (Perceptual Threshold) watermarker |

NeuTTS Air replaces Kokoro-82M from the original plan. It uses the same GGML/llama.cpp infrastructure as demucs.cpp, unifying the Tier 1 runtime. Kokoro-82M is smaller (82M vs 748M) but NeuTTS Air's Q4 quantization makes the practical size difference minimal, and quality is substantially better. It is the first TTS model designed from the ground up for on-device GGUF deployment.

---

### 3.2 Tier 2 — Python Sidecar

The Python sidecar runs as a background process communicating with the main app via IPC/gRPC. It handles GPU-accelerated, model-heavy tasks asynchronously.

#### Music Generation (Primary): ACE-Step 1.5 ⭐

| Property | Value |
|----------|-------|
| Released | January 28, 2026 |
| License | MIT (code + weights, trained on licensed/royalty-free data) |
| Architecture | Hybrid LM planner + Diffusion Transformer (DiT) |
| Speed | Full song in <2 sec on A100, <10 sec on RTX 3090 |
| VRAM | <4 GB — runs on consumer hardware |
| Duration | 10 seconds to 10 minutes |
| Languages | 50+ for lyrics/prompts |
| Repo | `github.com/ace-step/ACE-Step-1.5` |

ACE-Step 1.5 replaces DiffRhythm as the primary music generation engine. It outperforms most commercial music models on the SongEval benchmark (scoring 8.09, above Suno v5). Its LM planner uses chain-of-thought reasoning to convert simple prompts into structured song blueprints — metadata, lyrics, arrangement — which then condition the DiT for audio synthesis.

**Built-in DAW-ready features:**

- **Audio Repainting:** Selectively regenerate specific bars without affecting the rest. Maps to: right-click a section → "Regenerate bars 8–16."
- **Cover Generation:** Generate new compositions in a reference track's style via LoRA fine-tuning from just a few songs. Maps to: drag reference track → "Generate cover in this style."
- **Continuation:** Extend existing audio seamlessly. Maps to: "Extend this track by 30 seconds."
- **Vocal-to-BGM Conversion:** Extract vocal melody and generate matching accompaniment.
- **LoRA Fine-Tuning:** Users can train personalized style models from their own songs.

The LM planner partially overlaps with the Llama 4 orchestration role — investigate whether ACE-Step's planner can replace or supplement the orchestration stage for music-specific tasks.

#### Music Generation (Vocals): YuE

| Property | Value |
|----------|-------|
| Released | January 28, 2025 (Apache 2.0 since January 30, 2025) |
| License | Apache 2.0 |
| Architecture | LLaMA2-based, trillions of tokens, autoregressive |
| Output | Up to 5 min with synchronized vocals + accompaniment |
| VRAM | 8 GB minimum (quantized models) |
| Repo | `github.com/multimodal-art-projection/YuE` |

YuE is the dedicated vocal generation engine. While ACE-Step also generates vocals, YuE's vocals are more controllable via lyrics input. It supports multilingual lyrics, voice style transfer, and bidirectional generation. The YuE-UI (Gradio) supports batch generation, audio prompts, continuation, and session save/load.

Use for the "generate song with vocals from lyrics" workflow specifically.

#### Sound Effects / Ambient Audio: Bark (Suno)

| Property | Value |
|----------|-------|
| License | MIT (code + weights, commercial use allowed) |
| Architecture | GPT-style three-stage transformer (Text → Semantic → Coarse acoustic → Fine acoustic) |
| Audio Codec | EnCodec quantized representation |
| VRAM | Full: ~12GB, Small: ~8GB, CPU offload: ~2GB |
| Output | 24kHz mono audio |
| Repo | `github.com/suno-ai/bark` (38.9k stars) |

Bark is Suno's only open-source release. It is not a music generation model — it is a text-to-audio model that fills the gap between TTS and music generation. It excels at generating laughter, sighing, crying, background noise, simple musical elements, and ambient soundscapes.

**DAW use cases:**

- **Ad-lib generation:** `[laughs]`, `[sighs]`, `[gasps]` paralinguistic tags
- **Ambient soundscapes:** Rain, crowd noise, room tone, environmental audio
- **Simple musical elements:** Short melodic fragments, tonal textures
- **100+ speaker presets:** Built-in voice library across languages
- **Sound design layers:** Non-speech vocalizations for modern production

#### Code-to-Music Generation: Llama 4 + midiutil/pyo ⭐ Novel Differentiator

This is the feature that makes Wavy Labs fundamentally different from every competitor.

**Concept:** Use an LLM to generate executable Python code from natural language prompts, where the code programmatically synthesizes music. Instead of generating opaque audio waveforms, you generate an editable program that creates music.

**Pipeline: Prompt → LLM → Python Code → Audio Engine → Music**

```
User Prompt
"Create a chill lo-fi beat, 85 BPM,
 Rhodes piano chords with vinyl crackle"
        │
        ▼
┌─────────────────────┐
│  LLM (Llama 4 /     │  System prompt includes:
│  fine-tuned LoRA)    │  - Music theory knowledge
│                      │  - Code generation templates
│  Generates Python    │  - Available instruments/FX
│  code that creates   │  - Safety constraints
│  music               │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Sandbox Executor    │  Validates & runs generated
│  (Python subprocess) │  code in isolated environment
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐  ┌──────────────────────┐
│  MIDI Generation     │  │  Direct Synthesis     │
│  (midiutil)          │  │  (pyo / Tone.js)      │
└─────────┬───────────┘  └──────────┬───────────┘
          │                         │
          ▼                         ▼
┌─────────────────────┐  ┌──────────────────────┐
│  FluidSynth          │  │  WAV Output           │
│  (SoundFont render)  │  │  (direct audio)       │
└─────────┬───────────┘  └──────────┬───────────┘
          └────────────┬────────────┘
                       ▼
          ┌──────────────────────┐
          │  Audio Track          │
          │  + MIDI data          │
          │  + Source code         │
          │  (all editable)        │
          └──────────────────────┘
```

**Advantages over neural audio generation:**

1. **Deterministic & Reproducible:** Same code = same output. No random seeds or diffusion noise.
2. **Fully Editable:** The generated code IS the arrangement. Change BPM? Edit one number. Swap an instrument? Change one function call.
3. **Parametric Control:** Every musical parameter (tempo, key, velocity, timing, effects) is an explicit variable.
4. **Exportable as MIDI/Notation:** The code describes musical events, which trivially map to MIDI.
5. **Composable:** Chain multiple generated code blocks: intro_code + verse_code + chorus_code.
6. **Version Control Friendly:** Musical arrangements become diffable text files.

**DAW Integration — "Code View" tab:**

No existing DAW offers a code-to-music workflow. This becomes a defining feature:

- "Code View" tab alongside Piano Roll and Arrangement views
- User can toggle between generated audio and source code
- Editing code regenerates audio instantly
- Export as standalone Python scripts for reproducibility
- Appeals to technical musicians, producers, and the growing "vibe coding" culture

**Dependencies (all permissively licensed):**

| Library | Purpose | License |
|---------|---------|---------|
| midiutil | MIDI file generation | MIT |
| FluidSynth | MIDI → audio rendering via SoundFonts | LGPL (dynamic link) |
| pyo | Python-native audio DSP and synthesis | LGPL (dynamic link) |

**Training a Code-to-Music LoRA:**

Fine-tune Llama 4 Scout on (prompt, code) pairs:

- Sonic Pi tutorials & examples (thousands of annotated code snippets)
- SuperCollider documentation (extensive example library)
- MIDI datasets (Lakh MIDI, MAESTRO) — convert to midiutil Python code
- Music theory textbooks — formalize chord progressions/scales/rhythms as code
- Algorave/live-coding community — published performance code
- Custom pairs — Use ACE-Step to generate audio, manually create equivalent code
- Target: 10k-50k high-quality pairs, LoRA fine-tuning

#### Voice Cloning (Zero-Shot): Chatterbox Turbo

| Property | Value |
|----------|-------|
| Params | 350M |
| License | MIT |
| Latency | Sub-200ms |
| Features | 1-step diffusion decoder, paralinguistic tags ([laugh], [cough], [chuckle]) |
| Watermarking | Built-in PerTh neural watermarker |

Chatterbox Turbo has been benchmarked against ElevenLabs, with 63.75% of evaluators preferring Chatterbox in blind tests. The paralinguistic tags are uniquely valuable for music production — ad-libs, vocal textures, and non-speech vocalizations.

**Additional Chatterbox variants available:**

- **Chatterbox Original** (500M): English, emotion exaggeration control
- **Chatterbox Multilingual** (550M): 23 languages, full voice cloning + emotion control

All three variants are MIT licensed and hot-swappable.

#### Voice Cloning (Trained): RVC v2

| Property | Value |
|----------|-------|
| License | MIT |
| Repo Stars | 34.6k |
| Status | RVC v3 weights have NOT been released as of March 2026. Use v2. |

RVC v2 handles trained voice conversion — users provide training samples and get a fine-tuned voice model. Pair with Chatterbox Multilingual for zero-shot scenarios.

#### Voice Synthesis (Advanced): Qwen3-TTS

| Property | Value |
|----------|-------|
| Released | January 2026 |
| Languages | 10 (Chinese, English, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian) |
| Variants | Base (3-sec voice cloning), VoiceDesign (create voices from text descriptions), CustomVoice (9 presets) |

Qwen3-TTS VoiceDesign is specifically valuable for the "create voice from description" workflow (e.g., "warm male baritone with slight reverb"). Dual-track LM architecture enables ultra-low first-packet latency.

#### Audio-to-MIDI: basic-pitch (Spotify)

| Property | Value |
|----------|-------|
| License | Apache 2.0 |
| Developer | Spotify |
| Capability | Polyphonic audio → MIDI transcription |

Converts any generated or imported audio into editable MIDI. Enables workflows like: generate audio with ACE-Step → export as MIDI → edit in piano roll → re-render with different instruments. This replicates Udio's Audio-to-MIDI export feature using a fully open-source tool.

#### Mastering: Matchering + Essentia

| Component | License | Notes |
|-----------|---------|-------|
| Matchering | GPLv3 | Reference-based mastering (copyleft) |
| Essentia | AGPLv3 | Audio analysis/feature extraction (copyleft — SaaS implications, evaluate) |

#### LLM Orchestration: Llama 4 via Ollama

The agentic pipeline uses Llama 4 Scout as the orchestration LLM, running locally via Ollama. It interprets complex user requests, routes them to the appropriate model, chains multi-step workflows, and manages context.

Note: ACE-Step 1.5's built-in LM planner may reduce reliance on the orchestration LLM for music-specific tasks. Investigate whether the planner can handle prompt interpretation directly.

#### Future Addition — Real-Time Jam Mode: MagentaRT (Phase 3+)

| Property | Value |
|----------|-------|
| Released | June 2025 |
| License | Apache 2.0 (with some bespoke terms) |
| Architecture | 800M parameter autoregressive transformer |
| Output | 48 kHz stereo, real-time streaming |
| Training | ~190k hours of stock music (mostly instrumental) |

No DAW currently offers AI-driven live accompaniment. MagentaRT's real-time streaming architecture could power a "jam with AI" mode where the DAW generates backing tracks that morph in real time based on text or audio style prompts. Currently runs at 1.6x real-time factor on Colab TPUs; local inference is coming. Limited to instrumental and 10-second context window — not a replacement for ACE-Step, but a unique differentiating feature.

---

### 3.3 Tier 3 — Optional Cloud (BYO API Key)

Wavy Labs has **zero cloud dependency**. Every core feature runs locally. However, the architecture supports optional cloud integration for users who want it:

- **No-GPU fallback:** Users without a local GPU can connect to a hosted cloud endpoint for Tier 2 tasks
- **BYO API key:** Users can optionally plug in their own ElevenLabs, Suno (if/when available), or other API keys for premium cloud features — similar to how apps optionally support OpenAI keys

This is not a business dependency or required feature. It's a convenience layer that costs Wavy Labs nothing to maintain while serving users who want it.

---

### 3.4 Models Tracked But Not Integrated

These models are worth monitoring for future inclusion:

| Model | Why Track | Blocker |
|-------|-----------|---------|
| HeartMuLa 3B | Best lyrics controllability claim, HeartCLAP model useful for audio-text retrieval | Less mature than ACE-Step, only 3B version open |
| OpenAudio S1 / S1-mini (Fish Audio) | #1 on TTS-Arena2, best WER/CER scores | CC-BY-NC-SA-4.0 license (non-commercial weights) |
| InspireMusic (Alibaba) | Unified audio framework, 24/48 kHz | Less documented, early stage |
| CosyVoice2-0.5B (Alibaba) | 150ms streaming TTS, emotional control | Apache 2.0 — potential Tier 2 addition |
| IndexTTS-2 | Precise duration control, emotion-timbre disentanglement | Evaluate license terms |
| Dual-path Mamba networks | Better benchmark scores than Demucs for stem separation | No production-ready open-source implementation |
| Suno public API | If/when released, evaluate for optional Tier 3 | No public developer API as of March 2026 |

---

## 4. Feature Matrix — What Users Get

```
┌──────────────────────────────────────────────────────────┐
│              WAVY LABS — ALL FEATURES FREE                │
│              (Everything runs locally)                    │
│                                                          │
│  MUSIC GENERATION                                        │
│  ├─ Text-to-music (ACE-Step 1.5) ─── 50+ languages      │
│  ├─ Lyrics-to-song with vocals (YuE)                     │
│  ├─ Code-to-Music (Llama 4 + midiutil) ─── UNIQUE       │
│  ├─ Audio repainting (regenerate sections)               │
│  ├─ Cover generation (style transfer via LoRA)           │
│  ├─ Continuation (extend tracks seamlessly)              │
│  └─ Real-time jam mode (MagentaRT) ─── Phase 3+         │
│                                                          │
│  SOUND EFFECTS & AMBIENT                                 │
│  ├─ Text-to-SFX (Bark) ─── laughter, sighs, ambience    │
│  └─ 100+ speaker presets for vocal textures              │
│                                                          │
│  VOICE & TTS                                             │
│  ├─ On-device TTS (NeuTTS Air) ─── runs on CPU          │
│  ├─ Voice cloning, zero-shot (Chatterbox Turbo)          │
│  ├─ Voice cloning, multilingual (Chatterbox, 23 langs)   │
│  ├─ Voice cloning, trained (RVC v2)                      │
│  ├─ Voice design from description (Qwen3-TTS VoiceDesign)│
│  └─ Paralinguistic tags: [laugh] [cough] [chuckle]       │
│                                                          │
│  AUDIO TOOLS                                             │
│  ├─ Stem separation (demucs.cpp) ─── vocals/drums/bass   │
│  ├─ Noise reduction (ONNX Runtime)                       │
│  ├─ Audio-to-MIDI (basic-pitch) ─── edit any audio       │
│  └─ Reference mastering (Matchering)                     │
│                                                          │
│  AI ORCHESTRATION                                        │
│  ├─ Natural language commands (Llama 4)                   │
│  ├─ Multi-step workflow chaining                         │
│  └─ Intelligent model routing                            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 5. AI Task Routing Table

| AI Task | Tier | Engine | License |
|---------|------|--------|---------|
| Noise reduction | 1 (C++) | ONNX Runtime | MIT |
| Stem separation | 1 (C++) | demucs.cpp / GGML | MIT |
| TTS / voice preview | 1 (C++) | NeuTTS Air GGUF Q4 | Apache 2.0 |
| Music generation (primary) | 2 (Python) | ACE-Step 1.5 | MIT |
| Music generation (vocals) | 2 (Python) | YuE | Apache 2.0 |
| Sound effects / ambient | 2 (Python) | Bark (Suno) | MIT |
| Code-to-Music generation | 2 (Python) | Llama 4 + midiutil/pyo | MIT / LGPL |
| Voice cloning (zero-shot) | 2 (Python) | Chatterbox Turbo + Qwen3-TTS | MIT |
| Voice cloning (trained) | 2 (Python) | RVC v2 | MIT |
| Audio repainting/editing | 2 (Python) | ACE-Step 1.5 | MIT |
| Audio-to-MIDI | 2 (Python) | basic-pitch (Spotify) | Apache 2.0 |
| Mastering | 2 (Python) | Matchering + Essentia | GPLv3 / AGPLv3 |
| LLM orchestration | 2 (Python) | Llama 4 via Ollama | Open |
| Real-time jam mode | 2 (Python) | MagentaRT | Apache 2.0 |
| Heavy gen (no GPU) | 3 (Cloud) | Cloud endpoint | — |

---

## 6. Complete License Audit

Every model in the stack has been verified for commercial safety:

| Component | License | Commercial Safe? | Notes |
|-----------|---------|-----------------|-------|
| ACE-Step 1.5 | MIT | ✅ Yes | Trained on licensed/royalty-free data |
| YuE | Apache 2.0 | ✅ Yes | |
| Bark (Suno) | MIT | ✅ Yes | Code + weights |
| Chatterbox (all variants) | MIT | ✅ Yes | |
| NeuTTS Air | Apache 2.0 | ✅ Yes | |
| Qwen3-TTS | Open source | ✅ Yes | Check specific variant |
| RVC v2 | MIT | ✅ Yes | |
| Demucs / demucs.cpp | MIT | ✅ Yes | |
| basic-pitch (Spotify) | Apache 2.0 | ✅ Yes | |
| midiutil | MIT | ✅ Yes | |
| MagentaRT | Apache 2.0 (+terms) | ✅ Yes | Review bespoke terms |
| DiffRhythm | Apache 2.0 | ✅ Yes | Fallback if needed |
| HeartMuLa 3B | Apache 2.0 | ✅ Yes | Tracking only |
| Matchering | GPLv3 | ✅ Yes | Copyleft — distribute source |
| Essentia | AGPLv3 | ⚠️ Caution | Copyleft with SaaS implications |
| pyo | LGPL | ⚠️ Dynamic link only | Do not statically link |
| FluidSynth | LGPL | ⚠️ Dynamic link only | Do not statically link |
| OpenAudio S1 (Fish Audio) | CC-BY-NC-SA-4.0 | ❌ Non-commercial | Weights only — tracking |
| MusicGen | CC-BY-NC-4.0 | ❌ Non-commercial | Weights only — not used |

---

## 7. Competitive Positioning

### Why Open-Source Beats Proprietary for This Product

| Dimension | Suno / Udio / ElevenLabs | Wavy Labs |
|-----------|-------------------------|-----------|
| Runs offline | ❌ Cloud-only | ✅ 100% local |
| Output editable | ❌ Opaque audio blob | ✅ Code View + MIDI + stems |
| Vendor lock-in | ❌ Dependent on their pricing/availability | ✅ Zero dependency |
| Privacy | ❌ Audio uploaded to cloud | ✅ Never leaves device |
| Cost per generation | ❌ Per-track/per-character fees | ✅ Free after hardware |
| Customizable | ❌ No fine-tuning | ✅ LoRA from your own songs |
| Music quality | Suno v5 ELO 1,293 | ACE-Step 1.5 SongEval 8.09 (above Suno v5) |
| TTS quality | ElevenLabs Eleven v3 | Chatterbox preferred 63.75% in blind tests |
| Stem separation | ElevenLabs Stem API | Demucs htdemucs_ft (~9.2 dB SDR, industry standard) |
| Code-to-Music | ❌ Not offered | ✅ Unique — no competitor has this |

### What Competitors Cannot Replicate

1. **Code-to-Music pipeline** — generates editable programs, not opaque audio. Deterministic, reproducible, version-controllable.
2. **Fully local execution** — no internet, no API costs, no privacy concerns. Critical for professional studios.
3. **Unified open-source stack** — every component can be inspected, modified, and improved by the community.
4. **LoRA personalization** — users train style models on their own music. Your sound, your model, your device.

---

## 8. Phased Implementation

### Phase 1 — Foundation

- [ ] Tier 1 C++ core: demucs.cpp, ONNX noise reduction, NeuTTS Air GGUF
- [ ] Tier 2 Python sidecar: ACE-Step 1.5 integration, Chatterbox Turbo
- [ ] Basic DAW UI with track/arrangement view
- [ ] IPC/gRPC bridge between Tier 1 and Tier 2
- [ ] Bark integration for SFX/ambient generation

### Phase 2 — Full AI Suite

- [ ] YuE integration for vocal song generation
- [ ] Code-to-Music prototype (Llama 4 + midiutil, system prompt only)
- [ ] "Code View" tab in DAW UI
- [ ] basic-pitch audio-to-MIDI conversion
- [ ] RVC v2 trained voice cloning
- [ ] Qwen3-TTS VoiceDesign integration
- [ ] ACE-Step audio repainting, cover generation, continuation features
- [ ] Matchering reference mastering
- [ ] Llama 4 orchestration layer for natural language commands

### Phase 3 — Differentiation

- [ ] Code-to-Music LoRA fine-tuning (10k+ training pairs)
- [ ] MagentaRT real-time jam mode
- [ ] Advanced audio repainting UI (spectral selection → regenerate)
- [ ] Multi-model chaining (ACE-Step → Demucs → basic-pitch → Code View)
- [ ] Optional Tier 3 BYO-API-key connector
- [ ] LoRA marketplace / community sharing
- [ ] UVR5 MDX-Net models as alternative stem separation algorithm

---

## 9. Research Sources

This plan consolidates findings from two research rounds conducted March 2, 2026:

1. **RESEARCH_UPDATE_March2026.md** — Open-source model landscape analysis across music generation, stem separation, voice/TTS, and real-time audio. Identified ACE-Step 1.5, NeuTTS Air, Chatterbox family, YuE, MagentaRT, and other models.

2. **RESEARCH_Suno_ElevenLabs_Udio_March2026.md** — Commercial platform analysis of Suno, ElevenLabs, and Udio technology stacks, API capabilities, and open-source contributions. Originated the Bark integration recommendation, Code-to-Music pipeline design, and basic-pitch addition.

The strategic decision was made to prioritize the open-source-first approach, incorporating only the models and concepts from the commercial research that are themselves open source (Bark), use open-source tooling (Code-to-Music via midiutil/pyo), or fill gaps not covered by the primary research (basic-pitch for audio-to-MIDI).
