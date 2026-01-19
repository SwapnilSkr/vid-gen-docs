# Audio Test Utilities

The codebase includes a dedicated audio test service and routes for debugging audio merging.

## Endpoints

- `GET /api/audio/test`: generate a synthetic test composition with beeps
- `POST /api/audio/test`: custom test with a given video URL + audio configs
- `GET /api/audio/test/files`: list test output files
- `DELETE /api/audio/test/files`: delete test files

## How It Works

Implemented in `server/src/services/audio.service.ts`.

- Uses `ffmpeg` to generate sine wave audio files (lavfi)
- Merges them onto a sample or provided video
- Uses a fixed merge function that supports source videos with or without audio

## Why It Exists

The FFmpeg audio merge logic can be sensitive to source audio streams. These endpoints help validate the pipeline in isolation.
