# FFmpeg Processing

Implemented in `server/src/services/ffmpeg.service.ts`.

## Core Functions

- `convertToAspectRatio(input, screenType)`
  - Scales and crops to 9:16 (1080x1920) or 16:9 (1920x1080)

- `getVideoMetadata(filePath)`
  - Uses ffprobe to read duration, resolution, frame rate, codec

- `applyCharacterOverlays(videoPath, segments)`
  - Adds multiple character overlays via a single complex filter

- `mergeAudioTracks(videoPath, audioSegments)`
  - Mixes dialogue tracks onto the video
  - Reduces template audio to 30% if present

- `addSubtitlesToVideo(videoPath, subtitleContent, position)`
  - Converts SRT to ASS if needed
  - Burns ASS subtitles using FFmpeg `ass` filter

- `trimVideo(input, { trimStart, keepDuration, removeAudio })`
  - Trims or shortens template footage

- `finalizeVideo(input, output)`
  - Encodes with H.264 + AAC, sets `+faststart`

## Position Calculations

Character positions are stored as percent coordinates and anchors. FFmpeg translates them to pixel positions based on the target video size and overlay scale.

## Notes

- `addSubtitlesToVideo` logs alignment and margin values for debugging.
- Temporary ASS files are written to `config.processingPath` and removed after use.
