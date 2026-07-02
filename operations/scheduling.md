# Scheduling — recurring jobs

Three recurring jobs keep the farm running. Each has two modes:

- **In-process** (a `setInterval`-based scheduler started from `index.ts` on
  server boot) — simplest for a single-instance deploy.
- **External cron** driving a `scripts/*.ts` entrypoint — required once you
  run more than one server instance, since only one process should be
  writing to shared state (the story bank, the trend collections, S3).

All three default to the in-process mode. Switch to external cron by setting
the matching `*_ENABLED=false` env var and adding a crontab entry.

## 1. Story bank top-up

Keeps the Reddit story bank stocked so `topic="auto"` reels never block on
live generation.

| | |
|---|---|
| In-process | `startStoryTopUpScheduler()` (`server/src/services/story-scheduler.service.ts`), started from `index.ts` on boot. Checks the bank every `STORY_TOPUP_INTERVAL_MS` (default 30 min); tops up `STORY_TOPUP_COUNT` (default 15) via `topUpStoryBank` whenever `ready < STORY_TOPUP_THRESHOLD` (default 10). |
| Disable in-process | `STORY_TOPUP_ENABLED=false` |
| External cron script | `bun scripts/topup-stories.ts [count] [mode]` — `mode` = `llm` \| `hybrid` \| `verbatim` (default `llm`) |
| Suggested cadence | Every 30 min |

```cron
*/30 * * * * cd /path/to/server && bun scripts/topup-stories.ts 15 llm >> logs/topup-stories.log 2>&1
```

## 2. Trend scout (daily + weekly)

Pulls top-performing Reddit-story Shorts per genre from YouTube and refreshes
the per-genre `TrendInsight` digest that script/thumbnail prompts read. See
[docs/api/endpoints.md](../api/endpoints.md#trend-scout) for the underlying
API. Needs `YOUTUBE_DATA_API_KEY`.

| | |
|---|---|
| Manual one-off (run once, to seed data) | `bun scripts/trend-scout-backfill.ts` — last 30 days + last 48 hours, all genres |
| Daily | `bun scripts/trend-scout-daily.ts` — rolling 7-day window (catches fresh viral hits early) |
| Weekly | `bun scripts/trend-scout-daily.ts month` — rolling 30-day window (refreshes the longer leaderboard) |
| On-demand / from the UI | `POST /api/trends/scout {"window": "week"\|"month"}` (used by the Trends dashboard's "Run Scout" button) |
| Disable | `TREND_SCOUT_ENABLED=false` (no in-process scheduler exists for this yet — it's cron/on-demand only) |

```cron
# Daily at 03:00 — rolling week
0 3 * * * cd /path/to/server && bun scripts/trend-scout-daily.ts >> logs/trend-scout-daily.log 2>&1

# Weekly, Sunday at 03:30 — rolling month
30 3 * * 0 cd /path/to/server && bun scripts/trend-scout-daily.ts month >> logs/trend-scout-weekly.log 2>&1
```

Quota note: ~14 genres × (100-unit search + 1-unit videos.list) ≈ 1,400
units per run — comfortably inside the 10,000/day free tier even running
both jobs the same day.

## 3. S3 reconciliation

Sweeps `reels/`, `compositions/`, `audio/`, `subtitles/` for objects not
referenced by any current `Reel` or `Composition` document — catches the
class of orphan a cascading delete can never find (uploaded, then the
process crashed/stalled before the URL was saved to Mongo). Skips anything
uploaded in the last 2 hours so it never races an in-flight render. See
[docs/api/endpoints.md](../api/endpoints.md#maintenance--cleanup).

| | |
|---|---|
| Script | `bun scripts/reconcile-s3.ts` (dry run — reports only) / `bun scripts/reconcile-s3.ts --apply` (actually deletes) |
| On-demand | `GET /api/maintenance/s3-reconcile` (dry run) / `GET /api/maintenance/s3-reconcile?apply=true` |
| Disable | N/A — cron/on-demand only, no in-process scheduler (deletion should always be a deliberate, observable run, not a silent background loop) |
| Suggested cadence | Daily |

```cron
# Daily at 04:00 — after the trend-scout daily run
0 4 * * * cd /path/to/server && bun scripts/reconcile-s3.ts --apply >> logs/reconcile-s3.log 2>&1
```

Related: `POST /api/maintenance/reels/purge-failed` deletes every reel
currently `status: "failed"` (Mongo doc + its S3 assets + BullMQ job) — not
scheduled by default, since a failed reel is often worth inspecting before
deleting. Run it manually, or add it to the same cron entry if you'd rather
auto-purge failures older than a day:

```cron
# Optional: auto-purge failed reels daily, right after reconciliation
5 4 * * * curl -s -X POST http://localhost:3000/api/maintenance/reels/purge-failed >> logs/purge-failed.log 2>&1
```

## Full suggested crontab

```cron
*/30 * * * * cd /path/to/server && bun scripts/topup-stories.ts 15 llm >> logs/topup-stories.log 2>&1
0    3 * * * cd /path/to/server && bun scripts/trend-scout-daily.ts >> logs/trend-scout-daily.log 2>&1
30   3 * * 0 cd /path/to/server && bun scripts/trend-scout-daily.ts month >> logs/trend-scout-weekly.log 2>&1
0    4 * * * cd /path/to/server && bun scripts/reconcile-s3.ts --apply >> logs/reconcile-s3.log 2>&1
5    4 * * * curl -s -X POST http://localhost:3000/api/maintenance/reels/purge-failed >> logs/purge-failed.log 2>&1
```

If you disable the in-process story-topup scheduler (`STORY_TOPUP_ENABLED=false`
for multi-instance deploys), make sure exactly one instance's crontab runs
`topup-stories.ts` — same reasoning applies if a second instance's server
boot also starts the in-process scheduler.
