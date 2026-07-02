# Master Build Plan — Multi-Niche AI Short-Form Video Engine

_Synthesized June 2026 from: codebase audit + 3 research streams (market teardown, trend/format map, generation stack & costs)._

> **⚠️ Provider routing in §1–2 below is superseded.** This doc's original
> assumption (fal.ai primary, Inworld/ElevenLabs TTS) was corrected shortly
> after: **OpenRouter is the single provider** for LLM + images + TTS + video
> (`config/models.ts`, `resolveModels(tier)`). See
> [DECISIONS.md](DECISIONS.md) #9–14 for the current, accurate stack — the
> pricing tables below are the *pre-correction* numbers, kept for the
> original cost-modeling logic, not as a current price sheet.
>
> **Phase status as of 2026-07-02** (see §5 below for the original plan):
> Phase 0 (queue + scene graph + `GameplayOverlayStrategy`) — **done**.
> Phase 1 (Reddit shipped; facts/motivation niches — not built, `image_kenburns`
> strategy exists and could carry them but no recipe/planner work done yet).
> Phase 2 (`hybrid_scene`/`motion_graphics` strategies, horror/mythology/finance/
> movie-recap) — **not built**; `niche-styles.ts` already has recipes for
> horror/mythology/dark_history/finance/etc. declared, but their render
> strategies don't exist yet (only `image_kenburns` and `gameplay_overlay` are
> implemented). Phase 3 (cost ledger) — not built. Phase 4 (distribution) —
> YouTube Shorts publish **is built** (`youtube-publish.service.ts`), Instagram
> still deferred. Phase 5 (trend agent) — **built**: `trend-scout.service.ts`
> + `trend-insight.service.ts`, see DECISIONS.md #38–40.

---

## 0. The Bet (positioning)

The market has one **unserved quadrant**: nobody pairs **genuine motion AI-video quality** with **reliable auto-posting** at an **affordable tier**.

- Cheap tools (Crayo, AutoShorts, Reelfarm) = static-image / stock "slideshow slop" but they auto-post.
- Quality tools (FacelessReels, Vadoo, Neural Frames) = real AI video but **no scheduler**.
- The #1 complaint market-wide (Trustpilot, Reddit): **charged credits for failed/unusable generations, no refund, no pre-cost visibility.**

**Our edge = hybrid render engine (cheap stills + FFmpeg motion, one AI-video hero shot only) + per-niche polish + cost transparency + refund-on-failure + auto-posting as a wedge.**

Also critical: YouTube now **demonetizes templated AI "slop"** under its inauthentic-content policy. The engine must **vary the look** (per-niche caption skins, style rotation), never stamp out identical videos. Velocity (4–7 posts/day) + editing quality beat raw generation fidelity.

---

## 1. Economics (per ~45s / ~9-scene video)

| Tier | Images | Hero video | TTS | **Total** |
|---|---|---|---|---|
| **Cheap (default)** | 9× Flux-schnell $0.003 → Seedream $0.03 | none | Inworld Mini ($5/1M) ~$0.004 | **~$0.05–0.27** |
| **Value-premium** | FLUX.2 Pro stills + Veo 3.1 **Fast** hero $0.75 + Cartesia TTS | 1 clip | | **~$1.05** |
| **Premium (trend-gated)** | Nano Banana Pro stills + Veo 3.1 Standard hero $2 + ElevenLabs v3 | 1 clip | ~$0.14 | **~$2.94** |

The single hero AI-video clip dominates premium cost → **gate it behind a "trend demands it" flag.** Cheap tier is the workhorse.

---

## 2. Model Stack (concrete picks + routing)

**Routing correction to original assumption:** OpenRouter now *does* host some image models (Nano Banana, FLUX.2, Seedream) but bills per-token (unpredictable) and hosts **no video, no TTS**. Decision: **fal.ai is primary gen provider** (flat per-image pricing + full video catalog, one key); OpenRouter stays for LLM only.

| Layer | Cheap tier | Premium tier | Route |
|---|---|---|---|
| **LLM** (script + scene planning) | OpenRouter (current) | same | OpenRouter |
| **Image** | Flux-schnell $0.003 (simple) / **Seedream 4.x $0.03** (stylized: anime/horror/Pixar) | Nano Banana Pro / FLUX.2 Pro (~$0.10) | fal (Replicate fallback for schnell) |
| **Video (hero only)** | **Kling 2.5 Turbo Pro $0.35/5s** | Veo 3.1 Fast $0.75 → Standard $2 | fal |
| **TTS** | **Inworld TTS-1.5 Mini $5/1M** / Realtime TTS-2 | ElevenLabs Multilingual v3 $206/1M | direct |

⚠️ **Do not build core pipeline on Sora 2** — OpenAI API sunsets ~Sept 24, 2026.

---

## 3. Niche / Format Roadmap (build order by cost-to-produce & fit)

Ranked by engine fit (cheapest first) × virality/monetization. Each is a **FormatRecipe** (declarative config: style, image model tier, caption skin, motion params, hero policy, music bed).

| Wave | Format | Render recipe | Why first |
|---|---|---|---|
| **1** | **Reddit / AITA** | gameplay loop + TTS + bouncing word captions, **0 AI images** | ~90% of current code already does this |
| **1** | **"Did you know" / facts** | 3–6 stills, punch-in scale, big word-pop captions | cheapest, highest IG **save** rate |
| **1** | **Motivation / mindset** | 0–3 stills + stock b-roll, slow zoom, epic music, bold center captions | cheapest, shared reflexively → free reach |
| **2** | **Dark history** | 10–14 sepia stills, Ken Burns pan + grain, documentary VO | low fatigue, infinite supply |
| **2** | **Personal finance** | 2D motion-graphics, animated charts, highlighted-keyword captions | **highest CPM $8–20** (the money) |
| **2** | **Stoicism / philosophy** | marble/classical stills, very slow Ken Burns, serif captions | older buyers, loyal niche |
| **3** | **AI horror story** | 8–14 dark cinematic stills, push-in + film-grain, whispered TTS, **1 hero shot** | top watch-through, CPM $4–9 |
| **3** | **Mythology / dark fantasy** | painterly stills, parallax + embers, orchestral, **1 hero reveal** | billions of views proven |
| **3** | **Movie recap / what-if** | photoreal cinematic stills, push/pan + letterbox, **1 hero shot** | ~3× engagement |
| **4** | News/politics explainer, true crime, sci-fi lore, science/space, lo-fi loops | per-recipe | volume / niche expansion |

**Top 6 visual styles to support:** photoreal cinematic, Pixar/Disney-3D, Ghibli/hand-painted 2D, claymation/stop-motion, anime/cel-shaded, vintage + 2D motion-graphics.

**Winning conventions (engine requirements, not options):**
- Burned-in **word-by-word captions** (+15–25% retention), 4–7 words/line, per-niche skins.
- **Visual change every 2–4s** → with a stills engine, hit cadence via image swaps + Ken Burns direction changes.
- **Hook in first 3s** that lands sound-off (on-screen text + striking first frame). Structure = Hook → Body → Payoff, one idea per Short.

---

## 4. Architecture Overhaul

### 4.1 Current state (audit)
- **No real queue** — `job.service.ts` is an in-memory `Map` (lost on restart, single-process); the real pipeline is fire-and-forget `processComposition(id).catch()` writing status to the `Composition` doc.
- **Single hardcoded render path** — dialogue-script → ElevenLabs-per-line → FFmpeg character-overlay-on-gameplay. No concept of scenes/shots/visual prompts.
- **Data model married to brainrot format** — `Composition` → template(bg video) + characters(image+voiceId) + dialogue lines.

### 4.2 Target architecture

```
VideoProject (niche, formatRecipe, style, tier)
  └─ Scene[] { index, narration, visualPrompt, style, assetType(image|video|gameplay),
               assetUrl, motion(kenburns cfg), duration, captionStyle, isHero }

Pipeline (BullMQ + Redis, each stage idempotent & resumable):
  1. PlanScript      LLM (OpenRouter) → title + Scene[] with narration + visual prompts
  2. GenerateAssets  per scene: ImageProvider | VideoProvider | gameplay pick  (parallel)
  3. GenerateAudio   TTSProvider per scene narration                            (parallel)
  4. Render          RenderStrategy → FFmpeg (Ken Burns/parallax/captions/mix)
  5. Finalize        upload S3, cost reconcile
  6. Publish         (Wave 4) YouTube/IG auto-post
```

**Key abstractions to introduce:**
- **`RenderStrategy` interface** — pluggable per format. First strategy = current code refactored in (`GameplayOverlayStrategy` for Reddit). Then `ImageSlideshowKenBurnsStrategy`, `HybridSceneStrategy` (hero shots), `MotionGraphicsStrategy` (finance/news).
- **Provider layer** (mirrors existing TTS seam): `ImageProvider` (fal primary), `VideoProvider` (fal), `TTSProvider` registry (Inworld default, ElevenLabs premium), `LLMProvider` (OpenRouter).
- **`FormatRecipe` config** (Mongo collection) — the declarative per-niche spec. This is our "Template Library" + the **reference DB** for trend examples.
- **Cost ledger** — pre-estimate before run, track actual, **credits not consumed on failure** (directly attacks the #1 market complaint).

### 4.3 Data model changes
- Generalize `Composition` → `VideoProject` with a `scenes[]` subgraph (keep `Composition` shape as one strategy's internal form for back-compat).
- New collections: `FormatRecipe`, `TrendReference` (downloaded viral examples), `CostLedger`.

---

## 5. Phased Roadmap

- **Phase 0 — Foundation refactor** ✅ **done**: BullMQ+Redis queue; provider abstraction (OpenRouter, not fal — see routing note above); scene-graph data model shipped as `Reel`, not `VideoProject`; current pipeline is `gameplay_overlay` strategy.
- **Phase 1 — Cheap engine + Wave-1 niches** 🟡 **partial**: scene-planner LLM + `image_kenburns` (renamed from `ImageSlideshowKenBurnsStrategy`) both shipped. Reddit is fully built and is the only Wave-1 niche actually shipped — facts/motivation have no recipe/planner work done, though `niche-styles.ts` + `image_kenburns` could carry them with recipe work only, no new pipeline code.
- **Phase 2 — Styles + Wave-2/3 niches + hero shots** 🔴 **not built**: `niche-styles.ts` already declares recipes for dark_history/finance/stoicism/horror/mythology/movie_recap (styles, caption skins, voice overrides all specified), but their render strategies (`hybrid_scene` for hero shots, `motion_graphics` for finance) don't exist — only `image_kenburns` and `gameplay_overlay` are implemented. This is the next real gap for any non-Reddit niche.
- **Phase 3 — Cost transparency** 🔴 **not built**: no `CostLedger` collection, no pre-estimate, no refund-on-failure UX.
- **Phase 4 — Distribution** 🟡 **partial**: YouTube Shorts auto-post is **built and live** (`youtube-publish.service.ts`, wired into the review UI's Publish button). Instagram Reels still deferred (needs Business account + FB app review).
- **Phase 5 — Trend agent + reference DB** ✅ **done**: `trend-scout.service.ts` (YouTube Data API scraper) + `trend-insight.service.ts` (digest for prompts) + `/trends` dashboard. See [DECISIONS.md](DECISIONS.md) #38–40.

---

## 6. Immediate next step

~~Start Phase 0.~~ **Phase 0 is done** (see §5 status above). As of 2026-07-02
the highest-leverage next step is **Phase 2**: pick one non-Reddit niche
already declared in `niche-styles.ts` (dark_history is the cheapest —
`image_kenburns` already supports it, no new strategy needed, just recipe +
planner validation) and ship it end-to-end, before building the harder
`hybrid_scene`/`motion_graphics` strategies horror/finance need.
