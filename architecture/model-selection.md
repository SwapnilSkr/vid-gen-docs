# OpenRouter Model Selection (budget-first)

_Live pricing pulled from the OpenRouter API 2026-07-01. Swap models in one place: `server/src/config/models.ts` (or env). See "How to swap" at the bottom._

## The key budget insight

For a ~45s video, **LLM and TTS are effectively FREE** — fractions of a cent — because a script is ~500–1000 tokens and narration is ~700 characters. Your only real costs are:

1. **Images** — ~$0.02–0.13 each × ~5–9 per video
2. **Hero video clip** (Wave 3+ only) — the single genuinely expensive item

So: **don't cheap out on LLM/TTS — use good ones, it costs nothing.** Optimize images, and gate the hero clip. Kokoro was only the default because it was first, not because it saved meaningful money.

---

## 1. LLM (script + scene graph) — cost is negligible, pick for quality/JSON

| Model | $/Mtok in→out | Notes |
|---|---|---|
| `mistralai/mistral-small-24b` | 0.05 → 0.08 | cheapest usable |
| `qwen/qwen3-235b-a22b-2507` | 0.09 → 0.10 | big model, absurdly cheap |
| **`deepseek/deepseek-v4-flash`** ⭐ | 0.098 → 0.196 | **cheap workhorse, strong JSON** (registry `cheap`/`value` LLM could use this) |
| `google/gemini-2.5-flash-lite` | 0.10 → 0.40 | reliable, fast |
| `google/gemini-2.5-flash` | higher | **quality pick** (registry `value`/`premium`) |

Per video ≈ **$0.0005–0.002**. Negligible. Registry default: `deepseek-v4-flash` (cheap), `gemini-2.5-flash` (value/premium).

## 2. Images — your main cost lever

| Model | ~$/image | Notes |
|---|---|---|
| `google/gemini-3.1-flash-lite-image` | **~$0.02** | cheapest current-gen — best for broke bulk |
| **`google/gemini-2.5-flash-image`** (Nano Banana) ⭐ | ~$0.04 | proven in render lab, great look — `cheap` default |
| `google/gemini-3.1-flash-image` | ~$0.04–0.05 | newer Nano-Banana gen — `value` default |
| `google/gemini-3-pro-image` | ~$0.13 | 4K, best quality — `premium` |
| `openai/gpt-5-image-mini` / `gpt-5-image` | ~$0.02 / ~$0.10 | OpenAI alternatives |

5 images/video → **$0.10 (lite) to $0.20 (Nano Banana) to $0.65 (pro)**.

## 3. TTS — quality is basically free, upgrade away from Kokoro

| Model | $/Mtok | Quality | Format | Use |
|---|---|---|---|---|
| `hexgrad/kokoro-82m` | 0.0006 | robotic (you flagged this) | mp3 | kept only as an override/fallback |
| **`google/gemini-3.1-flash-tts`** ⭐ | 0.001→0.02 | natural (Charon/Puck/Zephyr) | **pcm** (auto-converted) | `cheap` + `value` default |
| `canopylabs/orpheus-3b` | 0.007 | expressive, emotive tags | mp3 | horror/drama |
| `mistralai/voxtral-mini-tts` | 0.016 | emotional (paul voices) | mp3 | drama |
| `microsoft/mai-voice-2` | 0.022 | high-quality, multilingual | mp3 | `premium` |

Per video (~700 chars) = **~$0.00002 even on the priciest**. Listen & pick: `docs/samples/tts/`. Default has moved off Kokoro to **Gemini Flash TTS** because the cost delta is a rounding error and the Reddit output was audibly too robotic.

## 4. Hero video (Wave 3+, the one real expense) — per second

| Model | ~$/sec | ~$/5s clip | Notes |
|---|---|---|---|
| `bytedance/seedance-2.0-fast` | ~0.022 | ~$0.11 | cheapest usable |
| `bytedance/seedance-1-5-pro` | ~0.047 | ~$0.24 | with audio |
| **`google/veo-3.1-fast`** ⭐ | ~0.09 | ~$0.45 | native audio, value hero |
| `google/veo-3.1` | ~0.18 | ~$0.90 | best cinematic |
| `kwaivgi/kling-v3.0`, `minimax/hailuo-2.3`, `alibaba/wan-2.6/2.7` | varies | — | alternatives |
| `openai/sora-2-pro` | — | — | ⚠️ **avoid — API sunsets ~Sept 2026** |

(Video pricing is computed per-request by resolution/duration; figures from research, confirm via `/api/v1/videos/models` at call time.)

---

## Per-video cost by tier

| Tier | LLM | 5 images | TTS | hero | **≈ total** |
|---|---|---|---|---|---|
| **cheap** | ~$0.001 | $0.20 (Nano Banana) | ~$0 | none | **~$0.20** |
| **value** | ~$0.002 | $0.20 | ~$0 | none | **~$0.20** |
| **premium** | ~$0.002 | $0.65 (pro) | ~$0 | +$0.45 (Veo fast) | **~$1.10** |

## How to swap (this is the whole "easy swap" answer)

Everything routes through `resolveModels(tier)` in `server/src/config/models.ts`. To change a model:

- **Pick a tier for a reel:** set `reel.tier` = `cheap|value|premium` (already wired — drives image + TTS + video).
- **Change a tier's default:** edit the `REGISTRY` table in `models.ts`.
- **Override one model globally without code:** env vars — `LLM_MODEL`, `IMAGE_MODEL`, `TTS_MODEL`, `TTS_VOICE`, `TTS_FORMAT`, `VIDEO_MODEL`, or `MODEL_TIER`.

No service code references a model slug directly, so swaps never touch the pipeline/render architecture.
