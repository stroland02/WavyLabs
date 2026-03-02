# Wavy Labs — Model Research Update

**Date:** March 2, 2026  
**Scope:** Newer open-source models across music generation, stem separation, voice/TTS, and real-time audio — models released or significantly updated since the v1.2 plan was written.

---

## 1. Music Generation — Major New Contenders

Your plan lists **DiffRhythm** (primary) and **MusicGen** (secondary). The landscape has shifted significantly since then. Three models now rival or exceed commercial systems like Suno, and all carry permissive licenses.

### ACE-Step 1.5 ⭐ Top Recommendation

| Property | Value |
|----------|-------|
| Released | January 28, 2026 |
| License | **MIT** (code and weights — trained on licensed/royalty-free data) |
| Architecture | Hybrid LM planner + Diffusion Transformer (DiT) |
| Speed | Full song in **<2 sec** on A100, **<10 sec** on RTX 3090 |
| VRAM | **<4 GB** — runs on consumer hardware |
| Duration | 10 seconds to 10 minutes |
| Languages | 50+ for lyrics/prompts |
| Repo | `github.com/ace-step/ACE-Step-1.5` |

**Why this matters for Wavy Labs:** ACE-Step 1.5 outperforms most commercial music models on the SongEval benchmark (scoring 8.09, above Suno v5). Its LM planner uses chain-of-thought to convert simple prompts into structured song blueprints — metadata, lyrics, arrangement — which then condition the DiT for audio synthesis. It supports LoRA fine-tuning from just a few reference songs, cover generation, audio repainting (regenerate specific bars), and vocal-to-BGM conversion. The MIT license and royalty-free training data make it safe for a commercial tier. At <4 GB VRAM, it could potentially be a Tier 1 candidate with further optimization.

**Impact on plan:** ACE-Step 1.5 should **replace DiffRhythm as the primary music generation engine**. DiffRhythm remains a solid fallback, but ACE-Step is faster, more controllable, and better quality. Its built-in LM planner partially overlaps with the Llama 4 orchestration role in the agentic pipeline — worth investigating whether ACE-Step's planner can replace or supplement that stage.

### YuE (乐) — Full-Song with Vocals

| Property | Value |
|----------|-------|
| Released | January 28, 2025 (Apache 2.0 since January 30, 2025) |
| License | **Apache 2.0** |
| Architecture | LLaMA2-based, trillions of tokens, autoregressive |
| Output | Up to 5 min with synchronized vocals + accompaniment |
| Repo | `github.com/multimodal-art-projection/YuE` |

YuE is the first open-source lyrics-to-song model that generates full songs with coherent vocals, matching some proprietary systems in musicality. It supports multilingual lyrics, voice style transfer, and bidirectional generation. It can run on as little as 8 GB VRAM using quantized models. A Gradio UI (YuE-UI) supports batch generation, audio prompts, continuation, and session save/load.

**Impact on plan:** YuE is the best option when **vocal generation** is needed (DiffRhythm and ACE-Step also generate vocals, but YuE's vocals are more controllable via lyrics). Consider as a Tier 2 option specifically for the "generate song with vocals from lyrics" workflow.

### HeartMuLa 3B — Lyrics-Conditioned Generation

| Property | Value |
|----------|-------|
| Released | January 2026 (RL-optimized version: January 23, 2026) |
| License | **Apache 2.0** |
| Architecture | Music language model + HeartCodec (12.5 Hz codec) |
| Extras | HeartCLAP (audio-text alignment), HeartTranscriptor (lyrics transcription) |
| Repo | `github.com/HeartMuLa/heartlib` |

HeartMuLa claims to be the best open-source model for lyrics controllability. The 7B internal version reportedly matches Suno in musicality, fidelity, and controllability — but only the 3B open-source version is available. Ships with its own audio codec, CLAP model, and lyrics transcriber. Reinforcement learning refined version available.

**Impact on plan:** Worth tracking, especially the CLAP model (HeartCLAP) which could enhance audio-text retrieval in the agentic pipeline. However, ACE-Step 1.5 is more mature and better documented for integration.

### MagentaRT (Google) — Real-Time Generation

| Property | Value |
|----------|-------|
| Released | June 2025 |
| License | **Apache 2.0** (with some bespoke terms) |
| Architecture | 800M parameter autoregressive transformer |
| Output | 48 kHz stereo, real-time streaming |
| Training | ~190k hours of stock music (mostly instrumental) |
| Repo | `github.com/magenta/magenta-realtime` |

MagentaRT is designed for live, real-time music generation — users can morph genres and styles dynamically by adjusting a style embedding. Currently runs at 1.6x real-time factor on Colab TPUs. Local inference is coming. Uses SpectroStream codec (successor to SoundStream) and MusicCoCa joint music+text embedding.

**Impact on plan:** Not a replacement for ACE-Step (limited to instrumental, 10-sec context window, no vocal support), but uniquely valuable for a **live jam/improv mode** in the DAW — a differentiated feature no other DAW offers. Could be a Phase 3+ addition.

### InspireMusic (Alibaba) — Unified Audio Framework

Open-sourced January 2026. Built on FunAudioLLM framework. Supports music, song, and general audio generation. Pre-trained models at 24 kHz and 48 kHz. Less documented than ACE-Step but worth monitoring as the Alibaba ecosystem (Qwen3-TTS, CosyVoice) matures.

---

## 2. Stem Separation — Demucs Still Dominates

Your plan's choice of **demucs.cpp** (Tier 1) and **Demucs v4 Python** (Tier 2 fallback) remains correct. No open-source model has dethroned Demucs for stem separation quality.

Key developments:

- **Demucs htdemucs_ft remains the best open-source stem separator** at ~9.2 dB SDR. Multiple independent reviews (MusicRadar, StemSplit, LANDR) confirm this as of early 2026.
- **MDX-Net / MDX'23** architectures are used alongside Demucs in tools like UVR5 (Ultimate Vocal Remover) and AudioStrip — different architectures can perform better for specific stems (e.g., MDX-Net for vocals, Demucs for drums).
- **Meta's SAM Audio** was tested for stem separation but found impractical — it only understands bass, drums, and vocals, requires separate runs per stem, takes 5+ hours for a 4-minute track, and has no "other" stem concept. Not a viable alternative.
- **AudioShake** (proprietary) claims ~2 dB better SDR than Demucs for vocals, but it's closed-source.
- **Dual-path Mamba networks** (reported in Nature) show better benchmark scores than Demucs, but no production-ready open-source implementation exists yet.

**Recommendation:** No changes needed. The demucs.cpp / ONNX approach is still the right call. Consider also bundling **UVR5's MDX-Net** models as an alternative algorithm option (UVR5 lets users pick which model to use per stem type — advanced users appreciate this).

---

## 3. Voice / TTS — Landscape Has Exploded

This is the area with the most dramatic change since your plan. Several new models significantly outperform your current selections.

### Chatterbox (Resemble AI) — Now a Full Family ⭐

| Variant | Params | Languages | Features | License |
|---------|--------|-----------|----------|---------|
| Chatterbox Original | 500M | English | Emotion exaggeration control, zero-shot cloning | **MIT** |
| Chatterbox Multilingual | 550M | **23 languages** | Full voice cloning + emotion control | **MIT** |
| Chatterbox Turbo | **350M** | English (23 via multilingual) | Sub-200ms latency, paralinguistic tags ([laugh], [cough], [chuckle]) | **MIT** |

Chatterbox has been benchmarked against ElevenLabs, with 63.75% of evaluators preferring Chatterbox in blind tests. The Turbo variant is particularly interesting — 350M params, 1-step diffusion decoder (distilled from 10 steps), and built-in PerTh watermarking for responsible AI. It's the fastest open-source TTS model available.

**Impact on plan:** Your plan already lists Chatterbox but underspecifies it. The Turbo variant is an excellent **Tier 1 candidate** alongside Kokoro-82M — its 350M size is larger but its quality is significantly better. For Tier 2, the Multilingual variant handles 23 languages with zero-shot cloning.

### NeuTTS Air (Neuphonic) — True On-Device TTS ⭐ Tier 1 Candidate

| Property | Value |
|----------|-------|
| Params | 748M (0.5B Qwen2 backbone) |
| License | **Apache 2.0** |
| Format | **GGUF quantizations (Q4/Q8)** — runs via llama.cpp |
| Cloning | 3 seconds of reference audio |
| Inference | Real-time on CPU, runs on Raspberry Pi |
| Languages | English (multilingual NeuTTS-Nano variants available) |
| Repo | `github.com/neuphonic/neutts` |

NeuTTS Air is the first TTS model designed from the ground up for on-device deployment via GGUF. It uses the same llama.cpp infrastructure as your demucs.cpp approach. The Q4 quantization brings it down to well under 1 GB. It runs on CPU in real time. Built-in Perth watermarking. It's the spiritual successor to what Kokoro-82M promises — but with a more mature GGUF deployment story.

**Impact on plan:** NeuTTS Air should **replace Kokoro-82M as the primary Tier 1 TTS candidate**. The GGUF format means it uses the same infrastructure (llama.cpp/GGML) as your Tier 1 demucs.cpp, simplifying the build system. Kokoro-82M is smaller (82M vs 748M) but NeuTTS Air's GGUF Q4 quantization makes the practical size difference minimal, and quality is substantially better.

### OpenAudio S1 / S1-mini (Fish Audio) — Top-Ranked TTS

| Variant | Params | License (Weights) | Notes |
|---------|--------|-------------------|-------|
| OpenAudio S1 | 4B | **CC-BY-NC-SA-4.0** | #1 on TTS-Arena2, full emotion/tone control |
| OpenAudio S1-mini | 0.5B | **CC-BY-NC-SA-4.0** | Distilled, Hugging Face Space available |
| Fish Speech v1.5 | — | **CC-BY-NC-SA-4.0** | Previous generation, still excellent |

OpenAudio S1 (rebranded from Fish Speech) achieved the #1 ranking on TTS-Arena2 with 0.008 WER and 0.004 CER on English — the best scores of any TTS model. It supports fine-grained emotion/tone markers (angry, sad, excited, sarcastic, etc.) and zero-shot cloning. Code is Apache 2.0 but weights are CC-BY-NC-SA-4.0 (non-commercial).

**Impact on plan:** The NC license limits this to the free/open-source tier only. Excellent quality for non-commercial use. S1-mini at 0.5B is a strong Tier 2 option.

### Qwen3-TTS — Your Plan Is Correct, But More Detail

Released January 2026. The family includes multiple specialized models: Base (3-second voice cloning), VoiceDesign (create voices from text descriptions), CustomVoice (9 preset voices). All support 10 languages and streaming. The dual-track LM architecture enables ultra-low first-packet latency.

**Impact on plan:** Your plan correctly lists Qwen3-TTS as Tier 2. It's a strong choice for zero-shot voice synthesis. Consider specifying the VoiceDesign variant for the "create voice from description" workflow (e.g., "warm male baritone with slight reverb").

### Other Notable TTS Models

| Model | Params | License | Key Feature |
|-------|--------|---------|-------------|
| CosyVoice2-0.5B (Alibaba) | 0.5B | Apache 2.0 | 150ms streaming latency, emotional control |
| IndexTTS-2 | — | Open source | Precise duration control, emotion-timbre disentanglement |
| VibeVoice-1.5B (Microsoft) | 1.5B | — | 90 min multi-speaker, 64K context |
| Kani-TTS-2 | 400M | Open source | 3 GB VRAM, audio-as-language architecture |
| Dia2 (Nari Labs) | — | Apache 2.0 | Streaming dialogue TTS, natural turn-taking |

### RVC v3 — Still "Coming Soon"

The RVC project's README still says "Please look forward to the pretrained base model of RVCv3, which has larger parameters, more training data, better results." As of March 2026, RVCv3 pretrained weights have **not been released**. The WebUI repo (34.6k stars) is actively maintained but the latest releases are still based on v2 models. Your plan should note this — RVC v2 is what's actually available and should be listed as such.

---

## 4. Revised Model Recommendations

### Tier 1 — Embedded C++ (ships in app)

| Feature | Current Plan | Recommended Update |
|---------|-------------|-------------------|
| Stem separation | demucs.cpp (GGML) | **No change** — still the best option |
| Noise reduction | ONNX Runtime | **No change** |
| TTS / voice preview | Kokoro-82M (ONNX) | **NeuTTS Air** (GGUF Q4, Apache 2.0) — same GGML stack as demucs.cpp, better quality |

### Tier 2 — Python Sidecar

| Feature | Current Plan | Recommended Update |
|---------|-------------|-------------------|
| Music generation (primary) | DiffRhythm | **ACE-Step 1.5** (MIT, <4 GB VRAM, faster, better quality) |
| Music generation (vocals) | MusicGen | **YuE** (Apache 2.0, full vocal songs from lyrics) |
| Mastering | Matchering + Essentia | **No change** |
| Voice cloning (training) | RVC v3 | **RVC v2** (v3 not released yet) + Chatterbox Multilingual (MIT, 23 languages, no training required) |
| Voice cloning (zero-shot) | Qwen3-TTS / Chatterbox | **Chatterbox Turbo** (MIT, sub-200ms, paralinguistic tags) + Qwen3-TTS |
| LLM orchestration | Llama 4 via Ollama | **No change** (though ACE-Step 1.5's built-in LM planner may reduce reliance on this) |

### Tier 3 — Cloud

No changes recommended. The same tiered approach applies.

---

## 5. New Opportunities Not in Original Plan

### ACE-Step's Built-In Editing Features

ACE-Step 1.5 supports audio repainting (selectively regenerate bars), cover generation, vocal-to-BGM conversion, and continuation — all from the same model. These features map directly to DAW workflows:

- **Right-click a section → "Regenerate bars 8–16"** (repainting)
- **Drag reference track → "Generate cover in this style"** (cover generation)
- **"Extend this track by 30 seconds"** (continuation)

### Real-Time Jam Mode (MagentaRT)

No DAW currently offers AI-driven live accompaniment. MagentaRT's real-time streaming architecture could power a "jam with AI" mode where the DAW generates backing tracks that morph in real time based on text or audio style prompts. This is a unique differentiator.

### Paralinguistic Tags (Chatterbox Turbo)

Chatterbox Turbo's native support for [laugh], [cough], [chuckle], [sigh] etc. enables more natural vocal generation for music production — ad-libs, vocal textures, and non-speech vocalizations that are commonly needed in modern production.

---

## 6. License Summary (Updated)

| Model | License | Commercial OK? |
|-------|---------|---------------|
| ACE-Step 1.5 | MIT | ✅ Yes |
| YuE | Apache 2.0 | ✅ Yes |
| HeartMuLa 3B | Apache 2.0 | ✅ Yes |
| MagentaRT | Apache 2.0 (+ bespoke terms) | ✅ Yes (review terms) |
| DiffRhythm | Apache 2.0 | ✅ Yes |
| Chatterbox (all variants) | MIT | ✅ Yes |
| NeuTTS Air | Apache 2.0 | ✅ Yes |
| Qwen3-TTS | Open source | ✅ Check specific model variant |
| OpenAudio S1 / Fish Speech | Code: Apache 2.0, Weights: **CC-BY-NC-SA-4.0** | ❌ Weights are non-commercial |
| MusicGen | Code: MIT, Weights: **CC-BY-NC-4.0** | ❌ Weights are non-commercial |
| RVC v2 | MIT | ✅ Yes |
| Demucs / demucs.cpp | MIT | ✅ Yes |
| Matchering | GPLv3 | ✅ Yes (copyleft) |
| Essentia | AGPL v3 | ⚠️ Copyleft (SaaS implications) |

---

## 7. Updated AI Task Routing Table

| AI Task | Tier | Implementation (Updated) |
|---------|------|--------------------------|
| Noise reduction | Tier 1 | ONNX Runtime C++ (no change) |
| Stem separation | Tier 1 | demucs.cpp / GGML (no change) |
| TTS / voice preview | Tier 1 | **NeuTTS Air GGUF Q4** (replaces Kokoro-82M) |
| Music generation | Tier 2 | **ACE-Step 1.5** (replaces DiffRhythm as primary) |
| Music generation (vocal songs) | Tier 2 | **YuE** (new addition) |
| Mastering | Tier 2 | Matchering + Essentia (no change) |
| Voice cloning (zero-shot) | Tier 2 | **Chatterbox Turbo** + Qwen3-TTS |
| Voice cloning (trained) | Tier 2 | **RVC v2** (v3 not yet available) |
| Audio repainting/editing | Tier 2 | **ACE-Step 1.5** (new capability) |
| Real-time jam mode | Tier 2/3 | **MagentaRT** (new capability, Phase 3+) |
| LLM orchestration | Tier 2 | Llama 4 via Ollama (no change) |
| Heavy generation (no GPU) | Tier 3 | Cloud endpoint (no change) |
