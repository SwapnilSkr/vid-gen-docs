# Subtitles (Karaoke Style)

Subtitles are generated in ASS format with per-word highlighting that advances with the spoken audio.

## Generation Algorithm

Implemented in `server/src/services/subtitle.service.ts`.

1. Split each dialogue line into chunks of 2-3 words (randomized).
2. Assign each chunk a duration: `dialogue.duration / chunks.length`.
3. Multiply chunk duration by `chunkSpeedMultiplier` to increase speed.
4. Split each chunk into words.
5. Apply animation based on `subtitleAnimation` (default: `none`):
   - `none`: Color change only (active word uses secondary color).
   - `pop`: Active word scales up (120%) then back down.
   - `shake`: Active word rotates ±8° left/right.
   - `reel`: Active word slides in from the left with a fade-in.
6. Create an ASS dialogue line for each word.

## ASS Header

- Uses `PlayResX: 1920`, `PlayResY: 1080`
- Style `Default` with font `Arial`, size 48, outline/shadow
- Alignment and vertical margin depend on subtitle position

## Positioning

Position is set by the composition's `subtitlePosition`:

- `top`: alignment 8, marginV 20
- `center`: alignment 5, marginV 0
- `bottom`: alignment 2, marginV 30

## Configuration

Defined in `server/src/config/index.ts`:

- `SUBTITLE_PRIMARY_COLOR` (default `#FFFFFF`)
- `SUBTITLE_SECONDARY_COLOR` (default `#00FF00`)
- `CHUNK_SPEED_MULTIPLIER` (default `0.75`)

## Output

- The ASS content is uploaded to S3 via `uploadSubtitles`
- The same content is burned into the video via FFmpeg

## Notes

- ASS detection is automatic: if content starts with `[Script Info]`, conversion is skipped.
- Karaoke subtitles are more verbose than plain SRT but render quickly in FFmpeg.
