# Repository Structure

This repository contains a backend API in `server/` and documentation in `docs/`.

## Top-Level

- `README.md`: project summary and quick start
- `server/`: Bun + Elysia API
- `docs/`: detailed architecture and feature documentation

## Server Structure (`server/src`)

- `index.ts`: app bootstrap, CORS, error handling, route mounting
- `config/`: environment configuration and validation
- `routes/`: HTTP routes grouped by domain
- `controllers/`: request handlers
- `services/`: business logic and external integrations
- `helpers/`: pipeline utilities
- `models/`: Mongoose schemas
- `db/`: database connection and seed script
- `middlewares/`: file upload and local file persistence
- `utils/`: filesystem and timestamp utilities
- `types/`: type declarations and request guard schemas

## Non-Code Assets

- `server/assets/`: fonts and other media assets
- `server/storage/`: local processing storage (ephemeral in production)
