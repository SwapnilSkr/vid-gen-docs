# Repository Structure

This repository contains a backend API in `server/`, a review frontend in
`client/`, and documentation in `docs/`.

## Top-Level

- `README.md`: project summary and quick start
- `server/`: Bun + Elysia API
- `client/`: Vite + React review studio — reel creation (`/reels/new`),
  the review dashboard (`/`, with status-filtered queue nav), the trends
  dashboard (`/trends`, `/trends/:genre`), thumbnail review, revoice, and
  YouTube publish actions. Routed with TanStack Router (`RootLayout` +
  mobile nav drawer, see `client/src/main.tsx`).
- `docs/`: detailed architecture and feature documentation

## Server Structure (`server/src`)

- `index.ts`: app bootstrap, CORS, error handling, route mounting, BullMQ worker startup
- `config/`: environment configuration + validation (`index.ts`), the
  OpenRouter model registry + TTS voice catalog (`models.ts`), and the
  per-niche style registry (`niche-styles.ts`)
- `routes/`: HTTP routes grouped by domain (includes `trend.routes.ts`,
  `meta.routes.ts`, `maintenance.routes.ts` alongside the original
  template/character/composition/reel/voice/audio routes)
- `controllers/`: request handlers
- `services/`: business logic and external integrations — the reel pipeline
  (`reel*.service.ts`), trend scout/insight, S3 (`s3.service.ts` +
  `s3-reconciliation.service.ts`), voice sampling, gameplay ingest
- `helpers/`: pipeline utilities
- `models/`: Mongoose schemas
- `db/`: database connection and seed script
- `queue/`: BullMQ queues + workers (`queues.ts`, `workers.ts`,
  `connection.ts`) — reel processing, publishing, revoicing, and legacy
  composition processing/regeneration all run through here, not
  fire-and-forget promises
- `middlewares/`: file upload and local file persistence
- `utils/`: filesystem and timestamp utilities
- `types/`: type declarations and request guard schemas

## Scripts (`server/scripts`)

One-off/cron entrypoints — see [operations/scheduling.md](../operations/scheduling.md)
for what runs on what cadence: `topup-stories.ts`, `trend-scout-backfill.ts`,
`trend-scout-daily.ts`, `refresh-trend-insights.ts`, `reconcile-s3.ts`,
`fetch-gameplay.ts`, `fetch-fonts.ts`, `ingest-gameplay.ts`, `youtube-authorize.ts`.

## Non-Code Assets

- `server/assets/`: fonts and other media assets
- `server/storage/`: local ephemeral storage — `processing/` (intermediate
  render files, cleaned up on both success and failure) and `gameplay/`
  (local cache of the S3 gameplay pool, cleared after each render since S3
  is the source of truth)
