# Templates

Templates are background videos that serve as the base for compositions. Templates store video metadata and the list of allowed characters.

## Data Model

Template fields (see `server/src/models/template.model.ts`):

- `name` (string)
- `description` (string)
- `videoUrl` (S3 URL)
- `thumbnailUrl` (optional)
- `duration` (number or null)
- `dimensions` (width/height)
- `frameRate`
- `characters` (array of Character ObjectIds)

## Endpoints

- `POST /api/templates`: upload template video
- `GET /api/templates`: list templates
- `GET /api/templates/:id`: fetch template
- `PUT /api/templates/:id`: update metadata and/or replace video
- `POST /api/templates/:id/characters`: attach characters
- `DELETE /api/templates/:id/characters`: detach characters
- `DELETE /api/templates/:id`: delete template and assets

See [docs/api/endpoints.md](../api/endpoints.md) for request/response details.

## Upload + Processing Flow

1. Request hits `template.routes.ts` and uses `localFileSave` middleware
2. Local file is saved to disk in `config.processingPath`
3. `createTemplateWithProcessing` applies optional processing:
   - `trimStart` (seconds)
   - `keepDuration` (seconds)
   - `removeAudio` (boolean)
4. Video is uploaded to S3 via `uploadVideo`
5. Metadata is read via `getVideoMetadata` (ffprobe)
6. Template record is created in MongoDB

Files: `server/src/services/template.service.ts`, `server/src/routes/template.routes.ts`.

## Update Behavior

- If a new video is provided, metadata is recomputed and the old S3 video is deleted.
- If `thumbnailUrl` changes, the old thumbnail is deleted from S3.

## Character Assignment

- Templates maintain a `characters` array of ObjectIds.
- When generating a composition, at least one character must be assigned.
