# Compositions

A Composition represents one generated video instance. It is created from a template and plot, then processed asynchronously.

## Data Model

Key fields (see `server/src/models/composition.model.ts`):

- `template`: Template ObjectId
- `plot`: plot/prompt string
- `title`: AI-generated or user-provided
- `screenType`: `mobile` or `desktop`
- `characterPositions`: Map of character IDs to normalized positions (includes scale, anchor, and animation config)
- `generatedScript`: dialogue lines with timings and speech URLs
- `subtitlePosition`: `top`, `center`, `bottom`
- `subtitleAnimation`: `none`, `pop`, `shake`, `reel`
- `status`: generation status
- `progress`: 0-100
- `outputUrl`: final video URL (S3)
- `subtitlesUrl`: subtitle URL (S3)

## Status Lifecycle

Statuses used in code:

- `pending`
- `generating_script`
- `generating_audio`
- `compositing`
- `completed`
- `failed`

The `progress` field is updated at key milestones.

## Endpoints

Primary endpoints:

- `POST /api/compositions`: start a new composition
- `GET /api/compositions`: list recent compositions
- `GET /api/compositions/:id/status`: status polling
- `GET /api/compositions/:id/download`: fetch output URL
- `POST /api/compositions/:id/regenerate`: regenerate without TTS

Alias endpoints:

- `POST /api/generate`: start generation
- `GET /api/generate/:id`: fetch status and URLs

Files: `server/src/controllers/composition.controller.ts`, `server/src/routes/composition.routes.ts`.

## Character Positioning

`characterPositions` is stored as a Map of `characterId` to position:

- `x`, `y`: percentage of frame (relative to total width/height)
- `scale`: image scale multiplier (e.g., 0.3 = 30%)
- `anchor`: `top-left`, `top-right`, `bottom-left`, `bottom-right`, `center`
- `animation`: Character entry animation style:
  - `none`: Character appears instantly at the start of their line.
  - `slide_in_left`: Character slides from off-screen left to their target `x` position.
  - `slide_in_right`: Character slides from off-screen right to their target `x` position.
- `animationDuration`: Time in seconds for the animation to complete (default: `0.3`).

Positions are normalized during creation and regeneration so that all fields are populated (defaults applied when missing). Animation defaults to `none`.

### Animation Mechanics

Animations use FFmpeg overlay expressions to interpolate the position between the start time and `startTime + animationDuration`. For slide-ins, the character starts from a calculated off-screen coordinate based on their scaled size and the video dimensions.
