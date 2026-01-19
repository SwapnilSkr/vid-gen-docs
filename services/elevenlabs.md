# ElevenLabs (Text-to-Speech)

Implemented in `server/src/services/elevenlabs.service.ts`.

## Purpose

- Generate speech audio for each dialogue line
- Cache repeated audio to reduce API calls

## Key Behaviors

- Uses `ElevenLabsClient` with `ELEVEN_LABS_API_KEY`.
- Per-line audio is generated via `client.generate`.
- Audio is saved to `config.processingPath` and then uploaded to S3.
- Duration is read using ffprobe; falls back to 3 seconds on failure.

## Audio Cache

An in-memory cache stores `{ voiceId + text -> local audio path }` so repeated identical lines do not re-trigger ElevenLabs.

## Voice Settings

If custom settings are provided, the following defaults apply:

- stability: 0.5
- similarityBoost: 0.75
- style: 0.5
- useSpeakerBoost: true

## Voice Listing

`getVoices()` fetches all available voices and returns a list of `{ id, name }`.
