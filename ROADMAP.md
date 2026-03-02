# Wavy Labs — Development Roadmap

---

## Phase 0 — Foundation (Weeks 1–4)

**Goal:** Working toolchain. Tracktion Engine builds on all 3 platforms. Empty Wavy Labs app window appears.

### Tasks
- [ ] Clone Tracktion Engine with submodules
- [ ] Build `DemoRunner` example on Windows
- [ ] Build `DemoRunner` example on macOS
- [ ] Build `DemoRunner` example on Linux
- [ ] Study LMN-3-DAW source (`AppLookAndFeel.cpp`, `TracksView.cpp`, `MixerView.cpp`)
- [ ] Scaffold WavyLabs CMakeLists.txt (see PLAN.md §6)
- [ ] Create `Main.cpp` and empty `MainWindow` component
- [ ] Create `WavyLookAndFeel` dark theme (colors, fonts, button styles)
- [ ] Confirm CLAP plugin loading works end-to-end
- [ ] Set up Python virtual environment for AI backend
- [ ] Confirm Demucs runs locally (`pip install demucs`)
- [ ] Confirm MusicGen loads (`pip install audiocraft`)
- [ ] Set up CI/CD (GitHub Actions) for Win/Mac/Linux builds

### Deliverable
Empty desktop window with Wavy Labs title, dark theme applied, no crashes.

---

## Phase 1 — Core DAW (Months 1–3)

**Goal:** A functional DAW — record, arrange, mix, export. No AI yet.

### Milestone 1.1 — Audio Engine Integration
- [ ] `EngineManager` singleton wrapping `te::Engine`
- [ ] `EditManager` — create, load, save projects (SQLite-backed)
- [ ] Audio device setup dialog (sample rate, buffer size, driver)
- [ ] MIDI device configuration

### Milestone 1.2 — Arrangement View
- [ ] `TimelineView` — scrollable track grid with beat/bar ruler
- [ ] `TrackView` — track row with header (name, mute, solo, arm)
- [ ] `ClipComponent` — audio clip with `AudioThumbnail` waveform display
- [ ] `MidiClipComponent` — MIDI clip with note preview
- [ ] `PlayheadComponent` — moving playhead during playback
- [ ] Loop markers (in/out points)
- [ ] Transport bar (play, stop, record, loop, BPM display)
- [ ] Multi-track audio recording (lock-free disk write)
- [ ] Undo/redo (Tracktion Engine built-in)

### Milestone 1.3 — CLAP/VST3 Plugin Host
- [ ] Plugin scanner (scans system plugin folders)
- [ ] Plugin browser panel (list + search)
- [ ] Add plugin to track (instrument or effect)
- [ ] Plugin window display (native plugin UI)
- [ ] Plugin delay compensation (automatic via Tracktion Engine)
- [ ] Plugin bypass

### Milestone 1.4 — Piano Roll
- [ ] `PianoRollView` — grid with note lanes
- [ ] Draw notes (click + drag)
- [ ] Select, move, resize notes
- [ ] Velocity editing
- [ ] Quantize
- [ ] MIDI input recording into piano roll

### Milestone 1.5 — Mixer
- [ ] `MixerView` — horizontal channel strip layout
- [ ] `ChannelStrip` — fader, pan, mute, solo
- [ ] `LevelMeter` — real-time RMS/peak display
- [ ] Send/return routing
- [ ] Master bus channel
- [ ] Audio export (bounce to WAV/MP3/FLAC)

### Deliverable
A usable DAW. Can record audio and MIDI, arrange clips, use VST3/CLAP plugins, mix, and export. Shown to 5+ beta testers for feedback.

---

## Phase 2 — First AI Features (Months 4–6)

**Goal:** Stem separation and AI mastering working inside the DAW. The AI pipeline (gRPC bridge) proven end-to-end.

### Milestone 2.1 — AI Infrastructure
- [ ] Define `wavy_ai.proto` gRPC service (see PLAN.md §7)
- [ ] Generate C++ and Python gRPC stubs
- [ ] `AICommandQueue` in C++ — non-blocking AI task submission
- [ ] `GrpcClient` in C++ — connects to Python backend
- [ ] Python FastAPI + gRPC server running locally
- [ ] Auto-start Python backend on app launch
- [ ] Health check / reconnect logic
- [ ] AI progress overlay (progress bar in clip while processing)

### Milestone 2.2 — Stem Separation
- [ ] Demucs v4 integrated in Python backend
- [ ] UI: right-click audio clip → "Separate Stems"
- [ ] 4-stem output: vocals, drums, bass, other
- [ ] Stems automatically placed as new tracks below source
- [ ] Source track muted after separation
- [ ] Spleeter fallback for faster/lighter processing

### Milestone 2.3 — AI Mastering
- [ ] Matchering 2.0 integrated in Python backend
- [ ] Essentia audio analysis integrated
- [ ] UI: `MasteringView` panel — drag reference track, set loudness target
- [ ] "Master" button → gRPC → Matchering → places mastered file
- [ ] Preview A/B (original vs mastered)

### Deliverable
Alpha release. Real producers can load tracks, separate stems, master against a reference. Collect feedback.

---

## Phase 3 — Advanced AI (Months 7–9)

**Goal:** Generative music and voice cloning fully integrated. Agentic pipeline working.

### Milestone 3.1 — Generative Music
- [ ] DiffRhythm integrated in Python backend
- [ ] MusicGen as secondary option
- [ ] `GenerateView` panel — text prompt, duration, BPM, key, genre
- [ ] Llama 3.1 (via Ollama) for LLM composition orchestration
- [ ] Agentic pipeline: LLM → DiffRhythm → Demucs → Matchering
- [ ] Generated stems placed directly into timeline as editable tracks
- [ ] "Variations" button — generate 3 alternatives at once
- [ ] Prompt history / favorites

### Milestone 3.2 — Voice / Instrument Cloning
- [ ] RVC v3 integrated in Python backend
- [ ] `VoiceClonerView` — drag sample audio to clone
- [ ] Training mode: 5–10 min sample → 1–2hr local training
- [ ] Inference mode: apply cloned voice to MIDI notes
- [ ] Voice library (save and load cloned voices)

### Milestone 3.3 — Polish & Beta Launch
- [ ] VST3 compatibility layer (for existing pro plugin collections)
- [ ] App preferences (theme options, default directories, AI settings)
- [ ] Crash reporting (local log, optional opt-in telemetry)
- [ ] Auto-update system
- [ ] Onboarding tutorial (first-launch walkthrough)
- [ ] Public beta release on GitHub
- [ ] Discord community setup

### Deliverable
Public beta. Full AI feature set available. Community feedback loop begins.

---

## Phase 4 — Scale & Monetize (Post-MVP)

**Goal:** Sustainable product. Growing user base. SaaS revenue beginning.

### Features
- [ ] Cloud GPU inference tier (SaaS subscription, $9–15/month)
- [ ] Cloud project sync and collaboration
- [ ] Community model marketplace
- [ ] Fine-tune MusicGen-small on curated datasets (proprietary model)
- [ ] Mobile companion app (project browser, AI generation on-the-go)
- [ ] API for third-party integrations
- [ ] University/education tier pricing

### Growth
- [ ] Product Hunt launch
- [ ] YouTube demo videos
- [ ] Reach out to music production YouTubers for coverage
- [ ] Submit to open source audio directories
- [ ] Apply for music tech accelerators/grants

---

## Version Targets

| Version | Phase | Description |
|---------|-------|-------------|
| v0.1 | Phase 0 | Empty app, builds on all platforms |
| v0.2 | Phase 1 | Core DAW: record, arrange, mix |
| v0.3 | Phase 2 | Stem separation + AI mastering |
| v0.4 | Phase 3 | Generative music + voice cloning |
| v1.0 | Phase 4 | Public stable release |
