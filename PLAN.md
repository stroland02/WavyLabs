# Wavy Labs — Full Implementation Plan

**Version:** 1.0
**Date:** March 1, 2026
**Status:** Planning Phase

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

### 2.4 AI Backend — Python FastAPI + ONNX Runtime

The AI backend runs as a separate Python process. The C++ audio engine communicates with it via **gRPC** on a worker thread — **never** on the real-time audio thread.

```
[Tracktion Engine — C++ audio thread]   ← never touched by AI
[Worker thread] → gRPC → [Python FastAPI] → [AI model inference]
                  ↑ async, non-blocking
```

**Why gRPC over REST:**
- 3–7x smaller payloads than REST/JSON
- 5–10x faster parsing
- Streaming support for real-time progress updates
- Language-agnostic (C++ client ↔ Python server)

### 2.5 Communication & Storage

| Layer | Technology | Reason |
|-------|-----------|--------|
| C++ ↔ Python | gRPC + Protocol Buffers | Low latency, streaming |
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

**Primary: Demucs v4 (Meta)**
- Repo: https://github.com/facebookresearch/demucs
- License: MIT
- Quality: 9.20 dB SDR — state of the art
- Speed: Real-time capable on GPU
- Output: Vocals, drums, bass, other (4-stem) or 6-stem

**Secondary: Spleeter (Deezer)**
- Repo: https://github.com/deezer/spleeter
- License: MIT
- Speed: 100x real-time on GPU — faster than Demucs
- Quality: 6.5 dB SDR (lower quality but much faster)
- Use case: Quick preview / weaker hardware

**UI integration:** Right-click any audio clip → "Separate Stems" → progress dialog → stems placed as new tracks.

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
│  AI BRIDGE (Worker Thread — C++)                                │
│                                                                 │
│  AICommandQueue → gRPC client → Python backend                  │
│  Results → ValueTree → UI update on message thread             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  AI INFERENCE LAYER (Python 3.10+ / FastAPI)                    │
│                                                                 │
│  ┌──────────────┐ ┌──────────┐ ┌────────────┐ ┌─────────────┐ │
│  │ DiffRhythm   │ │ Demucs   │ │ Matchering │ │   RVC v3    │ │
│  │ MusicGen     │ │ v4       │ │ + Essentia │ │ StyleTTS2   │ │
│  │ (generation) │ │ (stems)  │ │ (mastering)│ │ (cloning)   │ │
│  └──────────────┘ └──────────┘ └────────────┘ └─────────────┘ │
│  ONNX Runtime — cross-platform, CPU + GPU + Apple Silicon       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  STORAGE                                                        │
│  SQLite (local projects) · File system (audio/MIDI/models)     │
└─────────────────────────────────────────────────────────────────┘
```

### Real-Time vs Offline Split (Critical Rule)

| AI Task | Mode | Max Latency | Implementation |
|---------|------|-------------|----------------|
| Noise gate / de-noise | Real-time | <40ms | Quantized ONNX model, C++ |
| Stem separation | Offline | 5–60s OK | Python gRPC |
| Music generation | Offline | 10–30s OK | Python gRPC |
| Mastering | Offline | 30–120s OK | Python gRPC |
| Voice cloning (training) | Offline | 1–2 hours OK | Python gRPC |
| Voice cloning (inference) | Near-real-time | <500ms | Python gRPC |

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
│       ├── AICommandQueue.h/.cpp   ← queues async AI tasks
│       ├── GrpcClient.h/.cpp       ← talks to Python backend
│       └── proto/
│           └── wavy_ai.proto       ← gRPC service definition
│
├── ai_backend/
│   ├── requirements.txt
│   ├── main.py                     ← FastAPI app entry
│   ├── proto/
│   │   └── wavy_ai.proto
│   ├── musicgen/
│   │   └── generator.py            ← DiffRhythm + MusicGen
│   ├── demucs/
│   │   └── separator.py            ← Demucs v4 stem separation
│   ├── matchering/
│   │   └── mastering.py            ← Matchering + Essentia
│   └── rvc/
│       └── cloner.py               ← RVC v3 voice cloning
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

## 8. Python AI Backend Setup

```python
# ai_backend/main.py
from fastapi import FastAPI
import grpc
from concurrent import futures
import wavy_ai_pb2_grpc

app = FastAPI()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=4))
    wavy_ai_pb2_grpc.add_WavyAIServicer_to_server(WavyAIServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

# ai_backend/requirements.txt
audiocraft>=1.0.0      # MusicGen
demucs>=4.0.1          # Stem separation
matchering>=2.0.0      # Mastering
torch>=2.2.0           # PyTorch
torchaudio>=2.2.0
onnxruntime-gpu>=1.17  # Cross-platform inference
fastapi>=0.110.0
grpcio>=1.62.0
grpcio-tools>=1.62.0
numpy>=1.26.0
scipy>=1.12.0
```

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
| AI latency breaks recording | High | Hard separation: real-time DSP vs offline AI |
| MusicGen CC-BY-NC weights | Medium | Use DiffRhythm (Apache 2.0) for commercial tier |
| Plugin crashes destabilize DAW | High | Isolate plugins in separate process; timeout on AI gRPC |
| Model download size (1–10GB) | Medium | Lazy load on first use; quantized GGUF defaults |
| GPU requirement alienates users | Medium | CPU fallback for all AI; cloud tier for GPU access |
| Competing with established DAWs | Medium | AI-native angle + open source = unique positioning |

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

4. Run Demucs locally to confirm AI works:
   ```bash
   pip install demucs
   python -m demucs --two-stems=vocals my_song.mp3
   ```

5. Run MusicGen locally to confirm VRAM:
   ```bash
   pip install audiocraft
   python -c "from audiocraft.models import MusicGen; m = MusicGen.get_pretrained('small'); print('OK')"
   ```

6. Set up WavyLabs git repo and scaffold CMakeLists.txt

---

## 15. Reference Repositories

| Project | URL | Role |
|---------|-----|------|
| Tracktion Engine | github.com/Tracktion/tracktion_engine | Core audio engine |
| JUCE | github.com/juce-framework/JUCE | UI + audio framework |
| LMN-3-DAW | github.com/FundamentalFrequency/LMN-3-DAW | DAW structure reference |
| awesome-juce | github.com/sudara/awesome-juce | JUCE component library list |
| DiffRhythm | github.com/ASLP-lab/DiffRhythm | Music generation |
| AudioCraft | github.com/facebookresearch/audiocraft | MusicGen |
| Demucs | github.com/facebookresearch/demucs | Stem separation |
| RVC v3 | github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI | Voice cloning |
| Matchering | github.com/sergree/matchering | AI mastering |
| Essentia | github.com/MTG/essentia | Audio analysis |
| MusicGen fine-tune | github.com/ylacombe/musicgen-dreamboothing | Model training |
