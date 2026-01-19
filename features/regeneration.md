# Regeneration

Regeneration reuses existing speech audio to avoid repeating TTS generation.

## Endpoint

- `POST /api/compositions/:id/regenerate`

## Inputs

- `delays`: optional array of per-line delays (seconds)
- `screenType`: optional new screen type
- `subtitlePosition`: optional new subtitle position
- `subtitleAnimation`: optional new subtitle animation style
- `characterPositions`: optional overrides

## Flow

1. Load the existing Composition and Template (with characters).
2. Ensure each line has a `speechUrl` (required for regeneration).
3. Apply custom delays or update screen type/positions.
4. Recalculate start times for all lines.
5. Delete previous S3 outputs (video + subtitles) to avoid storage bloat.
6. Download audio files from S3 to local processing path.
7. Run the full FFmpeg pipeline again using existing audio.
8. Update Composition with new output URLs and mark completed.

Files: `server/src/services/composition.service.ts`.
