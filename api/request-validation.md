# Request Validation (Guards)

The API uses Elysia guards (`t.Object`) to validate request bodies and params.

Source files: `server/src/types/guards/*`.

## Common

- `IdParams`: `{ id: string }`

## Templates

- `CreateTemplateBody`:
  - `video`: File (required)
  - `name`: string
  - `description`: optional string

- `UpdateTemplateBody`:
  - Any subset of `CreateTemplateBody`

- `TemplateCharactersBody`:
  - `characterIds`: string[]

- `TemplateQuery`:
  - `trimStart`: optional number
  - `keepDuration`: optional number
  - `removeAudio`: optional boolean

## Characters

- `CreateCharacterBody`:
  - `image`: File (required)
  - `name`: string
  - `displayName`: string
  - `voiceId`: string

- `UpdateCharacterBody`:
  - `image`: optional File
  - `displayName`: optional string
  - `voiceId`: optional string

## Compositions

- `CharacterPositionSchema`:
  - `x`, `y`: optional number (0-100)
  - `scale`: optional number (0.01-2)
  - `anchor`: `top-left | top-right | bottom-left | bottom-right | center`
  - `animation`: optional `none | slide_in_left | slide_in_right`
  - `animationDuration`: optional number (0.1 - 3.0 seconds)

- `CreateCompositionBody`:
  - `templateId`: string
  - `plot`: string
  - `title`: optional string
  - `screenType`: optional `mobile | desktop`
  - `subtitlePosition`: optional `top | center | bottom`
  - `characterPositions`: optional record of `characterId -> position`

- `RegenerateCompositionBody`:
  - `delays`: optional number[]
  - `screenType`: optional `mobile | desktop`
  - `subtitlePosition`: optional `top | center | bottom`
  - `characterPositions`: optional record of `characterId -> position`

## Reels

- `CreateReelBody`:
  - `niche`: string
  - `genre`: optional string
  - `topic`: optional string
  - `tier`: optional `cheap | value | premium`
  - `source`: optional `llm | hybrid | verbatim`
  - `parts`: optional `off | auto | number`
