# TRP 1 — AI Content Generation Challenge — Submission Report

**Project:** trp1-ai-artist (AI Content Generator)  
**Challenge:** Environment setup, codebase exploration, content generation  
**Theme:** Ethiopia + 10 Academy learning

---

## 1. Environment Setup Documentation

### APIs Configured

| Provider | Purpose | Status |
|----------|---------|--------|
| **Google Gemini** | Music (Lyria), Video (Veo), Image (Imagen) | ✅ Configured in `.env` as `GEMINI_API_KEY` |
| **AIMLAPI** | Music with vocals (MiniMax) | ✅ Optional; configured as `AIMLAPI_KEY` if used |
| **xAI (Grok)** | Video (Grok Imagine) | Optional; `XAI_API_KEY` in `.env` for video when Veo quota is exceeded |
| **KlingAI** | Video (high quality) | Optional; `KLINGAI_API_KEY` / `KLINGAI_SECRET_KEY` |
| **Replicate** | Video (Luma Ray) | Optional; `REPLICATE_API_TOKEN` |

### Installation Steps Completed

1. **Clone and install**
   - `uv sync` (or `pip install -e .`) in project root.
2. **Environment**
   - `.env` created from `.env.example` with API keys (not committed).
3. **Verification**
   - `uv run ai-content --help`
   - `uv run ai-content list-providers`
   - `uv run ai-content list-presets`

### Setup Issues & Resolutions

| Issue | Resolution |
|-------|------------|
| **Windows UnicodeEncodeError** (Lyria logs with emojis) | Run once per terminal: `chcp 65001` and `$env:PYTHONIOENCODING = "utf-8"` before `uv run ai-content ...` |
| **Veo video generation** | `google.genai.types` has no attribute `GenerateVideoConfig` — SDK/version mismatch; config fix applied (person_generation omitted). Video still blocked by 429 quota or SDK; Grok/xAI can be used as alternative. |

---

## 2. Codebase Understanding

### Architecture Overview

The **ai_content** package is a multi-provider AI content framework:

- **CLI / User code** → **ProviderRegistry** (lyria, minimax, veo, kling, grok, replicate, svd, …) → **Provider protocols** (MusicProvider, VideoProvider, ImageProvider) → **Vendor APIs** (Google genai, AIMLAPI, Kling, xAI, Replicate, etc.).

Design principles:

- **Protocol-based interfaces** (PEP 544) for Music/Video/Image providers.
- **Registry pattern**: providers self-register via decorators (e.g. `@ProviderRegistry.register_music("lyria")`).
- **Async-first** I/O for concurrent calls and non-blocking operation.
- **Result objects** (`GenerationResult`) instead of exceptions for generation outcomes.
- **Job tracking** (SQLite) for long-running jobs, duplicate detection, and cost awareness.

### Package Structure (`src/ai_content/`)

| Module | Purpose |
|--------|---------|
| **cli/** | Typer CLI: `music`, `video`, `list-providers`, `list-presets`, `music-status`, `jobs`, `jobs-stats`, `jobs-sync`. |
| **config/** | YAML/config loading; Pydantic settings (API keys, provider options). |
| **core/** | Protocols, `ProviderRegistry`, `GenerationResult`, exceptions, `job_tracker`. |
| **integrations/** | Archive.org, FFmpeg/media, YouTube upload. |
| **pipelines/** | Orchestration: base, music, video, full (music + image + video + merge). |
| **presets/** | Music and video style presets (prompts, BPM, aspect ratios). |
| **providers/** | Implementations by vendor: google (Lyria, Veo, Imagen), aimlapi (MiniMax), kling, xai (Grok), replicate, svd. |
| **utils/** | File handlers, lyrics parser, retry. |

### Provider System

- **Music:** Lyria (instrumental, real-time), MiniMax (vocals, lyrics, reference audio).
- **Video:** Veo, Kling, Grok (xAI), Replicate (Luma Ray), SVD; several support image-to-video.
- **Image:** Imagen (Google).
- Pipelines (`pipelines/`) orchestrate multi-step workflows and optional provider comparison; **full** pipeline does music + image + video + FFmpeg merge.

### Presets

- **Music:** jazz, blues, ethiopian-jazz, cinematic, electronic, ambient, lofi, rnb, salsa, bachata, kizomba (BPM and mood in `presets/music.py`).
- **Video:** nature, urban, space, abstract, ocean, fantasy, portrait (aspect ratio and duration in `presets/video.py`).

Detailed exploration is in **`docs/PART2_CODEBASE_EXPLORATION.md`** and **`docs/architecture/ARCHITECTURE.md`**.

---

## 3. Generation Log

### Commands Executed

**Audio (Lyria — instrumental)**

1. **10 Academy Ethiopia theme**
   ```powershell
   uv run ai-content music --prompt "Hopeful inspirational music for learning, 10 Academy Ethiopia" --provider lyria --duration 20 --bpm 90
   ```
   **Result:** ✅ WAV exported (e.g. `exports\lyria_20260202_131850.wav`, ~3.30 MB, 20s).

2. **Jazz preset**
   ```powershell
   uv run ai-content music --prompt "Jazz" --style jazz --provider lyria --duration 20
   ```
   **Result:** ✅ WAV exported (e.g. `exports\lyria_20260202_131927.wav`, ~3.30 MB, 20s).

3. **Custom Ethiopian-inspired prompt** (shorter to avoid Lyria “no chunks” with long presets)
   ```powershell
   uv run ai-content music --prompt "Warm Ethiopian-inspired melody, hopeful and uplifting, 10 Academy learning journey, gentle piano and soft strings" --provider lyria --duration 20 --bpm 85
   ```
   **Result:** ✅ WAV exported (~3.30 MB, 20s).

**Audio with vocals (MiniMax)** — if AIMLAPI key is set

- Lyrics file: **`lyrics_10_academy_ethiopia.txt`** (Ethiopia + 10 Academy theme, with `[Verse]` / `[Chorus]` tags).
- Command:
  ```powershell
  uv run ai-content music --prompt "Inspirational pop, hopeful, 10 Academy Ethiopia" --provider minimax --lyrics lyrics_10_academy_ethiopia.txt
  ```
- Status/download: `uv run ai-content music-status <generation_id> --output exports/minimax_10_academy.mp3`

**Video**

- **Veo:** Config fixed (person_generation omitted). Blocked by SDK (`GenerateVideoConfig`) or 429 quota.
- **Grok:** Requires `XAI_API_KEY` in `.env`. Example:
  ```powershell
  uv run ai-content video --prompt "Students in Ethiopia learning..." --provider grok --duration 5 --aspect 16:9
  ```

**Music video (bonus)**

- When a video file is available (e.g. from Grok or Veo):
  ```powershell
  ffmpeg -i exports\video.mp4 -i exports\lyria_20260202_131850.wav -c:v copy -c:a aac -shortest exports\10_academy_music_video.mp4
  ```

### Artifacts Summary

| Type | File(s) | Status |
|------|---------|--------|
| Audio 1 | Lyria — “10 Academy Ethiopia” instrumental (20s WAV) | ✅ |
| Audio 2 | Lyria — jazz preset (20s WAV) | ✅ |
| Audio 3 | Lyria — Ethiopian-inspired custom prompt (20s WAV) | ✅ |
| Lyrics | `lyrics_10_academy_ethiopia.txt` | ✅ |
| Video | Veo (SDK/quota); Grok (optional with XAI_API_KEY) | ⚠️ Blocked / optional |
| Music video | After video is generated | ⏳ |

---

## 4. Challenges & Solutions

| Challenge | Cause | Solution / Workaround |
|-----------|--------|------------------------|
| **Windows UnicodeEncodeError** | Lyria logs use emojis; Windows console (e.g. cp1252) | `chcp 65001` and `$env:PYTHONIOENCODING = "utf-8"` before running CLI. |
| **Lyria “no audio” with long preset** | Ethiopian-jazz preset prompt very long; Lyria returned 0 chunks | Use shorter custom prompts (e.g. “Warm Ethiopian-inspired melody, hopeful and uplifting, 10 Academy learning journey…”). |
| **Veo video generation** | `google.genai.types` missing `GenerateVideoConfig` and/or 429 quota | Align `google-genai` with Veo provider expectations; use Grok (`XAI_API_KEY`) or other video providers as alternative. |
| **WAV playback** | Earlier runs had invalid WAV header | Fix applied in pipeline; exports now produce playable WAV. |

---

## 5. Insights & Learnings

- **Exploration:** Reading `src/ai_content/providers/` and `core/registry.py` was the fastest way to see which providers exist and how they register. Examples in `examples/` and the Makefile gave ready-to-run workflows.
- **Presets vs prompts:** Long preset prompts (e.g. ethiopian-jazz) can yield no Lyria output; shorter, focused custom prompts worked reliably.
- **Multi-provider design:** Registry + protocols make it easy to add or switch providers (e.g. Grok for video when Veo is unavailable) without changing core CLI or pipelines.
- **Job tracking:** Duplicate detection and job-status commands are useful for expensive or long-running providers (e.g. MiniMax).
- **Improvements:** Clearer SDK/version requirements in README or `pyproject.toml` for Google Veo would reduce setup friction; optional “short prompt” variants for presets could improve Lyria reliability.

---

## 6. Links

- **YouTube:** _[Add your YouTube link(s) here when you upload — title format: `[TRP1] Your Name - Content Description`]_
- **Repository:** _[Add your GitHub repo link with exploration artifacts and this submission]_

---

## 7. Related Documentation in This Repo

| Document | Description |
|----------|-------------|
| `README.md` | Quick start, features, CLI, presets, job tracking. |
| `docs/PART2_CODEBASE_EXPLORATION.md` | Package structure, providers, presets, CLI options. |
| `docs/PART3_GENERATION_LOG.md` | Step-by-step generation log, commands, challenges. |
| `docs/architecture/ARCHITECTURE.md` | System design, protocols, registry, async. |
| `docs/TRP1_AI_Content_Generation_Challenge.md` | Challenge brief and rubric. |
| `docs/TRP 1 - MCP Setup Challenge.md` | MCP setup and rules configuration. |
| `docs/FREE_ALTERNATIVES.md` | Free-tier and alternative API options. |
| `.agent/rules/RULES.md` | Agent rules. |
| `.agent/skills/`, `.agent/workflows/` | Skills and workflows for music/video generation. |

---

**End of submission report.**
