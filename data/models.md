# Data Models

This file documents the MongoDB schemas defined in `server/src/models`.

## Character

Schema: `server/src/models/character.model.ts`

Fields:

- `name` (string, unique, lowercase)
- `displayName` (string)
- `voiceId` (string)
- `imageUrl` (string)
- `createdAt`, `updatedAt` (timestamps)

## Template

Schema: `server/src/models/template.model.ts`

Fields:

- `name` (string)
- `description` (string)
- `videoUrl` (string)
- `thumbnailUrl` (string, optional)
- `duration` (number or null)
- `dimensions` (width, height)
- `frameRate` (number)
- `characters` (ObjectId[])
- `createdAt`, `updatedAt`

## Composition

Schema: `server/src/models/composition.model.ts`

Fields:

- `template` (ObjectId)
- `title` (string)
- `plot` (string)
- `screenType` (`mobile | desktop`)
- `characterPositions` (Map of `characterId -> position`)
- `generatedScript` (array of dialogue lines)
- `subtitlePosition` (`top | center | bottom`)
- `subtitleAnimation` (`none | pop | shake | reel`)
- `status` (`pending`, `generating_script`, `generating_audio`, `compositing`, `completed`, `failed`)
- `progress` (0-100)
- `outputUrl` (string, optional)
- `subtitlesUrl` (string, optional)
- `error` (string, optional)
- `createdAt`, `updatedAt`

### Dialogue Line

- `character` (ObjectId)
- `text` (string)
- `startTime` (number)
- `duration` (number)
- `delay` (number)
- `speechUrl` (string, optional)

### Character Position

- `x`, `y` (percentage 0-100)
- `scale` (0.01-2)
- `anchor` (`top-left`, `top-right`, `bottom-left`, `bottom-right`, `center`)

Note: positions are normalized so all fields are populated when stored.

## TrendReference

Schema: `server/src/models/trend-reference.model.ts`

Fields:

- `niche` (string)
- `genre` (string, optional)
- `sourceUrl` (string, unique)
- `platform` (`youtube_shorts | instagram_reels | tiktok | reddit | unknown`)
- `metrics` (`views`, `likes`, `comments`, `shares`, `saves`, `durationSec`, `postedAt`, `capturedAt`)
- `notes` (string, optional)
- `status` (`candidate`, `reviewed`, `approved`, `rejected`, `archived`)
- `createdAt`, `updatedAt`

Service helpers: `createTrendReference` and `listTrendReferences` in `server/src/services/trend-reference.service.ts`.
