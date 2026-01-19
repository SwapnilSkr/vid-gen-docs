# Local Development

## Prerequisites

- Bun v1+
- MongoDB (local or Atlas)
- FFmpeg available in PATH

## Setup

From `server/`:

```
bun install
cp .env.local .env.local  # edit values
bun run seed
bun run dev
```

## Notes

- The API runs on `http://localhost:3000` by default.
- `localFileSave` saves template videos to `config.processingPath` for preprocessing.
- Uploaded character images are stored in S3 via the `fileUpload` middleware.
