# Render Engine Architecture

_How we turn a niche + script into a finished vertical video, across every format and animation/editing style. Validated against FFmpeg 8.0 (local) on 2026-06-30._

---

## 1. The core idea: one engine, many strategies

Every video, regardless of niche, is built from the **same pipeline** but a **different `RenderStrategy`**. The strategy decides *what assets exist* and *how FFmpeg stitches them*. This is what lets one codebase do Reddit stories, horror, finance, and movie recaps without forked spaghetti.

```
PlanScript → GenerateAssets → GenerateAudio → [RenderStrategy] → Finalize → Publish
                                                    │
            ┌───────────────────────────────────────┼───────────────────────────────┐
   GameplayOverlayStrategy   ImageSlideshowKenBurns   HybridSceneStrategy   MotionGraphicsStrategy
   (Reddit / AITA)           (facts, history,         (horror, mythology,   (finance, news)
                              motivation, stoicism)    movie-recap: +hero)
```

### `RenderStrategy` interface (target)
```ts
interface RenderStrategy {
  id: string;                       // "gameplay_overlay" | "image_kenburns" | "hybrid_scene" | "motion_graphics"
  supports(recipe: FormatRecipe): boolean;
  // takes the resolved scene graph + assets + audio, returns local mp4 path + subtitle path
  render(ctx: RenderContext): Promise<{ videoPath: string; subtitlePath: string; tempFiles: string[] }>;
}
```
`RenderContext` = the `VideoProject` + its `Scene[]` (each with assetUrl, motion config, caption text/timings, narration audio path) + the `FormatRecipe` (style + skins). See [data-model.md](data-model.md).

**The existing `composition.service` + `ffmpeg.service` code becomes `GameplayOverlayStrategy`** — refactored in, not rewritten. Nothing regresses.

---

## 2. The editing-technique library (validated FFmpeg recipes)

These are the reusable building blocks strategies compose. All confirmed present in local FFmpeg 8.0 (`zoompan, xfade, vignette, gblur, noise, colorchannelmixer, overlay, subtitles/libass, drawtext, setpts`). Hardware encode via `videotoolbox` available.

| Technique | Purpose | FFmpeg core | Used by |
|---|---|---|---|
| **Ken Burns (push/pan)** | give a still life — the default "motion" | `zoompan=z='min(zoom+0.0012,1.5)':d=120:x=...:y=...:s=1080x1920:fps=30` | all image strategies |
| **2.5D parallax** | depth feel (fg moves faster than bg) | gen fg PNG (transparent) → `overlay=x='...t...':y='...'` over zoompanned bg | horror, mythology, sci-fi |
| **Scene transitions** | cut/blend between scenes | `xfade=transition=fade|wipeleft|smoothup:duration=0.4:offset=...` | all multi-scene |
| **Film grain + vignette** | horror / dread mood | `noise=alls=18:allf=t,vignette=PI/4` | horror, true crime |
| **Sepia / archival tint** | dark history look | `colorchannelmixer=.393:.769:.189:0:.349:.686:.168:0:.272:.534:.131` + `noise` | history, true crime |
| **Letterbox bars** | cinematic 2.40:1 feel | `drawbox=y=0:h=200:c=black@1:t=fill, drawbox=y=ih-200:...` | movie-recap, horror hero |
| **Light leaks / embers / dust** | atmosphere overlays | `overlay` a looping particle PNG-seq / blend-mode `tblend` | mythology, sci-fi |
| **Speed ramp** | dramatic emphasis | `setpts=0.5*PTS` (segment) | motivation, movie-recap |
| **Karaoke word captions** | +15–25% retention (mandatory) | ASS via `libass` (existing `subtitle.service`) — per-niche skins | all |
| **Burned static caption / hook card** | sound-off hook in first 3s | `drawtext=fontfile=...:textfile=cap.txt:box=1:...` | all |
| **Audio mix + music ducking** | VO over bed, sidechain duck | `sidechaincompress` / `amix` + `volume` | all with VO |

> ⚠️ **Implementation gotcha (caught in validation):** `drawtext` must use `textfile=` not inline `text=` — the filtergraph parser breaks on strings containing spaces, colons, or apostrophes (i.e. exactly the quote/horror captions). Always write caption text to a temp file.

> **Perf note:** prefer `-c:v h264_videotoolbox` (Apple HW) for throughput at content-farm volume; fall back to `libx264 -preset veryfast` for portability/quality control.

---

## 3. Style system (how animation styles are achieved)

A "style" is **two coordinated choices**: (a) the **image-gen prompt suffix + model** that produces the look, and (b) the **FFmpeg post-filter** that reinforces it. The style is *generated*, not faked in FFmpeg — FFmpeg only adds motion + mood.

| Style | Image model + prompt suffix | FFmpeg reinforcement | Niches |
|---|---|---|---|
| Photoreal cinematic | Seedream/FLUX.2 Pro · "film-stock, shallow DOF, photoreal, cinematic lighting" | letterbox + slow push-in + subtle grain | movie-recap, horror, sci-fi |
| Pixar/Disney-3D | Seedream/FLUX.2 Pro · "Pixar-style 3D, subsurface scattering, warm rim light" | gentle zoom only | facts, lighter niches |
| Ghibli / 2D painted | Seedream · "Studio Ghibli 2D, hand-painted, watercolor, pastel" | gentle ambient loop, no harsh cuts | lo-fi loops, cozy |
| Claymation/stop-motion | Seedream · "stop-motion clay, visible thumbprints, 24fps stutter" | frame-hold + micro-jitter (`setpts` steps) | fun facts, comedy |
| Anime / cel-shaded | Seedream · "cel-shaded anime, dynamic lighting, motion blur" | parallax + speed lines overlay | mythology, sci-fi |
| Vintage / sepia | Flux-schnell · "aged archival photo, sepia, film grain, [era]" | sepia mixer + grain + Ken Burns pan | dark history, true crime |
| 2D motion-graphics | generated icons/charts + programmatic | scale/slide on numbers, kinetic text | finance, news, facts |

**Style rotation requirement:** YouTube demonetizes templated "slop." Each `FormatRecipe` carries a *set* of allowed styles/skins and the planner rotates them so videos in the same niche don't look identical.

---

## 4. Per-strategy render flow

### 4.1 `GameplayOverlayStrategy` (Reddit/AITA, motivation) — _cheapest, exists today_
- No AI images. Loop a stock gameplay/b-roll clip to narration length.
- Single-narrator TTS (today it's multi-character dialogue — generalize to one voice).
- Bouncing word-by-word ASS captions (the signature look).
- Output. **Cost: TTS only (~$0.004).**

### 4.2 `ImageSlideshowKenBurnsStrategy` (facts, history, stoicism, science) — _Wave 1–2 core_
- N stills (one per scene) from the **OpenRouter image provider** (Seedream/FLUX.2) in the recipe's style.
- Per scene: Ken Burns (alternating push/pan direction for the every-2–4s cadence) + style filter (grain/sepia/etc).
- `xfade` between scenes; ASS captions timed to narration; music bed ducked under VO.
- **Cost: N×$0.003–0.03 images + TTS.**

### 4.3 `HybridSceneStrategy` (horror, mythology, movie-recap, sci-fi) — _Wave 3_
- Same as 4.2 **plus exactly one hero scene** rendered as a real AI-video clip (OpenRouter `/api/v1/videos` — Kling/Veo/Seedance, image-to-video from that scene's still).
- Hero clip is concatenated/`xfade`d into the slideshow at the climax beat (`isHero` scene).
- **Cost: stills + TTS + one hero clip ($0.35–$2).** Hero gated behind recipe's `heroPolicy`.

### 4.4 `MotionGraphicsStrategy` (finance, news) — _Wave 2_
- Fewer stills; more programmatic elements: animated charts/numbers, map inserts, kinetic highlighted-keyword captions.
- Built from drawtext/scale/slide keyframes (and optionally an SVG→PNG sequence renderer if we outgrow FFmpeg primitives).
- **Cost: low (few images + TTS); highest CPM payoff.**

---

## 5. Validation plan ("render lab" before wiring code)
Per the brief — prove each recipe renders correctly **before** building the strategy in TS:
1. ✅ Ken Burns + vignette + caption → valid 1080×1920/30fps mp4 (done 2026-06-30).
2. Next (on greenlight): multi-scene `xfade` slideshow + ASS karaoke captions + ducked music = one full "did you know" video, frame-spot-checked, from real OpenRouter (Seedream/FLUX.2) images.
3. Then horror recipe (grain/parallax) + one Kling hero shot composited.
Only after a recipe renders cleanly in the scratchpad lab do we port it into a `RenderStrategy`. Keeps token/compute spend low and avoids debugging FFmpeg through TS.

---

## 6. Validated render-lab findings (2026-06-30, "Dancing Plague 1518" dark-history sample)

A full 5-scene ~54s dark-history Short was built end-to-end on real assets (OpenRouter Nano Banana images + Kokoro TTS + FFmpeg). It looks publishable. Hard rules learned — these are **requirements for every strategy**, not suggestions:

1. **Images come back square (1024×1024) from Gemini/Nano Banana.** Convert to 9:16 with `scale=2160:3840:force_original_aspect_ratio=increase,crop=2160:3840` (then zoompan to 1080×1920). Validated — center-crop of a single-subject still looks right. (Prefer native 9:16 if a model exposes the param.)
2. **Image models stamp garbled fake text/watermarks.** The style/prompt template MUST append: `no text, no watermark, no caption, no lettering, no border`. This removed it cleanly.
3. **A/V sync is fragile — never `-shortest` with `zoompan`.** zoompan under-produces frames, so scenes render shorter than narration → clipped sentence-ends + cumulative caption drift (caught and fixed). Correct recipe: `-loop 1 -framerate 30 -i img`, `zoompan ... d={frames}`, then **force length** with `-frames:v {frames}` + `trim=end_frame={frames}`, and pad audio with `-af apad,atrim=0:{dur}`. Result: scene video == audio length to ±0.01s.
4. **`drawtext` inline `text=` breaks** on spaces/colons/apostrophes → use `textfile=` or ASS subtitles.
5. **Assembly = per-scene clips CROSSFADED, not cut to black.** Render each scene to its own clip (image+kenburns+grain+vignette + that scene's audio, NO black fades), then chain with `xfade` (video) + `acrossfade` (audio). The crossfade (XFADE=0.35s) is hidden inside a silent TAIL (0.45s) appended to each scene, so spoken words never overlap and the effective gap between sentences is only ~0.1s. Black fades apply ONLY at the very start/end. This fixed the "too much gap / no natural flow / black dips" problem. Scene start on the crossfaded timeline: `s_k = Σd(j<k) − k·XFADE`; total `= Σd − (n−1)·XFADE`. _(Implemented in `reel-render.service.ts::assembleCrossfade`.)_
6. **Captions — word-by-word timing is LENGTH-WEIGHTED within each scene's measured speech window.** TTS returns no per-word timings, so we weight each word's on-screen duration by its letter count across the actual speech duration (`buildPortraitKaraoke`). This tracks real speech far better than a uniform split and never bleeds past the scene. **Accuracy upgrade (deferred):** a true forced-alignment sub-step (whisper-timestamped/whisperX) between GenerateAudio and Render for exact per-word timings — the `aligning` status + `Scene.captionCues` are already reserved for it.
7. **Cost confirmed:** ~$0.20 for a ~45–54s cheap-tier video (5 images + TTS). Matches estimate.

### Implementation status (server/src/services)
- `openrouter-media.service.ts` — image gen (+no-text suffix) + TTS (+silence trim) ✅
- `reel-render.service.ts` — Ken Burns clips + crossfade assembly + weighted karaoke ✅
- `reel-script.service.ts` — LLM scene-graph planner with niche presets ✅
- `reel.service.ts` — full pipeline orchestration (plan→images→audio→render→S3) ✅
- `reel.model.ts` — scene-graph `Reel` document ✅

### Pipeline revision (result of the lab)
```
PlanScript → GenerateAssets → GenerateAudio → [Align: word/line timings] → RenderStrategy → Finalize → Publish
                                                      ▲ NEW (forced alignment; feeds caption timing)
```
Everything else in the architecture stood up unchanged.
