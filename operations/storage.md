# Storage Layout

## Local Processing

`config.processingPath` (default `./storage/processing`) is used for:

- downloaded template videos (after upload)
- generated speech audio
- intermediate FFmpeg outputs
- temporary ASS subtitle files

These files are cleaned up at the end of each pipeline run. The cleanup mechanism ensures that even on failure, local assets like speech audio and intermediate video tracks are unlinked to prevent disk bloat.

## Gameplay library and download cache

`storage/gameplay/` contains intentional, persistent library clips and is never
age-swept. Gameplay objects fetched from S3 are stored separately under
`storage/gameplay/cache/` and are disposable.

The disposable cache is swept on startup and by the hygiene scheduler. It uses
file modification time as last-access time and applies both an age limit and an
LRU size cap. Files used within the protection window are never removed, so a
cleanup cannot race an active FFmpeg render.

Configuration:

- `GAMEPLAY_CACHE_MAX_GB` — maximum cache size, default `5` GB.
- `GAMEPLAY_CACHE_MAX_AGE_DAYS` — idle retention, default `14` days.
- `GAMEPLAY_CACHE_PROTECT_HOURS` — recent-use protection, default `2` hours.

The cache can be inspected without deletion at
`GET /api/maintenance/gameplay-cache-cleanup`; add `?apply=true` to run it.

## S3 Layout

The S3 bucket stores canonical assets by prefix:

- `templates/` (uploaded background videos)
- `characters/` (character images)
- `audio/` (speech audio clips)
- `subtitles/` (ASS/SRT files)
- `compositions/` (composition output videos)
- `reels/` (generated reel output videos)

## Naming

Files use the `generateFilename(prefix, extension)` utility, which includes a timestamp and random suffix.
