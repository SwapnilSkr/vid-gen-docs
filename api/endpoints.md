# API Endpoints

Base URL: `http://localhost:3000`

Responses use the shape:

```
{ "success": true, "data": ... }
{ "success": false, "error": "message" }
```

## Health and Info

- `GET /health`
  - Returns status, timestamp, uptime
- `GET /`
  - Returns API info and usage hints

## Operations

- `GET /api/operations`
  - Query: `limit` (1–100), `before` (ISO timestamp), `level`, `scope`, `reelId`.
  - Returns redacted persistent backend API/queue/worker/provider log entries and an optional `nextBefore` cursor.
- `DELETE /api/operations/:id`
  - Permanently deletes one operation log entry.
- `DELETE /api/operations`
  - JSON body: `{ ids: ["..."] }` (up to 100). Permanently deletes selected entries.
- `DELETE /api/operations/all`
  - Permanently deletes every Operations entry. Successful Operations reads/deletes are not logged, so this can leave the collection empty.

## Templates

- `POST /api/templates`
  - Multipart body: `video` (file), `name` (string), `description` (optional)
  - Query params (optional): `trimStart`, `keepDuration`, `removeAudio`

- `GET /api/templates`

- `GET /api/templates/:id`

- `PUT /api/templates/:id`
  - Multipart body: `video` (optional), `name` (optional), `description` (optional)
  - Query params (optional): `trimStart`, `keepDuration`, `removeAudio`

- `POST /api/templates/:id/characters`
  - JSON body: `{ "characterIds": ["..."] }`

- `DELETE /api/templates/:id/characters`
  - JSON body: `{ "characterIds": ["..."] }`

- `DELETE /api/templates/:id`

## Characters

- `POST /api/characters`
  - Multipart body: `image` (file), `name`, `displayName`, `voiceId`

- `GET /api/characters`

- `GET /api/characters/:id`
  - Supports ID or name

- `PUT /api/characters/:id`
  - Multipart body: `image` (optional), `displayName` (optional), `voiceId` (optional)

- `DELETE /api/characters/:id`

## Compositions (Primary)

- `POST /api/compositions`
  - JSON body: `{ templateId, plot, title?, screenType?, subtitlePosition?, subtitleAnimation?, characterPositions? }`

- `GET /api/compositions`
  - Query: `limit` (default 50)

- `GET /api/compositions/:id/status`

- `GET /api/compositions/:id/download`

- `POST /api/compositions/:id/regenerate`
  - JSON body: `{ delays?, screenType?, subtitlePosition?, subtitleAnimation?, characterPositions? }`

## Generate (Alias)

- `POST /api/generate`
  - Same body as `POST /api/compositions`

- `GET /api/generate/:id`

## Reels (scene-graph pipeline — Reddit/AITA, dark history, etc.)

- `POST /api/reels`
  - JSON body: `{ niche, genre?, topic?, tier?, source?, parts?, gameplayKey? }` — `topic` omitted/empty/`"auto"` pulls
    from the story bank (Reddit niche) or lets the planner choose freely
    (other niches). `tier`: `cheap` (default) | `value` | `premium`.
    `source` is optional and only affects Reddit/gameplay reels:
    `llm` | `hybrid` | `verbatim`; if omitted, `STORY_MODE` is used as the
    default for auto Reddit sourcing.
    `genre` is optional and only affects Reddit/gameplay reels. Current IDs:
    `aita_family`, `relationship_drama`, `wedding_drama`, `petty_revenge`,
    `pro_revenge`, `malicious_compliance`, `entitled_people`,
    `choosing_beggars`, `workplace_justice`, `customer_service`,
    `confession`, `tifu`, `updates`, `neighbors`.
    `parts` is optional and only affects Reddit/gameplay reels: `"off"`/omitted
    creates one reel, `"auto"` creates a series only when the source warrants it,
    and a number forces two to four parts.
    `gameplayKey` is optional and only affects gameplay_overlay reels: an
    explicit S3 key (e.g. `gameplay/parkour_003.mp4`, from `GET /api/gameplay`)
    to use instead of a random pick from the pool. The resolved key is always
    recorded on the reel as `gameplayKey` (random or explicit), so revoice
    reuses the same clip.
    `ttsModel`/`ttsVoice`/`ttsFormat` are optional — an explicit voice pick
    from `GET /api/tts-voices`, overriding the tier default and any niche
    voice override (precedence: tier default < niche override < this).
    Recorded as `reel.voiceOverride`.

- `GET /api/reels`
  - Query: `limit` (default 50)

- `GET /api/reels/:id/status`
  - Includes render state, `outputUrl`, `subtitlesUrl`, `redditStory`, scene data,
    `review`, and `youtube` publish state.

- `GET /api/reels/:id/download`

- `GET /api/reels/:id/review`
  - Returns or lazily creates the review package for a completed reel:
    `{ title, description, tags, thumbnailUrl, thumbnailPrompt, visibilityNotes, status }`.

- `PUT /api/reels/:id/review`
  - JSON body: `{ title?, description?, tags?, thumbnailPrompt?, visibilityNotes?, status? }`
  - Used by the frontend review step before publishing.

- `POST /api/reels/:id/review/thumbnail`
  - JSON body: `{ title?, description?, tags?, thumbnailPrompt?, visibilityNotes?, status? }`
  - Saves the supplied review fields, generates an AI background from
    `thumbnailPrompt` (via the OpenRouter image model, prompt informed by the
    genre's trend-insight digest) with bold hook text composited on top,
    uploads it, and returns the updated review package. Falls back to the
    original flat mockup design if image generation fails.

- `POST /api/reels/:id/review/copy`
  - Generates a new YouTube Shorts title and description from the reel source.
  - Preserves upload tags, thumbnail, Instagram caption, destination state, and
    rendered media. Its LLM receipt is appended to the reel cost breakdown.

- `POST /api/reels/:id/instagram-caption`
  - Generates a new platform-specific Instagram caption from the reel's source
    material. It does not modify the YouTube review package or render media.
  - Returns the updated reel with `instagramSettings.caption`, provenance, and
    model. The LLM receipt is appended to the reel cost breakdown.

- `POST /api/reels/:id/outro/comment-prompt`
  - Available for `plan_review` and `completed` reels. Generates a short,
    story-and-part-specific branded-outro question using the cheap LLM.
  - Optional JSON body: `{ scope?: "primary" | "inheriting" | "all" }`.
    `primary` preserves existing extra-channel questions, `inheriting` updates
    only blank/inheriting extra drafts, and `all` deliberately replaces every
    destination's question with the new story-specific copy.
  - Appends its actual LLM receipt to the cost breakdown immediately. It does
    not touch story scenes, gameplay, or body narration. On a completed reel it
    invalidates only the primary output and extra destinations that inherit the
    primary blank prompt; an optional `outro_only` rerender then uses the cached
    body and skips ready explicit-override destinations.

- `POST /api/reels/:id/destinations/primary`
  - JSON body: `{ platform, channelId, previousPrimary: "keep" | "remove",
    scope: "reel" | "series" }`.
  - Promotes a connected account as the one required primary destination. With
    `keep`, the former primary becomes an extra destination and keeps its own
    output/audio. With `remove`, only that former primary's media for the
    selected reel or series is reclaimed from S3. Neither choice deletes or
    disconnects the globally connected social account.

- `POST /api/reels/:id/review/thumbnail/frame`
  - JSON body: `{ atSeconds }` — extracts that frame from the rendered video
    (`reel.outputUrl`), uploads it, and sets it as the review thumbnail
    instead of the AI/flat design.

- `POST /api/reels/:id/publish`
  - Queues YouTube Shorts publishing for a completed reel.
  - Uses the reviewed title, description, tags, and thumbnail when present.

- `POST /api/reels/:id/revoice`
  - Only for completed `gameplay_overlay` reels. JSON body:
    `{ variants: [{ model?, voice?, format?, label? }] }` (1-5 entries; any
    field omitted falls back to the reel's tier default).
  - Registers each as a `pending` voice variant and queues its render
    (BullMQ `reel-revoicing`). Reuses the exact same story and gameplay clip
    (`reel.gameplayKey`) as the original render — only the narration changes.
  - Returns the reel's `voiceVariants` array; poll `GET /api/reels/:id/status`
    for each variant's `status` (`pending` → `ready`/`failed`) and `videoUrl`.

- `POST /api/reels/:id/revoice/:variantId/promote`
  - Promotes a ready voice variant as the shared body and queues an outro-only
    rebuild. Every configured channel receives a fresh channel-specific final;
    the raw variant itself is never publishable.

- `DELETE /api/reels/:id`

## Trend scout

Populated by `trend-scout.service.ts` (YouTube Data API v3 `search.list`/`videos.list`
per Reddit genre — needs `YOUTUBE_DATA_API_KEY`) — see
`scripts/trend-scout-backfill.ts` (one-off: last 30 days + last 48 hours) and
`scripts/trend-scout-daily.ts` (recurring: `week` daily / `month` weekly cron).
Each scan also refreshes a per-genre `TrendInsight` digest
(`trend-insight.service.ts`) — a short LLM-distilled bullet summary of
winning title/hook patterns that `planRedditStory`/`generateStory`/
thumbnail-prompt generation read (cheap cached read, not raw reference dumps).

- `GET /api/trends`
  - Query: `niche?, genre?, platform?, status?, limit?` — raw `TrendReference` rows for dashboard browsing.

- `GET /api/trends/summary`
  - Query: `period` = `week` (default, rolling 7-day scans) | `month` (rolling 30-day scans).
  - Per genre: top 5 performers (title/thumbnail/views/postedAt) + a
    view-weighted posting-time histogram (`dayOfWeek`/`hourUtc`, UTC) with a
    `bestPostingTime` pick. Caveat: this clusters *competitors'* publish
    timestamps (correlational, not a real view-velocity signal) — treat as a
    starting prior, not ground truth; your own channel's YouTube Analytics
    data after publishing is the stronger long-term signal.

- `POST /api/trends/scout`
  - JSON body: `{ window? }` = `week` (default) | `month`. Runs the same
    scout-scan + insight-digest-refresh cycle as the cron scripts, on demand
    (used by the Trends dashboard's "Run Scout" button).

## Maintenance / cleanup

`deleteReel`/`DELETE /api/reels/:id` cascades to every S3 asset *recorded on
the doc* (output video, subtitles, thumbnail, scene stills/audio, voice
variants) plus the BullMQ job. It can't catch assets that were uploaded and
then the process crashed/stalled before the URL was ever saved to Mongo —
that's what the reconciliation sweep below is for. See also
`scripts/reconcile-s3.ts` (same logic, cron-friendly, dry-run by default).
Before deleting a Reddit reel, the service upserts durable `Story` source
history (including older manually pasted/replanned reels). This retains no S3
media and is separate from user-clearable Operations telemetry; Browse and
pasted-link duplicate checks therefore continue to reject a story that has
already been produced after its reel record is gone.

- `GET /api/maintenance/s3-reconcile`
  - Query: `apply?` = `"true"` to actually delete (default: dry run, report only).
  - Sweeps `reels/`, `compositions/`, `audio/`, `subtitles/` for objects not
    referenced by any current `Reel` or `Composition` document. Skips
    anything uploaded in the last 2 hours (might belong to an in-flight
    render). Deliberately excludes `voice-samples/` (permanent cache) and
    `gameplay/` (persistent clip pool) — those aren't tied to per-job docs.

- `POST /api/maintenance/reels/purge-failed`
  - Runs the full `deleteReel()` cascade on every reel currently `status: "failed"`.

## Gameplay & voices reference data

- `GET /api/gameplay`
  - Lists the S3 `gameplay/` clip pool: `{ key, url, filename }[]`. Powers the
    create-reel gameplay picker and revoice's clip reuse.

- `GET /api/tts-voices`
  - Curated cross-model OpenRouter TTS voice catalog (`config/models.ts`
    `TTS_VOICE_CATALOG`): `{ model, voice, format, label }[]`. Powers the
    revoice voice picker.

- `GET /api/voices`
  - Lists ElevenLabs voices (legacy Composition/character flow)

## Audio Tests

- `GET /api/audio/test`
- `POST /api/audio/test`
  - JSON body: `{ videoUrl?, audioConfigs: [{ startTime, frequency?, duration? }] }`
- `GET /api/audio/test/files`
- `DELETE /api/audio/test/files`
