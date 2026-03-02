# Wavy Labs 🎵

> An open-source, AI-native Digital Audio Workstation for professional musicians.

Wavy Labs combines the proven **Tracktion Engine** audio backbone with cutting-edge open-source AI models to deliver a DAW where AI is a first-class citizen — not a bolt-on feature. Generate full tracks from prompts, separate stems in one click, auto-master with a reference track, and clone voices or instruments, all from inside a professional-grade desktop DAW.

---

## Vision

| What | Detail |
|------|--------|
| **Platform** | Desktop — Windows, macOS, Linux |
| **Target user** | Professional musicians & producers |
| **License** | AGPL v3 (core) / Commercial (optional) |
| **Language** | C++20 (JUCE + Tracktion Engine) + Python (AI backend) |
| **Plugin support** | CLAP (primary) + VST3 + AU + LV2 |

---

## Core AI Features

- **Generative Music** — text-prompt → full multi-stem track placed directly in your timeline
- **Stem Separation** — right-click any audio clip → separate to vocals, drums, bass, other
- **Auto-Mixing & Mastering** — AI-informed EQ/compression + reference-based mastering
- **Voice / Instrument Cloning** — clone any voice or instrument from a short sample

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│  UI LAYER — JUCE Components + LookAndFeel   │
│  Timeline · Piano Roll · Mixer · AI Panel   │
├─────────────────────────────────────────────┤
│  AUDIO ENGINE — Tracktion Engine (C++20)    │
│  Tracks · Clips · Plugins · Transport       │
├─────────────────────────────────────────────┤
│  AI BRIDGE — gRPC async (worker thread)     │
│  Never blocks the real-time audio thread    │
├─────────────────────────────────────────────┤
│  AI BACKEND — Python FastAPI + ONNX Runtime │
│  MusicGen · Demucs · Matchering · RVC v3    │
└─────────────────────────────────────────────┘
```

---

## Repository Structure (Planned)

```
WavyLabs/
├── README.md
├── PLAN.md               ← full implementation plan
├── ROADMAP.md            ← phased milestones
├── RESEARCH.md           ← technology research & decisions
├── docs/
│   └── architecture.md
├── src/
│   ├── CMakeLists.txt
│   ├── Main.cpp
│   ├── engine/           ← Tracktion Engine wrappers
│   ├── ui/               ← all JUCE Component subclasses
│   │   ├── LookAndFeel/
│   │   ├── MainWindow/
│   │   ├── TrackView/
│   │   ├── MixerView/
│   │   ├── PianoRoll/
│   │   └── AIPanel/
│   └── ai/               ← gRPC client, AI command queue
├── ai_backend/           ← Python FastAPI AI inference server
│   ├── main.py
│   ├── musicgen/
│   ├── demucs/
│   ├── matchering/
│   └── rvc/
└── modules/              ← git submodules
    ├── JUCE/
    └── tracktion_engine/
```

---

## Getting Started (Coming Soon)

Full build instructions will be added in Phase 1.

**Dependencies you'll need:**
- CMake 3.16+
- C++20 compatible compiler (MSVC 2022, Clang 14+, GCC 12+)
- Python 3.10+
- CUDA-capable GPU recommended (CPU fallback available)

---

## Roadmap

See [ROADMAP.md](ROADMAP.md) for the full phased plan.

| Phase | Focus | Target |
|-------|-------|--------|
| 0 | Foundation & toolchain | Month 1 |
| 1 | Core DAW (record, play, mix) | Month 1–3 |
| 2 | First AI features (stem sep, mastering) | Month 4–6 |
| 3 | Advanced AI (generation, voice cloning) | Month 7–9 |
| 4 | Scale, marketplace, SaaS tier | Post-MVP |

---

## Research & Decisions

See [RESEARCH.md](RESEARCH.md) for full technology evaluation and decision log.

---

## Contributing

This project is in active planning. Watch this repo for updates.

---

*Wavy Labs — where AI and music production meet.*
