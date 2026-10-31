# LitePack — 02 · Technical Specification

> Part 2 of 3. Read `01-PRODUCT-BRIEF.md` first. This file is the **reference** for how everything works; `03-BUILD-PLAN.md` says **in what order** to build it.
> The code blocks are **skeletons**: they show signatures, data flow and the tricky parts. Fill in the rest following the stated rules.

---

## 1. Architecture

```
 TEACHER LAPTOP (online, occasionally)         STATIC HOSTING (GitHub Pages)          LEARNER PHONE (offline after 1st visit)
┌───────────────────────────────────┐        ┌─────────────────────────────┐        ┌──────────────────────────────────────┐
│ samples/<id>/lesson.toml + video   │        │ PWA app shell  (~100 KB gz) │──once─▶│ Service-worker cache: app shell      │
│        │  $ litepack build         │        │ packs/index.json  (catalog) │        │ IndexedDB: pack zips + manifests     │
│        ▼                           │  copy  │ packs/<id>.litepack.zip     │──────▶ │ Library · Lesson · Summary · Practice│
│ ffprobe → ffmpeg (audio, frames)   │ ─────▶ └─────────────────────────────┘        │ · Listen · Settings · About · Studio │
│ → faster-whisper / captions import │                                                └──────────────────▲───────────────────┘
│ → keyframe selection → WebP        │   …or hand the .zip over directly: USB, Bluetooth, WhatsApp, Files│
│ → LLM enrich + translate (optional)│ ──────────────────────────────────────────────────────────────────┘
│ → manifest.json → zip → report/VTT │
└───────────────────────────────────┘
```

**Principles**
- Heavy work happens once, on the teacher's machine. The learner's phone only reads small files.
- The pack is the contract. The pipeline writes `litepack/1` packs and the app reads them. Both sides validate.
- Nothing in the app needs a server: no API keys, accounts or tracking.
- Every AI output is cached, reviewable and labelled.

---

## 2. Tech stack

`VERIFY` versions at setup time; minimum versions are given where they matter.

### Teacher pipeline (`pipeline/`)
| Tool | Version | Purpose |
|---|---|---|
| Python | **3.12** (pin via `uv python pin 3.12`) | `ctranslate2` wheels lagged behind Python 3.13 in 2025; 3.12 is the safe choice |
| uv | latest | Environments, dependencies, `uv run` |
| ffmpeg + ffprobe | ≥ 6, **built with libopus** | Probe, audio extraction, frame sampling. FFmpeg 9.0 (Aug 2026) **removed `-vsync`**; our commands don't use it. If you copy old snippets, use `-fps_mode` instead. |
| faster-whisper | ≥ 1.2 | Speech-to-text (CTranslate2) |
| numpy, Pillow ≥ 10, ImageHash ≥ 4.3 | — | Frame diffs, WebP encoding, perceptual dedupe |
| openai (Python SDK) | ≥ 1.50 | Used as a **generic OpenAI-compatible client** |
| pydantic | v2 | Manifest models (the schema source of truth) and LLM output validation |
| textstat | latest | Flesch–Kincaid grade (English only) |
| python-dotenv | latest | Loads `pipeline/.env` |
| pytest | latest | Tests |

### Learner app (`app/`)
| Tool | Version | Purpose |
|---|---|---|
| Node.js | 24 LTS (26 becomes LTS on Oct 28, 2026; either works) | Tooling |
| Vite | **7.x preferred** (see note) | Dev server and build |
| preact + @preact/preset-vite + TypeScript | latest | UI |
| vite-plugin-pwa | 1.x | Service worker (Workbox `generateSW`) and web manifest |
| @vite-pwa/assets-generator | latest | PWA icons from one SVG |
| idb | 8.x | Small IndexedDB wrapper |
| fflate | latest | Zip and unzip (a few KB, tree-shakable) |
| json-schema-to-typescript | dev | Generates TS types from the pipeline's JSON Schema |
| vitest, @playwright/test, @axe-core/playwright | dev | Unit tests, end-to-end/offline tests, accessibility scans |

> **Vite 8 compatibility note:** Vite 8 (March 2026) switched to Rolldown. Both `vite-plugin-pwa` and `@preact/preset-vite` had open Vite 8 support issues in early 2026. After scaffolding, install `vite-plugin-pwa`. If npm reports a **peer-dependency conflict**, run `npm i -D vite@^7` rather than using `--force` or `--legacy-peer-deps`. Record the outcome in `docs/DECISIONS.md`.

### LLM provider (teacher side only)
Any **OpenAI-compatible Chat Completions** endpoint, configured with three environment variables (§5.6):

| Provider | `LITEPACK_LLM_BASE_URL` | Notes |
|---|---|---|
| **Google Gemini (default)** | `https://generativelanguage.googleapis.com/v1beta/openai/` | Free tier with an AI Studio key. `VERIFY` the model ID at <https://ai.google.dev/gemini-api/docs/models>; pick a current **Flash** or **Flash-Lite** model that supports image input. Free-tier limits change; check AI Studio for yours. |
| Ollama (fully local) | `http://localhost:11434/v1` | No quota, but weaker for Swahili. Use a vision-capable model for alt text. |
| OpenRouter / OpenAI / Groq | provider URL | Paid fallbacks |

---

## 3. Repository layout (monorepo)

```
litepack/
├── README.md                  # judge-facing: what, demo link, setup, limitations, credits, AI disclosure
├── LICENSE                    # MIT (code). Pack content keeps its own source license.
├── .gitignore                 # work/, dist/, *.mp4, *.mkv, *.webm (source videos), .env, node_modules/
├── docs/
│   ├── 01-PRODUCT-BRIEF.md  02-TECHNICAL-SPEC.md  03-BUILD-PLAN.md
│   ├── DECISIONS.md           # log of deviations from these docs
│   ├── TESTING-LOG.md         # device, a11y and user-test findings
│   └── screenshots/
├── schema/
│   └── litepack-1.schema.json # GENERATED by `litepack schema` (do not hand-edit)
├── pipeline/
│   ├── pyproject.toml
│   ├── .env.example
│   ├── src/litepack/
│   │   ├── __init__.py        # __version__ = "0.1.0"
│   │   ├── cli.py             # argparse entry point → subcommands
│   │   ├── config.py          # lesson.toml loading + defaults + env
│   │   ├── manifest.py        # pydantic models (§4.2) + cross-reference validator
│   │   ├── media.py           # ffprobe/ffmpeg wrappers (subprocess, check=True)
│   │   ├── transcribe.py      # faster-whisper + VTT/SRT import + segment normalisation
│   │   ├── keyframes.py       # sampling + selection algorithm (§5.4-D)
│   │   ├── images.py          # WebP encoding (§5.4-E)
│   │   ├── sections.py        # sections from keyframes + segment assignment
│   │   ├── llm.py             # OpenAI-compatible client, JSON helper, retries, cache
│   │   ├── enrich.py          # titles, summary, glossary, quiz, alt text
│   │   ├── translate.py       # string-table extraction, batching, application
│   │   ├── readability.py
│   │   ├── packer.py          # assemble → validate → zip
│   │   ├── vtt.py             # caption export with line wrapping
│   │   ├── report.py          # size and time report (json + md)
│   │   └── catalog.py         # builds app/public/packs/index.json
│   └── tests/
│       ├── fixtures/          # make_fixture_video.py, slides.vtt
│       ├── test_keyframes.py  test_manifest.py  test_translate.py  test_vtt.py  test_build_smoke.py
├── samples/
│   ├── SOURCES.md             # every source video: title, author, URL, license, how obtained
│   ├── self-recorded-fractions/lesson.toml      # video file gitignored
│   └── ocw-<topic>/lesson.toml
├── app/
│   ├── package.json  vite.config.ts  tsconfig.json  index.html  playwright.config.ts
│   ├── public/
│   │   ├── logo.svg  (+ generated PWA icons)
│   │   └── packs/index.json  packs/*.litepack.zip     # demo packs, committed
│   ├── src/  (see §6.1)
│   └── tests/  unit/*.test.ts  e2e/*.spec.ts  fixtures/sample.litepack.zip
└── .github/workflows/deploy.yml
```

---

## 4. Pack format `litepack/1`

### 4.1 Zip layout and limits

```
<id>.litepack.zip
├── manifest.json            DEFLATE (level 9)
├── frames/f01.webp …        STORED (already compressed)
└── audio.ogg                STORED, optional (Ogg Opus)
```

| Rule | Value |
|---|---|
| Allowed paths (regex, whole path) | `^manifest\.json$` · `^frames/f\d{2,3}\.webp$` · `^audio\.ogg$` |
| Max zip size | 30 MB |
| Max entries | 200 |
| Max single entry (uncompressed) | 12 MB |
| Max total (uncompressed) | 40 MB |
| Max `manifest.json` | 2 MB |
| Max frames | 60 (pipeline default cap: 30) |
| File name | `<id>.litepack.zip`; the `.zip` ending keeps it openable by any file manager |

### 4.2 Manifest models

`pipeline/src/litepack/manifest.py` is the **single source of truth**. `litepack schema` exports JSON Schema from it, and the app generates its TS types from that schema (03 · T14).

```python
from __future__ import annotations
from typing import Literal
from pydantic import BaseModel, Field, model_validator

L10n = dict[str, str]        # {"en": "...", "sw": "..."}; MUST contain languages.source
ReviewStatus = Literal["machine", "reviewed", "human"]
ReviewKey = Literal["transcript", "titles", "summary", "glossary", "quiz", "alt"]

class Source(BaseModel):
    title: str
    author: str
    url: str | None = None                 # https only
    license: str                           # e.g. "CC-BY-NC-SA-4.0", "Owned by the LitePack team"
    licenseUrl: str | None = None
    attribution: str                       # one-line credit shown to learners
    durationSec: float
    originalBytes: int                     # size of the source file we measured against
    originalDescription: str               # e.g. "MP4 640×360 H.264, publisher download page"

class Languages(BaseModel):
    source: str                            # BCP-47, e.g. "en"
    available: list[str]                   # includes source, e.g. ["en", "sw"]

class Generator(BaseModel):
    name: Literal["litepack-cli"] = "litepack-cli"
    version: str
    asr: str | None                        # "faster-whisper small int8" | "imported-captions"
    llm: str | None                        # model id, or None if run with --no-llm
    translator: str | None                 # "llm:<model>" | "opus-mt" | None

class Section(BaseModel):
    id: str = Field(pattern=r"^s\d{2,3}$")
    start: float
    end: float
    title: L10n
    frameId: str | None                    # None when frames.mode == "none"

class Segment(BaseModel):
    id: str = Field(pattern=r"^t\d{4}$")
    start: float
    end: float
    sectionId: str
    text: L10n

class Frame(BaseModel):
    id: str = Field(pattern=r"^f\d{2,3}$")
    t: float                               # seconds into the video
    file: str = Field(pattern=r"^frames/f\d{2,3}\.webp$")
    width: int
    height: int
    bytes: int
    alt: L10n                              # ≤ 125 chars per language
    boardText: L10n | None = None          # visible text/equations, transcribed

class Summary(BaseModel):
    plain: L10n
    keyPoints: list[L10n]

class GlossaryItem(BaseModel):
    term: L10n
    definition: L10n
    firstSegmentId: str | None = None

class QuizItem(BaseModel):
    id: str = Field(pattern=r"^q\d{1,2}$")
    question: L10n
    choices: list[L10n] = Field(min_length=3, max_length=4)
    answerIndex: int
    explanation: L10n
    refSegmentId: str | None = None

class Audio(BaseModel):
    file: Literal["audio.ogg"] = "audio.ogg"
    mime: Literal["audio/ogg; codecs=opus"] = "audio/ogg; codecs=opus"
    bitrateKbps: int
    bytes: int

class Review(BaseModel):
    status: dict[str, dict[ReviewKey, ReviewStatus]]   # lang → key → status
    reviewer: str | None = None
    reviewedAt: str | None = None          # ISO 8601
    notes: str | None = None

class Stats(BaseModel):
    originalBytes: int
    assetBytes: int                        # frames + audio
    readability: dict[str, float] = {}     # {"transcript_fk_grade": 9.8, "summary_fk_grade": 5.9}

class Manifest(BaseModel):
    format: Literal["litepack/1"] = "litepack/1"
    id: str = Field(pattern=r"^[a-z0-9][a-z0-9-]{2,63}$")
    title: L10n
    subject: str | None = None
    level: str | None = None
    languages: Languages
    createdAt: str                         # ISO 8601 UTC
    generator: Generator
    source: Source
    sections: list[Section]
    segments: list[Segment]
    frames: list[Frame]
    summary: Summary | None = None
    glossary: list[GlossaryItem] = []
    quiz: list[QuizItem] = []
    audio: Audio | None = None
    review: Review
    stats: Stats

    @model_validator(mode="after")
    def _cross_refs(self) -> "Manifest":
        # Raise ValueError with a clear message if ANY of these fail:
        # 1. languages.source ∈ languages.available; every L10n contains languages.source
        # 2. sections sorted, contiguous (s[i].end == s[i+1].start ± 0.01), s[0].start == 0, last.end == source.durationSec ± 1
        # 3. every segment.sectionId exists; segments sorted by start; start < end
        # 4. every section.frameId (if set) exists in frames; frame ids unique
        # 5. quiz: 0 ≤ answerIndex < len(choices); refSegmentId (if set) exists
        # 6. glossary.firstSegmentId (if set) exists
        # 7. review.status has an entry for every language in languages.available
        return self
```

> JSON Schema can't express rule 1–7 cross-checks. The app re-implements them in `validate.ts` (§6.5).

### 4.3 Example manifest (excerpt)

```json
{
  "format": "litepack/1",
  "id": "photosynthesis-intro",
  "title": { "en": "How plants make food", "sw": "Jinsi mimea inavyotengeneza chakula" },
  "subject": "Biology",
  "level": "Secondary (Form 2–3)",
  "languages": { "source": "en", "available": ["en", "sw"] },
  "createdAt": "2026-11-05T14:02:11Z",
  "generator": { "name": "litepack-cli", "version": "0.1.0", "asr": "faster-whisper small int8", "llm": "‹model-id›", "translator": "llm:‹model-id›" },
  "source": { "title": "…", "author": "…", "url": "https://…", "license": "CC-BY-NC-SA-4.0",
              "licenseUrl": "https://creativecommons.org/licenses/by-nc-sa/4.0/",
              "attribution": "Adapted from … by …, CC BY-NC-SA 4.0", "durationSec": 724.3,
              "originalBytes": 0, "originalDescription": "MP4 640×360, publisher download" },
  "sections": [ { "id": "s01", "start": 0, "end": 74.0, "title": { "en": "Why leaves are green", "sw": "…" }, "frameId": "f01" } ],
  "segments": [ { "id": "t0001", "start": 0.0, "end": 4.2, "sectionId": "s01", "text": { "en": "Today we will…", "sw": "Leo tuta…" } } ],
  "frames": [ { "id": "f01", "t": 71.0, "file": "frames/f01.webp", "width": 800, "height": 450, "bytes": 31234,
                "alt": { "en": "Diagram of a leaf cell with chloroplasts labelled", "sw": "…" },
                "boardText": { "en": "CO2 + H2O → glucose + O2", "sw": "CO2 + H2O → glukosi + O2" } } ],
  "summary": { "plain": { "en": "…", "sw": "…" }, "keyPoints": [ { "en": "…", "sw": "…" } ] },
  "glossary": [ { "term": { "en": "chlorophyll", "sw": "klorofili" }, "definition": { "en": "…", "sw": "…" }, "firstSegmentId": "t0012" } ],
  "quiz": [ { "id": "q1", "question": { "en": "…", "sw": "…" }, "choices": [ { "en": "…", "sw": "…" }, { "en": "…", "sw": "…" }, { "en": "…", "sw": "…" } ],
              "answerIndex": 1, "explanation": { "en": "…", "sw": "…" }, "refSegmentId": "t0042" } ],
  "audio": { "file": "audio.ogg", "mime": "audio/ogg; codecs=opus", "bitrateKbps": 16, "bytes": 0 },
  "review": { "status": {
      "en": { "transcript": "machine", "titles": "machine", "summary": "machine", "glossary": "machine", "quiz": "machine", "alt": "machine" },
      "sw": { "transcript": "machine", "titles": "machine", "summary": "machine", "glossary": "machine", "quiz": "machine", "alt": "machine" } },
    "reviewer": null, "reviewedAt": null, "notes": null },
  "stats": { "originalBytes": 0, "assetBytes": 0, "readability": { "transcript_fk_grade": 0, "summary_fk_grade": 0 } }
}
```

### 4.4 Review status

| Status | Meaning | What the learner sees |
|---|---|---|
| `machine` | Produced by speech-to-text or an LLM and not checked | Text badge: **"Machine-made — may contain mistakes"** (localised) |
| `reviewed` | Machine output that a named person checked or edited in Studio | Text badge: **"Checked by ‹reviewer›"** |
| `human` | Written by a person (imported human captions, teacher-written) | No badge |

Imported captions → `transcript: "human"` for the source language. Translations always start as `machine`.

### 4.5 Catalog `app/public/packs/index.json`

```json
{
  "format": "litepack-catalog/1",
  "updatedAt": "2026-11-12T10:00:00Z",
  "packs": [
    {
      "id": "photosynthesis-intro",
      "title": { "en": "How plants make food", "sw": "…" },
      "subject": "Biology", "level": "Secondary",
      "languages": ["en", "sw"],
      "durationSec": 724.3,
      "file": "photosynthesis-intro.litepack.zip",
      "bytes": 0,
      "originalBytes": 0,
      "hasAudio": true,
      "license": "CC-BY-NC-SA-4.0",
      "attribution": "…"
    }
  ]
}
```

`file` is relative to `packs/`. `bytes` is the exact zip size, shown to learners **before** they download.

---

## 5. Teacher pipeline

### 5.1 Input: `samples/<id>/lesson.toml`

```toml
id = "photosynthesis-intro"
video = "photosynthesis.mp4"          # relative to this file; gitignored
captions = ""                         # optional .vtt/.srt; if set, skips ASR (prefer human captions)
subject = "Biology"
level = "Secondary (Form 2–3)"
source_language = "en"
target_languages = ["sw"]
audio = true                          # include audio.ogg in the pack

[title]
en = "How plants make food"

[source]
title = "…"
author = "…"
url = "https://…"
license = "CC-BY-NC-SA-4.0"
license_url = "https://creativecommons.org/licenses/by-nc-sa/4.0/"
attribution = "Adapted from … by …, licensed CC BY-NC-SA 4.0"
original_description = "MP4 640×360 H.264, downloaded from the publisher's download link"

[asr]
model = "small"                       # tiny | base | small | medium | large-v3 | large-v3-turbo
device = "auto"                       # auto | cpu | cuda

[frames]
mode = "auto"                         # auto | interval | none
interval_sec = 60                     # used by mode = "interval"
cut_threshold = 0.10                  # tune with `litepack frames-debug`
min_scene_sec = 3
max_section_sec = 120
max_frames = 30
blank_std = 8                         # frames with grayscale std below this are treated as blank/fade
dedupe_hamming = 6                    # dHash distance (of 64 bits) at or below which adjacent frames are "the same"
extra_times = []                      # force extra frames at these seconds, e.g. [73.5, 410]
drop_ids = []                         # drop auto frames after reviewing debug output, e.g. ["f04"]

[llm]
enabled = true
alt_text = true                       # vision model required
```

### 5.2 CLI

```
litepack build   <lesson.toml> [--force STAGE ...] [--no-llm] [--out dist/]
litepack frames-debug <lesson.toml>        # writes work/<id>/keyframes/debug.html for tuning
litepack validate <pack.zip>               # schema + cross-refs + zip rules; exit 1 on failure
litepack report   <pack.zip> --original <video>   # re-print size report
litepack catalog  <dist-dir> --out ../app/public/packs   # copies zips + writes index.json
litepack schema   > ../schema/litepack-1.schema.json
```

Use `argparse` with subparsers. Print a one-line progress message per stage, plus timing. On error: a clear message and a non-zero exit code; no stack trace unless `--debug`.

### 5.3 Stages, cache and outputs

Every stage writes its result to `work/<id>/`. A stage is **skipped if its output exists**, unless `--force <stage>` (or `--force all`) is given. Re-running after a crash or prompt change is then cheap.

| # | Stage | Output in `work/<id>/` |
|---|---|---|
| A | probe | `probe.json` |
| B | audio | `audio16k.wav` (ASR input), `audio.ogg` (pack) |
| C | transcript | `transcript.json` (list of `{id,start,end,text}`) |
| D | keyframes | `keyframes.json`, `keyframes/raw_*.png`, `keyframes/debug.html` |
| E | images | `frames/fNN.webp`, `frames.json` |
| F | sections | `sections.json` |
| G–H | enrich | `enrich.json` (+ `llm_cache/*.json`) |
| I | alt text | `alt.json` |
| J | translate | `translate.<lang>.json` |
| K | readability | merged into manifest stats |
| L | pack | `dist/<id>/<id>.litepack.zip` |
| M | captions | `dist/<id>/captions/<id>.<lang>.vtt` |
| N | report | `dist/<id>/report.json`, `report.md` |

### 5.4 Stage details

#### A. Probe
```
ffprobe -v error -show_entries format=duration,size:stream=codec_type,codec_name,width,height -of json <video>
```
Store `durationSec`, `originalBytes` (file size on disk), and video width, height and codec. Fail fast if there is no audio stream.

#### B. Audio
```bash
# ASR input: 16 kHz mono PCM
ffmpeg -y -v error -i <video> -vn -ac 1 -ar 16000 -c:a pcm_s16le work/<id>/audio16k.wav
# Pack audio: Ogg Opus, speech-tuned, ~120 KB/min
ffmpeg -y -v error -i <video> -vn -ac 1 -c:a libopus -b:a 16k -application voip work/<id>/audio.ogg
```
If `libopus` is missing, `ffmpeg -hide_banner -encoders | grep -i opus` shows nothing. Install a full ffmpeg build (03 · Troubleshooting).

#### C. Transcript
**Path 1: imported captions (preferred when human-made).** Parse VTT/SRT with a small regex parser: cue timing `HH:MM:SS.mmm --> HH:MM:SS.mmm` (SRT uses a comma). Strip tags like `<c>` and `<i>`, join multi-line cue text, and drop empty cues. Set the review status `transcript: "human"`.

**Path 2: faster-whisper.**
```python
from faster_whisper import WhisperModel

def transcribe(wav: Path, lang: str, model_name: str = "small", device: str = "auto") -> list[dict]:
    dev = "cuda" if device == "auto" and _has_cuda() else ("cpu" if device == "auto" else device)
    model = WhisperModel(model_name, device=dev, compute_type="float16" if dev == "cuda" else "int8")
    segments, info = model.transcribe(
        str(wav), language=lang, beam_size=5,
        vad_filter=True,                    # skips silence → fewer hallucinations
        condition_on_previous_text=False,   # avoids repetition loops on long audio
    )
    out = []
    for i, s in enumerate(segments, 1):     # `segments` is a GENERATOR: iterating runs the model
        text = s.text.strip()
        if text:
            out.append({"id": f"t{i:04d}", "start": round(s.start, 2), "end": round(s.end, 2), "text": text})
        print(f"\r  ASR {s.end:6.0f}s / {info.duration:.0f}s", end="")
    return renumber(out)
```
- The first run downloads the model from Hugging Face; `small` is a few hundred MB. Mention this in the README.
- Use `small` on CPU. With a GPU, `large-v3-turbo` gives better accuracy.

**Normalisation (both paths):** sort by start; merge segments shorter than 1.0 s into the previous one; split segments longer than 20 s at sentence punctuation, dividing time in proportion to character count; renumber `t0001…`.

#### D. Key frames (selection algorithm)

Goal: one still image per meaningful "board state". For **slide** videos, that means each slide. For **handwriting** videos (Khan-style), it means the board **just before it is cleared or scrolled**, which is when it is most complete.

```python
import subprocess, numpy as np, imagehash
from PIL import Image
from dataclasses import dataclass

SAMPLE_FPS, W, H = 1.0, 160, 90

def sample_gray(video: Path) -> np.ndarray:
    """All frames at 1 fps as 160×90 grayscale, streamed via a pipe (no temp files)."""
    cmd = ["ffmpeg", "-v", "error", "-i", str(video),
           "-vf", f"fps={SAMPLE_FPS},scale={W}:{H},format=gray",
           "-f", "rawvideo", "-pix_fmt", "gray", "-"]
    raw = subprocess.run(cmd, check=True, capture_output=True).stdout
    n = len(raw) // (W * H)
    return np.frombuffer(raw[: n * W * H], dtype=np.uint8).reshape(n, H, W)

@dataclass
class Cand:
    idx: int          # sample index of the chosen frame
    start: float      # scene start (s)
    end: float        # scene end (s)
    @property
    def t(self) -> float: return self.idx / SAMPLE_FPS

def select(frames: np.ndarray, cfg) -> list[Cand]:
    n = len(frames)
    f = frames.astype(np.int16)
    diff = np.zeros(n)
    diff[1:] = np.abs(f[1:] - f[:-1]).mean(axis=(1, 2)) / 255.0      # 0..1 change vs previous second

    cuts = [0] + [i for i in range(1, n) if diff[i] > cfg.cut_threshold] + [n]
    scenes = [(a, b) for a, b in zip(cuts, cuts[1:]) if (b - a) / SAMPLE_FPS >= cfg.min_scene_sec]

    step = int(cfg.max_section_sec * SAMPLE_FPS)                       # split long scenes
    chunks = []
    for a, b in scenes:
        while b - a > step:
            chunks.append((a, a + step)); a += step
        chunks.append((a, b))

    cands = []
    for a, b in chunks:
        idx = most_stable(diff, a, b, window=int(5 * SAMPLE_FPS))     # last calm moment before the change
        if frames[idx].std() < cfg.blank_std:                          # skip black/fade/blank frames (blank_std≈8)
            continue
        cands.append(Cand(idx, a / SAMPLE_FPS, b / SAMPLE_FPS))

    cands = dedupe(cands, frames, max_hamming=cfg.dedupe_hamming)      # ≈ 6 bits of 64
    cands = cap(cands, cfg.max_frames)
    return cands

def most_stable(diff, a, b, window):
    lo = max(a + 1, b - window)
    if lo >= b:
        return max(a, b - 1)
    return lo + int(np.argmin(diff[lo:b]))

def dedupe(cands, frames, max_hamming):
    """Adjacent near-identical frames → keep the LATER one (more complete board) and extend its range back."""
    out = []
    for c in cands:
        h = imagehash.dhash(Image.fromarray(frames[c.idx]))
        if out and (h - out[-1][1]) <= max_hamming:
            prev, _ = out.pop()
            c = Cand(c.idx, prev.start, c.end)
        out.append((c, h))
    return [c for c, _ in out]

def cap(cands, max_frames):
    """While too many: merge the adjacent pair with the shortest combined duration (keep the later frame)."""
    while len(cands) > max_frames:
        i = min(range(len(cands) - 1), key=lambda k: cands[k + 1].end - cands[k].start)
        cands[i:i + 2] = [Cand(cands[i + 1].idx, cands[i].start, cands[i + 1].end)]
    return cands
```

Then:
- Apply `extra_times`: insert candidates at those times and split the containing range. Apply `drop_ids` after numbering.
- Extract each chosen frame at **full resolution** with a fast seek:
  `ffmpeg -y -v error -ss <t> -i <video> -frames:v 1 work/<id>/keyframes/raw_<NN>.png`
- `mode = "interval"`: one frame every `interval_sec`, using `most_stable` within each window. `mode = "none"`: no frames; sections every ~90 s.
- **`frames-debug`** writes `debug.html`: every candidate thumbnail with its time and range, plus a simple inline SVG polyline of `diff` with the threshold line. Use it to tune `cut_threshold` per video.

Known limitation: **talking-head videos** produce frames of a face. Recommend `mode = "none"` for them.

#### E. Frame images
```python
def encode_webp(src: Path, dst: Path, max_w=800, qualities=(50, 40, 30), max_bytes=60_000) -> dict:
    im = Image.open(src).convert("RGB")
    if im.width > max_w:
        im = im.resize((max_w, round(im.height * max_w / im.width)), Image.LANCZOS)
    a = np.asarray(im, dtype=np.int16)
    chroma = (np.abs(a[..., 0] - a[..., 1]) + np.abs(a[..., 1] - a[..., 2])).mean()
    if chroma < 6:                       # effectively grayscale → smaller file
        im = im.convert("L")
    for q in qualities:
        im.save(dst, "WEBP", quality=q, method=6)
        if dst.stat().st_size <= max_bytes:
            break
    return {"width": im.width, "height": im.height, "bytes": dst.stat().st_size}
```
Target: an average of **≤ 45 KB per frame**. Report the average in `report.md`.

#### F. Sections
- Section *i* spans `[previous end, cand_i.end)`. The first starts at 0 and the last ends at `durationSec`, so coverage has **no gaps**.
- Snap each internal boundary to the nearest segment boundary within ±3 s, so no sentence is cut in half.
- Assign each segment to the section that contains its **midpoint**.
- Sections with no speech are allowed; the board alone is useful.
- Default titles before the LLM runs: `"Part 3 (2:14–4:05)"`.
- Default alt text without a vision model: `"Board at 2:14. The transcript for this part follows."` (localised by translation).

#### G. LLM client (`llm.py`)
```python
import base64, hashlib, json, os, random, re, time
from pathlib import Path
from openai import OpenAI, APIConnectionError, APIStatusError, BadRequestError, RateLimitError
from pydantic import BaseModel, ValidationError

class LLM:
    def __init__(self, cache_dir: Path):
        self.model = os.environ["LITEPACK_LLM_MODEL"]
        self.client = OpenAI(base_url=os.environ["LITEPACK_LLM_BASE_URL"],
                             api_key=os.environ["LITEPACK_LLM_API_KEY"])
        self.min_interval = float(os.environ.get("LITEPACK_LLM_MIN_INTERVAL_SEC", "6"))  # ≈10 req/min
        self.json_mode, self._last, self.cache_dir = True, 0.0, cache_dir
        cache_dir.mkdir(parents=True, exist_ok=True)

    def _call(self, messages: list[dict]) -> str:
        for attempt in range(6):
            wait = self.min_interval - (time.monotonic() - self._last)
            if wait > 0:
                time.sleep(wait)
            self._last = time.monotonic()
            try:
                kw = {"response_format": {"type": "json_object"}} if self.json_mode else {}
                r = self.client.chat.completions.create(model=self.model, messages=messages, temperature=0.2, **kw)
                return r.choices[0].message.content or ""
            except BadRequestError as e:
                if self.json_mode and "response_format" in str(e).lower():
                    self.json_mode = False          # provider lacks JSON mode → rely on prompt + validation
                    continue
                raise
            except (RateLimitError, APIConnectionError, APIStatusError) as e:
                code = getattr(e, "status_code", None)
                if code is not None and code < 500 and code != 429:
                    raise
                time.sleep(min(60, 2 ** attempt + random.random()))
        raise RuntimeError("LLM request failed after retries")

    def json(self, system: str, user, schema: type[BaseModel], max_fix: int = 2) -> BaseModel:
        key = hashlib.sha256(json.dumps([self.model, system, user, schema.__name__], default=str).encode()).hexdigest()
        cached = self.cache_dir / f"{key}.json"
        if cached.exists():
            return schema.model_validate_json(cached.read_text())
        messages = [{"role": "system", "content": system}, {"role": "user", "content": user}]
        for _ in range(max_fix + 1):
            text = re.sub(r"^`{3}(?:json)?\s*|\s*`{3}$", "", self._call(messages).strip())   # strip code fences
            try:
                obj = schema.model_validate_json(text)
                cached.write_text(obj.model_dump_json())
                return obj
            except ValidationError as err:
                messages += [{"role": "assistant", "content": text},
                             {"role": "user", "content": f"Invalid JSON for the schema: {err.errors()[:5]}. Reply with corrected JSON only."}]
        raise RuntimeError(f"LLM returned invalid {schema.__name__}")

def image_part(jpeg_bytes: bytes) -> dict:
    return {"type": "image_url",
            "image_url": {"url": "data:image/jpeg;base64," + base64.b64encode(jpeg_bytes).decode()}}
```
- For vision calls, `user` is a list of content parts: text plus `image_part(...)`. Send each frame as JPEG q80 at ≤ 768 px wide; don't send the WebP.
- **Never send personal data.** Only lesson content.

#### H. Enrichment prompts (`enrich.py`)

Shared system prompt:
> You help a secondary-school teacher turn a recorded lesson into study materials for students with little or no internet. Use **only** facts stated in the transcript. If something is unclear, leave it out. Write for a reader aged 13–17 who may be reading in a second language. Reply with **JSON only**, matching the requested shape exactly.

The user message for each task contains the task instructions plus `{"sections":[{"id":"s01","text":"…"}, …]}` or the full transcript with segment IDs (`[t0042] text`), as indicated.

| Task | Input | Output shape (pydantic model) | Rules in the prompt |
|---|---|---|---|
| Titles | sections with text | `{"titles": {"s01": "…"}}` | ≤ 6 words, plain words, no numbering, one per section id |
| Summary | full transcript | `{"plain": "…", "keyPoints": ["…"]}` | `plain` ≤ 150 words; sentences ≤ 15 words; everyday words; explain any technical term in the same sentence; about a US grade-6 reading level. 3–6 key points of ≤ 20 words each |
| Glossary | transcript with ids | `{"items":[{"term","definition","firstSegmentId"}]}` | 5–10 terms that **appear in the transcript**; definitions ≤ 25 words; `firstSegmentId` = first segment containing the term |
| Quiz | transcript with ids | `{"items":[{"question","choices":[…],"answerIndex","explanation","refSegmentId"}]}` | Exactly 5 questions; 3–4 choices; exactly one correct; answerable from the transcript; no "all/none of the above"; explanation ≤ 30 words; `refSegmentId` = where the answer is said |
| Alt text (P1, vision) | 1 image + text of its section | `{"alt": "…", "boardText": "…"}` | `alt` ≤ 125 chars describing what matters for learning (not "image of"); `boardText` = all visible text and equations transcribed exactly (empty string if none). Batch up to 4 images per request to save quota. |

**Post-checks** (code, not the LLM): every id referenced exists; quiz `answerIndex` is in range; there are no duplicate choices; glossary terms really occur in the transcript (case-insensitive). If a check fails, retry the task once with the error appended; if it fails again, drop the offending item and log it.

#### J. Translation (`translate.py`)

All learner-visible text is flattened into a **string table** with stable keys, translated in batches, then written back. Alignment is guaranteed by the keys.

| Key | Source |
|---|---|
| `title` | pack title |
| `sec.<sid>.title` | section titles |
| `seg.<tid>` | transcript segments |
| `sum.plain`, `sum.kp.<i>` | summary |
| `gl.<i>.term`, `gl.<i>.def` | glossary |
| `q.<qid>.q`, `q.<qid>.c.<j>`, `q.<qid>.x` | quiz question, choices, explanation |
| `fr.<fid>.alt`, `fr.<fid>.board` | frame alt text, board text |

```python
class TranslationResult(BaseModel):
    items: dict[str, str]

def translate_table(llm, table: dict[str, str], src: str, tgt: str, fixed_terms: dict[str, str]) -> dict[str, str]:
    out: dict[str, str] = {}
    for chunk in chunks(table, max_items=40, max_chars=4000):          # keep each request small
        for attempt in range(2):
            res = llm.json(TRANSLATE_SYSTEM.format(src=NAMES[src], tgt=NAMES[tgt], terms=json.dumps(fixed_terms, ensure_ascii=False)),
                           json.dumps({"items": chunk}, ensure_ascii=False), TranslationResult)
            missing = set(chunk) - set(res.items)
            out.update({k: v for k, v in res.items.items() if k in chunk})
            if not missing:
                break
            chunk = {k: chunk[k] for k in missing}                      # retry only what's missing
        else:
            raise RuntimeError(f"untranslated keys: {sorted(missing)[:5]}…")
    return out
```

`TRANSLATE_SYSTEM`:
> Translate every JSON value from {src} to {tgt} for secondary-school students. Keep the keys exactly as given. Keep numbers, formulas, units, chemical symbols and code unchanged. Use clear, standard {tgt}. Always use these fixed term translations: {terms}. Return `{"items": {key: translation}}` containing every key.

Order of work: (1) translate the glossary **terms** first to get `fixed_terms`, (2) translate everything else using them. Set every `review.status[tgt][*] = "machine"`.

**Offline fallback (P2):** `--translator opus-mt` uses `Helsinki-NLP/opus-mt-en-sw` via `transformers` (an optional dependency group `offline-mt`). Same string table, one string at a time. Quality is lower; say so.

#### K. Readability
`textstat.flesch_kincaid_grade()` on the joined English transcript and on the English plain summary. Store both in `stats.readability`. If the summary grade is above 7, print a warning (no retry needed). The source language must be English for this metric; otherwise skip it.

#### L. Assemble, validate, zip
1. Build `Manifest(...)`. Pydantic validation plus `_cross_refs` must pass, or the build fails with a message.
2. Write `manifest.json` (UTF-8, `ensure_ascii=False`, no indentation).
3. Zip with `zipfile.ZipFile(..., "w")`: `manifest.json` as `ZIP_DEFLATED` (`compresslevel=9`); frames and audio as `ZIP_STORED`.
4. Re-open the zip and run the same checks as `litepack validate` (allowed paths, limits, every referenced file present).

#### M. Captions (VTT)
For each language, write `captions/<id>.<lang>.vtt`:
- Header `WEBVTT`, then cues `HH:MM:SS.mmm --> HH:MM:SS.mmm`.
- At most 2 lines of 42 characters each. Longer segments are split into consecutive cues, with time divided in proportion to characters.
- These are a **side output** for the teacher (e.g. to upload to the original video). They are not inside the pack, since the text is already in the manifest. We also use them to caption **our own demo video** (03 · T35).

#### N. Report
`report.json` and a human-readable `report.md`:
```
Lesson: How plants make food (12:04)
Original: 96.4 MB  (MP4 640×360 H.264, publisher download)
Pack (no audio): 1.31 MB  = 1.4% of original (73.6× smaller)
  manifest 0.21 MB · 18 frames 0.86 MB (avg 48 KB) 
Pack (with audio): 2.75 MB = 2.9% of original  · audio 1.44 MB @16 kbps
Estimated download (bytes×8 ÷ rate; ignores protocol overhead):
  @ 0.25 Mbps  video 51.4 min · pack 42 s
  @ 1 Mbps     video 12.9 min · pack 10.5 s
Readability (Flesch–Kincaid grade): transcript 9.8 → summary 5.9
Review status: sw = machine (all)
```
(The numbers above only illustrate the format. Real numbers come from the build.)

To report sizes "without audio", build the zip twice: once with and once without `audio.ogg`. Ship the audio version and record both.

#### O. Catalog
`litepack catalog dist/ --out ../app/public/packs` copies every `dist/*/*.litepack.zip` and writes `index.json` (§4.5) from each pack's manifest and its exact file size.

### 5.5 `pyproject.toml` essentials
```toml
[project]
name = "litepack"
version = "0.1.0"
requires-python = ">=3.11,<3.13"
dependencies = ["faster-whisper>=1.2", "numpy", "pillow>=10", "imagehash>=4.3",
                "openai>=1.50", "pydantic>=2.7", "textstat", "python-dotenv"]
[project.optional-dependencies]
offline-mt = ["transformers", "sentencepiece", "torch"]
[project.scripts]
litepack = "litepack.cli:main"
[dependency-groups]
dev = ["pytest"]
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### 5.6 Environment (`pipeline/.env`, gitignored; commit `.env.example`)
```
LITEPACK_LLM_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/
LITEPACK_LLM_API_KEY=            # from Google AI Studio; NEVER commit
LITEPACK_LLM_MODEL=              # VERIFY a current Flash/Flash-Lite model id with image input
LITEPACK_LLM_MIN_INTERVAL_SEC=6  # raise if you hit 429s on the free tier
HF_HOME=                         # optional: where Whisper models are cached
```

---

## 6. Learner app (PWA)

### 6.1 Source layout
```
app/src/
├── main.tsx              # render(<App/>, #app); import './styles/base.css'
├── app.tsx               # <SettingsProvider><Layout><RouteSwitch/></Layout></SettingsProvider>
├── router.ts             # hash router: useRoute(), navigate(), match('/pack/:id/:tab'), query params
├── settings.tsx          # context + localStorage('litepack.settings') + applies <html> attrs
├── i18n/ index.ts  en.json  sw.json        # t(key, vars) with fallback to en
├── lib/
│   ├── db.ts             # IndexedDB (idb) schema + CRUD (§6.4)
│   ├── validate.ts       # runtime manifest validation incl. cross-refs (§6.5)
│   ├── importPack.ts     # Blob → limits → unzip → validate → store (§6.5)
│   ├── openPack.ts       # usePack(id): unzip → object URLs; revoke on unmount (§6.6)
│   ├── catalog.ts        # fetch index.json; download zip with progress
│   ├── share.ts          # Web Share (files) → fallback download (§6.8)
│   ├── exportPack.ts     # (Studio) edited manifest + original assets → new zip
│   ├── l10n.ts           # pick(text: L10n, mode, langs) → [{lang, text}] for 1 or 2 languages
│   └── format.ts         # bytes ("1.3 MB"), mm:ss, relative dates
├── components/
│   ├── Layout.tsx  SkipLink.tsx  PackTabs.tsx  LangModeToggle.tsx  MachineBadge.tsx
│   ├── Figure.tsx  TranscriptBlock.tsx  PackCard.tsx  ImportButton.tsx  ShareButton.tsx
│   └── StorageMeter.tsx  SizeCompare.tsx  Status.tsx
├── pages/
│   ├── Library.tsx  Lesson.tsx  Summary.tsx  Practice.tsx  Listen.tsx
│   ├── Settings.tsx  About.tsx  NotFound.tsx
│   └── studio/Studio.tsx           # lazy-loaded: import('./pages/studio/Studio')
├── types/pack.gen.ts               # GENERATED from schema/litepack-1.schema.json
└── styles/base.css
```

### 6.2 `vite.config.ts`
```ts
import { defineConfig, type Plugin } from 'vite';
import preact from '@preact/preset-vite';
import { VitePWA } from 'vite-plugin-pwa';

// CSP as a <meta> tag (GitHub Pages can't set headers). Production only: dev uses inline HMR styles.
const csp = (): Plugin => ({
  name: 'csp-meta', apply: 'build',
  transformIndexHtml: (html) => html.replace('<head>', `<head>
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'; style-src 'self';
      img-src 'self' blob: data:; media-src 'self' blob:; connect-src 'self'; object-src 'none';
      base-uri 'self'; form-action 'none'">`),
});

export default defineConfig({
  base: process.env.BASE_PATH ?? '/',                  // CI sets "/<repo-name>/"
  build: { target: ['chrome87', 'edge88', 'firefox78', 'safari14'] },  // old Android phones
  plugins: [
    preact(),
    csp(),
    VitePWA({
      registerType: 'autoUpdate',
      injectRegister: 'script',                       // external registerSW.js → CSP-friendly
      includeAssets: ['logo.svg', 'favicon.ico', 'apple-touch-icon-180x180.png'],
      manifest: {
        name: 'LitePack: offline study packs', short_name: 'LitePack',
        description: 'Small offline study packs made from video lessons.',
        lang: 'en', start_url: '.', scope: '.', display: 'standalone',
        background_color: '#ffffff', theme_color: '#14532d',
        icons: [
          { src: 'pwa-192x192.png', sizes: '192x192', type: 'image/png' },
          { src: 'pwa-512x512.png', sizes: '512x512', type: 'image/png' },
          { src: 'maskable-icon-512x512.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
        ],
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,svg,png,ico,webmanifest}'],
        globIgnores: ['packs/**'],                    // NEVER precache packs: they'd download on install
        navigateFallback: 'index.html',
        cleanupOutdatedCaches: true,
        runtimeCaching: [{
          urlPattern: ({ url }) => url.pathname.endsWith('/packs/index.json'),
          handler: 'NetworkFirst',
          options: { cacheName: 'catalog', networkTimeoutSeconds: 5 },
        }],
      },
    }),
  ],
});
```
Icons: put a simple `public/logo.svg` in place, then run `npx @vite-pwa/assets-generator --preset minimal-2023 public/logo.svg`.

### 6.3 Routes (hash-based)

| Hash | Page | Notes |
|---|---|---|
| `#/` | Library | Default |
| `#/pack/:id` | → redirect to `/pack/:id/lesson` | |
| `#/pack/:id/lesson[?seg=t0042]` | Lesson | `seg` scrolls to and focuses that segment |
| `#/pack/:id/summary` | Summary + glossary | |
| `#/pack/:id/practice` | Quiz | |
| `#/pack/:id/listen` | Audio + synced transcript | Tab hidden if the pack has no audio |
| `#/settings` | Settings | |
| `#/about` | About, credits, licenses, AI disclosure | |
| `#/studio/:id` | Teacher Studio (P1, lazy) | Linked only from the pack's ⋯ menu |

On every route change: set `document.title` to "‹Page› · ‹Pack title› · LitePack", then move focus to the page's `<h1 tabIndex={-1}>`.

### 6.4 Data layer (`db.ts`)

```ts
import { openDB, type DBSchema } from 'idb';
import type { Manifest } from '../types/pack.gen';

export interface StoredPack {
  id: string;
  manifest: Manifest;
  zip: Blob;                        // the exact file, so sharing sends identical bytes
  bytes: number;
  importedAt: number;
  origin: 'catalog' | 'file' | 'studio';
}
export interface Progress {
  packId: string;
  lastSectionId?: string;
  doneSections: string[];
  quiz?: { answers: Record<string, number>; score: number; at: number };
}
interface LiteDB extends DBSchema {
  packs: { key: string; value: StoredPack };
  progress: { key: string; value: Progress };
}

export const db = openDB<LiteDB>('litepack', 1, {
  upgrade(d) {
    d.createObjectStore('packs', { keyPath: 'id' });
    d.createObjectStore('progress', { keyPath: 'packId' });
  },
});

export const listPacks = async () => (await db).getAll('packs');
export const getPack = async (id: string) => (await db).get('packs', id);
export const putPack = async (p: StoredPack) => (await db).put('packs', p);
export const deletePack = async (id: string) => {
  const d = await db; const tx = d.transaction(['packs', 'progress'], 'readwrite');
  await Promise.all([tx.objectStore('packs').delete(id), tx.objectStore('progress').delete(id), tx.done]);
};
```
- After the **first successful import**, call `navigator.storage?.persist?.()` once to ask the browser not to evict packs. Settings shows the result (`navigator.storage.persisted()`) and the usage (`navigator.storage.estimate()`).
- Settings live in `localStorage` (small, synchronous, read before first paint).

### 6.5 Import and validation (`importPack.ts`, `validate.ts`)

```ts
import { unzipSync, strFromU8 } from 'fflate';

const LIMITS = { zip: 30e6, entries: 200, entry: 12e6, total: 40e6, manifest: 2e6 };
const SAFE = /^(manifest\.json|frames\/f\d{2,3}\.webp|audio\.ogg)$/;
// No TS parameter properties: newer Vite templates enable `erasableSyntaxOnly`, which rejects them.
export class ImportError extends Error {
  code: ImportCode;
  constructor(code: ImportCode) { super(code); this.code = code; }
}
export type ImportCode = 'too_big' | 'not_zip' | 'no_manifest' | 'bad_manifest' | 'missing_file' | 'unsupported_version';

export async function importPack(file: Blob, origin: StoredPack['origin']): Promise<StoredPack> {
  if (file.size > LIMITS.zip) throw new ImportError('too_big');
  const buf = new Uint8Array(await file.arrayBuffer());
  let count = 0, total = 0;
  let files: Record<string, Uint8Array>;
  try {
    files = unzipSync(buf, {
      filter: (f) => {
        count++; total += f.originalSize;
        if (count > LIMITS.entries || total > LIMITS.total || f.originalSize > LIMITS.entry) throw new ImportError('too_big');
        return SAFE.test(f.name);                     // silently ignore anything else (e.g. __MACOSX/)
      },
    });
  } catch (e) { throw e instanceof ImportError ? e : new ImportError('not_zip'); }

  const raw = files['manifest.json'];
  if (!raw) throw new ImportError('no_manifest');
  if (raw.length > LIMITS.manifest) throw new ImportError('too_big');
  const manifest = validateManifest(JSON.parse(strFromU8(raw)));     // throws ImportError('bad_manifest' | 'unsupported_version')
  for (const fr of manifest.frames) if (!files[fr.file]) throw new ImportError('missing_file');
  if (manifest.audio && !files['audio.ogg']) throw new ImportError('missing_file');

  const pack: StoredPack = { id: manifest.id, manifest, zip: file, bytes: file.size, importedAt: Date.now(), origin };
  await putPack(pack);                                 // same id → replaces (newer version of the pack)
  return pack;
}
```

`validateManifest(x: unknown): Manifest`, hand-written with no validation library (to save bytes):
- `format === 'litepack/1'`, otherwise `unsupported_version`.
- Required fields have the right types (strings, finite numbers, arrays).
- The same cross-reference rules 1–7 as §4.2.
- Strings are used **only as text**. Never `dangerouslySetInnerHTML`. `source.url` is rendered as a link only if it starts with `https://`, with `rel="noopener noreferrer"`.

Error messages (localised) per code, e.g. `not_zip` → "This file isn't a LitePack pack. Ask your teacher for a file ending in .litepack.zip."

`<ImportButton>`: a visible `<label>` styled as a button wrapping `<input type="file">`. **Do not set a restrictive `accept`**; some Android pickers grey out `.zip` with certain accept values. Validate after the file is selected.

### 6.6 Opening a pack (`openPack.ts`)
```ts
export function usePack(id: string) {
  // state: { status: 'loading' | 'ready' | 'missing' | 'error', manifest, urls: Record<path, string> }
  // effect: getPack(id) → unzipSync(new Uint8Array(await pack.zip.arrayBuffer()))
  //         → for each frame/audio: URL.createObjectURL(new Blob([bytes], { type }))
  //         cleanup: revoke every object URL
}
```
Opening a ~3 MB pack this way takes milliseconds on desktop. Measure it on a 4× CPU-throttled profile (budget: ≤ 1 s).

### 6.7 Screens

All text comes through `t()`. All pack text goes through `l10n.pick()`, and every rendered pack string gets `lang={code}` and `dir="auto"`.

**Library (`#/`)**
- `h1` "Your lessons". A list of `PackCard`s showing: title, subject/level, duration, **size** ("1.3 MB"), languages, review badge, progress ("3 of 8 parts done"), and an **Open** link.
- "Get a lesson file" section with `<ImportButton>`, whose visible label is exactly **"Open a lesson file"** (the e2e tests select it by this label). On success it shows `role="status"` text **"Saved: ‹title›"**; on failure it shows `role="alert"` with the localised error.
- "Lessons you can download" (shown only when the catalog loads; otherwise "You're offline. Lessons you saved are still here."). Each item shows **"Save for offline · 1.3 MB"** plus `SizeCompare` ("The video is 96 MB"). Download uses `fetch` with a `ReadableStream` reader, showing progress in `<progress>` with a text percentage, then calls `importPack(blob, 'catalog')`.
- `StorageMeter` at the bottom.

**Pack header (all pack tabs)**
- `h1` pack title; `LangModeToggle`, a radio group labelled "Read in": *English* / *Kiswahili* / *Both*. Default: the device language if the pack has it, otherwise the source language. Stored per pack.
- `PackTabs`: `<nav aria-label="Lesson parts">` containing links with `aria-current="page"`: Lesson · Summary · Practice · Listen.
- A "⋯" menu with Share, Delete (with confirmation dialog), and Studio (P1).

**Lesson**
```
<section aria-labelledby="s01-h">
  <h2 id="s01-h">1. Why leaves are green <MachineBadge kind="titles"/></h2>
  <Figure>  <img src=blob: alt=… width height loading="lazy"> <figcaption>Board at 1:11</figcaption>
            <details><summary>Read the text on the board</summary><p lang>…</p></details>
  </Figure>
  <TranscriptBlock>  per segment: <p id="t0001" tabIndex={-1} lang="en">…</p>
                     in Both mode: source line, then translation, visually paired (translation indented, different colour, same font size)
  [Play from 1:11] (only if audio)   [Mark part as done ✓]
</section>
```
Record `lastSectionId` with an IntersectionObserver. "Continue where you left off" appears on the Library card.

**Empty states:** packs built with `--no-llm` have no summary, glossary or quiz. Summary and Practice must then render their `h1` plus a short explanation ("This lesson has no practice questions yet."), never a blank page or an error.

**Summary:** the plain summary, a "Key points" list, then the glossary as a `<dl>` (term `<dt>`, definition `<dd>`, and a "Hear it in the lesson" link to `?seg=firstSegmentId`). Readability is shown if present: "Reading level: about grade 6".

**Practice:** one question at a time.
- `<fieldset><legend>Question 2 of 5: …</legend>` with native radio inputs and labels, and a **Check answer** button.
- Feedback goes in `<p role="status">`: "Correct ✓" or "Not quite ✗", always as **text plus icon**, never colour alone. Then the explanation and a link "Review this part of the lesson" (`?seg=refSegmentId`).
- **Next** button. At the end: score, "Try again", and saved progress.

**Listen** (only if `manifest.audio`)
- Native `<audio controls preload="metadata" src=blob:>`. A speed `<select>` (0.75 / 1 / 1.25 / 1.5) sets `playbackRate`.
- The transcript is a list of `<button>`s ("1:11 Today we will…"); clicking one seeks.
- On `timeupdate`, binary-search the current segment and set `aria-current="true"` plus a highlight class. **No `aria-live` here**, which would flood screen readers.
- A "Follow along" checkbox (default on) calls `scrollIntoView({ block: 'nearest', behavior: reduced ? 'auto' : 'smooth' })`.
- Never autoplay.

**Settings:** UI language (English / Kiswahili); text size (100 / 125 / 150 / 200%) via `--text-scale`; line spacing (normal / relaxed); theme (system / light / dark / high contrast); "Show translation warnings" (default on, can't be hidden for `machine` content in the demo); storage usage and the "Keep my lessons safe" persist button.

**About:** what LitePack is, offline instructions ("Add to home screen"), credits (each pack's source attribution and license with link), open-source library licenses, AI disclosure, and contact/repo link.

### 6.8 Share / export (`share.ts`)
```ts
export async function sharePack(p: StoredPack): Promise<'shared' | 'downloaded' | 'cancelled'> {
  const file = new File([p.zip], `${p.id}.litepack.zip`, { type: 'application/zip' });
  if (navigator.canShare?.({ files: [file] })) {
    try { await navigator.share({ files: [file], title: pickTitle(p.manifest) }); return 'shared'; }
    catch (e) { if ((e as DOMException).name === 'AbortError') return 'cancelled'; /* else fall through */ }
  }
  const url = URL.createObjectURL(file);
  const a = Object.assign(document.createElement('a'), { href: url, download: file.name });
  document.body.append(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 10_000);
  return 'downloaded';
}
```
- Chrome only allows **certain file types** in Web Share; some (e.g. JSON) are rejected, and zip support is not guaranteed. **Always feature-detect with `canShare`.** The download fallback always works: the user then shares from their Files app by Bluetooth, WhatsApp, etc. After downloading, show a short instruction ("Saved to Downloads. Open Files → Downloads → press and hold the file → Share.").
- **Don't** build a Web Share *Target*. Chromium has open 2026 bugs where files shared into installed web apps are dropped. Receiving is done with `<ImportButton>`.

### 6.9 Teacher Studio (P1, `#/studio/:id`, lazy-loaded)
- Choose a language to review, then see a table per category (Titles, Transcript, Summary, Glossary, Quiz, Alt text). Columns: **Source** (read-only, `lang`) | **Translation** (`<textarea>`, `lang`) | status.
- Filter: "Only show machine-made".
- Each category gets a **"Mark as reviewed"** button that requires a reviewer name (stored in settings) and sets `review.status[lang][key] = 'reviewed'`, `reviewer` and `reviewedAt`.
- The quiz editor can change the question/choices text and `answerIndex`.
- **Save** re-validates with `validateManifest`, then calls `exportPack()`: unzip the original, replace `manifest.json`, `zipSync` (`manifest.json` level 9, assets level 0), then `putPack({ origin: 'studio' })`. **Download** offers the new file.
- Studio must not be in the initial bundle (`import()` only).

### 6.10 Accessibility requirements (WCAG 2.2 AA target)

1. `<html lang>` = UI language. Every pack text element has `lang` and `dir="auto"`. Screen readers switch pronunciation for Kiswahili vs English.
2. Landmarks: `<header>`, `<nav>`, `<main id="main">`, `<footer>`. **Skip link** ("Skip to content") is the first focusable element.
3. One `h1` per route, sections as `h2`, with no skipped levels. Focus moves to `h1` on route change, and `document.title` is updated.
4. Only native controls: `<a href>`, `<button>`, `<input>`, `<select>`, `<details>`. No click handlers on `div`/`span`.
5. Visible focus everywhere: `:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }`. Never `outline: none`.
6. Contrast ≥ 4.5:1 for text and ≥ 3:1 for UI components and focus rings, in **every** theme.
7. Primary touch targets ≥ 44×44 CSS px (WCAG 2.2 AA minimum is 24×24).
8. Reflow: no horizontal scrolling at 320 CSS px width **and** at 200% text size. Images use `max-width: 100%`.
9. Images: `alt` from the manifest, in `<figure>` with `<figcaption>` (time). Board text is in `<details>`.
10. Audio: native controls, transcript always visible, no autoplay, adjustable speed.
11. Status messages (import result, download progress, quiz feedback) use `role="status"`. Errors use `role="alert"`. Progress is a labelled `<progress>`.
12. Every input has a `<label>`. Errors are linked with `aria-describedby`.
13. Motion ≤ 200 ms and disabled under `prefers-reduced-motion`. Respect `prefers-color-scheme` and `prefers-contrast: more` (maps to the high-contrast theme).
14. Never convey meaning by colour alone (quiz results, badges, current line).
15. `MachineBadge` is real text, not just an icon. `aria-label` is not needed.
16. Dialogs (delete confirmation) use `<dialog>` with `showModal()`, which moves focus in and returns it after closing.
17. Don't use special "dyslexia fonts" (they add bytes, and evidence for them is mixed). Offer text-size and spacing controls instead.

### 6.11 Internationalisation
- Flat JSON string files: `en.json` (source) and `sw.json` (machine-translated by the pipeline's LLM using the same translation prompt, then reviewed). Mark the UI translation status on the About page.
- `t('library.size', { size: '1.3 MB' })` uses simple `{name}` interpolation. Missing keys fall back to `en`, with a console warning in dev.
- Plurals: keep phrasing plural-neutral ("Parts done: 3 of 8").
- Numbers and dates: `Intl.NumberFormat(uiLang)`, with a fallback to `en` if unsupported.

### 6.12 Styling
- One `base.css` (≤ 10 KB gzip) with custom properties: `--bg`, `--fg`, `--muted`, `--accent`, `--focus`, `--text-scale`, `--line`.
- Themes come from `html[data-theme=light|dark|contrast]`.
- System font stack (`system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", sans-serif`); **no web fonts, no icon fonts**. Inline SVG icons with `aria-hidden="true"`.
- `font-size: calc(1rem * var(--text-scale))`; line-height 1.6 (relaxed: 1.9); reading width `max-width: 70ch`.
- Mobile-first single column. Bottom-safe-area padding for installed mode.

### 6.13 Performance budgets (checked in CI or by script; see 03 · T31)

| Budget | Limit |
|---|---|
| Initial JS (all chunks loaded on `#/`), gzip | ≤ 60 KB |
| CSS, gzip | ≤ 10 KB |
| Total precache (app shell), gzip | ≤ 150 KB |
| Studio chunk | lazy only; not in the initial load |
| Open a 3 MB pack (4× CPU throttle) | ≤ 1 s to first section visible |
| Lighthouse mobile Performance (repeat visit) | ≥ 90 |
| Web fonts / third-party requests | 0 |

### 6.14 Browser support
- **Primary:** Chrome for Android 87+ and Samsung Internet. Chrome 138 was the last version for Android 8/9, Chrome 119 for Android 7, and current Chrome needs Android 10+. Our build target leaves a wide margin.
- **Secondary:** Firefox 78+, Safari 14+. Ogg Opus audio needs **Safari 18.4+** (macOS 15.4 / iOS 18.4); on older Safari, the Listen tab shows "Audio isn't supported on this device; the transcript has everything." Detect with `new Audio().canPlayType('audio/ogg; codecs=opus')`.
- No PWA install on desktop Firefox; the site still works in a tab.

### 6.15 Security and privacy
- Treat every pack as untrusted input: §6.5 limits, path allowlist, validation and text-only rendering.
- CSP meta (§6.2). No `eval`, no inline scripts, no third-party scripts, no analytics.
- No secrets in the app. The LLM key exists only on the teacher's machine.
- No personal data is collected. Progress stays on the device. The About page says so.
