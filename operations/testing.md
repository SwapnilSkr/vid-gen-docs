# Testing and Diagnostics

## API Sanity

- `GET /health` for service health
- `GET /` for API info
- `GET /api/ffmpeg` — FFmpeg / libass / fonts capability (gates create & produce)

## Caption burn smoke (required on every Mac / device)

Caption burns use FFmpeg's `ass` (libass) filter. Produce **hard-fails** on caption
burn errors so paid scene assets stay on S3 — use **Resume** after fixing FFmpeg.

Create / approve / regenerate / resume / apply-captions all call
`assertFfmpegReady()` first. The Studio and Create screens show an **FFmpeg required**
modal when the check fails.

### CLI

From `server/`:

```bash
bun run fetch-fonts          # once per machine if assets/fonts is empty
bun run smoke:captions       # exit 0 only when captions actually render
bun run smoke:captions -- --keep   # keep the sample MP4 for visual check
```

### Front → backend

```bash
curl -s http://localhost:3000/api/ffmpeg | jq
curl -s http://localhost:3000/api/maintenance/caption-smoke | jq
```

```ts
import { getFfmpegStatus, runCaptionSmokeTest, applyCaptions, resumeFailedReel } from "@/api/reels";

const ffmpeg = await getFfmpegStatus();
if (!ffmpeg.ok) throw new Error(ffmpeg.message);

await runCaptionSmokeTest();
await applyCaptions(reelId, { fontSize: 72 });
// After a caption-burn failure:
await resumeFailedReel(reelId); // reuses S3 assets, re-runs render only
```

## Audio Pipeline Tests

- `GET /api/audio/test` runs a synthetic test using beep tones
- `POST /api/audio/test` allows custom audio configs

## Lint and Typecheck

From `server/`:

```
bun run lint
bun run typecheck
```
