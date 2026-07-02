# Locked Decisions (plain English)

_Running record of what we've settled. Update as decisions are made. Last updated 2026-07-01._

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

### Deferred
23. **Trend-research agent + `TrendReference` DB:** foundation started 2026-07-01. `TrendReference` now exists as a Mongo collection with `{ niche, genre, sourceUrl, platform, metrics, notes, status }` plus create/list service helpers. External scraping/ingestion remains deferred to Phase 5; `FormatRecipe` is designed so it can consume trend data later without a reschema.
