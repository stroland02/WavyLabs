# Wavy Labs Research: Suno, ElevenLabs & Udio — Technology, Open Source & Integration Strategy

**Date:** March 2, 2026  
**Scope:** What Suno and ElevenLabs use/release as open source, ElevenLabs API integration plan, Udio technology analysis, and "Code-to-Music" pipeline design  
**Relation:** Supplements `RESEARCH_UPDATE_March2026.md` and `PLAN.md` v1.2

---

## 1. Suno: Technology & Open-Source Landscape

### 1.1 Proprietary Stack (Closed)

Suno does not publish official technical specifications. Their core music generation models — originally codenamed **Chirp** and now simply versioned as **Suno v3, v4, v4.5, v5** — are entirely proprietary and have never been open-sourced. There is no public API as of March 2026.

Based on observable behavior and third-party analysis, Suno's architecture is understood to be a **hybrid transformer + diffusion** pipeline:

- **Text Understanding Layer:** Interprets prompts, determines arrangement direction, section behavior, mood, genre, and instrumentation. Uses NLP to parse musical intent from natural language.
- **Audio Generation Layer:** Renders vocals, instruments, spatial mix, and sonic texture using diffusion models that generate raw audio waveforms (not MIDI).
- **Latent Diffusion Transformers:** Similar to what powers Stable Audio and ACE-Step, allowing direct waveform synthesis at high fidelity.
- **EnCodec-style quantization:** Audio representations are tokenized using neural codec approaches derived from Meta's EnCodec research.

Suno v5 (announced Sept 2025) achieved an ELO benchmark score of 1,293. The platform has reached nearly 100 million users with a $2.45 billion valuation after a $250 million Series C.

### 1.2 Suno's Open-Source Release: Bark

**Bark** is Suno's only open-source contribution — a **text-to-audio** model (not music generation).

| Property | Detail |
|----------|--------|
| Repo | `github.com/suno-ai/bark` (38.9k stars) |
| License | **MIT** (code + weights, commercial use allowed) |
| Architecture | GPT-style transformer, similar to AudioLM + Vall-E |
| Audio Codec | Quantized representation from **EnCodec** |
| Tokenizer | BERT (text → semantic tokens → coarse acoustic → fine acoustic) |
| VRAM | Full: ~12GB, Small: ~8GB, CPU offload: ~2GB |
| Output | 24kHz mono audio |
| Capabilities | Multilingual speech, laughter, sighing, crying, simple music, background noise, sound effects |
| Paralinguistic | Supports `[laughs]`, `[sighs]`, `[music]` tags in prompts |
| 100+ Speaker Presets | Built-in voice library across languages |
| HuggingFace | `suno/bark` — integrated with Transformers pipeline |

**Key architectural detail:** Bark uses a **three-stage transformer pipeline:**
1. **Text → Semantic tokens** (text encoder, BERT-based)
2. **Semantic → Coarse acoustic tokens** (autoregressive transformer)
3. **Coarse → Fine acoustic tokens** (autoregressive transformer → EnCodec decoder)

This is the same fundamental approach that later influenced models like Vall-E, Chatterbox, and others. Bark converts text prompts directly to audio without intermediate phoneme representation.

### 1.3 What Wavy Labs Should Take from Suno

**Architecture patterns to replicate (using open-source equivalents):**

| Suno Pattern | Open-Source Equivalent for Wavy Labs |
|--------------|--------------------------------------|
| Hybrid transformer + diffusion | **ACE-Step 1.5** (LM planner + Diffusion Transformer) |
| EnCodec audio tokenization | **EnCodec** (MIT, from Meta) or **HeartCodec** (from HeartMuLa) |
| GPT-style text-to-audio pipeline | **Bark** (MIT) for TTS/SFX; **YuE** for full song generation |
| "Scenes" multimodal input | LLM orchestration layer (Llama 4) with vision → prompt pipeline |
| Prompt → full song with vocals | **ACE-Step 1.5** + **YuE** combination |
| Audio repainting/inpainting | **ACE-Step 1.5** native audio repainting feature |

**Direct integration recommendation:** Bundle Bark as a Tier 2 sound effects / ambient audio generator alongside its existing role as a TTS model. Its ability to generate laughter, background noise, and simple musical elements makes it uniquely useful for **ad-lib generation** and **ambient soundscapes** in a DAW context.

---

## 2. ElevenLabs: Technology, Open Source & API Integration

### 2.1 ElevenLabs Open-Source Contributions

Unlike Suno, ElevenLabs has **not released any of its core AI models as open source**. Their models (Eleven v3, Multilingual v2, Flash v2.5, Turbo v2.5, Scribe v2, Eleven Music) are all proprietary and accessible only via their API.

What ElevenLabs **has** open-sourced:

| Project | Description | License |
|---------|-------------|---------|
| `elevenlabs-python` | Official Python SDK | MIT |
| `elevenlabs-js` | Official JavaScript/Node SDK | MIT |
| `elevenlabs-swift-sdk` | Official Swift SDK (iOS/macOS) | MIT |
| `elevenlabs-examples` | Example apps (Mac desktop app, React integration, dubbing, pronunciation) | MIT |
| `elevenlabs-ui` | Component library built on shadcn/ui for building agent UIs | MIT |
| ElevenLabs Agents SDK | TypeScript SDK for building voice agents | MIT |

ElevenLabs' CEO has publicly stated that audio models will be "commoditized" over time and that the company plans to work with open-source technologies and combine its audio expertise with other models' strengths. This signals a future where ElevenLabs may release models or contribute to open-source efforts, but nothing has materialized yet.

### 2.2 ElevenLabs API — Full Capability Map

The ElevenLabs API provides the most comprehensive commercial audio platform available. Here is the complete capability set relevant to Wavy Labs:

**Text-to-Speech Models:**

| Model | ID | Latency | Languages | Character Limit | Best For |
|-------|----|---------|-----------|-----------------|----------|
| Eleven v3 | `eleven_v3` | Standard | 70+ | 5,000 | Dramatic delivery, audiobooks, emotional dialogue |
| Multilingual v2 | `eleven_multilingual_v2` | Standard | 29 | 10,000 | Long-form, consistent quality |
| Flash v2.5 | `eleven_flash_v2_5` | ~75ms | 32 | 40,000 | Real-time, low latency (50% cheaper) |
| Turbo v2.5 | `eleven_turbo_v2_5` | ~250ms | 32 | 40,000 | Balance of quality and speed (50% cheaper) |

**Additional APIs:**

| Capability | Description | Key Features |
|------------|-------------|--------------|
| **Eleven Music** (`music_v1`) | Text-to-music generation | Full songs from prompts, vocals/instrumental, multilingual, 3s-5min, MP3 44.1kHz |
| **Music Inpainting** | Section-level editing | Modify specific sections, extend/trim, change lyrics, create loops |
| **Stem Separation** | Source separation API | Separate vocals, drums, bass, other from mixed audio |
| **Sound Effects** (`eleven_text_to_sound_v2`) | Text-to-SFX | Generate cinematic sound effects from descriptions |
| **Speech-to-Text** (Scribe v2) | Transcription | 90+ languages, speaker diarization (32 speakers), word timestamps |
| **Scribe v2 Realtime** | Live transcription | ~150ms latency, real-time streaming |
| **Voice Cloning (IVC)** | Instant voice clone | Upload samples → create voice |
| **Voice Design (TTV)** | Text-to-voice | Create voices from text descriptions |
| **Voice Changer (STS)** | Speech-to-speech | Replace one voice with another |
| **Voice Isolator** | Vocal isolation | Separate vocals from background |
| **Dubbing** | Translation + voice-over | Translate audio/video with matched voices |
| **Text-to-Dialogue** | Multi-speaker dialogue | Natural multi-speaker generation with v3 |
| **Forced Alignment** | Lyric timestamps | Precise word-level timing for sync |

### 2.3 ElevenLabs Integration Plan for Wavy Labs

**Tier 3 — Cloud API Integration (Premium Features)**

ElevenLabs should be integrated as the **premium cloud tier** for Wavy Labs, providing capabilities that exceed what local open-source models can deliver:

```
┌─────────────────────────────────────────────────────┐
│              WAVY LABS — ELEVENLABS TIER             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  MUSIC GENERATION (Eleven Music API)                │
│  ├─ Text prompt → full song (vocals + instrumental) │
│  ├─ Inpainting: edit specific sections in-place     │
│  ├─ Stem separation via API                         │
│  ├─ Lyric timestamps for sync                       │
│  └─ Composition plans for structured generation     │
│                                                     │
│  VOICE (Eleven v3 / Flash v2.5)                     │
│  ├─ Premium TTS with 70+ languages                  │
│  ├─ Instant voice cloning (upload → clone)          │
│  ├─ Voice design from text descriptions             │
│  ├─ Speech-to-speech voice conversion               │
│  ├─ Text-to-Dialogue (multi-speaker)                │
│  └─ Real-time streaming TTS (~75ms)                 │
│                                                     │
│  AUDIO TOOLS                                        │
│  ├─ Sound effects generation                        │
│  ├─ Voice isolation / vocal extraction              │
│  ├─ Dubbing / translation                           │
│  └─ Speech-to-text transcription (Scribe v2)        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Implementation approach using ElevenLabs Python SDK:**

```python
# Music generation
from elevenlabs.client import ElevenLabs

client = ElevenLabs(api_key="...")

# Generate music from prompt
audio = client.music.generate(
    prompt="Dreamy indie rock, reverb-soaked vocals, retro keys",
    duration_seconds=180,
    instrumental=False
)

# TTS with streaming for real-time preview
audio_stream = client.text_to_speech.stream(
    text="Verse one lyrics here...",
    voice_id="JBFqnCBsd6RMkjVDRZzb",
    model_id="eleven_v3",
    output_format="mp3_44100_128"
)

# Voice cloning
voice = client.voices.ivc.create(
    name="Custom Singer",
    files=["./sample_0.mp3", "./sample_1.mp3"]
)
```

**Pricing consideration:** ElevenLabs requires paid plans for Music API access. The API is billed per generation (music/SFX) or per character (TTS). This fits cleanly into Wavy Labs' Tier 3 cloud model — users who need premium quality or lack local GPU resources can access ElevenLabs through the app.

**Licensing:** Songs generated through the ElevenLabs Music API include commercial rights. Music was trained on licensed data in collaboration with labels, publishers, and artists. This is a significant advantage for users who need legally clear output. Enterprise plans are required for film and TV use.

### 2.4 What Open Source Models ElevenLabs' Quality Competes With

ElevenLabs doesn't open-source its own models, but the open-source community has models that approach or match its quality:

| ElevenLabs Feature | Best Open-Source Alternative | Quality Gap |
|-------------------|------------------------------|-------------|
| TTS (Eleven v3) | Chatterbox Turbo (MIT) — 63.75% preferred over ElevenLabs in blind tests | **Closed or surpassed** |
| TTS Quality | OpenAudio S1 (Fish Audio) — #1 on TTS-Arena2 | **Surpassed** (but NC license) |
| TTS Streaming | NeuTTS Air (Apache 2.0) — GGUF, runs on CPU | **Approaching** |
| Voice Cloning | Qwen3-TTS VoiceDesign / Chatterbox Multilingual | **Approaching** |
| Music Generation | ACE-Step 1.5 (MIT) | **Unknown** (different architecture) |
| Stem Separation | Demucs v4 (MIT) | **Comparable** |
| Sound Effects | Bark (MIT, Suno) | **ElevenLabs ahead** |
| Transcription | Whisper v3 (MIT, OpenAI) | **Comparable** |

---

## 3. Udio: Technology & Competitive Analysis

### 3.1 Architecture

Udio operates on a **Latent Diffusion Transformer (LDT)** architecture tailored for audio — the same fundamental approach as Stable Audio and ACE-Step, but with proprietary training and optimization.

**Udio v4 (2026 release) key specs:**
- 48kHz stereo audio output
- Extended context window for consistent songs up to 10 minutes
- Trained on authorized and licensed music (via UMG deal)

**Core features:**
- **Magic Edit (Inpainting):** Highlight a section in the spectral view → regenerate lyrics, melody, or instrumentation without altering the rest
- **Stem Separation 2.0:** Native export of discrete layers (Vocals, Bass, Drums, Other) with industry-leading phase coherence
- **Style Transfer:** Upload reference track → generate new compositions in that timbral palette without copying melody
- **Voice Cloning:** Upload 1 minute of audio → custom vocal avatar (added late 2025)
- **Audio-to-MIDI Conversion:** Export generated audio as MIDI for DAW editing
- **Multilingual Vocals:** 50+ languages with native accent emulation

### 3.2 Udio's Legal/Business Position

In October 2025, Universal Music Group and Udio settled copyright litigation and announced a landmark agreement:
- New platform launching in 2026 trained exclusively on **authorized and licensed music**
- Subscription service combining creation, consumption, and streaming
- Walled garden with fingerprinting, filtering, and artist consent mechanisms
- Revenue sharing for UMG artists and songwriters
- Warner Music Group also settled separately with Udio

This positions Udio as the first AI music platform operating under full major label licensing — a model Suno is also pursuing (Warner settled, Sony litigation ongoing).

### 3.3 Udio SDK/API (Enterprise)

Udio now offers a Python SDK and Enterprise API. Notable features:
- Programmatic music generation with prompt control
- Inpainting API for section-level editing
- Stem export with DAW integration
- Style transfer from reference tracks

### 3.4 What Wavy Labs Should Take from Udio

| Udio Feature | Wavy Labs Implementation |
|-------------|--------------------------|
| Spectral inpainting | ACE-Step 1.5 audio repainting (right-click → "Regenerate bars 8-16") |
| Native stem separation | Demucs v4 / demucs.cpp already in plan |
| Style transfer | ACE-Step 1.5 cover generation + LoRA fine-tuning |
| Audio-to-MIDI export | `basic-pitch` (Spotify, Apache 2.0) or `omnizart` for polyphonic transcription |
| Voice cloning | Chatterbox Turbo (MIT) + RVC v2 |
| Licensed training data | ACE-Step 1.5 already trained on licensed/royalty-free data |

---

## 4. The "Code-to-Music" Pipeline — A Unique Differentiator

### 4.1 Concept

The idea is to use an LLM to generate **executable code** from natural language prompts, where that code **programmatically synthesizes music**. This is fundamentally different from text-to-audio diffusion models. Instead of generating raw waveforms, you generate a **program** that creates music — giving you complete, deterministic, editable control.

Think of it as: **Prompt → LLM → Code → Audio Engine → Music**

This approach has several powerful advantages over pure neural audio generation:

1. **Deterministic & Reproducible:** Same code = same output. No random seeds or diffusion noise.
2. **Fully Editable:** The generated code IS the arrangement. Change BPM? Edit one number. Swap an instrument? Change one function call.
3. **Parametric Control:** Every musical parameter (tempo, key, velocity, timing, effects) is an explicit variable.
4. **Exportable as MIDI/Notation:** The code describes musical events, which trivially map to MIDI.
5. **Trainable:** Fine-tune an LLM on pairs of (description → music code) to learn musical patterns.
6. **Composable:** Chain multiple generated code blocks together for complex arrangements.

### 4.2 Target Languages/Frameworks

Several programming environments exist for algorithmic music creation:

| Language/Framework | Type | Best For | License |
|-------------------|------|----------|---------|
| **Sonic Pi** | Ruby-based live coding | Education, live performance, accessible syntax | MIT |
| **SuperCollider** | Audio programming language | Professional synthesis, granular control | GPL |
| **Csound** | C-based synthesis | Academic, extreme precision | LGPL |
| **FoxDot** | Python-based live coding | Python ecosystem integration | MIT |
| **TidalCycles** | Haskell-based patterns | Rhythmic pattern generation | GPL |
| **Tone.js** | JavaScript Web Audio | Browser-based, real-time | MIT |
| **FluidSynth** | MIDI → audio rendering | SoundFont playback | LGPL |
| **pyo** | Python audio DSP | Python-native synthesis | LGPL |
| **FAUST** | Functional audio DSP | DSP algorithm design, compiles to C++ | GPL |

### 4.3 Recommended Implementation: Python + pyo/FluidSynth

For Wavy Labs, the ideal code-to-music pipeline uses **Python** as the generated code language because:
- The Python sidecar already exists in the architecture
- LLMs are extremely proficient at generating Python
- Libraries like `pyo`, `midiutil`, and `FluidSynth` handle audio synthesis
- Generated code can be sandboxed for safety

**Pipeline architecture:**

```
┌──────────────────────────────────────────────────────────┐
│                CODE-TO-MUSIC PIPELINE                     │
│                                                          │
│  User Prompt                                             │
│  "Create a chill lo-fi beat, 85 BPM,                    │
│   Rhodes piano chords with vinyl crackle"                │
│         │                                                │
│         ▼                                                │
│  ┌─────────────────────┐                                 │
│  │  LLM (Llama 4 /     │  System prompt includes:       │
│  │  fine-tuned variant) │  - Music theory knowledge      │
│  │                      │  - Code generation templates   │
│  │  Generates Python    │  - Available instruments/FX    │
│  │  code that creates   │  - Safety constraints          │
│  │  music               │                                │
│  └─────────┬───────────┘                                 │
│            │                                             │
│            ▼                                             │
│  ┌─────────────────────┐                                 │
│  │  Sandbox Executor    │  Validates & runs generated    │
│  │  (Python subprocess) │  code in isolated environment  │
│  └─────────┬───────────┘                                 │
│            │                                             │
│            ▼                                             │
│  ┌─────────────────────┐  ┌──────────────────────┐      │
│  │  MIDI Generation     │  │  Direct Synthesis     │     │
│  │  (midiutil)          │  │  (pyo / Tone.js)      │     │
│  └─────────┬───────────┘  └──────────┬───────────┘      │
│            │                         │                   │
│            ▼                         ▼                   │
│  ┌─────────────────────┐  ┌──────────────────────┐      │
│  │  FluidSynth          │  │  WAV Output           │     │
│  │  (SoundFont render)  │  │  (direct audio)       │     │
│  └─────────┬───────────┘  └──────────┬───────────┘      │
│            │                         │                   │
│            └────────────┬────────────┘                   │
│                         ▼                                │
│            ┌──────────────────────┐                      │
│            │  Audio Track          │                     │
│            │  + MIDI data          │                     │
│            │  + Source code         │                     │
│            │  (all editable)        │                     │
│            └──────────────────────┘                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 4.4 Example: What the LLM Would Generate

**User prompt:** "Create a jazzy chord progression in Cmaj7, Dm7, G7, Cmaj7 at 90 BPM with a walking bass line"

**LLM-generated Python code:**

```python
from midiutil import MIDIFile
import fluidsynth

# Setup
midi = MIDIFile(2)  # 2 tracks: piano + bass
midi.addTempo(0, 0, 90)

# Track 0: Piano chords
chords = [
    [60, 64, 67, 71],  # Cmaj7
    [62, 65, 69, 72],  # Dm7
    [67, 71, 74, 77],  # G7
    [60, 64, 67, 71],  # Cmaj7
]
for bar, chord in enumerate(chords):
    for note in chord:
        midi.addNote(0, 0, note, bar * 4, 4, 80)

# Track 1: Walking bass
bass_notes = [48, 50, 52, 53, 50, 52, 53, 55, 55, 53, 52, 50, 48, 47, 45, 48]
for i, note in enumerate(bass_notes):
    midi.addNote(1, 0, note, i, 1, 90)

# Render to audio via FluidSynth
with open("output.mid", "wb") as f:
    midi.writeFile(f)

fs = fluidsynth.Synth()
sfid = fs.sfload("/path/to/jazz_soundfont.sf2")
fs.program_select(0, sfid, 0, 4)   # Rhodes piano
fs.program_select(1, sfid, 0, 32)  # Acoustic bass
# ... render to WAV
```

### 4.5 Training a Code-to-Music Model

To make the LLM excellent at generating music code, fine-tune it on:

**Dataset sources (all open/freely available):**
1. **Sonic Pi tutorials & examples** — thousands of annotated code snippets with musical descriptions
2. **SuperCollider documentation** — extensive example library
3. **MIDI datasets** (Lakh MIDI, MAESTRO) — convert to `midiutil` Python code programmatically
4. **Music theory textbooks** — formalize chord progressions, scales, rhythms as code patterns
5. **Algorave/live-coding community** — published performance code with descriptions
6. **Custom pairs** — Use ACE-Step/Suno to generate audio from prompts, then manually create equivalent code

**Fine-tuning approach:**
- Base model: **Llama 4 Scout** (already in plan for orchestration)
- Training format: `(prompt, code)` pairs where code produces the described music
- LoRA fine-tuning on ~10k-50k high-quality pairs
- Evaluation: Generated code must compile, produce valid MIDI/audio, and match prompt intent

### 4.6 Unique DAW Integration

No existing DAW offers a code-to-music workflow. This becomes a defining feature:

- **"Code View"** tab alongside Piano Roll and Arrangement views
- User can toggle between generated audio and the source code
- Editing the code regenerates audio instantly
- Code blocks can be chained: intro_code + verse_code + chorus_code
- Export as standalone Python scripts for reproducibility
- Version control friendly — musical arrangements become diffable text

---

## 5. Updated Integration Matrix

### Complete Tier Architecture with New Additions

| Feature | Tier | Engine | Source |
|---------|------|--------|--------|
| Noise reduction | 1 (C++) | ONNX Runtime | Existing |
| Stem separation | 1 (C++) | demucs.cpp / GGML | Existing |
| TTS / voice preview | 1 (C++) | NeuTTS Air (GGUF Q4) | Updated |
| Music generation (primary) | 2 (Python) | **ACE-Step 1.5** | Updated |
| Music generation (vocals) | 2 (Python) | **YuE** | Updated |
| Sound effects / ambient | 2 (Python) | **Bark** (Suno, MIT) | **NEW** |
| Code-to-music generation | 2 (Python) | **Llama 4 + midiutil/pyo** | **NEW** |
| Voice cloning (zero-shot) | 2 (Python) | Chatterbox Turbo + Qwen3-TTS | Updated |
| Voice cloning (trained) | 2 (Python) | RVC v2 | Updated |
| Audio repainting | 2 (Python) | ACE-Step 1.5 | Updated |
| Audio-to-MIDI | 2 (Python) | **basic-pitch** (Spotify) | **NEW** |
| Mastering | 2 (Python) | Matchering + Essentia | Existing |
| LLM orchestration | 2 (Python) | Llama 4 via Ollama | Existing |
| Real-time jam mode | 2/3 | MagentaRT | Phase 3+ |
| **Premium TTS** | **3 (Cloud)** | **ElevenLabs Eleven v3** | **NEW** |
| **Premium music gen** | **3 (Cloud)** | **ElevenLabs Eleven Music** | **NEW** |
| **Premium voice clone** | **3 (Cloud)** | **ElevenLabs IVC** | **NEW** |
| **Premium SFX** | **3 (Cloud)** | **ElevenLabs SFX API** | **NEW** |
| **Premium transcription** | **3 (Cloud)** | **ElevenLabs Scribe v2** | **NEW** |
| **Premium stem separation** | **3 (Cloud)** | **ElevenLabs Stem API** | **NEW** |
| Heavy generation (no GPU) | 3 (Cloud) | Cloud endpoint | Existing |

### ElevenLabs API Integration Points in Wavy Labs UI

```
┌─────────────────────────────────────────────────┐
│  WAVY LABS DAW — Feature Access                  │
│                                                  │
│  🟢 FREE (Local / Open Source)                   │
│  ├─ ACE-Step 1.5 music generation               │
│  ├─ Bark SFX / ambient generation               │
│  ├─ Code-to-Music generation                     │
│  ├─ Demucs stem separation                       │
│  ├─ NeuTTS Air / Chatterbox TTS                 │
│  ├─ RVC v2 voice cloning                         │
│  └─ Matchering mastering                         │
│                                                  │
│  ⭐ PREMIUM (ElevenLabs Cloud)                   │
│  ├─ Eleven Music: Licensed, studio-grade songs   │
│  ├─ Eleven v3 TTS: 70+ languages, emotional     │
│  ├─ Voice cloning: Instant from samples          │
│  ├─ Voice design: Create voice from description  │
│  ├─ Sound effects: Cinematic quality             │
│  ├─ Stem separation: Phase-coherent              │
│  ├─ Scribe v2: Production transcription          │
│  └─ Dubbing: Multilingual voice-over             │
│                                                  │
└─────────────────────────────────────────────────┘
```

---

## 6. License Summary (All New Additions)

| Component | License | Commercial Safe? |
|-----------|---------|-----------------|
| Bark (Suno) | MIT | ✅ Yes |
| ElevenLabs API | Proprietary (commercial license via API plans) | ✅ Yes (paid) |
| Eleven Music output | Commercial rights included (varies by plan) | ✅ Yes |
| basic-pitch (Spotify) | Apache 2.0 | ✅ Yes |
| midiutil | MIT | ✅ Yes |
| pyo | LGPL | ⚠️ Dynamic link only |
| FluidSynth | LGPL | ⚠️ Dynamic link only |
| Sonic Pi | MIT | ✅ Yes |
| Udio API | Proprietary | ❌ Not integrable (closed) |

---

## 7. Key Recommendations

### Immediate Actions (Phase 1-2)

1. **Integrate ElevenLabs Python SDK** into the Python sidecar as a premium cloud provider. Wrap all ElevenLabs calls behind an abstraction layer so the UI doesn't care whether audio comes from local or cloud.

2. **Add Bark** as a Tier 2 model for sound effects and ambient audio generation. It fills a gap between TTS and music generation.

3. **Begin Code-to-Music prototype** with Llama 4 + midiutil. Start collecting training data from Sonic Pi examples and MIDI datasets. This is a unique feature no competitor offers.

4. **Add basic-pitch** for audio-to-MIDI conversion — enables Udio-style MIDI export from any generated or imported audio.

### Medium-Term (Phase 2-3)

5. **Fine-tune a Code-to-Music LoRA** on Llama 4 using curated (prompt → code) pairs. Target 10k+ examples.

6. **Build "Code View" tab** in the DAW UI alongside Piano Roll and Arrangement views.

7. **Add ElevenLabs Music Inpainting** as a premium editing feature — users can highlight sections and regenerate via API.

### Strategic Notes

8. **Do NOT integrate Udio** — their platform is a walled garden with no public API for third-party integration. The UMG licensing deal is exclusive to their platform.

9. **Monitor Suno API** — as of March 2026, Suno still has no public developer API. If/when they release one, evaluate for Tier 3 cloud integration alongside ElevenLabs.

10. **The Code-to-Music pipeline is the true differentiator.** Every competitor (Suno, Udio, ElevenLabs) generates opaque audio. Wavy Labs generating *editable code that produces music* is fundamentally different and appeals to technical musicians, producers, and the growing "vibe coding" culture.
