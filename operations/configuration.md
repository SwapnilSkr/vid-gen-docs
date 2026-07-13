# Configuration and Environment Variables

Config source: `server/src/config/index.ts`.

## Server

- `PORT` (default: 3000)
- `HOST` (default: localhost)
- `NODE_ENV` (default: development)

## MongoDB

- `MONGODB_URI` (default: `mongodb://localhost:27017/video-generator`)
- `OPERATION_LOG_RETENTION_DAYS` (default: `30`) — TTL retention for Mongo-backed API/worker/fallback logs

## AWS S3

- `AWS_REGION` (default: `us-east-1`)
- `AWS_ACCESS_KEY_ID` (required in production)
- `AWS_SECRET_ACCESS_KEY` (required in production)
- `S3_BUCKET` (required in production)

## AI Services

- `OPENROUTER_API_KEY` (required for script generation)
- `OPENROUTER_MODEL` (default: `openai/gpt-4o-mini`)

## ElevenLabs

- `ELEVEN_LABS_API_KEY` (required for TTS)

## FFmpeg

- `FFMPEG_PATH` (default: `ffmpeg`)
- `FFPROBE_PATH` (default: `ffprobe`)

## Storage Paths

- `STORAGE_PATH` (default: `./storage`)
- `PROCESSING_PATH` (default: `./storage/processing`)

## Limits

- `MAX_FILE_SIZE_MB` (default: 2048)
- `MAX_VIDEO_DURATION` (default: 300)

## Subtitle Styling

- `SUBTITLE_PRIMARY_COLOR` (default: `#FFFFFF`)
- `SUBTITLE_SECONDARY_COLOR` (default: `#00FF00`)
- `CHUNK_SPEED_MULTIPLIER` (default: `0.75`)

## Validation

`validateConfig()` logs warnings if required values are missing; it does not halt the process.
