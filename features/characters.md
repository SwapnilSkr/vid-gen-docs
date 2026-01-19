# Characters

Characters are avatar images with associated voice IDs. They are linked to templates and appear as overlays in compositions.

## Data Model

Character fields (see `server/src/models/character.model.ts`):

- `name` (unique, lowercased)
- `displayName`
- `voiceId` (ElevenLabs voice ID)
- `imageUrl` (S3 URL)

## Endpoints

- `POST /api/characters`: create character (image upload + metadata)
- `GET /api/characters`: list characters
- `GET /api/characters/:id`: fetch by ID or by name
- `PUT /api/characters/:id`: update name, voice, or image
- `DELETE /api/characters/:id`: delete

## Upload Flow

1. Route uses `fileUpload` middleware to upload image to S3
2. Uploaded image URL is injected into the controller context
3. `createCharacter` persists the character in MongoDB

Files: `server/src/middlewares/upload.middleware.ts`, `server/src/services/character.service.ts`.

## Update Behavior

- If the image is replaced, the old image is deleted from S3.
- Character names are normalized to lowercase for lookup consistency.
