# Wavy Labs 🎵

> An open-source, AI-native Digital Audio Workstation — everything runs locally.

Wavy Labs combines the proven **Tracktion Engine** audio backbone with state-of-the-art open-source AI models to deliver a DAW where AI is a first-class citizen. Generate full tracks from prompts, write music as code, separate stems in one click, clone voices, auto-master — all running on your machine with **zero cloud dependency**.

**What makes Wavy Labs different:** Every feature runs locally using permissively licensed open-source models. No API keys. No subscriptions. No internet required. And a first-of-its-kind **Code-to-Music** pipeline that generates editable Python code instead of opaque audio — giving you deterministic, version-controllable compositions.

---

## Vision

| What | Detail |
|------|--------|
| **Platform** | Desktop — Windows, macOS, Linux |
| **Target user** | Professional musicians & producers |
| **License** | AGPL v3 (core) / Commercial (optional) |
| **Language** | C++20 (JUCE 8 + Tracktion Engine) + Python (AI sidecar) |
| **Plugin support** | CLAP (primary) + VST3 + AU + LV2 |
| **Philosophy** | Open-source first. Every core feature runs locally. |

---

## Core AI Features

All features are free, local, and run without internet:

- **🎵 Music Generation** — text prompt → full song placed in your timeline (ACE-Step 1.5, MIT)
- **💻 Code-to-Music** — LLM generates editable Python code that creates music — deterministic, reproducible, version-controllable *(unique — no other DAW offers this)*
- **🎤 Vocal Song Generation** — lyrics → full song with synchronized vocals (YuE, Apache 2.0)
- **🔊 Sound Effects & Ambient** — text → laughter, sighs, ambient soundscapes, ad-libs (Bark, MIT)
- **✂️ Stem Separation** — right-click any clip → vocals, drums, bass, other (demucs.cpp, MIT)
- **🗣️ Voice Cloning** — zero-shot from 3-second sample or trained from custom audio (Chatterbox/RVC v2, MIT)
- **🎹 Audio-to-MIDI** — convert any audio into editable MIDI (basic-pitch, Apache 2.0)
- **🎚️ Auto-Mastering** — reference-based mastering (Matchering, GPLv3)
- **🎸 Real-Time Jam Mode** — AI generates backing tracks that morph in real time *(Phase 3+)*

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  UI LAYER — JUCE 8 Components + Custom LookAndFeel       │
│  Timeline · Piano Roll · Mixer · AI Panel · Code View    │
├──────────────────────────────────────────────────────────┤
│  AUDIO ENGINE — Tracktion Engine (C++20)                 │
│  Tracks · Clips · Plugins · Transport                    │
├──────────────────────────────────────────────────────────┤
│  TIER 1 — Embedded C++ (zero setup, ships in binary)     │
│  demucs.cpp · ONNX Runtime · NeuTTS Air (GGUF)          │
├──────────────────────────────────────────────────────────┤
│  AI BRIDGE — gRPC async (never blocks audio thread)      │
├──────────────────────────────────────────────────────────┤
│  TIER 2 — Python Sidecar (auto-installs on first use)    │
│  ACE-Step 1.5 · YuE · Bark · Llama 4 · Chatterbox      │
│  basic-pitch · RVC v2 · Matchering · Code-to-Music       │
└──────────────────────────────────────────────────────────┘
```

**Key design principle:** The real-time audio thread is never touched by AI. Tier 1 models run in-process on worker threads. Tier 2 models run in a separate Python process via gRPC with streaming progress.

---

## AI Model Stack

Every model has been verified for commercial safety under permissive licenses:

| Feature | Model | License | Tier |
|---------|-------|---------|------|
| Music generation | ACE-Step 1.5 | MIT | 2 (Python) |
| Vocal song generation | YuE | Apache 2.0 | 2 (Python) |
| Code-to-Music | Llama 4 + midiutil/pyo | MIT/LGPL | 2 (Python) |
| Sound effects / ambient | Bark (Suno) | MIT | 2 (Python) |
| Stem separation | demucs.cpp | MIT | 1 (C++) |
| TTS / voice preview | NeuTTS Air (GGUF) | Apache 2.0 | 1 (C++) |
| Voice cloning (zero-shot) | Chatterbox Turbo | MIT | 2 (Python) |
| Voice cloning (trained) | RVC v2 | MIT | 2 (Python) |
| Audio-to-MIDI | basic-pitch (Spotify) | Apache 2.0 | 2 (Python) |
| Noise reduction | ONNX Runtime | MIT | 1 (C++) |
| Mastering | Matchering | GPLv3 | 2 (Python) |
| LLM orchestration | Llama 4 via Ollama | Open | 2 (Python) |
| Real-time jam (Phase 3+) | MagentaRT | Apache 2.0 | 2 (Python) |

---

## Repository Structure

```
WavyLabs/
├── README.md                 ← you are here
├── PLAN.md                   ← consolidated implementation plan (v2.0)
├── ROADMAP.md                ← phased milestones
├── RESEARCH.md               ← technology research & decision log
├── research/
│   ├── open_source_models.md ← open-source model landscape analysis
│   └── commercial_platforms.md ← Suno/ElevenLabs/Udio analysis
├── docs/
│   └── architecture.md
├── src/
│   ├── CMakeLists.txt
│   ├── Main.cpp
│   ├── engine/               ← Tracktion Engine wrappers
│   ├── ui/
│   │   ├── LookAndFeel/
│   │   ├── MainWindow/
│   │   ├── TrackView/
│   │   ├── MixerView/
│   │   ├── PianoRoll/
│   │   ├── CodeView/         ← Code-to-Music editor
│   │   └── AIPanel/
│   └── ai/                   ← gRPC client, AI command queue
├── ai_backend/               ← Python sidecar
│   ├── main.py
│   ├── ace_step/             ← music generation
│   ├── yue/                  ← vocal song generation
│   ├── bark/                 ← sound effects / ambient
│   ├── code_to_music/        ← LLM → Python → audio pipeline
│   ├── chatterbox/           ← voice cloning
│   ├── basic_pitch/          ← audio-to-MIDI
│   ├── matchering/           ← mastering
│   └── rvc/                  ← trained voice cloning
└── modules/                  ← git submodules
    ├── JUCE/
    └── tracktion_engine/
```

---

## Getting Started (Coming Soon)

Full build instructions will be added in Phase 1.

**Dependencies you'll need:**
- CMake 3.16+
- C++20 compatible compiler (MSVC 2022, Clang 14+, GCC 12+)
- Python 3.10+ (for Tier 2 AI features)
- CUDA-capable GPU recommended (CPU fallback available for all features)

**Tier 1 features work immediately** — stem separation and TTS run on CPU with zero setup.

---

## Roadmap

See [ROADMAP.md](ROADMAP.md) for the full phased plan.

| Phase | Focus | Key Deliverable |
|-------|-------|-----------------|
| 0 | Foundation & toolchain | App window builds on all platforms |
| 1 | Core DAW + first AI | Record, mix, stem separation, Bark SFX |
| 2 | Full AI suite | ACE-Step music gen, Code-to-Music, voice cloning |
| 3 | Differentiation | Real-time jam mode, LoRA marketplace, Code-to-Music LoRA |
| 4 | Scale | Community, optional cloud tier, fine-tuned models |

---

## Documentation

| Document | Description |
|----------|-------------|
| [PLAN.md](PLAN.md) | Consolidated implementation plan (v2.0) |
| [ROADMAP.md](ROADMAP.md) | Phased development milestones |
| [RESEARCH.md](RESEARCH.md) | Technology research & decision log |
| [research/open_source_models.md](research/open_source_models.md) | Open-source AI model landscape (March 2026) |
| [research/commercial_platforms.md](research/commercial_platforms.md) | Suno, ElevenLabs, Udio analysis |

---

## Why Open Source?

| | Cloud AI (Suno/Udio/ElevenLabs) | Wavy Labs |
|--|--------------------------------|-----------|
| Runs offline | ❌ | ✅ |
| Output editable | ❌ Opaque audio | ✅ Code + MIDI + stems |
| Vendor lock-in | ❌ | ✅ Zero dependency |
| Privacy | ❌ Audio uploaded | ✅ Never leaves device |
| Cost per generation | ❌ Per-track fees | ✅ Free after hardware |
| Customizable | ❌ | ✅ LoRA from your songs |
| Code-to-Music | ❌ | ✅ Unique |

---

## Contributing

This project is in active development. Watch this repo for updates.

---

*Wavy Labs — where AI and music production meet. Locally.*
