# Operations logging

The Operations screen (`/operations`) is a read-only viewer for the backend system record of requests and asynchronous work. Every durable server event is also mirrored as a redacted, compact line in the backend console. Browser CORS `OPTIONS` preflights are deliberately excluded: they are transport setup rather than an application operation and otherwise obscure the request that follows.

## What is stored

`OperationLog` is a MongoDB collection with these sources:

| Source | Meaning | Examples |
| --- | --- | --- |
| `api` | Every inbound HTTP request | request method/path/status/duration, request ID |
| `queue` | Queue acceptance and duplicate suppression | plan, produce, publish, revoice enqueued |
| `worker` | Background job start, completion, retry, final failure | reel worker retry, publish worker failure |
| `external` | A provider or storage fallback/failure that changed behavior | Reddit public fallback, OpenRouter TTS fallback, S3 cleanup failure |
| `system` | A degraded server path that retains a prior valid state | subtitle update skipped after an outro-only rerender |

Entries include a timestamp, severity, event name, readable message, optional reel/job/request IDs, and bounded metadata. API responses include `X-Request-Id`; use it to connect a visible error to its request entry.

In-process story-bank and hygiene schedulers also emit `system` error events. Expected local temporary-file cleanup (for example, deleting a file that has already been removed) is intentionally not logged one line at a time; those cases are non-user-visible and would create misleading noise. Failed S3 cleanup that can leave storage behind is recorded.

## Privacy and size controls

- Metadata is redacted before Mongo writes. Keys such as token, secret, password, authorization, cookie, API key, signature, and credential are replaced. URL query strings (including signed URLs) are removed.
- Strings, stacks, object depth, arrays, and key counts are bounded so an exception or provider response cannot create a giant document.
- `OPERATION_LOG_RETENTION_DAYS` defaults to `30`. A Mongo TTL index removes expired entries automatically.
- `OPERATION_LOG_CONSOLE` defaults to `true`. Set it to `false` only when another server-side collector already mirrors the redacted Mongo events.
- The Operations screen supports filtering by level/source/reel, opening full details, deleting one entry, deleting selected entries, and permanently deleting every entry.

## Deliberate boundaries

The browser never writes Operations records. It shows local failures directly; API failures that reach the server, queue/worker outcomes, provider failures, and behavior-changing server fallbacks are logged by the backend. Successful `GET`/`DELETE` requests to `/api/operations` itself are intentionally excluded: the screen polls its own data and manages deletion, and recording those management-plane calls would refill a feed the user just cleared. Failed Operations requests still log. The system does **not** persist raw request bodies, raw provider responses, access tokens, or every successful individual provider call; that would expose story/private data and make the database grow at render-segment rate. Provider failures and behavior-changing fallbacks are recorded. Metered provider spend remains in the reel cost ledger.

If an entire server is unavailable, no backend process can log the browser's failed connection; the UI still presents the local failure. Once the server is reachable, normal API failures and worker errors are recorded by the backend.

## Retention operations

Use Operations for day-to-day deletion. For an emergency wipe, delete `OperationLog` documents directly from Mongo only after stopping the server or agreeing the scope; do not remove the TTL index. The collection is operational telemetry, not an audit trail intended for compliance retention.
