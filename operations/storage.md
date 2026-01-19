# Storage Layout

## Local Processing

`config.processingPath` (default `./storage/processing`) is used for:

- downloaded template videos (after upload)
- generated speech audio
- intermediate FFmpeg outputs
- temporary ASS subtitle files

These files are cleaned up at the end of each pipeline run. The cleanup mechanism ensures that even on failure, local assets like speech audio and intermediate video tracks are unlinked to prevent disk bloat.

## S3 Layout

The S3 bucket stores canonical assets by prefix:

- `templates/` (uploaded background videos)
- `characters/` (character images)
- `audio/` (speech audio clips)
- `subtitles/` (ASS/SRT files)
- `compositions/` (final videos)

## Naming

Files use the `generateFilename(prefix, extension)` utility, which includes a timestamp and random suffix.
