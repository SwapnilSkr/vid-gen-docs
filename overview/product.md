# Product Overview

## Purpose

The Templated AI Video Generator is a backend API that creates short-form videos from a text plot, a background video template, and a set of character assets. The API generates a script, synthesizes audio, and renders a final video with character overlays and karaoke-style subtitles.

## Core Concepts

- **Template**: A background video with metadata (duration, frame rate, dimensions). Templates define which characters are allowed in a composition.
- **Character**: An avatar image and a voice ID. Characters are linked to templates and assigned on a per-composition basis.
- **Composition**: A single generated video instance from a plot. It stores script, positions, and output URLs.

## Typical Flow

1. Upload a template background video.
2. Create characters (upload images + voice IDs).
3. Associate characters with a template.
4. Submit a plot to generate a composition.
5. Poll for status until completed.

## Outputs

- **Final video** (S3 URL)
- **Subtitles** (ASS format, S3 URL)
- **Script** (dialogue lines with timings)

## Key Features

- One-command generation (`POST /api/generate`)
- AI-driven script + title generation
- TTS with ElevenLabs
- Karaoke-style subtitles with word highlighting
- Aspect ratio conversion (mobile/desktop)
- Regeneration using cached speech

## Non-Goals (Current Code)

- No user authentication/authorization layer
- No billing or quotas
- No multi-tenant isolation (consider API gateway)
