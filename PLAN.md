# Wavy Labs — Full Implementation Plan

**Version:** 1.1
**Date:** March 1, 2026
**Status:** Planning Phase
**Changelog:** v1.1 — AI backend revised from Python-only to tiered C++/Python architecture

---

## 1. Project Summary

Wavy Labs is an open-source, AI-native Desktop DAW targeting professional musicians on Windows, macOS, and Linux. The strategy is to combine proven open-source codebases — rather than build from scratch — accelerating development while maintaining full control over the AI-native user experience.

**Core philosophy:** AI is a first-class citizen, not a bolt-on. Every workflow in Wavy Labs should be faster and more creative because of AI.

---

## 2. Technology Stack (Final Decisions)

### 2.1 Audio Engine — Tracktion Engine

| Property | Value |
|----------|-------|
| Repo | https://github.com/Tracktion/tracktion_engine |
| Stars | 1.4k |
| Language | C++20 |
| License | Dual: GPLv3 / Commercial |
| Last commit | February 18, 2026 (2 weeks ago — actively maintained) |
| Built on | JUCE framework |

**Why Tracktion Engine over forking a complete DAW:**
- It is a library, not a finished product — you build your own UI and UX from scratch
- 115,000+ lines of production-tested C++, 15+ years of development
- Clean `Engine → Edit → Track → Clip` hierarchy designed for custom DAWs
- Tracktion's own commercial product (Waveform Free/Pro) is built on it
- Dual license allows commercial SaaS tier later
- Actively maintained (Feb 2026 commit)

**Core class hierarchy:**
```
Engine              ← singleton, initializes everything
└── Edit            ← a project/session
    ├── TempoTrack
    ├── MarkerTrack
    ├── AudioTrack
    │   ├── AudioClip / MIDIClip / StepClip / EditClip
    │   └── Plugin (VST3 / AU / LV2 / LADSPA / CLAP)
    └── RackType    ← instrument/effect racks
```

**Built-in features:**
- Multi-CPU audio engine with lock-free disk recording
- WAV / AIFF / FLAC / OGG / MP3 / CAF / REX playback
- Time-stretching & pitch-shifting (Elastique, Rubber Band, SoundTouch)
- Perfect plugin delay compensation (PDC)
- MIDI clips, step clips, groove quantization
- MIDI pattern generation (bass, melody, chord, arp progressions)
- Clip launching (Ableton Live style)
- ARA2 support (Melodyne-style integration)
- Full transport with scrubbing and looping

### 2.2 UI Framework — JUCE

**Decision: JUCE UI (not Qt6)**

JUCE is part of the same framework as Tracktion Engine — zero integration friction. The plan is to build with JUCE UI first, and optionally migrate to Qt6/QML for Phase 4+ polish.

**Key JUCE UI components for DAW building:**

| Component | DAW Use |
|-----------|---------|
| `Component` | Base class for all views |
| `AudioThumbnail` | Waveform rendering in clips |
| `LookAndFeel_V4` | Global dark theme + custom styling |
| `Viewport` | Scrollable timeline and track list |
| `ListBox` | Track headers, plugin list, file browser |
| `Slider` | Volume faders, pans, effect parameters |
| `OpenGLContext` | GPU-accelerated rendering for timeline |
| `TabbedComponent` | Arrange / Mixer / AI tabs |

**Custom components to build (no JUCE built-in exists):**
- Timeline / Arrangement view
- Clip component with waveform
- Piano roll (MIDI editor)
- Mixer channel strip
- Level meter
- AI Panel (prompt input, progress, results)
- Playhead component

**Primary reference project:** LMN-3-DAW (`github.com/FundamentalFrequency/LMN-3-DAW`)
- Real DAW built on Tracktion Engine + JUCE
- GPL-3.0, 530 stars
- Study: `AppLookAndFeel.cpp`, `TracksView.cpp`, `MixerView.cpp`

**Key resource:** awesome-juce (`github.com/sudara/awesome-juce`) — check before building any component.

### 2.3 Plugin Standard — CLAP (primary) + VST3 (compatibility)

| Standard | License | Why |
|----------|---------|-----|
| CLAP | MIT | Open, modern, multi-threading, MIDI 2.0, per-note automation |
| VST3 | Proprietary SDK (free) | Compatibility with existing professional plugins |
| AU | Apple (macOS only) | Required for macOS users |
| LV2 | ISC (Linux) | Linux compatibility |

CLAP is the future-facing choice. It is MIT licensed, supported by Bitwig, REAPER, and growing. Adopted by 15 DAWs and 393 plugins as of 2026.

### 2.4 AI Backend — Tiered C++ / Python Architecture

Not all AI features require Python. The backend is split into three tiers based on what C++ inference can realistically handle today vs what still requires Python.

#### The Three Tiers

```
TIER 1 — Embedded C++ (ships inside the app binary, zero user setup)
  demucs.cpp   → stem separation (GGML weights, MIT)
  ONNX Runtime → noise reduction, small real-time models (C++ API)
  No Python. No install step. Works on every user's machine out of the box.

TIER 2 — Python sidecar (auto-installs on first use, runs locally)
  DiffRhythm / MusicGen  → music generation (transformers, complex ONNX graph)
  Matchering + Essentia   → mastering & audio analysis
  RVC v3 / StyleTTS2      → voice cloning (training loop requires Python)
  Ollama / Llama 3.1      → LLM orchestration for agentic pipeline
  Communicates with C++ app via gRPC — never blocks audio thread.

TIER 3 — Cloud fallback (optional SaaS subscription)
  Larger model variants for users without a GPU
  Managed inference endpoints — user pays per generation
```

#### Communication Pattern

```
[Tracktion Engine — C++ real-time audio thread]   ← AI NEVER touches this
[Worker thread]
    ├── Tier 1: calls demucs.cpp / ONNX Runtime directly (in-process)
    └── Tier 2: gRPC → Python FastAPI sidecar → model inference
                ↑ async, streaming progress back to UI
```

#### Why gRPC for Tier 2 (not REST)
- 3–7x smaller payloads than REST/JSON
- 5–10x faster parsing
- Native streaming — progress updates flow back continuously ("Generating stems… 47%")
- Strongly typed `.proto` contract between C++ and Python

#### Why Not All-Python
Python was the original plan because every AI library ships Python-first. However:
- Stem separation (`demucs.cpp`, MIT) is already a mature C++17 port with GGML — zero Python needed
- Mixxx (open source DJ software) completed a GSoC 2025 project converting Demucs v4 to ONNX for C++ — ONNX weights are available
- ONNX Runtime has a full C++ API, confirmed working in JUCE audio plugins
- Shipping stem separation in pure C++ means the DAW's most-used AI feature works for every user with no setup, no Python, no GPU required

Music generation and voice cloning stay in Python for now — no production-ready C++ equivalent exists for transformer-scale models yet. That can be migrated to ONNX/C++ as the models mature.

### 2.5 Communication & Storage

| Layer | Technology | Reason |
|-------|-----------|--------|
| Tier 1 AI | ONNX Runtime C++ / demucs.cpp | In-process, zero latency |
| Tier 2 AI | gRPC + Protocol Buffers | Low latency, streaming, typed |
| Local storage | SQLite | Single-file projects, fast |
| Cloud sync (future) | PostgreSQL + S3 | Optional SaaS tier |

---

## 3. AI Feature Modules

### 3.1 Generative Music / Beat Creation

**Primary: DiffRhythm**
- Repo: https://github.com/ASLP-lab/DiffRhythm
- License: Apache 2.0 (fully permissive, commercial friendly)
- Quality: Excellent — generates a 4:45 full song in ~10 seconds
- Architecture: Latent diffusion model (similar to Suno's approach)

**Secondary: AudioCraft / MusicGen (Meta)**
- Repo: https://github.com/facebookresearch/audiocraft
- License: MIT code / CC-BY-NC-4.0 weights (non-commercial restriction on weights)
- Quality: Excellent — industry benchmark for text-to-music
- Note: CC-BY-NC weights limit commercial use — use DiffRhythm for commercial tier

**Agentic Music Generation Pipeline (Wavy Labs differentiator):**
```
User Prompt: "lo-fi hip hop, 90 BPM, chill study vibes"
    ↓
LLM Agent (Llama 3.1 via Ollama — local, private):
  → Song structure, chord progression, BPM, key, mood
    ↓
DiffRhythm / MusicGen:
  → Full backing track audio
    ↓
Demucs stem separation:
  → Splits into drums, bass, melody, other stems
    ↓
Matchering:
  → AI mastering against reference
    ↓
Output: Multi-stem project placed directly in timeline as editable tracks
```

This produces editable stems — not just a flat audio file. No other product does this.

### 3.2 Stem Separation

**Tier 1 — C++ (ships in app, no Python required): demucs.cpp**
- Repo: https://github.com/sevagh/demucs.cpp
- License: MIT, 160 stars
- What: C++17 port of Demucs v3 and v4 using GGML + Eigen3 — same tech as llama.cpp
- Quality: Same model weights as Python Demucs v4 (9.20 dB SDR)
- Setup: Load GGML-format weights at startup, call from C++ worker thread
- Powers: freemusicdemixer.com (live web demo proving it works)

**Tier 1 — ONNX alternative: Demucs v4 ONNX (Mixxx GSoC 2025)**
- Mixxx's GSoC 2025 project successfully converted Demucs v4 to ONNX format
- ONNX weights can be loaded via ONNX Runtime C++ API — no Python
- Reference: mixxx.org/news/2025-10-27-gsoc2025-demucs-to-onnx-dhunstack

**Tier 2 — Python fallback: Demucs v4 (Meta) via gRPC**
- Repo: https://github.com/facebookresearch/demucs
- License: MIT — use if C++ tier needs fallback or for 6-stem output
- Speed: Real-time capable on GPU

**Tier 2 — Fast fallback: Spleeter (Deezer)**
- Repo: https://github.com/deezer/spleeter
- License: MIT, 100x real-time on GPU
- Quality: 6.5 dB SDR — lower but much faster, good for quick preview

**UI integration:** Right-click any audio clip → "Separate Stems" → Tier 1 C++ runs immediately, stems appear as new tracks. No Python install required.

### 3.3 Auto-Mixing & Mastering

**Mastering: Matchering 2.0**
- Repo: https://github.com/sergree/matchering
- License: GPLv3
- Concept: Match your track's loudness, spectrum, and dynamics to a reference track
- Usage: User drags reference track → AI applies loudness/EQ normalization

**Mixing analysis: Essentia (MTG)**
- Repo: https://github.com/MTG/essentia
- License: AGPL v3
- Purpose: Analyze incoming audio (instrument detection, key/tempo, spectral features)
- Usage: Feed analysis to AI mixing suggestions (EQ, compression settings)

**iZotope-inspired three-stage pattern:**
1. User preference: target genre, reference track, loudness target
2. ML analysis (Essentia): classify instruments, analyze spectrum
3. Intelligent DSP: apply Matchering + parametric suggestions

### 3.4 Voice / Instrument Cloning

**Primary: RVC v3**
- Repo: https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI
- License: MIT
- Quality: Excellent — real-time capable
- Training: Only 5–10 minutes of audio needed to clone a voice
- Usage: Record/import sample → train model (1–2 hours locally) → apply to MIDI notes

**Secondary: StyleTTS2**
- License: MIT
- Quality: Matches ElevenLabs Professional quality
- Advantage: Transformer-based, better prosody control

**Avoid: Coqui XTTS-v2** — non-commercial license restriction; Coqui AI shut down December 2025.

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  WAVY LABS — COMPLETE SYSTEM ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│  UI LAYER (JUCE Components)                                     │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │ Timeline │ │  Mixer   │ │  Piano   │ │    AI Panel      │  │
│  │  View    │ │  View    │ │   Roll   │ │ Generate · Stems │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  AUDIO ENGINE (Tracktion Engine + JUCE — C++20)                 │
│                                                                 │
│  Engine → Edit → Tracks → Clips → Plugin Chain                  │
│  CLAP / VST3 / AU / LV2 plugin hosting                          │
│  Real-time audio thread (NEVER touched by AI code)              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  AI WORKER THREAD (C++) — dispatches to correct tier            │
│                                                                 │
│  AICommandQueue → routes by feature → Tier 1 or Tier 2         │
│  Results → ValueTree → UI update on message thread              │
│                                                                 │
├──────────────────────┬──────────────────────────────────────────┤
│  TIER 1 — C++        │  TIER 2 — Python sidecar (gRPC)          │
│  (embedded in app)   │  (auto-installs on first AI use)         │
│                      │                                          │
│  demucs.cpp          │  DiffRhythm / MusicGen (generation)      │
│  (stem separation)   │  Matchering + Essentia (mastering)       │
│                      │  RVC v3 / StyleTTS2 (voice cloning)      │
│  ONNX Runtime C++    │  Ollama / Llama 3.1 (LLM orchestration)  │
│  (noise reduction,   │                                          │
│   small models)      │  FastAPI gRPC server on localhost:50051  │
│                      │                                          │
│  Zero dependencies   │  Requires Python 3.10+ + GPU recommended │
├──────────────────────┴──────────────────────────────────────────┤
│  TIER 3 — Cloud (optional SaaS subscription)                    │
│  Managed GPU inference for heavy models / users without GPU     │
├─────────────────────────────────────────────────────────────────┤
│  STORAGE                                                        │
│  SQLite (local projects) · File system (audio/MIDI/models)      │
└─────────────────────────────────────────────────────────────────┘
```

### AI Task Routing Table

| AI Task | Tier | Latency Budget | Implementation |
|---------|------|----------------|----------------|
| Noise reduction | Tier 1 | <40ms | ONNX Runtime C++ (in-process) |
| Stem separation | Tier 1 | 5–60s OK | demucs.cpp (GGML, C++17) |
| Music generation | Tier 2 | 10–30s OK | DiffRhythm via Python gRPC |
| Mastering | Tier 2 | 30–120s OK | Matchering via Python gRPC |
| Voice cloning training | Tier 2 | 1–2 hours OK | RVC v3 via Python gRPC |
| Voice cloning inference | Tier 2 | <500ms OK | RVC v3 via Python gRPC |
| Heavy generation (no GPU) | Tier 3 | 30–120s OK | Cloud endpoint |

---

## 5. Project Folder Structure

```
WavyLabs/
├── CMakeLists.txt                  ← root build
├── README.md
├── PLAN.md                         ← this file
├── ROADMAP.md
├── RESEARCH.md
│
├── modules/                        ← git submodules
│   ├── JUCE/                       ← git submodule
│   └── tracktion_engine/           ← git submodule
│
├── src/
│   ├── CMakeLists.txt
│   ├── Main.cpp                    ← JUCEApplication entry point
│   │
│   ├── engine/                     ← Tracktion Engine wrappers
│   │   ├── EngineManager.h/.cpp    ← singleton Engine wrapper
│   │   ├── EditManager.h/.cpp      ← Edit create/load/save
│   │   └── TrackManager.h/.cpp
│   │
│   ├── ui/
│   │   ├── LookAndFeel/
│   │   │   └── WavyLookAndFeel.h/.cpp   ← dark theme, all styling
│   │   ├── MainWindow/
│   │   │   └── MainWindow.h/.cpp
│   │   ├── TrackView/
│   │   │   ├── TimelineView.h/.cpp
│   │   │   ├── TrackView.h/.cpp
│   │   │   ├── ClipComponent.h/.cpp
│   │   │   ├── MidiClipComponent.h/.cpp
│   │   │   └── PlayheadComponent.h/.cpp
│   │   ├── MixerView/
│   │   │   ├── MixerView.h/.cpp
│   │   │   ├── ChannelStrip.h/.cpp
│   │   │   └── LevelMeter.h/.cpp
│   │   ├── PianoRoll/
│   │   │   └── PianoRollView.h/.cpp
│   │   └── AIPanel/
│   │       ├── AIPanelView.h/.cpp
│   │       ├── GenerateView.h/.cpp
│   │       ├── StemSeparatorView.h/.cpp
│   │       ├── MasteringView.h/.cpp
│   │       └── VoiceClonerView.h/.cpp
│   │
│   └── ai/
│       ├── AICommandQueue.h/.cpp   ← routes tasks to Tier 1 or Tier 2
│       ├── GrpcClient.h/.cpp       ← Tier 2: talks to Python sidecar
│       ├── cpp/
│       │   ├── DemucsRunner.h/.cpp ← Tier 1: wraps demucs.cpp
│       │   └── OnnxRunner.h/.cpp   ← Tier 1: ONNX Runtime C++ wrapper
│       └── proto/
│           └── wavy_ai.proto       ← gRPC service definition
│
├── ai_backend/                     ← Tier 2: Python sidecar
│   ├── requirements.txt
│   ├── main.py                     ← gRPC server entry point
│   ├── proto/
│   │   └── wavy_ai.proto
│   ├── musicgen/
│   │   └── generator.py            ← DiffRhythm + MusicGen
│   ├── matchering/
│   │   └── mastering.py            ← Matchering + Essentia
│   ├── rvc/
│   │   └── cloner.py               ← RVC v3 voice cloning
│   └── llm/
│       └── orchestrator.py         ← Ollama / Llama 3.1 agentic pipeline
│
├── vendor/                         ← Tier 1: C++ AI dependencies
│   ├── demucs.cpp/                 ← git submodule (MIT)
│   └── onnxruntime/                ← ONNX Runtime headers + libs
│
└── tests/
    ├── engine_tests/
    └── ai_backend_tests/
```

---

## 6. Build System Setup (CMakeLists.txt skeleton)

```cmake
cmake_minimum_required(VERSION 3.22)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

project(WavyLabs VERSION 0.1.0)

# Submodules
add_subdirectory(modules/tracktion_engine/modules/juce)
add_subdirectory(modules/tracktion_engine)

# Main app
juce_add_gui_app(WavyLabs
    PRODUCT_NAME "Wavy Labs"
    COMPANY_NAME "Wavy Labs"
    BUNDLE_ID "com.wavylabs.daw"
)

target_compile_features(WavyLabs PRIVATE cxx_std_20)

target_sources(WavyLabs PRIVATE
    src/Main.cpp
    src/engine/EngineManager.cpp
    src/ui/LookAndFeel/WavyLookAndFeel.cpp
    src/ui/MainWindow/MainWindow.cpp
    # ... add files as you build them
)

target_link_libraries(WavyLabs
    PRIVATE
        juce::juce_audio_basics
        juce::juce_audio_devices
        juce::juce_audio_formats
        juce::juce_audio_processors
        juce::juce_audio_utils
        juce::juce_gui_basics
        juce::juce_gui_extra
        juce::juce_opengl
        tracktion::tracktion_engine
)
```

---

## 7. gRPC AI Bridge (wavy_ai.proto)

```protobuf
syntax = "proto3";
package wavyai;

service WavyAI {
  rpc GenerateMusic (GenerateRequest) returns (stream GenerateProgress);
  rpc SeparateStems (StemRequest) returns (stream StemProgress);
  rpc MasterTrack (MasterRequest) returns (stream MasterProgress);
  rpc CloneVoice (CloneRequest) returns (stream CloneProgress);
}

message GenerateRequest {
  string prompt = 1;
  float duration_seconds = 2;
  float bpm = 3;
  string key = 4;
  string genre = 5;
}

message GenerateProgress {
  float progress = 1;         // 0.0 – 1.0
  string status_message = 2;
  bytes audio_data = 3;       // populated when complete
  repeated string stem_paths = 4;
}

message StemRequest {
  string audio_file_path = 1;
  int32 num_stems = 2;        // 2, 4, or 6
}

message StemProgress {
  float progress = 1;
  string status_message = 2;
  repeated string output_paths = 3;
}

message MasterRequest {
  string audio_file_path = 1;
  string reference_file_path = 2;
  float target_loudness_lufs = 3;
}

message MasterProgress {
  float progress = 1;
  string status_message = 2;
  string output_path = 3;
}

message CloneRequest {
  string sample_audio_path = 1;
  string mode = 2;            // "train" or "infer"
  bytes audio_to_convert = 3; // for infer mode
}

message CloneProgress {
  float progress = 1;
  string status_message = 2;
  bytes converted_audio = 3;
}
```

---

## 8. AI Backend Setup

### Tier 1 — C++ (added to CMakeLists.txt)

```cmake
# Add demucs.cpp as submodule
add_subdirectory(vendor/demucs.cpp)

# Add ONNX Runtime
find_package(onnxruntime REQUIRED)

target_link_libraries(WavyLabs
    PRIVATE
        # ... existing JUCE/Tracktion links ...
        demucs_cpp          # Tier 1 stem separation
        onnxruntime         # Tier 1 ONNX inference
)
```

**Adding demucs.cpp submodule:**
```bash
git submodule add https://github.com/sevagh/demucs.cpp vendor/demucs.cpp
git submodule update --init --recursive
```

### Tier 2 — Python Sidecar

```python
# ai_backend/main.py — gRPC server, no Python setup for end user until first AI use
import grpc
from concurrent import futures
import wavy_ai_pb2_grpc

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=4))
    wavy_ai_pb2_grpc.add_WavyAIServicer_to_server(WavyAIServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()
```

```
# ai_backend/requirements.txt
# NOTE: Demucs is NOT here — stem separation runs via Tier 1 (demucs.cpp)
audiocraft>=1.0.0      # MusicGen / DiffRhythm (music generation)
matchering>=2.0.0      # AI mastering
essentia>=2.1          # Audio analysis
torch>=2.2.0           # PyTorch (for MusicGen, RVC)
torchaudio>=2.2.0
ollama>=0.1.0          # Local LLM (Llama 3.1 orchestration)
grpcio>=1.62.0
grpcio-tools>=1.62.0
numpy>=1.26.0
scipy>=1.12.0
```

**Auto-install pattern:** The C++ app checks if Python sidecar is running on localhost:50051. If not, it launches a bundled install script on first use of a Tier 2 feature, showing an "Installing AI features…" progress dialog.

---

## 9. Quality Benchmarks (Reference Products)

| Reference | What to match | Open source equivalent |
|-----------|--------------|----------------------|
| **Suno AI** | Full song from text, realistic vocals | DiffRhythm (generation) + RVC v3 (vocals) |
| **ElevenLabs** | Voice cloning from short sample | RVC v3 / StyleTTS2 |
| **Songboy** | (Needs clarification from user) | TBD |

**Suno's architecture (confirmed via research):**
1. Audio codec (EnCodec-style) → discrete audio tokens
2. Transformer autoregressive model → generates tokens from text
3. Diffusion upscaler → high-fidelity audio
4. Neural singing synthesizer → vocals from melody + lyrics

**DiffRhythm matches this approach** with a latent diffusion architecture (Apache 2.0 license).

---

## 10. Custom AI Model Training Strategy

Wavy Labs should develop fine-tuned models over time for a proprietary quality edge.

### Base Model: MusicGen (AudioCraft)

Fine-tuning is well-documented and achievable:
- Guide: activeloop.ai — "Fine-Tuning MusicGen for Text-Based Music Generation"
- Repo: `github.com/ylacombe/musicgen-dreamboothing`
- Framework: MuseControlLite (ICML 2025) — low-cost controllable training

### Training Datasets (Open License)

| Dataset | Tracks | License | Content |
|---------|--------|---------|---------|
| Free Music Archive (FMA) | 106,574 | CC licenses | Diverse genres |
| MTG-Jamendo | 55,000 | CC BY | Full songs |
| MusicNet | 330 | CC0 | Classical + annotations |
| Slakh2100 | 2,100 | CC BY 4.0 | Multi-track MIDI + audio |
| NSynth (Google) | 300,000 | CC BY 4.0 | Individual instrument notes |

### Compute Requirements

| Goal | GPU | Time | Approx Cost |
|------|-----|------|-------------|
| Fine-tune MusicGen-small (300M) | 1x A100 | 8–24 hours | ~$50 cloud |
| Fine-tune MusicGen-medium (1.5B) | 4x A100 | 3–5 days | ~$600 cloud |
| Train DiffRhythm from scratch | 8x A100 | 2–4 weeks | ~$10,000 cloud |

**Start with:** MusicGen-small fine-tuned on curated FMA+Jamendo genre subsets. Achievable with $50 cloud compute. Produces genre-specialized output that sounds distinct from base models.

---

## 11. Licensing & Monetization

### Open Source Core — AGPL v3
- Full DAW features free and open source
- AGPL copyleft ensures forks stay open (prevents proprietary lock-in of improvements)
- Builds community trust (no telemetry, no tracking)

### Premium SaaS Tier — $9–15/month
- Cloud GPU access for faster inference
- Cloud project sync and collaboration
- Advanced AI model library (larger, higher quality models)
- Priority support

### Additional Channels
- Community model marketplace (revenue share)
- Professional studio support contracts
- University/education licensing
- Donation fund (Blender Foundation model)

---

## 12. Hardware Requirements

| Tier | Spec | Capability |
|------|------|-----------|
| Minimum | CPU-only, 16GB RAM | Basic DAW + Spleeter stem separation |
| Recommended | NVIDIA RTX 3060 12GB | Full AI features, sequential |
| Professional | NVIDIA RTX 4090 24GB | Multiple simultaneous AI models |
| Cloud tier | NVIDIA A40/H100 | Commercial SaaS inference |

Apple Silicon (M1/M2/M3) via Metal backend on ONNX Runtime — fully supported.

---

## 13. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| AI latency breaks recording | High | Hard separation: real-time DSP vs offline AI; Tier 1 never blocks |
| MusicGen CC-BY-NC weights | Medium | Use DiffRhythm (Apache 2.0) as primary for commercial tier |
| Plugin crashes destabilize DAW | High | Isolate plugins in separate process; timeout on AI gRPC calls |
| Model download size (1–10GB) | Medium | Lazy load on first use; quantized GGUF defaults for Tier 1 |
| Python setup friction (Tier 2) | Medium | Tier 1 covers stem separation with zero Python; Tier 2 only needed for generation/cloning |
| GPU requirement alienates users | Medium | Tier 1 runs CPU-only; Tier 2 has CPU fallback; Tier 3 cloud for heavy lifting |
| Competing with established DAWs | Medium | AI-native + open source + no-setup stem separation = clear differentiation |

---

## 14. Immediate Next Steps (Week 1)

1. Clone Tracktion Engine with submodules:
   ```bash
   git clone --recurse-submodules https://github.com/Tracktion/tracktion_engine.git
   ```

2. Build the `DemoRunner` example on your target OS — confirms toolchain works

3. Study these two files in LMN-3-DAW:
   - `Source/Views/LookAndFeel/AppLookAndFeel.cpp` — dark theme pattern
   - `Source/Views/Edit/Tracks/TracksView.cpp` — timeline/arrangement pattern

4. Build and test demucs.cpp (Tier 1 — no Python):
   ```bash
   git clone --recurse-submodules https://github.com/sevagh/demucs.cpp
   cd demucs.cpp && mkdir build && cd build
   cmake .. && cmake --build . --config Release
   # Download GGML weights, then:
   ./demucs.cpp.main my_song.mp3
   ```

5. Run MusicGen locally (Tier 2) to confirm VRAM:
   ```bash
   pip install audiocraft
   python -c "from audiocraft.models import MusicGen; m = MusicGen.get_pretrained('small'); print('OK')"
   ```

6. Test ONNX Runtime C++ with a simple audio model:
   ```bash
   # Download ONNX Runtime headers/libs from onnxruntime.ai
   # Link against onnxruntime in a test CMake project
   # Load a small .onnx model and run inference on a buffer
   ```

7. Set up WavyLabs git repo and scaffold CMakeLists.txt with demucs.cpp submodule

---

## 15. Reference Repositories

| Project | URL | Role |
|---------|-----|------|
| Tracktion Engine | github.com/Tracktion/tracktion_engine | Core audio engine |
| JUCE | github.com/juce-framework/JUCE | UI + audio framework |
| LMN-3-DAW | github.com/FundamentalFrequency/LMN-3-DAW | DAW structure reference |
| awesome-juce | github.com/sudara/awesome-juce | JUCE component library list |
| **demucs.cpp** | **github.com/sevagh/demucs.cpp** | **Tier 1: C++ stem separation (MIT)** |
| **ONNX Runtime** | **onnxruntime.ai** | **Tier 1: C++ model inference** |
| DiffRhythm | github.com/ASLP-lab/DiffRhythm | Tier 2: Music generation |
| AudioCraft | github.com/facebookresearch/audiocraft | Tier 2: MusicGen |
| Demucs (Python) | github.com/facebookresearch/demucs | Tier 2: Stem sep fallback |
| RVC v3 | github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI | Tier 2: Voice cloning |
| Matchering | github.com/sergree/matchering | Tier 2: AI mastering |
| Essentia | github.com/MTG/essentia | Tier 2: Audio analysis |
| MusicGen fine-tune | github.com/ylacombe/musicgen-dreamboothing | Model training reference |
| Demucs ONNX (Mixxx) | mixxx.org/news/2025-10-27-gsoc2025-demucs-to-onnx | Demucs v4 ONNX conversion guide |
