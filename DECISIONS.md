# Locked Decisions (plain English)

_Running record of what we've settled. Update as decisions are made. Last updated 2026-07-02._

### Product
1. **What we're building:** a multi-niche AI short-form video engine for **Instagram Reels + YouTube Shorts** (TikTok is banned in our market, ignore it).
2. **Business model:** **Content farm FIRST** — we run our own faceless channels and automate generation + posting. Build with **room for SaaS later** (sell the tool), but don't slow down the farm to build SaaS features now.
3. **The edge:** hybrid render (cheap AI stills + FFmpeg motion, only ONE AI-video "hero shot" per video) + per-niche polish + auto-posting. We avoid the "slideshow slop" the cheap tools ship, and we have the auto-posting the quality tools lack.
4. **Anti-slop rule:** YouTube demonetizes templated/identical AI videos, so every niche rotates through several looks/caption-skins. Never stamp out twins.

### Niches (build order — cheapest & best-fit first)
5. **Wave 1:** Reddit/AITA, "Did you know" facts, Motivation. (Reddit ≈ 90% of current code already.)
6. **Wave 2:** Dark history, Personal finance (highest CPM $8–20), Stoicism.
7. **Wave 3:** Horror, Mythology, Movie-recap (these get the single AI-video hero shot).
8. **Wave 4+:** News/politics, true crime, sci-fi lore, science/space, lo-fi loops.

### Tech stack
9. **Keep:** Bun + ElysiaJS + MongoDB + FFmpeg + S3 + OpenRouter (now the single AI provider: LLM + images + TTS + video).
10. **Job queue:** **Redis + BullMQ** (Redis confirmed available). Replaces the in-memory map; stages are resumable.
11. **Provider routing (corrected again — SINGLE PROVIDER):** **OpenRouter does everything** — LLM, images, TTS, AND video. It added a TTS endpoint (`/api/v1/audio/speech`) and an async video endpoint (`/api/v1/videos`) in 2026. One key (`OPENROUTER_API_KEY`), one bill. No fal, no extra gen accounts.
   - The async `/api/v1/videos` submit→poll→download shape maps cleanly onto our BullMQ resumable-stage queue.
12. **Images (OpenRouter):** Seedream 4.x (~$0.03, best style range) workhorse; FLUX.2 Pro / Nano Banana Pro premium. Token-billed (pennies). (Replicate Flux-schnell $0.003 = optional cheap-bulk fallback only.)
13. **TTS (OpenRouter):** **Gemini Flash TTS** is now the default even on cheap tier; Kokoro 82M is kept only as an override/fallback because its robotic delivery hurts Reddit retention and the cost delta is negligible. MAI-Voice-2 remains premium. ElevenLabs/Inworld direct keys (already in env) are optional premium narration upgrades, not dependencies.
14. **Hero video (Wave 3+, OpenRouter):** Seedance 1.5/2.0 (cheapest, ~$0.022–0.047/s) / Kling v3 / Veo 3.1 Fast ($0.09/s) → Standard for premium. **Do NOT build core pipeline on Sora 2** (on OpenRouter but API sunsets ~Sept 2026). Use `/api/v1/videos/models` to read live pricing/params.
15. **FFmpeg confirmed:** local FFmpeg 8.0 has every filter we need (Ken Burns, captions, transitions, grain, tints, HW encode). Validated a real Ken Burns + caption render on 2026-06-30.

### Costs (per ~45s video)
16. **Cheap tier ~$0.05–0.27** (default workhorse). **Premium ~$1–3**, dominated by the one hero clip → gate hero behind a "trend demands it" flag.

### Architecture
17. **Render engine = one pipeline + pluggable `RenderStrategy`** (Gameplay / ImageKenBurns / HybridScene / MotionGraphics). Current code becomes `GameplayOverlayStrategy`, refactored not rewritten. See [architecture/render-engine.md](architecture/render-engine.md).
18. **Data = scene graph:** `VideoProject` owns ordered `Scene`s; niches defined by declarative `FormatRecipe`. `Composition` kept for back-compat. See [architecture/data-model.md](architecture/data-model.md).
19. **Validate recipes in a scratchpad "render lab" before wiring TS** (per request — saves tokens/compute, avoids debugging FFmpeg through code).
    - ✅ **Done 2026-06-30:** full 54s dark-history sample ("Dancing Plague 1518") built on real OpenRouter images + Kokoro TTS + FFmpeg. Looks publishable. Cost ~$0.20 (matches cheap-tier estimate). Findings → [architecture/render-engine.md](architecture/render-engine.md) §6.
    - **One architecture addition the lab forced:** a **forced-alignment stage** (whisper/aeneas) between audio-gen and render, because TTS returns no word timings and the signature word-by-word captions need them. New `aligning` status + `Scene.captionCues`. Everything else stood up unchanged.
    - Hard FFmpeg rules locked: never `-shortest` with `zoompan` (use `-frames:v`+`apad`); image prompts must say "no text/watermark"; images come back square → scale+crop to 9:16; `drawtext` needs `textfile=`.
    - ✅ **Built in code** (`server/src/services/`): `openrouter-media`, `reel-render`, `reel-script`, `reel.service`, `reel.model`. Full pipeline plan→images→audio→render→S3.
    - ✅ **Flow fixes (v2, from user feedback):** scenes now **crossfade** (xfade+acrossfade hidden in a silent tail) instead of cutting to black → no gaps/black dips; captions are **length-weighted** to track real speech → no subtitle bleed. Sample: `docs/samples/dancing_plague_1518_v2.mp4`.

### Cost/credit system (clarification)
20. A **lightweight cost ledger is built now** even for the farm — we need real unit economics (what each video actually costs). The **credit balance + refund-on-failure UX is SaaS-facing** and deferred until we sell the tool. Same `CostLedger` model serves both; only the UI/enforcement differs.

### Reddit / gameplay strategy (built 2026-07-01)
29. **`gameplay_overlay` strategy is built** (`reel-gameplay.service.ts`): LLM story (`planRedditStory`) → per-sentence TTS → concat → looping cropped gameplay bg → **premium Reddit post card** (`reddit-card.service.ts`, SVG→PNG via `@resvg/resvg-js`: rounded corners, avatar, subreddit/username, upvote/comment row, auto-wrap no overflow) overlaid during the title + bouncing amber word captions. Wired into `reel.service` (niche `reddit`/`aita`). Samples: `docs/samples/reddit_*.mp4`.
30. **Gameplay assets pipeline:** `gameplay-ingest.service.ts` + `scripts/ingest-gameplay.ts` crop→9:16, strip audio, split into ~75s segments, upload to `s3://.../gameplay/` + local cache. `pickGameplay` prefers local, falls back to S3/CDN. Sourced footage MUST be license-clear (CC-BY / creator-released / owned) — current pool is CC-BY Minecraft parkour from YouTube "Orbital - No Copyright Gameplay" (**attribution required** per CC-BY). yt-dlp binary at `server/bin/yt-dlp`.
31. **CloudFront CDN created** for the bucket: distribution `E1HKTHTBXTDXPM` → `d3106f4xmrxo1r.cloudfront.net` (PriceClass_100, cache in front of the already-public bucket — no OAC/policy change). `CLOUDFRONT_URL` in env; `config.cdnUrl` + `cdnUrlFor(key)` in s3.service. Verified 200.

### Story engine (Reddit stories agent, built 2026-07-01)
32. **Pluggable story sourcing** (`story.service.ts` + `Story` model): 3 selectable modes — `llm` (original from a theme bank, no API, zero copyright risk), `hybrid` (real Reddit post as inspiration → LLM rewrites original), `verbatim` (real Reddit post narrated near-verbatim). **All 3 verified working 2026-07-01** with a script-type Reddit app (app-only client_credentials OAuth, read-only). Default mode = `config.storyMode` (env `STORY_MODE`, default `llm`).
    - ⚠️ **UA gotcha:** Reddit's WAF *blocks* its own recommended `platform:app:version (by /u/user)` User-Agent at some IPs (returns a "Blocked" HTML page, not a JSON error). A **browser User-Agent** gets through — that's the default now (`config.redditUserAgent`).
33. **Story bank + dedup + queue:** generated stories are deduped (normalized `premiseKey` + `seedUrl`) and banked in Mongo; `takeNextStory` pops the oldest unused (marks used). Reddit reels with no explicit topic (or `topic="auto"`) auto-pull from the bank.
34. **The "agent" = scheduled top-up, now wired in-process.** `startStoryTopUpScheduler()` (`server/src/services/story-scheduler.service.ts`) runs on server boot: checks the bank every `STORY_TOPUP_INTERVAL_MS` (default 30min) and tops up `STORY_TOPUP_COUNT` (default 15) via `topUpStoryBank` whenever `ready < STORY_TOPUP_THRESHOLD` (default 10). Set `STORY_TOPUP_ENABLED=false` for multi-instance deploys and drive `scripts/topup-stories.ts [count] [mode]` from external cron instead (only one process should top up the shared Mongo bank). Becomes a repeatable BullMQ job once the queue (blocker #1) lands.
35. **Genre-based Reddit sourcing added 2026-07-01:** requests can pass `{ genre, source, parts }`. `REDDIT_GENRES` maps lanes like `aita_family`, `wedding_drama`, `petty_revenge`, `malicious_compliance`, `confession`, and `updates` to multiple subreddit targets. Fetching prioritizes Reddit `/top` candidates, then filters/scores by usable body text, keyword fit, engagement, word-count fit, update/cliffhanger markers, and dedupe. Legacy theme bank remains for backward compatibility.
36. **Multipart Reddit reels added 2026-07-01:** `parts: 2|3|4|"auto"` creates a series of reel docs sharing `seriesId`. Non-final parts now append a spoken "Stay tuned for part N" outro plus a centered end card. Verbatim parts use only original Reddit text; LLM only selects cut points.

### Reel API (wired 2026-07-01)
37. **`/api/reels` routes added** (`reel.routes.ts` + `reel.controller.ts`, mirrors the `compositions` pattern): `POST /api/reels` (`{niche, genre?, topic?, tier?, source?, parts?}`), `GET /api/reels`, `GET /api/reels/:id/status`, `GET /api/reels/:id/download`, `DELETE /api/reels/:id`. Previously `createReel()` existed only as an internal service function with no HTTP entrypoint — verified end-to-end via curl (create → poll status → download) before landing.

### Distribution
21. **YouTube Shorts auto-posting: CONFIRMED feasible** via YouTube Data API v3 (`videos.insert`). Quota dropped from ~1,600 → ~100 units/call (Dec 2025), so the free tier allows **~100 uploads/day** — plenty for the farm. OAuth required; vertical ≤3min = Short.
22. **Instagram Reels auto-posting: deferred** (Graph API needs Business account + FB app review, or a paid aggregator). Revisit after the render engine works.

### Models
24. **Central model registry** (`server/src/config/models.ts`): all model choices resolve through `resolveModels(tier)`. Swap = edit one file or set env (`MODEL_TIER`, `IMAGE_MODEL`, `TTS_MODEL`, `TTS_VOICE`, `TTS_FORMAT`, `VIDEO_MODEL`). No service code hardcodes a slug → swapping never harms the architecture. Narration is format-aware (Gemini TTS pcm auto-converts to mp3). Full research: [architecture/model-selection.md](architecture/model-selection.md).
25. **Budget insight:** LLM + TTS are effectively FREE per video (fractions of a cent). Real costs = images (~$0.02–0.13 ea) and the hero video clip. So use GOOD llm/tts (deepseek/gemini + Gemini-TTS or MAI-Voice), optimize images, gate the hero clip. Kokoro TTS has been replaced as default.

### Style system (per-niche art direction)
26. **Per-niche style registry** (`server/src/config/niche-styles.ts`): each niche = a `NicheRecipe` with a POOL of visual styles (rotated per-video for anti-slop, consistent within a video) + strategy + image tier + caption skin + music mood + hero policy + voice + script guide. Research + full mapping: [architecture/style-system.md](architecture/style-system.md).
27. **Locked style choices (research-backed):** horror leads with **analog/liminal** (Backrooms trend) ✅ validated; dark history/true crime = **vintage sepia**; finance/news = **clean 2D motion-graphics/data-viz** (NOT anime/sepia — highest RPM, needs charts); fun_facts = **claymation/Pixar** ✅ validated; mythology = painterly/anime; lo-fi = Ghibli cozy. Trending styles (claymation, pixar_3d, ghibli, anime_cel, diorama) are first-class pool entries → chase a trend by adding one line, no code change.
28. **Adding a niche or style = edit the registry only.** planner + image gen + render all read from it. Image model auto-picks the niche's tier; voice can be overridden per niche (e.g. horror → deeper Kokoro voice until TTS upgrade).

### Trend scout — BUILT 2026-07-02 (was "Deferred" #23, no longer deferred)
38. **`trend-scout.service.ts` built and live-tested.** Pulls top-viewed Reddit-story YouTube Shorts per genre via YouTube Data API v3 (`search.list`+`videos.list`, needs `YOUTUBE_DATA_API_KEY` — separate read-only key from the OAuth publish creds), filters to true Shorts (≤183s), upserts into the extended `TrendReference` model (`externalId`, `title`, `thumbnailUrl`, `channelTitle`, `tags`, `dayOfWeek`/`hourUtc`, `scanWindow`). Cadence: daily = rolling 7-day window, weekly = rolling 30-day window (`scripts/trend-scout-daily.ts [month]`), plus a one-off `scripts/trend-scout-backfill.ts`. `getTrendSummary(period)` gives per-genre top performers + a view-weighted posting-time histogram — explicitly a **competitor-publish-timestamp correlational prior**, not a real view-velocity signal (the public API doesn't expose that); real signal is your own channel's YouTube Analytics once publishing.
39. **`trend-insight.service.ts` (context-window-safe RAG) built.** One cheap LLM call per genre per scout run distills the top ~12 `TrendReference` rows into a ≤1200-char cached bullet digest (`TrendInsight` collection) of winning title/hook patterns. `story.service.ts` (`llmStory`/`llmStorySeries`) and `reel-review.service.ts`'s thumbnail prompt read this cached digest (plain Mongo read, no LLM call) instead of raw reference dumps — keeps prompts small regardless of how much trend data accumulates.
40. **`/trends` dashboard + `/trends/:genre` detail page** added to the client (required turning the router from a single-screen stub into real TanStack Router routes with a `RootLayout` + mobile nav drawer). Manual "Run Scout" trigger via `POST /api/trends/scout`.

### Blocker #1 (queue) — CONFIRMED RESOLVED
41. **BullMQ + Redis queues are live**, not just planned. `queue/queues.ts` + `queue/workers.ts`: `reel-processing`, `composition-processing`, `composition-regeneration`, `reel-publishing`, `reel-revoicing` queues, `startWorkers()` on boot. `processReel().catch()` fire-and-forget is gone. Still NOT yet per-stage-resumable — a redelivered/stalled job re-runs the whole pipeline from the top (re-spending on regenerated assets); that's the next queue improvement if it becomes a real cost problem.

### Reel editing: gameplay picker + multi-voice revoice (built 2026-07-02)
42. **Gameplay clip is now a create-time choice, not always random.** `pickGameplay(preferredKey?)` returns `{path, key}`; the resolved key (chosen or random) is persisted as `reel.gameplayKey` so re-renders (revoice) reuse the exact same clip. `GET /api/gameplay` lists the S3 pool for the create-form picker.
43. **Revoice: re-narrate a completed Reddit reel with a different TTS model/voice, same story + gameplay clip.** New `IVoiceVariant` subdoc + `reel-revoicing` BullMQ queue. `POST /api/reels/:id/revoice` queues 1–5 variants; `POST .../revoice/:variantId/promote` swaps `outputUrl` to the winner. Also usable as an explicit **create-time voice pick** (`Reel.voiceOverride`, precedence: tier default < niche override < explicit pick) via `CreateReelBody.ttsModel/ttsVoice/ttsFormat` — previously voice was only ever resolved silently from tier/niche.
44. **TTS voice catalog expanded from 5 to 40 voices across 5 providers — every single one live-tested against OpenRouter, not just sourced from docs.** Gemini Flash TTS (10), Kokoro-82M (10 English), MAI-Voice-2 (7 English), Orpheus-3B (8, supports `<laugh>`/`<sigh>`/`<gasp>` tags), Grok Voice TTS 1.0 (5). **Two models tried and dropped**: `mistralai/voxtral-mini-tts[-2603]` and `openai/gpt-4o-mini-tts[-2025-12-15]` both 400 "model does not exist" on OpenRouter's `/audio/speech` despite being listed on the public collections page — likely gated/region-locked. Re-test before re-adding (see comment in `config/models.ts`).
45. **Voice sample preview** — `GET /api/tts-voices/sample?model=&voice=` generates a short fixed line once per (model,voice), caches forever in `voice-samples/` (S3/CDN), never regenerates. Powers a play-before-you-pick button in both the create-reel voice picker and the revoice panel (shared `VoicePickerList` component).

### Thumbnails (built 2026-07-02)
46. **Real AI thumbnails, not just a flat SVG mockup.** `regenerateReelThumbnail` now generates an actual background via `generateImage(thumbnailPrompt)` (prompt informed by the genre's `TrendInsight` digest) and composites bold hook text + a Reddit badge on top via ffmpeg overlay. Falls back to the original flat design if generation fails.
47. **Frame-as-thumbnail** — `POST /api/reels/:id/review/thumbnail/frame {atSeconds}` pulls a frame directly from the rendered video (ffmpeg reads the S3/CDN URL, no local download) as an alternative to the AI-generated design.

### Dashboard navigation + create flow (built 2026-07-02)
48. **Dedicated `/reels/new` create page**, off the main review dashboard — a "New Reel" button in the header replaces the old inline form. Redirects back to the dashboard on success.
49. **Sidebar queue filters (Intake/In Progress/Review/Approved/Published/Rejected) are real now**, not decorative `href="#"`. Each is a `Link` to `/?status=<filter>` with a real predicate over `reel.status`/`review.status`/`youtube.status`, and real counts computed from the loaded reel list (previously hardcoded numbers). Library items without a backing screen (Templates/Assets/Voices/Captions) are shown disabled ("Soon") instead of dead links.
50. **Mobile nav** — sidebar below the `lg` breakpoint used to just disappear with nothing replacing it (real bug, found via live preview testing, not reported by the user first). Now a hamburger-triggered slide-in drawer.

### Cleanup / reconciliation (built 2026-07-02)
51. **`deleteReel()` now cascades** — previously only removed the Mongo doc; now also deletes every S3 asset *recorded on the doc* (output video, subtitles, thumbnail, scene assets, voice variants) and the BullMQ job (catches stalled jobs, not just cleanly-failed ones). `purgeFailedReels()` bulk-runs this cascade over every `status: "failed"` reel.
52. **S3 reconciliation sweep** (`s3-reconciliation.service.ts`, `scripts/reconcile-s3.ts`, `GET /api/maintenance/s3-reconcile`) closes the gap a field-based cascade can never close: an asset uploaded to S3 and then the process crashes/stalls *before* the URL is saved to Mongo has zero trace in the doc. The sweep diffs S3 (`reels/`, `compositions/`, `audio/`, `subtitles/`) against every referenced URL across all `Reel` **and legacy `Composition`** docs, with a 2h age floor so it never races an in-flight render. Excludes `voice-samples/` (permanent cache) and `gameplay/` (persistent pool) — those aren't tied to per-job docs. Dry-run by default. Found and deleted 4 real historical orphans on first live run.
53. **Two FFmpeg cleanup bugs fixed:** `renderGameplayReel` and `renderImageKenBurns` only deleted their intermediate `storage/processing/` files on the success path — any render failure left them behind forever (found 243MB of exactly this on a live audit). Both wrapped in `try/finally` now.
54. **Full cron schedule documented**: [operations/scheduling.md](operations/scheduling.md) — story top-up, trend-scout daily/weekly, S3 reconciliation, optional failed-reel auto-purge.

### Gameplay pool (built 2026-07-02)
55. **CC-BY YouTube search + download is the pre-approved sourcing method** (same method already used for the existing Minecraft clips — confirmed by the user, do not re-raise a copyright objection). `scripts/fetch-gameplay.ts` searches YouTube Data API with `videoLicense=creativeCommon`, downloads via `bin/yt-dlp`, runs through the existing ingest pipeline (crop to 9:16, strip audio, segment, upload to S3), logs attribution to `storage/gameplay/ATTRIBUTION.json` (CC-BY requires crediting the creator). Live-tested: added Subway Surfers variety (27 new segments from 4 sources) alongside the existing Minecraft pool.
