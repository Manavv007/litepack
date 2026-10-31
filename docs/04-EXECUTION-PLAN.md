# LitePack — 04 · Execution Plan

> Part 4. Written Thu Oct 29, 2026, after reading 01–03 end to end and checking every `VERIFY` item against current sources.
> It does **not** replace 01–03. Tasks keep their T-numbers and their "Done when" checks. This file:
> 1. explains the whole system on one page (§1),
> 2. records what changed since the docs were written, with sources (§2),
> 3. lists gaps found in the docs (§3),
> 4. groups the tasks into phases with exit gates (§4–§5),
> 5. lists lead-time items, new risks and Windows notes for this machine (§6–§8).
>
> Updated Oct 29 with your answers: solo team, photosynthesis, **Hindi instead of Kiswahili**, OnePlus Nord 2T + Pixel 9, repo `litepack`. All decisions live in [`DECISIONS.md`](DECISIONS.md) (D1–D18). **Wherever 01–03 say Kiswahili / `sw`, read Hindi / `hi`.**

---

## 1. The project end to end

**One sentence.** A teacher turns a heavy video lesson into a small `.litepack.zip` with one CLI command; a learner opens it in an offline web app on a shared phone, in English, Hindi or both, with no data needed after one download.

### 1.1 Data flow

1. **Input.** The teacher writes `samples/<id>/lesson.toml` and puts the video next to it (gitignored).
2. **Build.** `litepack build` runs stages A–N and caches each result in `work/<id>/`:
   probe → audio (16 kHz WAV for ASR, 16 kbps Opus for the pack) → transcript (imported human captions, else faster-whisper) → key frames (1 fps diffs, scene cuts, "last calm moment", dedupe, cap) → WebP images → sections (frames + segment assignment) → LLM enrichment (titles, summary, glossary, quiz, alt text) → translation (ID-keyed string table) → readability → manifest + zip + validation → VTT captions → size report.
3. **Publish.** `litepack catalog` copies zips into `app/public/packs/` and writes `index.json`. A push to `main` runs CI, which tests and deploys the PWA to GitHub Pages.
4. **First visit.** The service worker precaches the app shell only. Packs are never precached.
5. **Save.** "Save for offline · N KB" (or "Open a lesson file") checks limits, unzips, validates and stores the **zip blob + parsed manifest** in IndexedDB.
6. **Study offline.** Opening a pack unzips it in memory and creates blob URLs (revoked on leave). Screens: Lesson, Summary + glossary, Practice, Listen. Progress stays in IndexedDB.
7. **Pass it on.** Share sends the exact stored zip to another phone (download fallback), which imports it with no data.
8. **Review (P1).** Studio edits translations and quiz, marks items reviewed, re-exports a pack that `litepack validate` accepts.

### 1.2 Invariants that hold the system together

| Invariant | Enforced by |
|---|---|
| `litepack/1` is the only interface between pipeline and app | `manifest.py` (pydantic) → JSON Schema → generated TS types. Cross-reference rules 1–7 implemented twice: `_cross_refs` (Python) and `validate.ts` (app) |
| Every learner-visible string has a stable ID | `sNN`, `tNNNN`, `fNN`, `qN`, and string-table keys (02 §5.4-J), so translations can never drift out of alignment |
| Machine text is labelled until a person checks it | `review.status[lang][key]` + `MachineBadge` |
| Nothing large downloads without consent | `globIgnores: ['packs/**']`; catalog shows exact bytes before download |
| An untrusted pack cannot break the app | size/entry limits, path allowlist, validation, text-only rendering, CSP |
| Every number is measured | `report.json`; `‹MEASURE›` placeholders stay until T34 |

### 1.3 How the entry will be judged

From the [Devpost overview](https://accessed-hacks.devpost.com/) and [rules](https://accessed-hacks.devpost.com/rules), re-read Oct 29: four criteria, equal weight, scored 1–5. Ties go to Barrier Removed, then Learning Value. Unverifiable claims and nonfunctional core features can lower scores, and technical complexity alone earns no bonus. Judging runs Nov 21–22, so **links must stay live through Nov 22**. Participants: 111 (01 said ~98). Eligibility: students at a recognized institution, above the age of majority.

| Criterion | Evidence we will show | Built in |
|---|---|---|
| Barrier Removed | measured MB before/after, download-time estimate, airplane-mode recording | T11, T34, T35 |
| Learning Value | Lesson + Summary + glossary + quiz, with the quiz checked by a person | T17, T21, T23, T24 |
| Inclusive Experience | §6.10 checklist, axe results, keyboard/TalkBack/NVDA logs, user-test findings | T27, T32, T33 |
| Working Prototype & Explanation | live URL, 2-minute try-it steps, limitations, credits, AI disclosure | T19, T37, T38 |

---

## 2. Research results (checked Oct 29, 2026)

### 2.1 This machine

| Tool | Found | Status |
|---|---|---|
| git | 2.47.1 | OK |
| Node / npm | 24.13.1 / 11.1.0 | Node OK. **Upgrade npm** before the first lockfile (D7) |
| Python | system 3.13.1; uv-managed 3.11.15 and 3.12 | OK. Pipeline pins 3.12 |
| uv | 0.11.20 | OK |
| ffmpeg | 8.1.1 full build (gyan.dev), `libopus` encoder present | OK (≥ 6 required) |
| GPU | no NVIDIA driver found | ASR runs on CPU: `small`, int8 |
| gh CLI | not installed | Optional: `winget install GitHub.cli` |
| Shell | Windows PowerShell 5.1 | `&&` doesn't work; see §8 |

### 2.2 `VERIFY` items and version drift

| Item | Docs assume | Current state | Action |
|---|---|---|---|
| Gemini model ID | "a current Flash or Flash-Lite with image input" | `gemini-3.8-flash` (newest stable Flash) and `gemini-3.5-flash-lite` both have free-tier pricing and image input. 2.5 models are limited to projects that used them before; 2.0 Flash IDs are shut down. ([models](https://ai.google.dev/gemini-api/docs/models), [pricing](https://ai.google.dev/gemini-api/docs/pricing), [summary](https://www.aifreeapi.com/en/posts/google-gemini-api-free-tier)) | D1 |
| Free-tier limits | "check AI Studio" | Google publishes no fixed free-tier table. Limits are per project (not per key); requests-per-day reset at midnight Pacific. ([rate limits](https://ai.google.dev/gemini-api/docs/rate-limits)) | Record your RPM/RPD in DECISIONS during T20 |
| Sampling parameters | `temperature=0.2` in `llm.py` | For Gemini 3.6 Flash and later, custom temperature/topP/topK are ignored, and Google says future models will reject them. ([Google Cloud docs](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/content-generation-parameters)) | D2 |
| Free-tier data use | content may be used by Google | Still true: unpaid-tier content is used to improve products. ([pricing](https://ai.google.dev/gemini-api/docs/pricing)) | No change. Only openly licensed lesson content |
| OpenAI-compatible endpoint | `…/v1beta/openai/` | Same URL. Images go in as `image_url` data URIs. `reasoning_effort` maps to Gemini's thinking level; thinking can't be turned off on Gemini 3. ([docs](https://ai.google.dev/gemini-api/docs/openai)) | Optional D3 |
| Python / ctranslate2 | 3.12, because 3.13 wheels lagged | ctranslate2 4.8.4 now ships cp313 wheels, Windows included. faster-whisper 1.2.1 is still the latest. numpy 2.5 and PyAV 19 require ≥ 3.12. | Keep 3.12 (the decision stands). D4 |
| Vite 8 vs plugins | prefer Vite 7; pin `vite@^7` on a peer conflict | `vite-plugin-pwa` 2.0.0 (Oct 3, 2026) and `@preact/preset-vite` 2.10.6 both list Vite 8 as a peer. The current `create-vite` `preact-ts` template (create-vite 9.3.3 on npm) uses Vite ^8.3.5, Preact ^11, TypeScript ~7.0. | D5: stay on Vite 8 |
| vite-plugin-pwa | 1.x | 2.0.0. Release notes list only a Node ≥ 20.19 requirement and the assets-generator ^2 peer. ([releases](https://github.com/vite-pwa/vite-plugin-pwa/releases)) | Use 2.x with the 02 §6.2 config |
| Preact | "latest" | Preact 11: Chrome ≥ 71 / Safari ≥ 12.1 / Firefox ≥ 69 without polyfills; ESM only; `useRef` needs an initial value; `JSX.*` helper types moved to `preact`; `useEffect` cleanup now runs after paint. ([upgrade guide](https://preactjs.com/guide/v11/upgrade-guide/)) | D6 |
| fflate | latest | **CVE-2026-45820**: a 38-byte zip makes `unzip`/`unzipSync` loop forever in versions ≤ 0.8.2. Fixed in 0.8.3. ([write-up](https://www.seal.security/blog/bad-things-also-come-in-small-packages-a-38-byte-dos-in-fflate-cve-2026-45820)) | D8 |
| GitHub Actions | checkout@v4, setup-node@v4, upload-pages-artifact@v3, deploy-pages@v4 | Latest releases: checkout v7.0.1, setup-node v7.1.0, upload-pages-artifact v5.0.0, deploy-pages v5.0.1 | D9 |
| Node LTS | 24 (26 becomes LTS Oct 28) | Node 26.13.0 became LTS on Oct 28, 2026 | Keep 24 locally and in CI |
| Web Share with a zip | "zip support is not guaranteed" | Chromium's permitted share list has no `.zip` ([FILE_TYPES.md](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/modules/webshare/FILE_TYPES.md)). On Chrome for Android, `canShare` will return false for a pack. | D10 |
| Ogg Opus on Safari | 18.4+ | Confirmed | No change |
| Playwright offline | sketch uses `setOffline` | Works with service workers in Chromium; a WebKit bug exists ([#42775](https://github.com/microsoft/playwright/issues/42775)) | Keep e2e on Chromium only |
| npm lockfiles across OSes | not covered | Older npm drops platform metadata for native optional deps (fixed in npm 11.11, [ref](https://github.com/dependabot/dependabot-core/issues/15318)). Vite 8's Rolldown ships native per-OS bindings, so a lockfile written on Windows can break `npm ci` on Linux CI. | D7 |
| Lesson B source | MIT OCW as an example | MIT OCW RES.TLL-004 "STEM Concept Videos": most under 15 minutes ([MIT Open Learning](https://openlearning.mit.edu/news/mit-takes-new-approach-stem-concept-videos)), with publisher-hosted `.srt` caption files on ocw.mit.edu. ([video list](https://ocw.mit.edu/courses/res-tll-004-stem-concept-videos-fall-2013/video_galleries/videos/videos/)) | Shortlist in §6.3 |

Other current versions (npm/PyPI, Oct 29): preact 11.1.0, idb 8.0.4, fflate 0.8.3, json-schema-to-typescript 16.0.0, vitest 5.0.5 (Vite 8 peer OK), @playwright/test 1.64.1, @axe-core/playwright 4.14.0, fake-indexeddb 6.2.6, typescript 7.0.2, openai (Python) 3.41.0 (the error classes `llm.py` imports still exist), pydantic 2.14.2, pillow 13.0.0, imagehash 4.3.2.

### 2.3 Decisions (copied into `DECISIONS.md`)

All dated 2026-10-29. `DECISIONS.md` is the authoritative copy.

- **D1 · LLM model.** `LITEPACK_LLM_MODEL=gemini-3.5-flash-lite` for text tasks and translation. Switch the quiz and alt text to `gemini-3.8-flash` if a human check finds weak output. Why: both are stable, free-tier, image-capable; 2.0 IDs are shut down and 2.5 is restricted for new projects. Confirm both appear for your project in AI Studio.
- **D2 · No `temperature`.** Remove `temperature=0.2` from `LLM._call`. Why: ignored by current Gemini models and slated to cause HTTP 400 on future ones. Determinism comes from the disk cache instead.
- **D3 · Optional reasoning effort.** Add `LITEPACK_LLM_REASONING_EFFORT` (unset by default). Pass `reasoning_effort` only when it's set, e.g. `low`. Why: Gemini 3 always thinks; `low` reduces latency and token use for simple JSON tasks. Other providers never receive the field.
- **D4 · `requires-python = ">=3.12,<3.13"`** instead of `>=3.11,<3.13`. Why: matches the 3.12 pin, and stops uv from resolving older numpy/PyAV versions just to satisfy 3.11.
- **D5 · Vite 8.** Keep the template's Vite 8 + `vite-plugin-pwa` 2.x + `@preact/preset-vite` 2.10.x. The 02 §2 fallback (`vite@^7`) applies only if `npm run build` actually breaks. Why: the peer conflict the fallback guarded against is gone.
- **D6 · Preact 11 + TypeScript 7** (template defaults). Use `useRef<T | null>(null)`; import helper types from `preact`, not `JSX`. Since unmount cleanup runs after paint, `usePack` tests must flush effects before asserting that URLs were revoked. If TS 7 or `json2ts` output fails to compile, pin an earlier TypeScript major and log it.
- **D7 · npm ≥ 11.11 before the first `npm install`.** Run `npm i -g npm@11` first. If CI `npm ci` fails with a missing `@rolldown/binding-linux-x64-gnu` (or similar), delete `package-lock.json` and `node_modules`, reinstall, and commit the new lockfile.
- **D8 · `fflate@^0.8.3` minimum.** Add the 38-byte CVE payload as a T15 unit test that must reject quickly with `not_zip`. Also add a test where an entry's declared size is smaller than what it inflates to, and confirm the import still respects the 40 MB total.
- **D9 · CI action versions.** `actions/checkout@v7`, `actions/setup-node@v7`, `actions/upload-pages-artifact@v5`, `actions/deploy-pages@v5`. Set `concurrency.cancel-in-progress: false` so a PR run can't cancel a deploy from `main` in progress. Watch the first run for input changes.
- **D10 · Share is download-first on Android.** Keep the `canShare` check, but design and test the download + instructions path as the main path. Rehearse the second-phone handover (Files → Downloads → Share → Quick Share/Bluetooth/WhatsApp) for the demo.
- **D11 · UTF-8 and POSIX paths in the pipeline.** Every `open`/`read_text`/`write_text` passes `encoding="utf-8"`. Set `PYTHONUTF8=1` in `pipeline/.env.example` as a safety net. Manifest `file` values and zip entry names are built as strings (`f"frames/{fid}.webp"`) or with `.as_posix()`, never `str(Path)`. Why: Windows defaults to cp1252 (breaks Devanagari text and `→`), and `str(Path)` gives backslashes that fail the `^frames/f\d{2,3}\.webp$` rule.
- **D12 · Lesson A topic:** photosynthesis (accepted).
- **D13 · Hindi (`hi`) replaces Kiswahili.** Brief §8 fallback: you read Hindi and Gujarati, and there's no Kiswahili reviewer. You review the Hindi; the submission discloses that the reviewer is on the team. Gujarati moves to P2. Offline MT (P2) becomes `Helsinki-NLP/opus-mt-en-hi`.
- **D14 · New personas and evidence for India** (Pooja, 16, and Mr. Sharma). India's data is among the cheapest per GB, so the pitch leads with weak signal, shared phones and language, not data price. Evidence with sources is in `DECISIONS.md`.
- **D15 · Solo team** with AI coding assistance (disclosed). Schedule in §6.1.
- **D16 · Devices:** OnePlus Nord 2T is the learner phone; Pixel 9 is the second device. Neither is old, so old-device speed is a throttled estimate (T31).
- **D17 · Name and repo:** LitePack, repo `litepack`, local root `newHack/litepack/`.
- **D18 · Branch:** `main`, via `git init -b main`.

---

## 3. Gaps and inconsistencies found in 01–03

1. **Lesson A topic.** T03 suggests "Adding fractions", but the persona, example manifest, demo script and user-test task 1 all assume photosynthesis. *Resolved: photosynthesis (D12).*
2. **"12-minute video".** Lesson A is 5–8 minutes. The demo narration must use the real duration and sizes from `report.json` (Brief §0 rule 2).
3. **`temperature=0.2`** in `llm.py` (D2).
4. **Sharing a zip** won't open the share sheet on Chrome for Android (D10). The demo segment at 2:05–2:20 will go through the download fallback.
5. **Axe route list** (03 §3) omits `#/pack/fixture/listen`. The fixture has a silent audio track and `audio = true` by default, so the Listen tab exists. Add it, and add `#/studio/fixture` once T29 lands.
6. **Offline test selector.** `page.getByRole('status')` can match several regions on Library (import result, download progress). Scope it to the import section or use `getByText(/saved:/i)` to avoid a strict-mode failure.
7. **Optional fields in generated types.** Pydantic's default (validation-mode) schema marks defaulted fields (`format`, `glossary`, `quiz`, `stats.readability`) as optional, so the TS types allow `undefined`. Have `validate.ts` normalise them (`glossary ?? []`, `quiz ?? []`) so screens never handle `undefined`.
8. **CI concurrency.** `cancel-in-progress: true` at workflow level lets a PR run cancel a `main` deploy (D9).
9. **Timeline.** 03 was written for Oct 28; today is Oct 29, the last M0 day. Recording lesson A will probably slip to Oct 31 – Nov 1. The fixture video (T12) and lesson B keep pipeline work moving.
10. **Windows file I/O** (D11). The 02 skeletons call `write_text()` without an encoding, which misbehaves on Windows.

---

## 4. Phases at a glance

| Phase | Dates (2026) | Goal | Tasks | Exit gate |
|---|---|---|---|---|
| **P0** Foundation | Thu Oct 29 – Fri Oct 30 | Repo, tools, decisions, accounts, content started | T01–T03 | G0 |
| **P1** Pack contract | Fri Oct 30 – Sat Oct 31 | Freeze the interface between pipeline and app | T04–T05 | G1 |
| **P2** Pipeline core (no AI) | Sat Oct 31 – Tue Nov 3 | One command → valid pack, report, VTT | T06–T12 | G2 |
| **P3** App core, offline, deployed | Wed Nov 4 – Sun Nov 8 (scaffold can start Oct 31) | Live URL works in airplane mode on a real phone | T13–T19 | **G3: minimum demoable product** |
| **P4** AI and remaining screens | Mon Nov 9 – Thu Nov 12 | Summary, glossary, quiz, Hindi; Summary/Practice/Listen | T20–T25 | **G4: feature-complete MVP** |
| **P5** Accessibility, settings, share, P1 | Fri Nov 13 – Sun Nov 15 | Audit passes; P1 as far as time allows | T26–T30 | G5 |
| **P6** Testing and measurement | Mon Nov 16 – Wed Nov 18 | Real devices, real people, final numbers | T31–T34 | G6 |
| **P7** Submission | Thu Nov 19 (buffer Fri Nov 20, deadline 5:00 PM ET = Sat Nov 21, 3:30 AM IST) | Video, screenshots, README, Devpost | T35–T38 | 03 §10 checklist |

Gates are hard stops: don't start the next phase's MVP work with a red gate. Two exceptions are planned in §6.1: the app scaffold (T13) starts during P2, and P2 can pass on lesson B and the fixture if lesson A's recording is late.

---

## 5. Phase details

### P0 · Foundation (Oct 29–30): T01, T02, T03

**Steps**
1. *Done:* repo root is `newHack/litepack/`, docs moved into `docs/`, with `LICENSE` (MIT), `.gitignore`, `.gitattributes`, `README.md`, `docs/DECISIONS.md`, `docs/TESTING-LOG.md`. Empty folders from the 02 §3 layout get created by the tasks that fill them (T04, T05, T13).
2. **You:** create the public GitHub repo `litepack` **empty** (no README, license or .gitignore), and send the URL. Kiro adds the remote and pushes the first commit. Then Settings → Pages → Source: **GitHub Actions**.
3. *Done:* T02 versions and D1–D18 are in `DECISIONS.md`.
4. **You:** create the AI Studio project and key; put it in `pipeline/.env` only (Kiro creates `.env.example` in T04). Open the rate-limit view for `gemini-3.5-flash-lite` and `gemini-3.8-flash` and note the numbers.
5. **You:** pick lesson B from the §6.3 shortlist and download the video **and** its `.srt` from the OCW page into `samples/ocw-<topic>/`. Kiro writes the `lesson.toml` (`captions = "<file>.srt"`, every `[source]` field) and the `samples/SOURCES.md` entry from the page details you send.
6. **You:** record lesson A on photosynthesis: 5–8 minutes, 6–10 distinct slides, clear English voice. Use slide text of 24 pt or larger. Export as **MP4, 854×480 (480p), H.264 + AAC**, so the size comparison is against a fair streaming quality (03 §7). Kiro can draft the slide script.
7. **You:** start recruiting 3–5 user testers now (§6.3), including at least one Hindi reader.

**G0 (exit):** repo public with the first commit pushed · `DECISIONS.md` complete · lesson B `lesson.toml` complete · lesson A recorded, or recording set for no later than Nov 1 · `git status` clean with videos ignored.

### P1 · Pack contract (Oct 30–31): T04, T05

**Notes**
- `uv init --package --name litepack --python 3.12` inside `pipeline/`, then edit `pyproject.toml` per 02 §5.5 with D4, then `uv python pin 3.12` and `uv sync`.
- `cli.py`: argparse subparsers; `--debug` shows stack traces, otherwise a one-line error and a non-zero exit.
- `config.py`: `tomllib` → defaults → resolve `video`/`captions` relative to the toml file → `load_dotenv(pipeline/.env)`.
- `manifest.py`: copy 02 §4.2, implement the 7 rules with messages that name the failing ID.
- `test_manifest.py`: one valid minimal manifest plus **one failing case per rule** (at least 7 tests).
- Commit `schema/litepack-1.schema.json`. This is what unblocks the app's T14.

**G1:** `uv run litepack --help` lists every subcommand · `build` prints the parsed config for both lessons · schema committed · `uv run pytest tests/test_manifest.py` passes.

### P2 · Pipeline core without AI (Oct 31 – Nov 3): T06–T12

**Notes per task**
- **T06** `media.py`: `run()` raises with ffmpeg's stderr; `probe()` fails fast without an audio stream; stage cache = "output exists" check plus `--force`. Check `audio.ogg` size against 16 kbps × duration ÷ 8 (±20%).
- **T07** `transcribe.py`: the VTT/SRT parser must accept CRLF line endings, a comma in SRT timings, and tags like `<i>`. Lesson B goes through captions import (`transcript: "human"`). Lesson A goes through faster-whisper on CPU. **Time the ASR run and log it.** If it's too slow, use `base` (02 §12). The first run downloads the model to the Hugging Face cache.
- **T08** `keyframes.py`: `frames-debug` is the tuning tool. Get lesson A to "each slide exactly once" before moving on. Record `cut_threshold` per lesson. If lesson B shows faces, use `extra_times`/`drop_ids` or `mode = "interval"`.
- **T09** `images.py`: print the average KB; check that board text is readable at 800 px.
- **T10** `sections.py`: assert full coverage of `[0, duration]` and exactly one section per segment; snap boundaries to segment edges within ±3 s.
- **T11** `packer.py`, `vtt.py`, `report.py`: build with and without audio; `validate` re-opens the zip and checks allowed paths and limits; check the VTT in VLC.
- **T12** fixture: `id = "fixture"`, 3 Pillow slides, `slides.vtt`. Copy the built pack to `app/tests/fixtures/sample.litepack.zip`.

**G2 (pipeline MVP):** `uv run litepack build ../samples/<A>/lesson.toml --no-llm` produces the zip, `report.md` and VTT · `litepack validate` exits 0 · `uv run pytest` passes · first real size numbers exist in `report.md`. If lesson A isn't recorded yet, pass G2 on lesson B and the fixture, then re-run for A.

### P3 · App core, offline, deployed (Nov 4–8; scaffold from Oct 31): T13–T19

**Notes per task**
- **T13** `npm create vite@latest app -- --template preact-ts`, then `npm i idb fflate@^0.8.3` and the dev deps from 03 T13. Write `vite.config.ts` from 02 §6.2. Build `base.css` with all themes and focus styles on day one (Brief §0 rule 5). Run `npm run preview` and check the console for CSP violations.
- **T14** `gen:types`; `db.ts` uses the generated `Manifest`.
- **T15** `validate.ts` implements rules 1–7 and the default normalisation from §3 item 7. Tests: the 8 cases in 03 T15, plus the CVE payload and the size-mismatch case (D8).
- **T16** catalog download with a `ReadableStream` progress reader; `<dialog>` delete confirmation; call `navigator.storage.persist()` once after the first import.
- **T17** Lesson: `lang` and `dir="auto"` on every pack string, `?seg=` focus, focus to `h1` and `document.title` on route change, object URLs revoked on leave.
- **T18** offline e2e (with the §3 item 6 selector fix) and the manual DevTools check.
- **T19** `deploy.yml` with D9. Build lesson A without AI, run `litepack catalog`, commit `app/public/packs/`. Check that the zip sizes in `index.json` match the files.

**Run early:** the axe test (T27) on `#/` and `#/pack/fixture/lesson` as soon as T17 lands. It's cheaper to fix problems on two screens than on eight.

**G3 (minimum demoable product):** on the OnePlus Nord 2T, the live URL loads, saves lesson A, and opens it in airplane mode · Lesson works keyboard-only · axe reports 0 serious/critical on Library and Lesson · URL is in the README. This gate covers the "never cut" items except the quiz and translation.

### P4 · AI and remaining screens (Nov 9–12): T20–T25

**Order (solo)**
- **Pipeline first:** T20 `llm.py` (D1–D3, cache, stub-client tests for 429 backoff and the JSON-fix loop) → T21 `enrich.py` + `readability.py` + vision alt text in batches of 4 → T22 `translate.py` (glossary terms first, then the rest; `hi` VTT).
- **Then the app:** T23 Summary, T24 Practice, T25 Listen. They're built against a hand-edited copy of the fixture manifest with a summary, glossary and quiz, so they don't depend on LLM output being final.

**Quota planning (estimate from 02's batch sizes, not measured):** per lesson, about 4 text-task calls, ceil(frames ÷ 4) vision calls, and one translation call per ~40 keys or ~4,000 characters. That's on the order of 10–20 requests per lesson and language. The cache makes re-runs free; `--force enrich` spends quota again.

**Human checks (you, Nov 11–12):** check every quiz answer against the transcript (T21), then review the Hindi: 10 random segments plus the whole quiz at minimum, fixing what's wrong by editing `translate.hi.json` and re-running with `--force pack` (T22). Log both in `TESTING-LOG.md`.

**G4 (feature-complete MVP):** both demo packs built with AI and `hi` · Summary, Practice and Listen work offline on the Nord 2T · every machine-made text shows a badge · empty states render for `--no-llm` packs · redeployed.

### P5 · Accessibility, settings, share, P1 (Nov 13–15): T26–T30

**Order:** T26 Settings → T27 accessibility pass (MVP) → T28 Share (D10) → T30 UI translation + About → T29 Studio. Studio comes last because it's the first item on the Brief §5 cut line (Studio → vision alt text → UI translation → glossary). The About page's credits and AI disclosure are required for submission, so build About even if UI translation is cut.

**G5:** axe 0 serious/critical on every route (including Listen, and Studio if built) · Lighthouse Accessibility ≥ 95 on Library, Lesson, Practice · no horizontal scroll at 320 px with 200% text · §6.10 checklist in `TESTING-LOG.md` · share path recorded from a real second device.

### P6 · Testing and measurement (Nov 16–18): T31–T34

- **T31** `size-check.mjs` in CI; Lighthouse mobile repeat visit; time to open the biggest pack with 4× CPU slowdown.
- **T32** the "Pooja path" (open the app → save a lesson → read section 2 in Hindi → open the board text → answer the quiz) with keyboard only, TalkBack on the Nord 2T (Hindi voice installed, D13), and NVDA on this Windows machine.
- **T33** 3+ testers per 03 §8; fix the top 3 issues or list them as limitations.
- **T34** `--force all` rebuild of both packs → catalog → redeploy → copy numbers into the README and the Devpost draft.

**G6:** size check passes in CI · three assistive-tech runs logged · 3+ user sessions logged · `git grep "‹MEASURE"` and `git grep "‹CALC"` return nothing.

### P7 · Submission (Nov 19; buffer Nov 20): T35–T38

- Record the demo video on the evening of Nov 18 or the morning of Nov 19, after T34, so the numbers on screen are final.
- Caption it with LitePack's own pipeline (stage M), correct by hand, upload the VTT, paste the transcript into Devpost.
- Screenshots with alt text (03 T36), README (03 T37), Devpost (03 §9).
- After submitting, open the public page logged out and check every link. **Don't push risky changes to `main` until results are out (Nov 23)**; judges test the live URL Nov 21–22.

---

## 6. Schedule, critical path and lead time

### 6.1 Solo schedule: who does what

Kiro writes the code, tests and docs. These tasks need you in person: accounts, recordings, phones, reviews, testers and the submission.

| Dates | You | Kiro |
|---|---|---|
| Oct 29–30 | Create the empty `litepack` repo and send the URL · AI Studio key in `pipeline/.env` · pick and download lesson B (video + `.srt`) | First push · T04–T05 |
| Oct 31 – Nov 1 | Record lesson A (photosynthesis, 480p MP4) | T06–T07 on lesson B and the fixture · T13 scaffold |
| Nov 2–3 | Look at `debug.html` for both lessons and confirm each slide appears once | T08–T12 · lesson A build |
| Nov 4–8 | **Nov 8:** live URL on the Nord 2T, save lesson A, airplane mode (G3); log Android and Chrome versions | T14–T19 |
| Nov 9–12 | **Nov 11–12:** check every quiz answer; review the Hindi | T20–T25 |
| Nov 13–15 | **Nov 14:** share test, Nord 2T → Pixel 9 · review `hi.json` UI strings · practise TalkBack | T26–T30 |
| Nov 16–18 | **Nov 16–17:** 3+ user tests · TalkBack and NVDA runs · keyboard run | T31 · fixes · T34 final rebuild |
| Nov 19 | Record the demo video · take phone screenshots · submit on Devpost | T35 captions · T36 alt text · T37 README · T38 draft text |

Long CPU jobs (Whisper on lesson A) run in the background while other tasks continue.

### 6.2 Critical path

T04 → T05 → T06–T11 → T19 (deployed with lesson A) → T20–T22 → T34 → T35 → T38.

The tightest stretch is P4: the quiz must be checked and the Hindi reviewed before T34's final rebuild, and the video (T35) needs the final packs. If T22 slips past Nov 12, cut along the Brief §5 line rather than squeezing P6.

### 6.3 Lead-time items (start in P0)

| Item | Needed by | Note |
|---|---|---|
| Hindi review time (you) | Nov 11–12 (T22), Nov 15 (T30) | Block about 2 hours for each |
| 3–5 testers: someone on a weak connection, someone using a screen reader or large text, at least one Hindi reader | Nov 16–17 | Verbal consent; record no names or faces |
| OnePlus Nord 2T (learner phone) and Pixel 9 (second device) | Nov 8 (T19), Nov 14 (T28) | Install the Hindi voice in Speech Services by Google on both |
| Lesson A recording | Nov 1 | Needed to tune ASR and key frames |
| Lesson B video + `.srt` | Oct 30 | Shortlist below |
| AI Studio project + key | Nov 9 (check limits on Oct 30) | Per-project quota |
| TalkBack practice | Nov 16 | Practise on the fixture pack during P5 |

**Lesson B shortlist** (MIT OCW RES.TLL-004 STEM Concept Videos; open each video page and confirm the license line, the length and the presence of a caption file before choosing): *Conservation of Mass*, *Latent Heat*, *What is Temperature?*, *Feedback Loops* (biology). Preview the footage first. Prefer the one with the most diagrams or board content, since presenter-heavy shots produce face frames (Brief §9 limitation). Download from the OCW page's own download link, not YouTube.

---

## 7. Risks added by this review

These extend 03 §11.

| Risk | Likelihood | Mitigation |
|---|---|---|
| Repeated 503s from `gemini-3.8-flash` on the free tier (reported Oct 2026, [forum](https://discuss.ai.google.dev/t/repeated-503-unavailable-on-gemini-3-8-flash-free-tier-including-minimal-text-only-requests/189051/1)) | Medium | `_call` already retries 5xx with backoff; default to `gemini-3.5-flash-lite` (D1); the cache prevents repeat calls |
| Lockfile written on Windows breaks `npm ci` on Linux CI | Medium | D7 |
| Windows default encoding corrupts Devanagari text or symbols | High unless handled | D11 |
| Backslashes in manifest paths | Medium on Windows | D11; the validator's regex catches it |
| Share sheet never opens for zips on Chrome Android | High (by design) | D10; tested download path |
| TypeScript 7 or Preact 11 type changes break template code or generated types | Low–Medium | D6 fallback: pin an earlier TypeScript major |
| Lesson B footage is mostly a presenter | Medium | Preview before choosing; `interval` mode; `extra_times`/`drop_ids` |
| Judges see the translation reviewer is on the team | Certain | Disclose it plainly; ask a Hindi-reading tester whether they'd trust the translation (T33) and quote the answer |
| Both test phones are fairly new, so "works on old phones" isn't shown directly | Certain | Report the 4× CPU-throttle timing as an estimate (T31); keep the build target at Chrome 87 |
| Judges outside India assume Indian data is expensive | Medium | Pitch says plainly that data is cheap per GB in India and the barrier is signal, shared phones and language (D14) |

---

## 8. Windows notes for this machine

- **PowerShell 5.1 has no `&&`.** The docs' `cd pipeline && uv init …` becomes `cd pipeline; uv init …`, or run the commands one by one. `grep` becomes `Select-String`, e.g. `ffmpeg -hide_banner -encoders | Select-String libopus`. `git grep` works as written.
- **`.gitattributes`** (created) keeps text files LF and marks zip, WebP, PNG, ICO, Ogg and WAV as binary.
- **Python encoding:** D11. Keep `PYTHONUTF8=1` in `pipeline/.env`. If Hindi text prints garbled in the terminal, also set `PYTHONIOENCODING=utf-8`.
- **Hugging Face cache:** set `HF_HOME` to a drive with a few GB free. A warning about symlinks on Windows is harmless.
- **System Python is 3.13.** Always go through `uv run` inside `pipeline/` so the pinned 3.12 environment is used.
- **uv location:** the `uv` on your PATH is `C:\Users\BAPS\AppData\Local\hermes\bin\uv.exe`, installed by another tool. If a `uv` command hangs, check for another `uv` process holding its cache lock (`Get-Process uv`). The idle processes noted earlier have since exited.

---

## 9. Open questions

Answered Oct 29: solo team · photosynthesis · Hindi (you review) · OnePlus Nord 2T + Pixel 9 · name LitePack, repo `litepack` · root `newHack/litepack/`.

Still needed from you:
1. **The repo URL**, after you create the empty public repo `litepack`.
2. **Lesson B choice** from the §6.3 shortlist, after previewing the videos.
