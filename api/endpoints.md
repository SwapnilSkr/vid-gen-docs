# API Endpoints

Base URL: `http://localhost:3000`

Responses use the shape:

```
{ "success": true, "data": ... }
{ "success": false, "error": "message" }
```

## Health and Info

- `GET /health`
  - Returns status, timestamp, uptime
- `GET /`
  - Returns API info and usage hints

## Templates

- `POST /api/templates`
  - Multipart body: `video` (file), `name` (string), `description` (optional)
  - Query params (optional): `trimStart`, `keepDuration`, `removeAudio`

- `GET /api/templates`

- `GET /api/templates/:id`

- `PUT /api/templates/:id`
  - Multipart body: `video` (optional), `name` (optional), `description` (optional)
  - Query params (optional): `trimStart`, `keepDuration`, `removeAudio`

- `POST /api/templates/:id/characters`
  - JSON body: `{ "characterIds": ["..."] }`

- `DELETE /api/templates/:id/characters`
  - JSON body: `{ "characterIds": ["..."] }`

- `DELETE /api/templates/:id`

## Characters

- `POST /api/characters`
  - Multipart body: `image` (file), `name`, `displayName`, `voiceId`

- `GET /api/characters`

- `GET /api/characters/:id`
  - Supports ID or name

- `PUT /api/characters/:id`
  - Multipart body: `image` (optional), `displayName` (optional), `voiceId` (optional)

- `DELETE /api/characters/:id`

## Compositions (Primary)

- `POST /api/compositions`
  - JSON body: `{ templateId, plot, title?, screenType?, subtitlePosition?, characterPositions? }`

- `GET /api/compositions`
  - Query: `limit` (default 50)

- `GET /api/compositions/:id/status`

- `GET /api/compositions/:id/download`

- `POST /api/compositions/:id/regenerate`
  - JSON body: `{ delays?, screenType?, subtitlePosition?, characterPositions? }`

## Generate (Alias)

- `POST /api/generate`
  - Same body as `POST /api/compositions`

- `GET /api/generate/:id`

## Voices

- `GET /api/voices`
  - Lists ElevenLabs voices

## Audio Tests

- `GET /api/audio/test`
- `POST /api/audio/test`
  - JSON body: `{ videoUrl?, audioConfigs: [{ startTime, frequency?, duration? }] }`
- `GET /api/audio/test/files`
- `DELETE /api/audio/test/files`
