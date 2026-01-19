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
- An `AudioSegment` object is created containing:
  - `characterId`: ID of the speaking character.
  - `text`: Dialogue text.
  - `audioPath`: Local path to the generated MP3.
  - `speechUrl`: S3 URL for persistence.
  - `startTime` & `duration`: Temporal markers.
- Update line duration in the database based on actual audio length.

After all lines:

- Recalculate timings using `recalculateTimings` to incorporate actual audio durations.

## 4) Video Processing (FFmpeg)

### 4.1 Convert to target aspect

- `convertToAspectRatio` crops/scales the template to either:
  - `mobile`: 1080x1920 (9:16)
  - `desktop`: 1920x1080 (16:9)

### 4.2 Overlay characters

- Build `VideoSegment` objects per line using the previously generated `AudioSegment` list.
- Includes character image path, temporal range (`startTime` to `endTime`), and a normalized `IRequiredCharacterPosition`.
- Apply overlays using FFmpeg's `overlay` filter with support for dynamic positioning:
  - For `none` animation: Simple static `x:y` pixel coordinates.
  - For `slide_in_left/right`: Complex if-then-else expressions that use the `t` (time) variable to interpolate `x` from off-screen to the target coordinate over `animationDuration`.
- The filter is enabled using the `:enable='between(t,start,end)'` option.

### 4.3 Merge audio

- Mix all dialogue audio tracks from the `AudioSegment` list (and reduce template audio if present).

### 4.4 Generate subtitles

- Generate karaoke ASS subtitles with highlighted words.
- Upload the subtitle file to S3.
- Burn subtitles into the video.

### 4.5 Trim and finalize

- Trim video to exact conversation duration.
- Finalize encoding (H.264 + AAC).

## 5) Upload and Cleanup

- Upload final video to S3.
- Delete all temporary processing files (audio segments, intermediate video tracks).
- Update Composition status to `completed`.

## Error Handling & Asset Cleanup

The pipeline implements robust error handling to ensure orphaned local files are removed even on failure:

1.  **Pipeline-Level Catch**: Errors caught in `processComposition` or `regenerateCompositionAsync` trigger a cleanup of all local `AudioSegment` files.
2.  **Helper-Level Catch**: Inside `processVideoWithAudioAndSubtitles`, a recursive `tempFiles` tracker is used. If any intermediate FFmpeg step fails (aspect correction, overlays, audio merging), all files generated _up to that point_ are immediately unlinked from the local disk.
3.  **Status Update**: The Composition record is updated with status `failed` and the error message is stored in the `error` field for diagnosis.
4.  **S3 Resilience**: During regeneration, if old S3 assets exist for that composition, they are deleted before the new ones are uploaded to save storage costs.
