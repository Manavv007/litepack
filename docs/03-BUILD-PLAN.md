# LitePack — 03 · Build Plan

> Part 3 of 3. Do the tasks **in order**. Each task lists what to do, the files involved, and a **Done when** check that must actually pass, by running it or looking at it, before moving on. **Reference convention:** a bare `§` points to `02-TECHNICAL-SPEC.md`; `Brief §` points to `01-PRODUCT-BRIEF.md`; `this file §` points to sections of this document.

---

## 0. Timeline

Today is Wed Oct 28, 2026. The hard deadline is **Fri Nov 20, 5:00 PM ET**; we submit on **Thu Nov 19**.

| Milestone | Dates | Outcome |
|---|---|---|
| **M0** Setup and content | Oct 28–29 | Repo, tools, demo videos with licenses recorded |
| **M1** Pipeline core (no AI) | Oct 30 – Nov 3 | `litepack build --no-llm` produces a valid pack, report and VTT |
| **M2** App core, offline, deployed | Nov 4–8 | Live URL: import or download a pack, read the Lesson offline |
| **M3** AI and remaining screens | Nov 9–12 | Titles, summary, glossary, quiz and translation; Summary, Practice, Listen screens |
| **M4** Accessibility, settings, share, Studio | Nov 13–15 | All MVP done; P1 where possible |
| **M5** Testing and measurement | Nov 16–18 | Device, screen-reader and user tests; final numbers; fixes |
| **M6** Submission | Nov 19 (buffer Nov 20) | Video, screenshots, README, Devpost submitted |

With 2+ people, run M1 (pipeline) and M2 (app) in parallel from Oct 30. The app can start against the fixture pack from T12.

---

## 1. Prerequisites (one-time)

```bash
# ffmpeg with libopus
sudo apt-get install -y ffmpeg            # macOS: brew install ffmpeg · Windows: winget install Gyan.FFmpeg
ffmpeg -version | head -1
ffmpeg -hide_banner -encoders | grep -i libopus      # MUST print a line

# Python via uv
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.12

# Node 24 LTS (via nvm, fnm or the installer)
node -v                                    # v24.x (v26 also fine)

# Accounts / keys
#  - GitHub repo (public) with Pages enabled: Settings → Pages → Source: "GitHub Actions"
#  - Google AI Studio API key (free tier) → pipeline/.env (never commit)
```

---

## 2. Tasks

### M0: Setup and content

**T01 · Repository scaffold**
- Do: Create the repo `litepack` with the layout in §3. Add an MIT `LICENSE` and a `.gitignore` (`work/`, `dist/`, `**/.env`, `node_modules/`, `*.mp4`, `*.mkv`, `*.webm`, `*.mov`, `__pycache__/`, `.venv/`). Copy these three docs into `docs/`. Create `docs/DECISIONS.md` and `docs/TESTING-LOG.md` with headings only.
- Done when: the first commit is pushed and the repo is public.

**T02 · Tooling check**
- Do: Run every command in this file §1 and paste the versions into `docs/DECISIONS.md` under "Environment".
- Done when: `libopus` is listed and the Python 3.12 and Node versions are recorded.

**T03 · Demo content**
- Do:
  1. Record **lesson A** yourselves: 5–8 min, slides plus voice, on a topic you know (e.g. "Adding fractions"). Speak clearly; use 6–10 distinct slides.
  2. Choose **lesson B**: an openly licensed video of 15 minutes or less, downloaded from the **publisher's own download link** (e.g. MIT OpenCourseWare). Prefer one with human captions available. Never use a YouTube downloader (Brief §8).
  3. For each, create `samples/<id>/lesson.toml` (§5.1) and an entry in `samples/SOURCES.md` covering title, author, URL, license and license URL, how it was obtained, and the date.
- Done when: both `lesson.toml` files have every `[source]` field filled, and the videos are present locally but **not** tracked by git (`git status` is clean).

### M1: Pipeline core (no AI)

**T04 · Package skeleton and CLI**
- Do: `cd pipeline && uv init --package --name litepack --python 3.12`, then edit `pyproject.toml` to match §5.5 and run `uv sync`. Implement `cli.py` with argparse subcommands from §5.2 (stubs that print "not implemented") and `config.py` (load `lesson.toml` with `tomllib`, apply defaults, resolve `video` relative to the toml file, load `.env` with python-dotenv). Add `.env.example` (§5.6).
- Done when: `uv run litepack --help` lists all subcommands, and `uv run litepack build ../samples/<id>/lesson.toml` prints the parsed config.

**T05 · Manifest models and schema**
- Do: Implement `manifest.py` exactly as in §4.2, including `_cross_refs`. Implement `litepack schema` (`Manifest.model_json_schema()` → stdout, indent 2). Write the schema to `schema/litepack-1.schema.json`.
- Done when: the schema file exists; `test_manifest.py` passes, with one valid minimal manifest plus one failing case for each cross-reference rule.

**T06 · Probe and audio (stages A–B)**
- Do: `media.py` with `run(cmd)` (subprocess, `check=True`, raises a readable error with stderr), `probe()`, `extract_asr_audio()` and `extract_pack_audio()` per §5.4-A/B. Add stage caching (§5.3) and `--force`.
- Done when: for lesson A, `probe.json` shows the right duration; `audio.ogg` is ≈ 16 kbps × duration ÷ 8 bytes (±20%) and plays in a browser; re-running prints "skipped (cached)".

**T07 · Transcript (stage C)**
- Do: `transcribe.py`: captions import (VTT and SRT), the faster-whisper path, and normalisation (§5.4-C). Use captions when `captions` is set; otherwise ASR.
- Done when: lesson A has a `transcript.json` whose text you spot-check against the audio at 3 random timestamps; IDs are `t0001…` in order; a second run is instant (cached).

**T08 · Key frames (stage D) and debug view**
- Do: `keyframes.py` per §5.4-D, including `extra_times`, `drop_ids`, `interval` and `none` modes, and full-resolution extraction. Implement `litepack frames-debug`.
- Done when: on lesson A (slides), `debug.html` shows **each slide exactly once** with no blank frames. On lesson B, the frames look like useful "board states". Tune `cut_threshold` per lesson in its `lesson.toml` and record the chosen values in `DECISIONS.md`.

**T09 · Frame images (stage E)**
- Do: `images.py` per §5.4-E.
- Done when: all frames are WebP with width ≤ 800 and average size ≤ 45 KB (printed by the stage). Visually check that board text is still readable.

**T10 · Sections (stage F)**
- Do: `sections.py` per §5.4-F, with default titles and default alt text.
- Done when: an assertion in code confirms the sections cover `[0, duration]` with no gaps or overlaps and every segment belongs to exactly one section.

**T11 · Pack, validate, VTT, report (stages L–N) without AI**
- Do: `packer.py`, `vtt.py`, `report.py`, `litepack validate`, `litepack report` (§5.4-L/M/N). With `--no-llm`: no summary, glossary or quiz; `languages.available = [source]`. Build both "with audio" and "without audio" sizes for the report.
- Done when: `uv run litepack build ../samples/<A>/lesson.toml --no-llm` produces `dist/<A>/<A>.litepack.zip`, `report.md` and `captions/<A>.en.vtt`; `litepack validate` exits 0; the VTT opens correctly when attached to the video in a player (e.g. VLC: Subtitle → Add file).

**T12 · Fixture and smoke test**
- Do: `tests/fixtures/make_fixture_video.py`:
  1. Draw 3 clearly different 640×360 slides with Pillow (e.g. a black rectangle on the left, a circle on the right, horizontal stripes) as `slide1.png`–`slide3.png`.
  2. Run `ffmpeg -y -v error -framerate 1/10 -i slide%d.png -f lavfi -i anullsrc=r=16000:cl=mono -c:v libx264 -r 25 -pix_fmt yuv420p -c:a aac -shortest slides.mp4`.

  Add `slides.vtt` (3 cues) and a `lesson.toml` with `id = "fixture"` (the app's accessibility tests use routes like `#/pack/fixture/lesson`) and `captions = "slides.vtt"`. Then write the tests:
  - `test_keyframes.py`: unit tests on synthetic numpy frames, covering cuts, `min_scene_sec`, dedupe and cap.
  - `test_build_smoke.py`: full build `--no-llm` on the fixture; asserts 3 frames (±1), validation passes, and the zip contains only allowed paths.
  - Copy the resulting pack to `app/tests/fixtures/sample.litepack.zip` for the app's tests.
- Done when: `uv run pytest` passes.

### M2: App core, offline, deployed

**T13 · App scaffold**
- Do:
  1. `npm create vite@latest app -- --template preact-ts`, then `cd app && npm i && npm i idb fflate && npm i -D vite-plugin-pwa @vite-pwa/assets-generator json-schema-to-typescript vitest fake-indexeddb @playwright/test @axe-core/playwright && npx playwright install chromium`.
  2. Apply the Vite 8 compatibility note (§2) if npm reports a peer-dependency conflict.
  3. Write `vite.config.ts` from §6.2 and generate the icons.
  4. Create `base.css` (§6.12), `router.ts`, `settings.tsx`, `i18n/`, `Layout` with `SkipLink`, an empty `Library`, `About` and `NotFound`.
  5. Add the npm script `"build:check": "vite build"` now; CI calls it from T19. T31 extends it to `vite build && node scripts/size-check.mjs`.
- Done when: `npm run build` succeeds; `npm run preview` shows the Library; Chrome DevTools → Application shows a valid manifest and an activated service worker; keyboard Tab order starts with "Skip to content".

**T14 · Generated types**
- Do: Add the script `"gen:types": "json2ts -i ../schema/litepack-1.schema.json -o src/types/pack.gen.ts"` and run it.
- Done when: `Manifest` and the related types compile and are used by `db.ts`.

**T15 · Storage, import and validation**
- Do: `db.ts` (§6.4), `validate.ts` and `importPack.ts` (§6.5), `ImportButton`, and localised error messages. Unit tests (`vitest` + `fake-indexeddb/auto`) building test zips with `fflate.zipSync`:
  - imports a valid pack
  - rejects non-zip input
  - rejects a missing manifest
  - rejects the wrong `format`
  - rejects a broken cross-reference
  - rejects a missing frame file
  - rejects oversized input
  - **ignores** `../evil.txt` and `__MACOSX/` entries
- Done when: `npx vitest run` passes, and importing `tests/fixtures/sample.litepack.zip` in the browser shows it in the Library.

**T16 · Library and catalog download**
- Do: `PackCard`, `catalog.ts` (fetch `packs/index.json`, download with a progress stream, then `importPack(..., 'catalog')`), offline message, `StorageMeter`, delete with a `<dialog>` confirmation, and a one-time `navigator.storage.persist()` after the first import.
- Done when: with `app/public/packs/` containing the fixture pack plus `index.json`, "Save for offline · N KB" downloads it with visible progress, and the card shows size, languages and badge. Delete removes the pack and its progress.

**T17 · Lesson screen**
- Do: `usePack` (§6.6), the pack header (`LangModeToggle`, `PackTabs`), and the `Lesson` page (§6.7) with `Figure`, `TranscriptBlock`, `MachineBadge`, `?seg=` deep link with focus, "Mark part as done", and `lastSectionId`.
- Done when: lesson A's pack (built in T11) renders every section with its image, alt text and transcript. `?seg=t0010` scrolls to and focuses that line. Object URLs are revoked when leaving the page (check DevTools → Memory or the console log in dev).

**T18 · Offline**
- Do: Confirm the Workbox precache list contains only the app shell (DevTools → Application → Cache Storage) and that `packs/**` is excluded. Write the Playwright offline test (this file §3).
- Done when: the e2e test passes **and** a manual check works: open the preview, import a pack, DevTools → Network → **Offline**, reload, open the pack. Everything works.

**T19 · Deploy to GitHub Pages**
- Do: Add `.github/workflows/deploy.yml` (this file §4) and follow the deployment checklist (this file §5). Build lesson A's pack (no AI is fine for now), run `litepack catalog`, and commit `app/public/packs/`.
- Done when: `https://<user>.github.io/<repo>/` loads on a **real phone**, saves the pack, and opens it in airplane mode. Put the URL in the README.

### M3: AI and remaining screens

**T20 · LLM client**
- Do: `llm.py` per §5.4-G, including the disk cache. Add a `litepack llm-ping` debug subcommand that sends a tiny JSON request and prints the model's reply.
- Done when: `llm-ping` works with the Gemini OpenAI-compatible endpoint (`VERIFY` the model ID first). A second identical call is served from cache. A fake 429 (unit test with a stub client) triggers backoff.

**T21 · Enrichment**
- Do: `enrich.py` per §5.4-H (titles, summary, glossary, quiz, post-checks), plus `readability.py`. Then P1: alt text with vision in batches of up to 4 images.
- Done when: lesson A builds with real titles and summary, a glossary of 5–10 terms, and 5 quiz questions. Every `refSegmentId` and `firstSegmentId` exists (validator passes), and `report.md` shows the summary's FK grade (aim ≤ 7). A human reads the quiz and confirms every answer is correct per the transcript; log any fixes in `TESTING-LOG.md`.

**T22 · Translation**
- Do: `translate.py` per §5.4-J, with glossary-first fixed terms. Set review statuses to `machine`. Generate the `sw` VTT.
- Done when: `languages.available == ["en","sw"]`; every key has a translation (code check); a fluent reader spot-checks 10 random segments plus the quiz and records their notes in `TESTING-LOG.md`. (Edits happen in T29 or by editing `translate.sw.json` and re-running with `--force pack`.)

**T23 · Summary screen**
- Done when: the summary, key points and glossary (`<dl>`) render in all three language modes, and the glossary's "Hear it in the lesson" links land on the right segment.

**T24 · Practice screen**
- Done when: all 5 questions can be answered **with the keyboard only**; feedback is announced by a screen reader (`role="status"`); the score is saved; "Review this part" links work.

**T25 · Listen screen**
- Done when: audio plays from the blob URL; tapping a line seeks to it; the current line is highlighted with `aria-current`; speed control works; with "Follow along" off, the page doesn't scroll by itself; the tab is hidden for packs without audio.

### M4: Accessibility, settings, share, Studio

**T26 · Settings**
- Done when: text size (up to 200%), spacing, theme and UI language persist across reloads and apply to every page, with no horizontal scroll at 320 px width and 200% text.

**T27 · Accessibility pass**
- Do: Go through §6.10 item by item and fix gaps. Add the Playwright axe test over every route (this file §3).
- Done when: the axe test reports 0 serious/critical violations; Lighthouse Accessibility is ≥ 95 on Library, Lesson and Practice; the §6.10 checklist is copied into `TESTING-LOG.md` with each item ticked and how it was verified.

**T28 · Share / export**
- Do: `ShareButton` per §6.8, with post-download instructions.
- Done when: on an Android phone, either the share sheet opens with the file, or the file lands in Downloads with instructions shown. Sending it to a **second device** (Bluetooth, USB or a messaging app) and importing it there works. Record which path happened in `TESTING-LOG.md`.

**T29 · Teacher Studio (P1)**
- Done when: a reviewer can edit three translations and one quiz choice, mark "Transcript" reviewed with their name, save, and export. Re-importing the exported file shows "Checked by ‹name›" badges, and `litepack validate` accepts the exported file. The Studio chunk is absent from the Library page's network requests.

**T30 · UI translation and About page**
- Do: Generate `sw.json` from `en.json` with the pipeline's translation function (add a small `litepack translate-ui` subcommand), then have it reviewed. Build the About page per §6.7.
- Done when: switching the UI to Kiswahili leaves no English strings visible (except proper nouns), and About lists every pack's attribution and license, the library licenses, and the AI disclosure.

### M5: Testing and measurement

**T31 · Performance and old-device check**
- Do: Add `app/scripts/size-check.mjs`: gzip-size every file in `dist/` except `packs/`, `sw.js` and `workbox-*.js`. Sum the initial JS (excluding the Studio chunk), the CSS and the total, and **exit 1 if any §6.13 budget is exceeded**. Change the npm script to `"build:check": "vite build && node scripts/size-check.mjs"`. Lighthouse mobile run (repeat visit). DevTools profile: 4× CPU slowdown + "3G" throttling; measure the time to open the biggest pack.
- Done when: the size check passes in CI, and the measured values are written in `TESTING-LOG.md`.

**T32 · Screen-reader and keyboard testing**
- Do: Complete the "Amina path" (open the app → save a lesson → read section 2 in Kiswahili → open the board text → answer the quiz) with: (a) keyboard only on desktop, (b) **TalkBack** on Android, and (c) NVDA (Windows) or VoiceOver (macOS).
- Done when: all three completions are logged with any issues found and fixed.

**T33 · User testing** (protocol in this file §8)
- Done when: 3+ sessions are logged, and the top 3 issues are fixed or listed as limitations.

**T34 · Final numbers**
- Do: Rebuild both demo packs from clean (`--force all`), run `litepack catalog`, redeploy, and copy the numbers from each `report.md` into the README and the Devpost draft (Brief §4 table).
- Done when: no `‹MEASURE›` placeholders remain anywhere.

### M6: Submission

**T35 · Demo video (2–3 min) with captions**: script in this file §6.
- Done when: the video is uploaded (YouTube unlisted or public), with captions **generated by running LitePack's own pipeline on the demo video** (VTT from stage M, corrected by hand) and uploaded as a caption track. The transcript is also pasted into the Devpost page.

**T36 · Screenshots** (`docs/screenshots/`, each with alt text in the Devpost page):
  1. `report.md`, the size comparison
  2. Library with the size shown before download
  3. Lesson in "Both" mode (English + Kiswahili) with a board image
  4. Practice feedback
  5. Listen with the highlighted line
  6. Airplane-mode phone showing the lesson working
  7. Studio review (if built)
  8. TalkBack in use (photo of the phone)

**T37 · README (judge-facing)**
Sections:
- What it is, as a one-line pitch plus the before/after table
- **Try it in 2 minutes** (demo link, steps, airplane-mode test)
- Build a pack (setup and commands)
- How it works (diagram from §1)
- Accessibility
- **Limitations** (Brief §9)
- **Credits & disclosures**: libraries with licenses, content sources with licenses, AI use (Whisper; LLM for titles, summary, glossary, quiz, translation and alt text; AI coding assistance), translation reviewers (with permission), and **pre-existing work: none**

**T38 · Devpost submission:** fill in the template in this file §9. Add every team member. Submit by **Nov 19**. Afterwards, open the public page logged out and check that the links, video and images all work.

---

## 3. Testing strategy (lean; focused on what the demo depends on)

| Layer | Tool | What it covers |
|---|---|---|
| Pipeline unit | pytest | keyframe selection, manifest cross-refs, translation key round-trip (stub LLM), VTT wrapping, LLM retry/JSON-fix (stub client) |
| Pipeline smoke | pytest + ffmpeg | fixture video → valid pack (`--no-llm`, captions import) |
| App unit | vitest + fake-indexeddb | `validateManifest`, `importPack` limits and path safety, `l10n.pick`, time/byte formatting |
| App e2e | Playwright (Chromium, Pixel 5 profile) against `vite preview` | import → lesson renders; **offline reload works**; quiz answerable by keyboard |
| Accessibility | @axe-core/playwright on every route + manual keyboard/TalkBack/NVDA | WCAG 2.2 AA (§6.10) |
| Budgets | `size-check.mjs` | §6.13 |

`app/playwright.config.ts`:
```ts
import { defineConfig, devices } from '@playwright/test';
export default defineConfig({
  testDir: 'tests/e2e',
  use: { ...devices['Pixel 5'], baseURL: 'http://localhost:4173' },
  webServer: { command: 'npm run build && npm run preview -- --port 4173 --strictPort', port: 4173, reuseExistingServer: !process.env.CI },
});
```

Offline test (sketch):
```ts
test('lesson works offline after first visit', async ({ page, context }) => {
  await page.goto('/');
  await page.evaluate(() => navigator.serviceWorker.ready);
  await page.reload();                                         // now controlled by the SW
  await page.getByLabel(/open a lesson file/i).setInputFiles('tests/fixtures/sample.litepack.zip');
  await expect(page.getByRole('status')).toContainText(/saved/i);
  await context.setOffline(true);
  await page.reload();
  await page.getByRole('link', { name: /open/i }).first().click();
  await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
  await expect(page.locator('figure img').first()).toBeVisible();
});
```

Axe test (sketch):
```ts
for (const route of ['#/', '#/pack/fixture/lesson', '#/pack/fixture/summary', '#/pack/fixture/practice', '#/settings', '#/about']) {
  test(`a11y ${route}`, async ({ page }) => {
    /* import fixture once (beforeEach), then: */ await page.goto('/' + route);
    const r = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa', 'wcag21aa', 'wcag22aa']).analyze();
    expect(r.violations.filter(v => ['serious', 'critical'].includes(v.impact ?? ''))).toEqual([]);
  });
}
```

---

## 4. CI

One workflow, `.github/workflows/deploy.yml`, runs app tests on every push and deploys on `main`.

```yaml
name: Test and deploy app
on:
  push: { branches: [main] }
  pull_request:
  workflow_dispatch:
permissions: { contents: read, pages: write, id-token: write }
concurrency: { group: pages, cancel-in-progress: true }
jobs:
  build:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: app } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm, cache-dependency-path: app/package-lock.json }
      - run: npm ci
      - run: npx vitest run
      - run: npm run build:check            # build + size budgets
        env: { BASE_PATH: "/${{ github.event.repository.name }}/" }
      - run: npx playwright install --with-deps chromium && npx playwright test
      - uses: actions/upload-pages-artifact@v3
        if: github.ref == 'refs/heads/main'
        with: { path: app/dist }            # NOTE: path is relative to repo root, not working-directory
  deploy:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    environment: { name: github-pages, url: "${{ steps.deployment.outputs.page_url }}" }
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```
`VERIFY` the current major versions of these actions; bump them if GitHub shows deprecation warnings. The pipeline doesn't run in CI because it needs models and API keys; pipeline tests run locally with `uv run pytest`.

Playwright in CI builds with `BASE_PATH=/` (its own `webServer` command), so e2e and the Pages build are separate builds. That's expected.

---

## 5. Deployment checklist
- [ ] Pages source is "GitHub Actions".
- [ ] `BASE_PATH` matches the repo name; assets load (no 404s in DevTools).
- [ ] `packs/index.json` and the zips are committed, and the zip sizes in `index.json` match the files.
- [ ] HTTPS URL opens on a phone; "Add to Home screen" works; the installed app opens offline.
- [ ] After a redeploy, the phone picks up the new version on next open (`autoUpdate`); packs survive the update.

---

## 6. Demo video script (target 2:30)

| Time | Show | Say (summary) |
|---|---|---|
| 0:00–0:15 | Title card: Amina (fictional), a phone, a data bundle | "Amina's teacher assigned a 12-minute video. On her prepaid bundle, that's ‹X› MB." |
| 0:15–0:35 | Video page on throttled "3G" in DevTools, buffering | The barrier: cost, buffering, English only, gone when the signal drops |
| 0:35–1:00 | Teacher terminal: `litepack build` (sped up) → `report.md` | "One command on the teacher's laptop: ‹X› MB → ‹Y› MB." |
| 1:00–1:50 | Phone in **airplane mode** (show the icon): Library → Lesson in Both mode → board text → Listen (highlighted line) → Practice question | It works offline in Kiswahili and English; audio is optional |
| 1:50–2:05 | TalkBack reading a section; text size at 200% | Accessible by design |
| 2:05–2:20 | Share file → second phone imports it | No data needed to pass it on |
| 2:20–2:30 | "Machine-made" badge → Studio "Checked by …"; limitations card | Honest about AI; teachers stay in control |

Record the phone screen with Android's built-in screen recorder. Keep narration slow and clear. Then caption it with LitePack itself (T35).

---

## 7. Measurement protocol (where every number comes from)

| Number | How to measure | Notes |
|---|---|---|
| Original video size | File size on disk of the source used for the build (`probe.json`) | Compare against a **fair** quality: 360p/480p, not 1080p. State the format. |
| Pack size (with/without audio) | `report.json` | Both are built automatically |
| Ratio | `original ÷ pack` | Report both "×" and "%" |
| Download time | `bytes × 8 ÷ rate` at 0.25 and 1 Mbps | Label as an estimate; state the assumptions |
| App shell size | `size-check.mjs` output, and DevTools Network "transferred" on first load | |
| Offline works | Airplane-mode recording on a real phone | |
| Pack open time | DevTools Performance with 4× CPU slowdown | |
| Readability | `report.json` FK grades | English only |
| Optional real streaming cost | Android Settings → Network → Data usage → Chrome, before and after streaming the video once at 360p | A strong real-world number if you can do it |

---

## 8. User-testing protocol (3–5 people, about 15 minutes each)

- **Who:** classmates, ideally including someone on a slow connection, someone who uses a screen reader or large text, and a Kiswahili reader. Get verbal consent. Record **no names or faces**; use "Tester 1–5".
- **Setup:** their own Android phone if possible; the app URL; Wi-Fi on for the first step.
- **Tasks** (read aloud; don't help unless they're stuck for more than 2 minutes):
  1. "Save the photosynthesis lesson so you can use it later."
  2. "Turn on airplane mode. Open the lesson and find what the board shows in part 2."
  3. "Switch the lesson to Kiswahili, then to both languages."
  4. "Answer the practice questions."
  5. "Make the text bigger."
- **Record per task:** success (yes/with help/no), time, and what confused them.
- **Ask afterwards:** "What would stop you or a friend using this?", "What was easiest?", "Would you trust the translation? Why?"
- **Write up** in `TESTING-LOG.md`: findings → changes made → quotes used in Devpost (with permission).

---

## 9. Devpost page template

- **Name:** LitePack
- **Tagline (≤ 200 chars):** Turn a heavy video lesson into a tiny offline study pack (transcript, board images, audio and a quiz) in the learner's language.
- **Inspiration:** a short description of the data-cost barrier, with cited stats (Brief §3).
- **Who it's for:** Amina (fictional learner) and Mr. Otieno (fictional teacher); say they're composites.
- **What it does:** 4–6 bullets.
- **Before and after:** the filled table from Brief §4, plus the one-line pitch.
- **How we built it:** pipeline stages (ffmpeg → faster-whisper → keyframes → LLM → zip) and the PWA (Preact, service worker, IndexedDB). Link the diagram.
- **Accessibility:** what we did (§6.10 highlights) and how we tested it (TalkBack, NVDA/VoiceOver, axe, keyboard).
- **Challenges:** e.g. choosing key frames for handwriting videos, keeping translations aligned, staying within budget on old phones.
- **What we learned:** honest, specific points.
- **Limitations and what's next:** from Brief §9, plus the P2 list.
- **Try it:** demo URL, a 3-step guide including the airplane-mode test, and setup instructions for building packs (link to README).
- **Credits and disclosures:** libraries, content licenses, AI use, reviewers, and "No pre-existing work; built Oct 28 – Nov 19, 2026 (see commit history)."
- **Media:** screenshots with alt text, the video with captions, and the transcript.
- **Built with (tags):** python, ffmpeg, whisper, gemini, preact, typescript, vite, pwa, indexeddb, workbox, github-pages.

---

## 10. Final checklist (Nov 19)

- [ ] Every MVP item from Brief §5 works on the **live URL** on a real phone in airplane mode
- [ ] No `‹MEASURE›`/`‹CALC›` placeholders remain; numbers match `report.json`
- [ ] Every machine-made text shows a badge; reviewed content shows the reviewer
- [ ] Every pack shows the source attribution and license; About lists them all
- [ ] axe clean; keyboard and TalkBack paths are logged
- [ ] README: try-it steps, setup, limitations, credits, AI disclosure, pre-existing work statement
- [ ] Video has captions **and** a transcript; screenshots have alt text
- [ ] All team members are added on Devpost; submitted; public page checked while logged out
- [ ] No secrets in the repo (`git grep -i "api_key\|AIza"` returns nothing sensitive)

---

## 11. Risks and fallbacks

| Risk | Likelihood | Fallback |
|---|---|---|
| Free-tier LLM quota/429s or model renamed | Medium | Cache means each call happens once; raise `MIN_INTERVAL`; switch to a Flash-Lite model; Ollama locally; worst case `--no-llm` plus teacher-written quiz in Studio |
| Poor Swahili translation quality | Medium | Human review in Studio; badges; or switch to a language the team reads (Brief §8) |
| Speech-to-text errors | Medium | Import human captions where available; `small` → `medium`/`large-v3-turbo`; Studio edits |
| Key frames miss or duplicate content | Medium | Tune with `frames-debug`; `extra_times` / `drop_ids`; `interval` mode |
| `vite-plugin-pwa` incompatible with installed Vite | Low–Med | Pin `vite@^7` (§2) |
| Service worker caching stale builds | Low | `autoUpdate` + `cleanupOutdatedCaches`; in testing, DevTools → Application → "Update on reload" |
| Browser evicts stored packs | Low | `storage.persist()`; the pack file can always be re-imported |
| Web Share rejects zip files | Medium | Download fallback with instructions (already built in) |
| Ogg Opus unsupported (old Safari) | Low for target users | Message plus transcript; documented limitation |
| Scope slip | Medium | Cut line (Brief §5) |
| Licensing doubt about lesson B | Low | Use only lesson A plus a second self-recorded lesson |

---

## 12. Troubleshooting

| Symptom | Fix |
|---|---|
| `pip`/`uv` can't find `ctranslate2` wheels | Use Python 3.12 (`uv python pin 3.12`), then `uv sync` |
| `Unrecognized option 'vsync'` | FFmpeg 9 removed it; delete it, or use `-fps_mode vfr` |
| `Unknown encoder 'libopus'` | Install a full ffmpeg build (Homebrew, apt `ffmpeg`, gyan.dev builds on Windows) |
| Whisper repeats the same sentence | Confirm `condition_on_previous_text=False` and `vad_filter=True` |
| Whisper is very slow | Use a smaller model (`base`), use a GPU, or import captions |
| LLM returns non-JSON | The JSON-fix loop handles it; if it persists, lower the batch size and check that the model ID supports JSON output |
| `400 … response_format` | The client auto-disables JSON mode (§5.4-G); check the logs for the fallback message |
| App assets 404 on GitHub Pages | `BASE_PATH` must be `/<repo>/` with both slashes |
| App not working offline | DevTools → Application → Service Workers must show "activated"; check that the precache list includes `index.html`; reload once online first |
| Changes don't appear after deploy | Close all tabs of the app and reopen (or enable "Update on reload" while testing) |
| File picker greys out the pack on Android | Remove the `accept` attribute (§6.5) |
| `navigator.share` throws `NotAllowedError` | Must be called from a user tap; otherwise use the fallback download |
| Audio won't seek | Make sure `src` is a Blob URL of the whole file, and check `preload="metadata"` |
