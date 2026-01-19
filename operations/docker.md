# Docker Deployment

Docker files live in `server/Dockerfile` and `server/docker-compose.yml`.

## Dockerfile Highlights

- Multi-stage build (Bun)
- FFmpeg installed in runtime image
- Non-root `vidgen` user
- Storage directories created at `/app/storage/*`

## docker-compose

```
docker compose up --build
```

Environment variables are passed through from your shell. Key ones:

- `ELEVEN_LABS_API_KEY`
- `OPENROUTER_API_KEY`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `S3_BUCKET`

## Health Check

- `/health` endpoint is used for container health
