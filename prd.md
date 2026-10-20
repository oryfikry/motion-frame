# Motion Frame — PRD + Implementation Plan

## Context
Want local prompt→video tool on top of HeyGen **hyperframes** (Apache-2.0; HTML+GSAP compositions rendered deterministically by headless Chrome + FFmpeg). Hyperframes ships a CLI + Studio code editor, but no "type prompt, pick ratio, get narrated video" UI, no OpenAI-compatible gateway config, no batch queue UI. Goal: single-user web app (`C:\f\ai\motion-frame`, currently empty) that turns prompts into narrated MP4s, picks Lottie / Three.js / text elements via an LLM "skill", supports batch + logs.

Decisions (confirmed): own app with hyperframes as npm dep · Vite+React+daisyUI frontend, Hono API · TTS via gateway **or** local Kokoro (selectable) · single-user, no auth.

## Product requirements

| # | Feature | Acceptance |
|---|---|---|
| F1 | **Prompt input** | Textarea + template select + ratio + duration hint + narration toggle/voice → "Generate" creates job, redirects to job page |
| F2 | **Ratio** | Landscape 16:9 (1920×1080), Portrait 9:16 (1080×1920), Square 1:1 (1080×1080); sets root `data-width/height` |
| F3 | **AI gateway settings** | Base URL, API key (masked after save), chat model, TTS model, voice, "Test connection" (`GET {base}/models`). Any OpenAI-compatible endpoint (OpenAI, OpenRouter, LiteLLM, Ollama…) |
| F4 | **Scene-planner skill** | LLM receives skill doc (when to use Lottie vs Three.js vs text/CSS vs image) + tool `search_lottie`; returns validated scene JSON |
| F5 | **LottieFiles search** | Tool queries LottieFiles public GraphQL, returns top N (name, preview, JSON url); chosen JSON downloaded into job dir (no network at render time) |
| F6 | **Three.js element** | 3 seekable presets (particles, rotating shape/logo, globe) driven by timeline time, three vendored locally |
| F7 | **Narration** | Per-scene narration text → TTS (gateway `/audio/speech` or `npx hyperframes tts` Kokoro) → scene durations fitted to audio length (ffprobe); `<audio>` track in composition. Optional background music w/ `data-fx-carve` ducking |
| F8 | **Templates** | 4 video templates (Minimal, Bold Promo, Explainer, Social Story) whose palettes come from daisyUI themes; gallery page with thumbnail |
| F9 | **Batch** | Paste multi-line prompts or upload CSV (`prompt,ratio,template,voice`), optional "all ratios" fan-out → N jobs in queue; configurable concurrency |
| F10 | **Logs** | Every step (LLM call, tool call, TTS, lint, render progress, errors) written to DB; live SSE stream on job page; global Logs page with level/job filter |
| F11 | **Jobs** | Table w/ status/progress, video player, download MP4, open composition HTML, retry, cancel (AbortSignal) |
| F12 | **App UI themes** | daisyUI theme switcher (light/dark/cupcake/synthwave…), stored in localStorage |
| F13 | **GPU auto-detect** | On startup + "Re-scan" button: find GPU and hardware encoders; Settings shows result; render uses GPU when found (`useGpu`), CPU otherwise; mode `auto` (default) / `on` / `off` |

Non-goals v1: auth/multi-user, timeline editor (hyperframes Studio exists — link "Open in Studio" via `npx hyperframes preview <jobdir>`), cloud/lambda rendering, LLM writing freeform HTML (add later as "Freeform" template, gated by `hyperframes lint`).

## Architecture

```
web (Vite+React+daisyUI) ──HTTP/SSE──> server (Hono, Node 22, tsx)
                                         ├─ db.ts       node:sqlite  (settings, jobs, logs)
                                         ├─ queue.ts    in-process queue, concurrency from settings
                                         ├─ pipeline.ts plan → assets → tts → compose → lint → render
                                         ├─ llm.ts      fetch {base}/chat/completions w/ tools
                                         ├─ lottie.ts   GraphQL search + download
                                         ├─ tts.ts      gateway /audio/speech | `hyperframes tts`
                                         ├─ gpu.ts      detect GPU + HW encoders, cached result
                                         └─ render.ts   @hyperframes/producer createRenderJob/executeRenderJob
```
No OpenAI SDK — plain `fetch` (OpenAI-compatible JSON is tiny). No ORM — `node:sqlite`. No queue lib — array + running counter.

### Folder layout
```
motion-frame/
  package.json            # scripts: dev (concurrently vite + tsx watch), build, start
  server/
    index.ts  db.ts  queue.ts  pipeline.ts  llm.ts  lottie.ts  tts.ts  render.ts  compose.ts  gpu.ts
    skills/scene-planner.md     # the "skill": when to use which element + JSON schema
    three-presets/*.js          # particles.js, shape.js, globe.js (seekable: render(t))
  templates/<id>/template.html  meta.json  thumb.png
  web/  index.html  src/{main.tsx, App.tsx, api.ts, pages/{Generate,Batch,Jobs,Job,Logs,Settings,Templates}.tsx}
  data/ app.db  jobs/<jobId>/{index.html, assets/, narration/, out.mp4}
```

### Data model (node:sqlite)
- `settings(key PK, value)` — base_url, api_key, chat_model, tts_provider(`gateway|kokoro`), tts_model, tts_voice, concurrency, fps, quality, music_path, gpu_mode(`auto|on|off`), gpu_info_json (cached detection)
- `jobs(id, batch_id, status[queued|planning|assets|tts|composing|rendering|done|failed|cancelled], progress, prompt, ratio, template, narration, voice, plan_json, output_path, error, created_at, updated_at)`
- `logs(id, job_id NULL, ts, level, step, message, data_json)`

### API (Hono)
`GET/PUT /api/settings` (key masked on GET) · `POST /api/settings/test` · `GET /api/templates` · `POST /api/jobs` · `POST /api/batches` (json lines or CSV multipart) · `GET /api/jobs?batch=` · `GET /api/jobs/:id` · `POST /api/jobs/:id/retry|cancel` · `GET /api/jobs/:id/events` (SSE logs+progress) · `GET /api/logs?job=&level=` · `GET /files/jobs/:id/*` (static video/html) · `GET /api/lottie/search?q=` (manual preview in UI) · `GET /api/gpu` (cached) · `POST /api/gpu/scan` (re-detect)

### Job pipeline (`pipeline.ts`)
1. **Plan** — `llm.chat()` with system = `scene-planner.md` + template meta + ratio; tools: `search_lottie(query)`, `list_three_presets()`. Loop tool calls (max 6 rounds). Final answer = JSON `{title, scenes:[{id, narration, headline, body?, element:{type:"lottie"|"three"|"text"|"image", lottieUrl?, preset?, params?}, durationHint}]}`. Validate by hand-written check; on bad JSON re-ask once with error, then fail. Log every request/tool call.
2. **Assets** — download Lottie JSON to `assets/`, copy `lottie-web` + `three` builds from node_modules to `assets/lib/`.
3. **TTS** — per scene: gateway `POST {base}/audio/speech {model, voice, input}` → mp3, or `npx hyperframes tts "<text>" -o x.wav --voice v`. `ffprobe` duration → scene duration = max(hint, audio + 0.4s pad). Skip if narration off.
4. **Compose** — `compose.ts` fills `template.html` (string replace of `{{SCENES}}`, `{{W}}`, `{{H}}`, `{{DURATION}}`, theme vars) producing root with `data-composition-id`, `data-width/height/duration`, scene clips with `data-start/data-duration/data-track-index`, `<audio>` per narration clip, and one paused GSAP timeline registered on `window.__timelines[id]`. Lottie scenes: `lottie.loadAnimation({autoplay:false})` + timeline `onUpdate` → `anim.goToAndStop(frame, true)`. Three scenes: preset `render(t)` called from timeline `onUpdate`. (Check first whether hyperframes' built-in Lottie/Three adapters make onUpdate glue unnecessary — prefer them if so.)
5. **Lint** — `npx hyperframes lint <jobdir>` (+ `check`); failures logged, job failed with message.
6. **Render** — `createRenderJob({fps, quality, format:"mp4", entryFile:"index.html", useGpu})` → `executeRenderJob(job, jobDir, out.mp4, (j,msg)=>log+progress, abortSignal)`. Progress pushed via SSE. `useGpu` = `on` → true, `off` → false, `auto` → result from gpu.ts. If a GPU render throws, log a warning and retry the job once on CPU.

### GPU detection (`gpu.ts`)
Runs once at startup, cached in `settings.gpu_info_json`, re-run via `POST /api/gpu/scan`. Each probe is a `child_process.execFile` with a 5s timeout; a failure means "not found":
- **GPU name/vendor** — `nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv,noheader`; Windows fallback `powershell Get-CimInstance Win32_VideoController | Select Name,AdapterRAM`; Linux fallback `lspci | grep -i vga`; macOS `system_profiler SPDisplaysDataType`.
- **HW encoders** — `ffmpeg -hide_banner -encoders`, look for `h264_nvenc|hevc_nvenc` (NVIDIA), `h264_qsv` (Intel), `h264_amf` (AMD), `h264_videotoolbox` (Apple). Then confirm with a 1-frame test encode (`ffmpeg -f lavfi -i color=c=black:s=256x256:d=0.1 -c:v <enc> -f null -`), because ffmpeg lists encoders the driver may not support.
- **Chrome GPU** — skipped at first; rely on hyperframes `--gpu`. Add a check only if renders show it's needed.
- Result `{vendor, name, vramMB, encoders:[...], usable:boolean, checkedAt}`, logged at `info` level.
- **First task:** check what hyperframes `useGpu`/`--gpu` actually turns on (Chrome GPU raster, the ffmpeg encoder, or both) and whether it detects anything itself. If producer already picks the encoder, gpu.ts only decides `useGpu` and reports to the UI.
- Settings UI: a `stat` card (GPU name, VRAM, encoders, badge "GPU in use" / "CPU only"), a `select` for gpu_mode, and a "Re-scan" button. The Settings page also shows a GPU-vs-CPU benchmark button (`npx hyperframes benchmark` if it fits, otherwise a 3s test render timed both ways).

### Skill doc (`server/skills/scene-planner.md`) — key rules
- **Lottie**: icons, illustrated characters, UI micro-interactions, checkmarks/loaders, friendly/explainer tone. Always `search_lottie` first; pick result whose name matches concept; ≤1 Lottie per scene.
- **Three.js**: abstract tech/AI/space/data themes, product hero, intro/outro backgrounds; use preset + params (color, speed); max 2 per video (render cost).
- **Text/CSS (GSAP)**: titles, stats, lists, quotes, CTA — default when no visual concept is clear.
- **Image**: only if user prompt supplies URL.
- Narration: 1–2 sentences/scene, ~150 wpm; total ≈ requested duration.
- Portrait: fewer words per line, stacked layout; landscape: split layout.
- Also borrow guidance from upstream `skills/hyperframes-animation` & `hyperframes-core` (determinism: no Date.now/random/network).

### LottieFiles search
Public GraphQL `https://graphql.lottiefiles.com/2022-08` query `searchPublicAnimations(query, first)` → `jsonUrl, name, previewUrl`. **Verify endpoint/shape at start of implementation**; fallback: scrape-free option = curated local Lottie set in `assets/lottie-library/` with tag index.

### UI (daisyUI 5 + Tailwind 4)
- `navbar` + `drawer` sidebar: Generate · Batch · Jobs · Logs · Templates · Settings; theme `select` (`data-theme`).
- Generate: `textarea`, ratio as `join` of radio buttons with aspect icons, template `card` picker, narration `toggle` + voice `select`, `range` duration.
- Batch: `textarea` (1 prompt/line) or `file-input` CSV, ratio checkboxes (fan-out), preview `table` of N jobs, Submit.
- Jobs: `table` + `progress` + `badge` status; Job page: `<video>`, `steps` (plan→assets→tts→compose→render), live log `mockup-code`, scene plan `collapse`.
- Logs: `table` with level/job filters, auto-refresh.
- Settings: `fieldset` forms, Test connection button → `alert`.

## Milestones
1. **Skeleton** — package.json, Hono server, node:sqlite schema, Vite+daisyUI shell, Settings page + test connection.
2. **Render spike + GPU** — hard-coded composition (text + Lottie + Three preset) rendered via producer in 3 ratios. Confirms adapter approach before building templates. gpu.ts + Settings GPU card; same spike rendered with GPU and CPU and the times compared.
3. **Pipeline** — llm.ts tools loop, lottie.ts, compose.ts, 1 template, Generate + Job pages, SSE logs.
4. **Narration** — tts.ts both providers, duration fitting, audio in composition.
5. **Batch + queue + Jobs/Logs pages**, cancel/retry.
6. **Remaining 3 templates + gallery thumbnails** (`npx hyperframes snapshot`).

## Risks
- LLM JSON quality varies across gateway models → strict schema in prompt + one repair retry; `response_format: json_object` when supported.
- Render CPU heavy → default concurrency 1, workers from settings.
- Kokoro needs local python/onnx install → `hyperframes doctor` check shown on Settings page.
- LottieFiles API is undocumented/public — may change; fallback library.
- API key stored plaintext in local SQLite — acceptable single-user; never returned to client unmasked.

## Verification
- `npx hyperframes doctor` passes (Node 22+, ffmpeg, chrome).
- `npm run dev` → open UI, Settings: point to gateway, Test connection green.
- Generate "30s explainer about solar panels", portrait, narration on → job reaches `done`, MP4 1080×1920 plays with voice, `ffprobe` shows audio stream + expected duration.
- Same prompt landscape/square → correct dimensions.
- Prompt mentioning "3D AI network" → plan uses Three preset; "checklist onboarding" → plan uses Lottie (check logs show `search_lottie` call).
- Batch: 3 prompts × 2 ratios = 6 jobs, concurrency 2 respected, all done; cancel one mid-render → `cancelled`.
- Logs page shows LLM/tool/TTS/render entries; failing gateway key → job `failed` with clear error.
- GPU: Settings shows the detected GPU and encoders (cross-check against `nvidia-smi` / Device Manager). In `auto` mode the render log shows `useGpu=true` and is faster than `off`. On a machine without a GPU, or with an ffmpeg that has no HW encoders, the card says "CPU only" and renders still work. A forced GPU failure falls back to CPU once and logs a warning.
- Small `server/compose.test.ts` (node:test): scene JSON → HTML has correct data-attributes/durations sum.