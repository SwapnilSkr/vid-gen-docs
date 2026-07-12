# Docker Deployment (server only)

Docker files live in `server/Dockerfile` and `server/docker-compose.yml`.

**Local development stays native:** `bun dev` + `.env.local` (local Redis by default).  
**Docker / shared testing:** image + `.env.docker` (Redis Cloud URI, same Atlas Mongo).

Typical workflow: smoke-test Docker on your laptop → wipe local images so disk stays clean → run the same image on a server so others can hit the API.

## One-time setup

```bash
cd server
cp .env.docker.example .env.docker
# Edit .env.docker: paste secrets from .env.local, set REDIS_URL to Redis Cloud
```

Do **not** commit `.env.docker`. Copy it to the server securely (scp, secrets manager, etc.).

Caption fonts are downloaded **during the Docker build** (`FETCH_FONTS_STRICT=1 bun run scripts/fetch-fonts.ts`). You do not need to run `fetch-fonts` manually for Docker. Local `bun dev` still benefits from `bun run fetch-fonts` once if you want matching caption fonts outside Docker.

## Env split

| File | Used by | Redis |
|------|---------|-------|
| `.env.local` | `bun dev` | `redis://localhost:6379` (default if unset) |
| `.env.docker` | `docker compose` | Redis Cloud URI |

Mongo can be the same Atlas URI in both files. TLS for Redis Cloud is enabled automatically for `*.redis.io` / `*.redislabs.com` (and for `rediss://` / `REDIS_TLS=true`). Redis Cloud certificates often lack a matching DNS SAN (`ERR_TLS_CERT_ALTNAME_INVALID`), so hostname verification is skipped for those hosts — traffic is still encrypted. Set `REDIS_TLS=false` to force plain, or `REDIS_TLS_REJECT_UNAUTHORIZED=true` if you mount a proper CA.

---

## 1. Smoke-test Docker locally

```bash
cd server
bun run docker:up
# or: docker compose up --build
```

Check health:

```bash
curl http://localhost:3000/health
```

Point the client at that API if you want an end-to-end check. Your laptop can keep using `bun dev` + local Redis for day-to-day work; this Docker run is only to verify the image.

Stop when done:

```bash
bun run docker:down
# or: docker compose down
```

---

## 2. Delete local Docker images (avoid disk bloat)

After a local test, remove this project's container + image:

```bash
cd server
docker compose down --rmi local --volumes --remove-orphans
```

Optional — reclaim unused Docker disk system-wide (other projects' unused images/cache too):

```bash
docker image prune -a -f
docker builder prune -a -f
```

`compose down --rmi local` only removes images built for this compose project. The prune commands are broader; use them when you want a full cleanup.

---

## 3. Run on a server so others can test

On a machine with Docker installed (VPS, home server, etc.):

```bash
# clone / pull the repo, then:
cd server
cp .env.docker.example .env.docker
# fill .env.docker (same secrets as local docker env; Redis Cloud + Atlas)

docker compose up --build -d
# or: bun run docker:up   # foreground; prefer -d on a server
```

Others hit `http://YOUR_SERVER_IP:3000` (or whatever you put behind nginx/Caddy). Your laptop stays on native `bun dev` if you prefer.

Useful server commands:

```bash
docker compose logs -f          # follow logs
docker compose ps               # status
docker compose down             # stop (keeps image)
docker compose down --rmi local # stop + remove this project's image
```

### Optional: registry push/pull

If you build once and want other hosts to pull instead of rebuild:

```bash
docker build -t vid-template-gen-server:latest .
docker tag vid-template-gen-server:latest YOUR_REGISTRY/vid-template-gen-server:latest
docker push YOUR_REGISTRY/vid-template-gen-server:latest
```

On the server:

```bash
docker pull YOUR_REGISTRY/vid-template-gen-server:latest
# ensure .env.docker exists, then:
docker compose up -d
```

---

## Health & runtime paths

- Container healthcheck hits `/health`
- FFmpeg inside the image: `/usr/bin/ffmpeg`, `/usr/bin/ffprobe`
- Storage mounts (compose): `./storage/{templates,characters,output,gameplay}` → `/app/storage/...`
- **Processing scratch** uses a Docker named volume (`processing_data`), not a host bind mount — bind-mounting ffmpeg temp files through Docker Desktop is a common cause of multi-minute stalls after scene clips finish
- Final encode preset defaults to `veryfast` via `FFMPEG_PRESET` (override in `.env.docker` if needed)
- `QUEUE_CONCURRENCY` defaults to `1` in compose so two ffmpeg jobs don't thrash the same CPUs
