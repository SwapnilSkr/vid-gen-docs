# Data Model — Scene Graph

_The generalized model that replaces the brainrot-only `Composition`. This is the hardest thing to change later, so it's specced before code._

---

## 1. Why change

Today `Composition` = template(bg video) + characters(image+voiceId) + dialogue lines. That can only express "characters talking over gameplay." A finance explainer or horror story has **scenes with narration + generated visuals + per-scene motion/captions** — a different shape. So we generalize to a **scene graph**: a `VideoProject` owns an ordered list of `Scene`s.

The old `Composition` shape is kept as `GameplayOverlayStrategy`'s internal form (back-compat); new niches use the scene graph.

---

## 2. Collections

```
VideoProject ──< Scene[]            (the job + its ordered scenes)
FormatRecipe                        (declarative per-niche config = "Template Library" + reference DB)
Asset                               (generated image/video/audio, S3-backed, reusable/cacheable)
CostLedger                          (per-project spend: estimate -> actual; light for farm, full for SaaS)
TrendReference                      (viral/trend examples feeding recipe tuning; scraping deferred)
```

---

## 3. Core interfaces (target TS / Mongoose)

```ts
// ---- VideoProject: the unit of work (replaces Composition as the top-level job) ----
interface VideoProject {
  _id: ObjectId;
  niche: string;                    // "horror" | "finance" | "reddit" | ...
  recipe: ObjectId;                 // -> FormatRecipe
  strategy: RenderStrategyId;       // resolved from recipe: "image_kenburns" | "hybrid_scene" | ...
  tier: "cheap" | "value" | "premium";
  style: string;                    // chosen from recipe.allowedStyles (rotated to avoid sameness)

  topic: string;                    // the seed idea / prompt ("the Dyatlov Pass incident")
  title?: string;                   // LLM-generated
  hook?: string;                    // first-3s sound-off hook text
  script?: string;                  // full narration (planner output)

  scenes: Scene[];                  // the ordered scene graph
  aspect: "9:16";                   // vertical default
  durationSec?: number;

  status: ProjectStatus;            // pending | planning | generating_assets | generating_audio
                                    // | aligning | rendering | uploading | completed | failed
                                    // ('aligning' = forced word/line alignment; added per render-lab finding)
  progress: number;
  stageState?: Record<string, unknown>; // for resumable queue stages

  outputUrl?: string;
  subtitlesUrl?: string;
  costEstimateUsd?: number;         // shown BEFORE run
  costActualUsd?: number;
  error?: string;

  // multi-tenant seam (content-farm now; SaaS later)
  ownerId?: ObjectId;               // null/system for our own farm channels
  channelId?: ObjectId;             // which farm channel/brand this belongs to

  createdAt: Date; updatedAt: Date;
}

// ---- Scene: one beat = one visual + its narration + motion + captions ----
interface Scene {
  index: number;
  narration: string;                // VO text for this beat
  visualPrompt: string;             // image/video-gen prompt (style suffix applied at gen time)

  assetType: "image" | "video" | "gameplay" | "stock";
  asset?: ObjectId;                 // -> Asset (resolved after GenerateAssets)
  isHero: boolean;                  // true => render as real AI-video clip (HybridScene)

  motion: {                         // how FFmpeg animates a still
    type: "ken_burns" | "parallax" | "static" | "loop_ambient" | "mograph";
    direction?: "in" | "out" | "left" | "right" | "up" | "down";
    intensity?: number;             // 0..1
  };

  startTime: number;                // resolved after audio durations known
  duration: number;

  captionStyle?: string;            // overrides recipe default skin if needed
  audioUrl?: string;                // S3 narration audio for this scene
  captionCues?: { t: number; end: number; text: string }[]; // from Align stage (word/line timings → ASS)
}

// ---- FormatRecipe: the declarative niche spec (what makes a niche a niche) ----
interface FormatRecipe {
  _id: ObjectId;
  niche: string;                    // "horror"
  displayName: string;              // "AI Horror Story"
  strategy: RenderStrategyId;

  allowedStyles: string[];          // for rotation (anti-slop)
  imageTier: "cheap" | "value" | "premium";
  sceneCountRange: [number, number];// e.g. [8,14]
  heroPolicy: "never" | "one_climax" | "trend_gated";

  captionSkin: string;              // "serif_lowerthird" | "word_pop" | "epic_titlecard" | "bouncing"
  motionDefaults: Scene["motion"];
  musicMood: string;                // "ambient_drone" | "orchestral" | "upbeat_minimal" | ...
  voicePreset: { provider: "inworld" | "elevenlabs"; voiceId: string; style?: string };

  scriptSystemPrompt: string;       // niche-specific planner instructions (hook/structure rules)
  active: boolean;
}

// ---- Asset: every generated artifact, cached + reusable ----
interface Asset {
  _id: ObjectId;
  kind: "image" | "video" | "audio" | "music" | "stock";
  provider: string;                 // "fal" | "inworld" | "openrouter" | ...
  model?: string;                   // "seedream-4" | "kling-2.5" | ...
  promptHash?: string;              // dedupe / cache key
  s3Url: string;
  costUsd: number;
  meta?: Record<string, unknown>;   // dims, duration, seed, etc.
  createdAt: Date;
}

// ---- CostLedger: unit economics (farm) + credit/refund (SaaS later) ----
interface CostLedger {
  _id: ObjectId;
  project: ObjectId;
  estimateUsd: number;              // computed pre-run
  lines: { stage: string; provider: string; model?: string; usd: number }[];
  actualUsd: number;
  refundedOnFailure: boolean;       // SaaS-facing; for farm = pure accounting
  createdAt: Date;
}

// ---- TrendReference: reference examples for future trend-aware recipes ----
interface TrendReference {
  _id: ObjectId;
  niche: string;                    // "reddit" | "finance" | "horror" | ...
  genre?: string;                   // "wedding_drama" | "petty_revenge" | ...
  sourceUrl: string;                // original example URL, unique
  platform: "youtube_shorts" | "instagram_reels" | "tiktok" | "reddit" | "unknown";
  metrics: {
    views?: number;
    likes?: number;
    comments?: number;
    shares?: number;
    saves?: number;
    durationSec?: number;
    postedAt?: Date;
    capturedAt?: Date;
  };
  notes?: string;                   // human/agent observations: hook, pacing, caption style, etc.
  status: "candidate" | "reviewed" | "approved" | "rejected" | "archived";
  createdAt: Date;
  updatedAt: Date;
}
```

---

## 4. How the planner fills the graph
`PlanScript` (LLM via OpenRouter) takes `{niche, recipe, topic}` and returns: `title`, `hook`, `script`, and a `Scene[]` where each scene has `narration` + `visualPrompt` + `motion` (+ marks one `isHero` if `heroPolicy` allows). This mirrors FacelessReels' Idea → Script → Scene Planner flow, but as one structured LLM call validated against the recipe's `sceneCountRange`.

## 5. Migration note
`Composition` is **not deleted**. `GameplayOverlayStrategy` keeps reading it. New niches write `VideoProject`. A thin adapter can later backfill old compositions into the new shape if needed. This avoids a risky big-bang migration.
