# Decisions log

Changes to 01–03, as Brief §0 rule 1 requires: date, what changed, why. Research behind D1–D11 is in [04 §2](04-EXECUTION-PLAN.md#2-research-results-checked-oct-29-2026).

**Reading rule (from D13):** wherever 01–03 say *Kiswahili* or `sw`, read *Hindi* or `hi`. Wherever they say *Amina* or *Mr. Otieno*, read the personas in D14.

---

## Environment

Recorded 2026-10-29 (T02) on the development machine (Windows 11, PowerShell 5.1).

| Tool | Version |
|---|---|
| git | 2.47.1.windows.2 |
| Node.js / npm | v24.13.1 / 11.1.0 (upgrade npm to ≥ 11.11 before the first `npm install`, D7) |
| Python (pipeline) | 3.12.13, managed by uv |
| Python (system, unused) | 3.13.1 |
| uv | 0.11.20 |
| ffmpeg | 8.1.1-full_build-www.gyan.dev |
| libopus encoder | present (`A....D libopus  libopus Opus (codec opus)`) |
| GPU | none (no NVIDIA driver); ASR runs on CPU |

---

## Decisions

### D1 · LLM model — 2026-10-29
`LITEPACK_LLM_MODEL=gemini-3.5-flash-lite` for text tasks and translation. Switch the quiz and alt text to `gemini-3.8-flash` if a human check finds weak output. **Why:** both are stable, free-tier and image-capable; the 2.0 Flash IDs are shut down and 2.5 models are restricted for new projects.

### D2 · No `temperature` — 2026-10-29
Remove `temperature=0.2` from `LLM._call` (02 §5.4-G). **Why:** current Gemini models ignore sampling parameters, and Google says future models will reject them with HTTP 400. Repeatability comes from the disk cache.

### D3 · Optional reasoning effort — 2026-10-29
Add `LITEPACK_LLM_REASONING_EFFORT` (unset by default). Send `reasoning_effort` only when it's set. **Why:** Gemini 3 models always think; `low` cuts latency and tokens for simple JSON tasks without affecting other providers.

### D4 · Python range — 2026-10-29
`requires-python = ">=3.12,<3.13"` instead of `">=3.11,<3.13"` (02 §5.5). **Why:** matches the 3.12 pin; numpy 2.5 and PyAV 19 need ≥ 3.12.

### D5 · Stay on Vite 8 — 2026-10-29
Use the template's Vite 8 with `vite-plugin-pwa` 2.x and `@preact/preset-vite` 2.10.x. The `vite@^7` fallback (02 §2) applies only if the build actually breaks. **Why:** both plugins now list Vite 8 as a supported peer.

### D6 · Preact 11 + TypeScript 7 — 2026-10-29
Keep the template defaults. Use `useRef<T | null>(null)`; import helper types from `preact`, not `JSX`. Tests must flush effects before checking that object URLs were revoked (cleanup now runs after paint). If TS 7 or the generated types fail to compile, pin an earlier TypeScript major and log it here.

### D7 · npm ≥ 11.11 before the first install — 2026-10-29
**Why:** older npm drops platform metadata for native optional dependencies (Vite 8's Rolldown bindings), so a lockfile written on Windows can break `npm ci` on Linux CI. If CI fails anyway, delete `package-lock.json` and `node_modules`, reinstall, commit.

### D8 · `fflate@^0.8.3` — 2026-10-29
**Why:** CVE-2026-45820, where a 38-byte zip loops forever in ≤ 0.8.2. T15 adds that payload as a test (must reject quickly with `not_zip`), plus a test where an entry inflates beyond its declared size.

### D9 · CI action versions — 2026-10-29
`actions/checkout@v7`, `actions/setup-node@v7`, `actions/upload-pages-artifact@v5`, `actions/deploy-pages@v5`; `cancel-in-progress: false`. **Why:** current majors; and a PR run must not cancel a `main` deploy.

### D10 · Share is download-first on Android — 2026-10-29
Keep the `canShare` check, but treat download + instructions as the main path, and rehearse it for the demo. **Why:** Chromium's permitted Web Share file types don't include `.zip`.

### D11 · UTF-8 and POSIX paths in the pipeline — 2026-10-29
Pass `encoding="utf-8"` to every file read/write; set `PYTHONUTF8=1` in `.env.example`. Build manifest and zip paths as strings (`f"frames/{fid}.webp"`) or `.as_posix()`, never `str(Path)`. **Why:** Windows defaults to cp1252 and backslash paths, which corrupt Devanagari text and fail the `^frames/f\d{2,3}\.webp$` rule.

### D12 · Lesson A topic: photosynthesis — 2026-10-31
Changes T03's example ("Adding fractions"). **Why:** the persona, example manifest, demo script and user-test task 1 all assume photosynthesis.

### D13 · Target language: Hindi (`hi`) instead of Kiswahili (`sw`) — 2026-10-31
The documented fallback in Brief §8: no fluent Kiswahili reviewer is available, and the team member reads Hindi and Gujarati fluently.
- `target_languages = ["hi"]`; files become `translate.hi.json`, `captions/<id>.hi.vtt`, `app/src/i18n/hi.json`. Language name for prompts: `Hindi`; autonym in the UI: `हिन्दी`. The `LangModeToggle` reads *English / हिन्दी / Both*.
- Hindi was chosen over Gujarati because it reaches far more learners and has far more training data for machine translation. **Gujarati (`gu`) moves to P2** ("more target languages"); it needs a toggle redesign for three languages.
- **Reviewer:** the team member (Manav) reviews all Hindi output. The submission discloses that the reviewer is on the team. During user tests (T33), at least one Hindi-reading tester is asked "Would you trust the translation? Why?" as an independent check.
- P2 offline MT becomes `Helsinki-NLP/opus-mt-en-hi` (exists on Hugging Face, Apache-2.0). No `opus-mt-en-gu` model exists.
- Extra checks: Devanagari renders with the system font (no web fonts), and TalkBack switches to a Hindi voice on `lang="hi"` text (install the Hindi voice in Speech Services by Google on both phones).

### D14 · Personas and evidence for India — 2026-10-31
Replaces Brief §3. Personas are **fictional composites**; the submission says so.

- **Learner: Pooja, 16.** Class 10, in a village in north India. She is taught science in English and speaks Hindi at home. She doesn't own a phone; she uses her father's Android phone in the evenings. Mobile signal at home is weak and drops often.
- **Teacher: Mr. Sharma.** Secondary science teacher with a laptop and school Wi-Fi a few times a week.
- **Task:** watch a video lesson on photosynthesis before Thursday's class.
- **Barrier:** the video needs a steady connection she doesn't have at home, in English only, on a phone she has for a short time each evening, and leaves nothing to review offline.

**Evidence** (re-check every number before submitting):
- ASER 2024 (rural India, 14–16-year-olds): 89.1% have a smartphone at home and 82.2% can use one, but only 31.4% of those who can use one own it (36.2% of boys, 26.9% of girls). 65.9% could bring a smartphone to do the survey's digital tasks. ([ASER 2024 summary, Ideas for India](https://www.ideasforindia.in/topics/human-development/youth-s-digital-readiness-in-rural-india); [ASER Centre](https://asercentre.org/about-aser-2024/))
- TRAI (June 2025): rural teledensity 58.8% vs urban 125.3%; Bihar lowest at 56.1%. ([Ookla, citing TRAI](https://www.ookla.com/articles/india-mobile-connectivity-1h2025))
- Ookla (1H 2025): 4G signal samples stronger than −110 dBm were recorded in 88.9% of India's villages. ([Ookla](https://www.ookla.com/articles/india-mobile-connectivity-1h2025))
- India's mobile data is among the cheapest per GB in the world ([Mappr 2026](https://www.mappr.co/mobile-data-pricing-by-country/)). **So the pitch leads with signal, shared phones and language, not the price of data.** Smaller downloads still matter because they finish in the short time a phone has signal.

**Effect on 01 §4 (before/after):** the *Language* row becomes "English audio only → English + Hindi text, side by side". The one-line pitch becomes "‹X› MB that needs a steady signal → ‹Y› MB that works offline, in Hindi."

### D15 · Solo team — 2026-10-31
One team member (Manav) plus AI coding assistance (disclosed). Tasks run in 03 order with no parallel tracks. Work that needs a person (recording, phones, accounts, reviews, user tests, video, Devpost) is listed with dates in [04 §6.1](04-EXECUTION-PLAN.md#61-solo-schedule-who-does-what).

### D16 · Test devices — 2026-10-31
**OnePlus Nord 2T** is the learner phone for every real-device check. **Google Pixel 9** is the second device for the share test (T28) and a second TalkBack run. Neither is an old phone, so old-device performance comes from the 4× CPU-throttle measurement (T31) and is labelled as an estimate. Record each phone's Android and Chrome versions in `TESTING-LOG.md` at T19.

### D17 · Name and repository — 2026-10-31
Project name stays **LitePack**. GitHub repo: `litepack` (public), so `BASE_PATH=/litepack/`. Local repo root: `newHack/litepack/`, with these docs in `docs/`. The npm name `litepack` belongs to an unrelated package and a few small GitHub repos use the name; this doesn't matter because nothing is published to npm or PyPI.

### D18 · Default branch `main` — 2026-10-31
Created with `git init -b main` (the global git default stays unchanged). **Why:** 03 §4 CI triggers on `main`.
