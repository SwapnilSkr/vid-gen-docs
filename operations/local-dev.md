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

From `client/`:

```
npm install
npm run dev
```

## Notes

- The API runs on `http://localhost:3000` by default.
- The review frontend runs on `http://localhost:5173` by default and calls
  `http://localhost:3000/api`. Override with `VITE_API_BASE_URL` if needed.
- `localFileSave` saves template videos to `config.processingPath` for preprocessing.
- Uploaded character images are stored in S3 via the `fileUpload` middleware.
