# FORK-PLAN — Matyss/meetily customizations

## ⏸️ STATUS: PARKED 2026-08-09 — but roadmap expanded 2026-08-10 (Matt wants WS3 + WS2)
- **WS1 (Polish) = DONE without the fork** — solved in the community app by placing a Whisper model file (see UPDATE 2 below). Daily tool = stock community Meetily.
- **WS3 (auto-stop + auto-Enhance on stop) = Matt's #1 wanted feature (2026-08-10).** Matt keeps forgetting to stop the recording; wants the meeting to end + get Whisper-enhanced + summarized automatically. Bundle with WS2 — both are on-stop post-processing.
- **WS2 (diarization / speaker labels) = wanted, pairs with WS3.** Un-park when ready to build WS3+WS2 together (one on-stop pipeline). Check Teams/Meet native transcript first for naming accuracy.

### ▶️ How to resume (build WS3 + WS2 together) when you pick this up
1. **Build/run the fork in dev:** `cd ~/Documents/Personal/Projects/meetily/frontend && bash dev-gpu.sh` (hot reload; Step 0 toolchain installed — rust/cmake/pnpm/node; sidecar builds via the script). **Not** `pnpm tauri:build` (skips sidecar). Fork already builds → `meetily.app` (Step 0 done).
2. **Rebase on upstream FIRST** (`git fetch upstream && git rebase upstream/main`) — upstream is active (building diarization into PRO; may add auto-stop). Check what landed free before doing work.
3. Build the on-stop pipeline (see WS2+WS3 section below). Effort ~3–5 sessions combined.

> ⚠️ Also fully valid: **don't fork at all — use the non-fork interim options** (calendar-end reminder + folder-watch auto-Enhance to vault). See "Non-fork alternatives" section. Decide fork-vs-scripts before diving in.

---

Our fork of Zackriya-Solutions/meetily. Workstreams community lacks/gates:
**WS1 — Whisper large-v3-turbo backend** (Polish; ✅ DONE via model-file placement, no fork) · **WS2 — free speaker diarization** (pyannote) · **WS3 — auto-stop + auto-Enhance-on-stop** (Matt's top ask).

---

## 🎯 WS2 + WS3 — the on-stop pipeline (roadmap detail for next agent)

Both are **post-recording-stop** work → build as ONE pipeline. Target end-to-end:
```
recording stops (manually OR auto)                     ← WS3 auto-stop
   └─ on `recording-stop-complete` event:
        1. Whisper large-v3-turbo re-transcribe saved audio   ← WS1 engine (exists)
        2. pyannote diarization → speaker turns, mic="You"    ← WS2
        3. merge speakers into transcript by timestamp
        4. summary (Built-in Qwen / Ollama) with speaker-attributed action items
   └─ all automatic, no clicks
```

### WS3 — auto-stop + auto-Enhance
- **Auto-Enhance (easier half):** on stop, auto-run the Whisper retranscribe + summary instead of only the Parakeet auto-summary.
  - Code anchors: `frontend/src/contexts/RecordingPostProcessingProvider.tsx` (listens `recording-stop-complete`), `isAutoSummary` in `contexts/ConfigContext.tsx` (existing auto-summary-on-stop toggle), `audio/retranscription.rs` + `start_retranscription_command` (Whisper retranscribe path already works). → Add a config toggle "auto re-transcribe with Whisper on stop" that fires the retranscribe (provider=whisper, model=large-v3-turbo) in the post-processing provider, then summarize.
- **Auto-stop (harder half):** community has **no** auto-stop (verified — no silence/idle/call-end detection). Add one:
  - **Silence/inactivity:** transcript already emits `[Silence]` segments (`TranscriptView.tsx`) + there's an audio level meter (`AudioLevelMeter.tsx`, `level_monitor.rs`). Trigger stop after N min of continuous silence.
  - **Calendar-end (optional):** stop at the active meeting's scheduled end (reuse the collision-guard's icalBuddy read).
  - Stop entry point exists: `src-tauri/src/audio/recording_commands.rs::stop_recording` (also called from `tray.rs`). Wire the silence-watcher to call it.

### WS2 — diarization (pairs into step 2 above)
- Engine present but **unwired**: `audio/stt.rs` (pyannote Segmentation+Embedding+identify), DB has a `speaker` field, `retranscription.rs` is the hook. Wire pyannote on the saved audio → align speaker turns to Whisper timestamps → store `speaker` per segment → UI show/rename → mic="You". Cross-meeting embeddings can auto-name recurring people (`EmbeddingManager`).
- Accuracy: good-not-perfect on mixed system audio (anonymous Speaker 1/2/3 + reliable "You").

### Priority for next agent
1. WS3 auto-Enhance-on-stop (small, high daily value — kills the "forgot to stop → only Parakeet" gap).
2. WS3 auto-stop on silence (medium — the real hands-off win).
3. WS2 diarization (medium — speaker-attributed action items). Do after 1–2 since it slots into the same on-stop pipeline.

---

## Non-fork alternatives (interim, or if we decide against maintaining a fork)
Considered 2026-08-10 — usable NOW without touching Meetily's code:
- **Alt-A — calendar-end reminder** (~30 min): launchd + icalBuddy (reuse collision-guard) → at each meeting's scheduled end (+buffer) iMessage "🔴 '<title>' ended — stop Meetily". Fixes the *forgetting*, not auto-stop. Robust.
- **Alt-B — folder-watch auto-Enhance to vault** (~1 h): launchd/fswatch on `~/Movies/meetily-recordings/` → on a finished `audio.mp4` run `whisper-cli large-v3-turbo` + Ollama summary → write a clean note into the Brain vault. Fully automates the Enhance step **once a recording is stopped** (so pair with Alt-A). Output lives in the vault, not Meetily's UI.
- **Dependency:** nothing processes until a recording is stopped → Alt-B needs Alt-A (nudge) or WS3 (auto-stop). Full hands-off = fork (WS3).

> Adopt Meetily as-is for daily use NOW; this fork adds the two gaps. Build from source = a dedicated session (Rust/Tauri toolchain). Keep synced with upstream (rebase, don't diverge hard).

---

## 1. Investigation (measured on Mini M4 16GB, 2026-08-09)

### Transcription benchmark (31.6s PL+EN sample, same content as our custom A/B)
| Model | Speed | RAM | Polish tech-jargon |
|---|---|---|---|
| Parakeet `parakeet-tdt-0.6b-v3-int8` (community default) | ~7× realtime | ~1 GB | ❌ "flamy gorka play" (Playwright), "Piteling Compinow" (pipeline CI), missed "piątku" |
| **Whisper large-v3-turbo** (whisper.cpp, Metal) | **2.5× realtime** (12.8s / 31.6s) | ~2 GB | ✅ "PlayWrit", "pipeline Continuous Integration", caught "do piątku" — near-perfect |
| Whisper large-v3 (full) | ~1× realtime (est.) | ~3 GB | best, but too slow for live |

**Lag implication:** Parakeet live lag ~2-3s (streaming). Whisper isn't natively streaming → live lag ≈ window + processing (~5-8s with turbo). Turbo *keeps up* (2.5×) but laggier live. → **Use Whisper for post-pass, keep Parakeet live.**

### Fork structure (grounded)
- **Transcription model registry:** `frontend/src-tauri/src/config.rs` → `DEFAULT_PARAKEET_MODEL = "parakeet-tdt-0.6b-v3-int8"`; `parakeet_engine` module; provider set in `onboarding.rs` (`provider=parakeet`). Adding Whisper = new provider/engine + model registry + onboarding/settings option.
- **Diarization code ALREADY PRESENT:** `frontend/src-tauri/src/audio/stt.rs` uses `pyannote::{models(Embedding,Segmentation), embedding::EmbeddingExtractor, identify::EmbeddingManager, segment::SpeechSegment}` — downloads embedding + segmentation models. So the engine exists; **"PRO" is likely a UI/license gate, not missing code.** Early spike: find where diarization is hidden/disabled in community build.
- **Re-transcribe path exists:** `audio/retranscription.rs` + Beta "Import & Retranscribe" → natural home for a Whisper post-pass.
- **Whisper infra partly present:** `backend/whisper-custom/server` (whisper.cpp server).
- **Build:** `docs/BUILDING.md`, `building_in_linux.md`, `GPU_ACCELERATION.md`, `architecture.md`. Rust/Tauri + Next.js frontend, Python backend, SQLite (sqlx).

---

## 1b. SPIKE RESULTS (2026-08-09) — reshapes the plan

- **Toolchain installed:** cmake 4.4, pnpm 11.20, rust 1.97, node 26. `pnpm install` (frontend deps) ✅. Full `pnpm tauri:build` kicked off (Step 0).
- **WS1 (Whisper) is ALREADY BUILT + UI-WIRED upstream — probably zero fork code needed:**
  - `frontend/src-tauri/src/whisper_engine/` = full whisper.cpp integration; **already supports `large-v3-turbo` + `large-v3`** (hardcoded download URLs in `whisper_engine.rs`).
  - `audio/retranscription.rs` routes to a `whisper` provider (`get_or_init_whisper`, "default to whisper"); `start_retranscription_command` is a **registered Tauri command** exposed to the UI.
  - Frontend: `constants/modelDefaults.ts` → `DEFAULT_WHISPER_MODEL = 'large-v3-turbo'`; ConfigContext `whisperModel`. Beta **"Import & Retranscribe"** feature is the entry point.
  - → **Action: TEST in the community app first** (Beta → Import & Retranscribe with Whisper large-v3-turbo on a Polish meeting). If it fixes PL (benchmark says yes), WS1 needs no fork.
- **WS2 (diarization) — the real fork work, but NOT a paywall:**
  - pyannote code exists in `audio/stt.rs` (Segmentation+Embedding+identify) **but is dead/unwired** — zero callers in the current parakeet/whisper pipeline (grep found none). Looks screenpipe-derived legacy.
  - **No PRO/license gate in the code** — "diarization = PRO" is marketing for their hosted build. In open code it's simply *not connected*.
  - → Work = **wire the unused pyannote path into the retranscription flow** (post-pass: Whisper transcript + pyannote speakers), + mic="Me". Real integration, not a flag flip.

**Revised recommendation:** WS1 = validate via existing community feature (no build). Fork's real job = WS2 (wire diarization). Full build (Step 0) still useful as the WS2 dev base.

### UPDATE (2026-08-09, after UI check + real-world test) — fork IS needed for Polish
- **Real-world PL test PASSED for Whisper:** on a real Polish YT talk ("Podstawy Amazon AWS", 100s), `whisper-cli large-v3-turbo -l pl` = near-perfect incl. jargon (AWS/EC2/instancje/Firewall/Ubuntu/Monitoring/CPU), **~15× realtime** (6.5s/100s). Parakeet mangles this jargon. (Earlier synthetic `say` failure was TTS + silence-gap loops — not representative.)
- **BUT community UI exposes Parakeet ONLY** — Import→Advanced Options→Model dropdown has a single option (`parakeet-tdt-0.6b-v3-int8`); "Language selection isn't supported for Parakeet". Whisper engine + `large-v3-turbo` exist in code but are **not surfaced**. → **Polish fix is NOT reachable without the fork.**
- **So the fork is justified for BOTH, and WS1 is now FIRST + small:**
  - **WS1 (do first, small):** surface Whisper large-v3-turbo in the transcription/import Model dropdown + model download + enable language selection for whisper. Engine/model-URLs/retranscribe path already exist — mostly UI + model-management wiring.
  - **WS2 (after, bigger):** wire the dead pyannote diarization code.
- Endpoint: fork becomes primary app (community redundant).

### UPDATE 2 (2026-08-09) — WS1 SOLVED IN COMMUNITY, ZERO FORK ✅
The dropdown filters to **downloaded** whisper models (`whisper_get_available_models` → status Available = `.bin` exists in `<app_data>/models`). Community just had no whisper model downloaded. **Fix = place the file** (we had OSW's): `cp ggml-large-v3-turbo.bin ~/Library/Application Support/com.meetily.ai/models/`. After relaunch, **🏠 Whisper: large-v3-turbo appears in the Import/retranscribe Model dropdown + Language selector enables.** Imported the real Polish AWS clip with Whisper → **perfect PL + jargon** (Amazon Web Services, EC2, Management Console, wirtualne maszyny, instancje). "Import complete, 4 segments."
- **Consequence: WS1 needs NO fork.** Community app does Polish via import/retranscribe with Whisper. Workflow = Parakeet live + Whisper retranscribe.
- **Fork's ONLY remaining value = WS2 (diarization).** Optional: a fork could add a Whisper-model *download button* in the UI (community has the download command, no button), but placing the file manually already works.

## 2. Architecture / approach

### Target: 3-layer pipeline (each layer runs on progressively better data)
```
Layer 1 — LIVE      Parakeet streaming → instant live overview during the call (~2-3s lag)
Layer 2 — POST      Whisper large-v3-turbo on saved audio → clean transcript
                    + pyannote diarization → who-said-what (person recognition)
Layer 3 — SUMMARY   LLM summarizes the CLEAN + DIARIZED transcript (Layer 2 output, NOT live)
                    → best quality, correct names/jargon, speaker-attributed action points
```
Key: the **summary is the last layer and consumes the post-processed transcript**, so it inherits Whisper's PL accuracy + diarization ownership. Live panel stays Parakeet (low lag); heavy work is post-meeting.

### Summary backend (Layer 3) — decision
- **Default: Built-in Qwen 3.5 4B** (offline, ~2.6 GB, 32k ctx). A 1-hr meeting ≈ 12-15k tokens → 32k is enough for a single meeting; output quality already good.
- **Switch to Ollama when:** (a) higher quality on messy transcripts (qwen3:8b/14b), or (b) **long context** (whole-day meetings, or future "previous meetings with this client" cross-context — where Ollama's configurable context length, up to 128k+, matters). Built-in can't tune context beyond its fixed window; Ollama can.
- Meetily exposes this as a dropdown (Built-in / Ollama / Claude / OpenAI / Groq / OpenRouter) → runtime-switchable, not locked. Keep everything local (Built-in or Ollama) for NDA.
- **RAM note:** Layers 2+3 are post-meeting (sequential), not concurrent with a live call → Whisper (~2GB) + Ollama big model fit 16GB when run after the meeting. Avoid a huge Ollama model + 128k context during a live call.


### WS1 — Whisper large-v3-turbo backend
- **Primary: post-meeting re-transcribe** (recommended, lowest risk). On stop (or on demand), run Whisper large-v3-turbo over the saved audio → write a clean canonical transcript, replacing the Parakeet live transcript. Zero live-lag cost; fixes PL jargon exactly where it's kept. Leverages existing `retranscription.rs`.
  - Auto-trigger option: "re-transcribe with Whisper on finish" toggle.
- **Optional: selectable live model.** Expose Whisper-turbo as a live transcription provider for PL-heavy calls (accept ~5-8s lag). Behind a setting; Parakeet stays default.
- Model mgmt: reuse OSW's `ggml-large-v3-turbo.bin` or download to Meetily's models dir; register as a transcription provider in `config.rs`/onboarding.

### WS2 — free diarization
- **Spike first:** determine if community diarization is (a) code-complete but UI/license-gated, or (b) partially wired. `stt.rs` suggests (a).
- Un-gate + wire diarization into the pipeline + summary (speaker labels in transcript + summary "who said what").
- **Mic = "Me" for free:** Meetily already captures mic separately → label the mic channel "Me"; pyannote handles the others (Speaker 1/2…). Manual rename in UI; later OCR active-speaker name from Teams/Meet.
- Post-process at meeting end (heavy; not live).

---

## 3. Acceptance Criteria
- **AC1 (WS1 quality):** on the PL jargon sample, Whisper post-pass gets Playwright/pipeline-CI/"piątku" right (WER ≪ Parakeet).
- **AC2 (WS1 lag):** live path unchanged (Parakeet ~2-3s); post-pass completes at ≥2× realtime (30-min meeting re-transcribes in ≤15 min) without blocking the UI.
- **AC3 (WS1 RAM):** Whisper post-pass peak fits 16GB alongside the app (turbo ~2GB) — no thrash.
- **AC4 (WS2):** diarization produces speaker-segmented transcript locally, free; "Me" correctly mapped from mic; no PRO/license prompt.
- **AC5 (integration):** SQLite schema stores speaker + which-engine-transcribed; summaries reference speakers; archive/search still work.
- **AC6 (upstream):** fork builds from source and can rebase on upstream without losing WS1/WS2.

## 4. Tests
- T1 Whisper post-pass on saved PL audio → compare WER vs Parakeet (AC1).
- T2 Lag: live Parakeet latency unchanged; post-pass wall-time on a 30-min recording (AC2).
- T3 RAM soak during post-pass + app open (AC3).
- T4 Diarization on a 2-3 speaker sample (mic + system) → correct segments, "Me" mapped (AC4).
- T5 DB/summary/search regression after schema use (AC5).
- T6 Clean `cargo tauri build` + rebase-on-upstream dry run (AC6).

## 5. Steps (chunks)
- [x] **Step 0 — toolchain + deps** (cmake/pnpm/rust/node, `pnpm install`). ⚠️ **Build gotcha:** on macOS use **`frontend/build-gpu.sh`** (release) or **`dev-gpu.sh`** (debug/hot-reload) — NOT `pnpm tauri:build`. The npm script (`scripts/tauri-auto.js`) only detects GPU + runs tauri; it does **not** build the `llama-helper` sidecar, so `pnpm tauri:build` fails with `binaries/llama-helper-aarch64-apple-darwin doesn't exist`. The `.sh` scripts build the sidecar (`cd llama-helper && cargo build --release --features metal`) + copy it to `src-tauri/binaries/llama-helper-<triple>` first. (docs/BUILDING.md macOS section is wrong on this.) ✅ **`build-gpu.sh` produces a working `target/release/bundle/macos/meetily.app`** (sidecar + Metal, ad-hoc signed). Only the final **DMG packaging fails** (`bundle_dmg.sh`) — cosmetic/installer-only, `.app` runs fine; ignore for dev. For WS2 dev use **`dev-gpu.sh`** (hot reload). ⚠️ **Name/bundle-id collision:** fork build = same name + `com.meetily.ai` as community app → do NOT install fork to /Applications (clobbers community / "in use"). Keep community in /Applications as daily tool; run fork from build folder or `dev-gpu.sh`. To coexist as separate apps later: rebrand fork (rename + new bundle id in `tauri.conf.json`).
- [ ] **Step 1 — diarization spike:** locate the gate in `stt.rs`/pipeline/frontend; determine effort to enable (WS2 may be quick).
- [ ] **Step 2 — WS1 post-pass:** wire Whisper large-v3-turbo into `retranscription.rs` + a "re-transcribe on finish" toggle. (T1-T3)
- [ ] **Step 3 — WS2 enable + wire diarization + mic="Me".** (T4-T5)
- [ ] **Step 4 — optional live Whisper model** behind a setting.
- [ ] **Step 5 — regression + upstream rebase check.** (T6)

## 6. Risks / notes
- **Fork maintenance:** upstream is active — they may ship Whisper/diarization themselves; check their roadmap before heavy work (maybe just wait/PR). Prefer minimal diffs + rebase.
- **Build complexity:** Rust/Tauri first build is heavy; GPU_ACCELERATION.md for Metal.
- **Recordings location:** stays `~/Movies/meetily-recordings` (Matt's call); no change.
- Daily use: Meetily community as-is until this lands.
