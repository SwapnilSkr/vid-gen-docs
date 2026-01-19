# Testing and Diagnostics

## API Sanity

- `GET /health` for service health
- `GET /` for API info

## Audio Pipeline Tests

- `GET /api/audio/test` runs a synthetic test using beep tones
- `POST /api/audio/test` allows custom audio configs

## Lint and Typecheck

From `server/`:

```
bun run lint
bun run typecheck
```
