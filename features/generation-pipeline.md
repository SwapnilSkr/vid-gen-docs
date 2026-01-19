# Generation Pipeline

This document walks through the full asynchronous pipeline used to generate a video.

Source files:

- `server/src/services/composition.service.ts`
- `server/src/helpers/composition.helper.ts`
- `server/src/services/ffmpeg.service.ts`
- `server/src/services/subtitle.service.ts`

## 1) Composition Creation

- Validate the template exists and has at least one character.
- Populate template characters to compute default positions.
- Merge any provided positions and normalize with defaults.
- Create a Composition document in MongoDB with status `pending`.

## 2) Script Generation (OpenRouter)

- Generate script from plot + character list.
- Validate dialogue character names.
- Calculate initial timings using `calculateDialogueTiming` (delay + duration).
- Save `generatedScript` with character references.

## 3) Speech Generation (ElevenLabs)

For each line:

- Generate speech audio using `generateSpeech`.
- Upload to S3 and store `speechUrl`.
- Update line duration based on actual audio length.

After all lines:

- Recalculate timings using `recalculateTimings` to incorporate actual audio durations.

## 4) Video Processing (FFmpeg)

### 4.1 Convert to target aspect

- `convertToAspectRatio` crops/scales the template to either:
  - `mobile`: 1080x1920 (9:16)
  - `desktop`: 1920x1080 (16:9)

### 4.2 Overlay characters

- Build `VideoSegment` objects per line and position.
- Apply overlays with FFmpeg complex filters.

### 4.3 Merge audio

- Mix all dialogue audio tracks (and reduce template audio if present).

### 4.4 Generate subtitles

- Generate karaoke ASS subtitles with highlighted words.
- Upload the subtitle file to S3.
- Burn subtitles into the video.

### 4.5 Trim and finalize

- Trim video to exact conversation duration.
- Finalize encoding (H.264 + AAC).

## 5) Upload and Cleanup

- Upload final video to S3.
- Delete all temporary processing files.
- Update Composition status to `completed`.

## Error Handling

Errors are caught at the pipeline level and written to the Composition record with status `failed`.
