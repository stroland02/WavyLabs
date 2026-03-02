# Wavy Labs — Development Roadmap

**Version:** 2.0  
**Updated:** March 2, 2026

---

## Phase 0 — Foundation (Weeks 1–4)

**Goal:** Working toolchain. Tracktion Engine builds on all 3 platforms. Empty Wavy Labs app window appears.

### Tasks
- [ ] Clone Tracktion Engine with submodules
- [ ] Build `DemoRunner` example on Windows, macOS, Linux
- [ ] Study LMN-3-DAW source (`AppLookAndFeel.cpp`, `TracksView.cpp`, `MixerView.cpp`)
- [ ] Scaffold WavyLabs CMakeLists.txt (see PLAN.md §2)
- [ ] Create `Main.cpp` and empty `MainWindow` component
- [ ] Create `WavyLookAndFeel` dark theme (colors, fonts, button styles)
- [ ] Confirm CLAP plugin loading works end-to-end
- [ ] Build and test demucs.cpp (Tier 1 — no Python):
  ```bash
  git clone --recurse-submodules https://github.com/sevagh/demucs.cpp
  cd demucs.cpp && mkdir build && cd build
  cmake .. && cmake --build . --config Release
  ./demucs.cpp.main my_song.mp3
  ```
- [ ] Test NeuTTS Air GGUF Q4 via llama.cpp (Tier 1 — no Python)
- [ ] Set up Python virtual environment for Tier 2 AI backend
- [ ] Confirm ACE-Step 1.5 loads and generates locally
- [ ] Set up CI/CD (GitHub Actions) for Win/Mac/Linux builds

### Deliverable
Empty desktop window with Wavy Labs title, dark theme applied, no crashes. Tier 1 models (demucs.cpp + NeuTTS Air) confirmed working standalone.

---

## Phase 1 — Core DAW + First AI (Months 1–3)

**Goal:** A functional DAW with the first AI features integrated. Stem separation and SFX generation working inside the app.

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
- [ ] Loop markers, transport bar, multi-track recording
- [ ] Undo/redo (Tracktion Engine built-in)

### Milestone 1.3 — CLAP/VST3 Plugin Host
- [ ] Plugin scanner, browser panel, add to track
- [ ] Plugin window display (native plugin UI)
- [ ] Plugin delay compensation + bypass

### Milestone 1.4 — Piano Roll
- [ ] `PianoRollView` — grid with note lanes
- [ ] Draw, select, move, resize notes
- [ ] Velocity editing, quantize, MIDI input recording

### Milestone 1.5 — Mixer
- [ ] `MixerView` with channel strips (fader, pan, mute, solo)
- [ ] `LevelMeter` — real-time RMS/peak display
- [ ] Send/return routing, master bus
- [ ] Audio export (bounce to WAV/MP3/FLAC)

### Milestone 1.6 — First AI Features (Tier 1 + Tier 2 Infrastructure)
- [ ] Define `wavy_ai.proto` gRPC service
- [ ] Generate C++ and Python gRPC stubs
- [ ] `AICommandQueue` + `GrpcClient` in C++
- [ ] Python FastAPI + gRPC sidecar server
- [ ] Auto-start Python backend on app launch
- [ ] **Stem Separation:** demucs.cpp (Tier 1, in-process C++)
  - Right-click audio clip → "Separate Stems"
  - 4-stem output placed as new tracks
- [ ] **Bark SFX/Ambient:** Bark integration in Python sidecar
  - AI Panel: text prompt → sound effect / ambient audio
  - `[laughs]`, `[sighs]`, `[music]` paralinguistic tags
  - 100+ speaker presets available

### Deliverable
A usable DAW with stem separation (Tier 1, instant) and SFX generation (Bark). Can record, arrange, use plugins, mix, and export. Shown to 5+ beta testers.

---

## Phase 2 — Full AI Suite (Months 4–6)

**Goal:** All core AI features integrated. Code-to-Music prototype working. Music generation, voice cloning, and audio-to-MIDI operational.

### Milestone 2.1 — Music Generation (ACE-Step 1.5)
- [ ] ACE-Step 1.5 integrated in Python sidecar
- [ ] `GenerateView` panel — text prompt, duration, genre, BPM, key
- [ ] Generated audio placed directly into timeline as editable tracks
- [ ] "Variations" button — generate multiple alternatives
- [ ] ACE-Step audio repainting: right-click section → "Regenerate bars 8–16"
- [ ] ACE-Step continuation: "Extend this track by 30 seconds"
- [ ] ACE-Step cover generation: drag reference → "Generate in this style"
- [ ] Prompt history / favorites

### Milestone 2.2 — Code-to-Music (Prototype)
- [ ] Llama 4 (via Ollama) integration for code generation
- [ ] System prompt with music theory knowledge + midiutil/pyo templates
- [ ] Sandbox executor (Python subprocess with safety constraints)
- [ ] MIDI generation via midiutil → FluidSynth SoundFont rendering
- [ ] **"Code View" tab** alongside Piano Roll and Arrangement views
- [ ] Toggle between generated audio and source code
- [ ] Edit code → regenerate audio instantly
- [ ] Export as standalone Python scripts

### Milestone 2.3 — Vocal Song Generation (YuE)
- [ ] YuE integrated in Python sidecar
- [ ] Lyrics input panel → full song with synchronized vocals
- [ ] Genre/style control, multilingual lyrics support

### Milestone 2.4 — Voice Cloning
- [ ] Chatterbox Turbo (zero-shot, MIT) — 3-second voice clone
- [ ] Chatterbox Multilingual — 23 language voice cloning
- [ ] Paralinguistic tags: [laugh], [cough], [chuckle] in generated speech
- [ ] RVC v2 (trained) — users provide training samples
- [ ] Qwen3-TTS VoiceDesign — create voices from text descriptions
- [ ] Voice library (save and load cloned voices)

### Milestone 2.5 — Audio-to-MIDI + Mastering
- [ ] basic-pitch (Spotify) integration — any audio → editable MIDI
- [ ] Workflow: generate audio → export as MIDI → edit in piano roll → re-render
- [ ] Matchering 2.0 — reference-based mastering
- [ ] Essentia — audio analysis for mixing decisions
- [ ] `MasteringView` — drag reference track, set loudness target, A/B preview

### Milestone 2.6 — LLM Orchestration
- [ ] Llama 4 (via Ollama) as orchestration LLM
- [ ] Natural language commands: "Generate a verse, separate the drums, add reverb"
- [ ] Multi-step workflow chaining: LLM → ACE-Step → Demucs → Matchering
- [ ] Intelligent model routing based on user intent

### Deliverable
Full-featured AI DAW. All core features working. Code-to-Music prototype functional. Public alpha release.

---

## Phase 3 — Differentiation (Months 7–9)

**Goal:** Polish unique features. Code-to-Music LoRA fine-tuning. Real-time jam mode. Community building.

### Milestone 3.1 — Code-to-Music LoRA
- [ ] Curate training dataset: Sonic Pi examples, MIDI datasets → midiutil code pairs
- [ ] Target: 10k–50k high-quality (prompt, code) pairs
- [ ] LoRA fine-tune Llama 4 Scout on music code generation
- [ ] Evaluate: code compiles, produces valid audio, matches prompt intent
- [ ] Code blocks chainable: intro_code + verse_code + chorus_code
- [ ] Version control integration — arrangements as diffable text

### Milestone 3.2 — Real-Time Jam Mode (MagentaRT)
- [ ] MagentaRT integration — AI-driven live accompaniment
- [ ] Dynamic style morphing via text/audio prompts
- [ ] 48 kHz stereo real-time streaming
- [ ] "Jam with AI" mode in DAW

### Milestone 3.3 — Advanced Editing
- [ ] Spectral selection → ACE-Step audio repainting UI
- [ ] UVR5 MDX-Net models as alternative stem separation algorithm
- [ ] Multi-model chaining UI (visual pipeline builder)
- [ ] ACE-Step LoRA fine-tuning from user's own songs

### Milestone 3.4 — Polish & Beta Launch
- [ ] App preferences (theme, directories, AI settings)
- [ ] Crash reporting, auto-update system
- [ ] Onboarding tutorial (first-launch walkthrough)
- [ ] Optional Tier 3 BYO-API-key connector (ElevenLabs, etc.)
- [ ] Public beta release on GitHub
- [ ] Discord community setup

### Deliverable
Public beta with unique Code-to-Music feature and real-time jam mode. Community feedback loop active.

---

## Phase 4 — Scale & Community (Post-MVP)

**Goal:** Sustainable product. Growing user base. Community-driven development.

### Features
- [ ] LoRA marketplace — community shares custom style models
- [ ] Optional cloud GPU tier for users without local GPU (BYO API key model)
- [ ] Cloud project sync and collaboration
- [ ] Fine-tune ACE-Step on curated datasets (proprietary Wavy Labs model)
- [ ] Mobile companion app (project browser, AI generation on-the-go)
- [ ] API for third-party integrations
- [ ] University/education tier pricing

### Growth
- [ ] Product Hunt launch
- [ ] YouTube demo videos (focus on Code-to-Music as differentiator)
- [ ] Music production YouTuber outreach
- [ ] Open source audio directory submissions
- [ ] Music tech accelerator/grant applications

---

## Version Targets

| Version | Phase | Description |
|---------|-------|-------------|
| v0.1 | Phase 0 | Empty app, builds on all platforms, Tier 1 models confirmed |
| v0.2 | Phase 1 | Core DAW + stem separation + Bark SFX |
| v0.3 | Phase 2 | Full AI suite: ACE-Step, Code-to-Music, YuE, voice cloning |
| v0.4 | Phase 3 | Code-to-Music LoRA, real-time jam, public beta |
| v1.0 | Phase 4 | Public stable release, community marketplace |
