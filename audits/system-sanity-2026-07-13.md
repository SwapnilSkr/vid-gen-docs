# System sanity audit — 2026-07-13

## Scope and method

This review covered the complete application source tree at the time of the audit: 130 server files and 97 client files. It includes all registered Elysia routes, controllers, services, Mongo models, BullMQ queues/workers, client API modules, Zustand state, polling hooks, screens, Studio panels, thumbnail tooling, and route wiring.

The review used static control-flow and error-path inspection, route-to-client tracing, Mongo state inspection for the affected Reddit series, live API checks against the local server, and TypeScript/lint/build verification. It did **not** deliberately invoke paid rendering, social publishing, OAuth exchange, or destructive maintenance endpoints. “Verified” below means source/control-flow verified or locally API-verified—not a claim that an external provider could never fail.

## Executive result

The product has real capability, but its reliability was undermined by fragmented state ownership and insufficient observability. A number of earlier features were correct at the service level but appeared broken because a sibling reel, cached client state, background retry, or fallback path was invisible to the user.

This audit adds durable operations logging and fixes several confirmed reliability defects. The remaining highest-risk work is transactional series editing, authentication/authorization, and a unified UI mutation/error contract.

## Coverage matrix

| Area | Reviewed paths | Current assessment |
| --- | --- | --- |
| Application bootstrap, config, Mongo, CORS | `server/src/index.ts`, `config`, `db` | Request lifecycle now logged; no application authentication exists. |
| Reels and Studio mutations | `reel.routes`, `reel.controller`, `reel.service`, `reel-edit.service`, `reel-edit-draft.service` | Rich functionality; cross-document series edits remain non-transactional. |
| Reddit discovery, updates, planning, splitting | `story.service`, `reddit-update-discovery.service`, `RedditSourcePanel` | OP comments are excluded; manual updates are source-post deduped and force-included; fallbacks now log. |
| Gameplay, covers, captions, outros, revoice | `reel-gameplay`, `reel-shorts-cover`, `reel-outro`, `reel-revoice`, Studio panels | Separate cached body/audio paths are deliberate, but every invalidation rule needs regression tests. |
| Image/horror/motion rendering | `reel-render`, `reel-motion`, `reel-hybrid`, `reel-script`, OpenRouter media | Provider and reference-image fallbacks now log; rendering is still long-running external work. |
| Cost accounting | `reel-cost`, OpenRouter media, planning/review paths | Actual and estimated lines are supported; late TTS receipts use a labelled character-price estimate. |
| Queueing and retries | `queue/queues.ts`, `queue/workers.ts` | Duplicate produce protection exists; retry status bug fixed; no explicit dead-letter UI yet. |
| YouTube/Instagram publishing | publish services, account routes/screens | Destination-specific status exists; external polling/retry behavior needs focused provider integration tests. |
| Destination output isolation | `reel-outro`, render, publish services, destination/publish Studio panels, S3 reconciliation | Exact channel output is now enforced; the state contract is documented in `architecture/destination-output-state.md`. |
| S3/local assets/hygiene | S3, local cleanup, reconciliation, maintenance routes | Superseded-delete failures now log; best-effort cleanup still has no retry queue. |
| YouTube import | import service/routes, polling hooks/screens | Polling is deduped in detail view; server API failures are logged at the backend boundary. |
| Trends, references, model/meta catalogues | trend services/routes/screens, meta routes | Functional but fallback/error feedback has been inconsistent with Studio. |
| Client data flow and UX | API modules, Zustand, polling hooks, all screens/panels | Studio has targeted race guards; there is no application-wide mutation state/error surface. |

## Confirmed fixes made during this audit

| ID | Finding | Impact | Change |
| --- | --- | --- | --- |
| OBS-001 | There was no persistent backend request log. Console output vanished and users had to inspect the Network tab. | High diagnostic cost; errors looked like no-ops. | Added Mongo `OperationLog`, request IDs, backend API/queue/worker/provider logging, complete queue enqueue/start/outcome tracing, redacted backend-console mirroring, retention, and `/operations` as a viewer. |
| OBS-002 | Browser-originated fallback/error telemetry mixed client state with backend operational history. | The Operations feed could imply that the server observed a browser-only event. | Browser telemetry and the `/client-events` endpoint were removed by design. The UI shows local failures; all server-reachable errors and fallbacks are logged by the backend. |
| OBS-003 | Operations polling and CORS preflights could crowd the new feed with diagnostic transport noise. | The action/error that a user needs to inspect could be pushed out of view. | Body-less reads no longer preflight; preflights and successful Operations-management reads/deletes are deliberately excluded, so browsing or clearing the feed does not recreate it. |
| REL-011 | Client API wrappers declared JSON content for body-less reads. | Cross-origin GETs unnecessarily performed a CORS preflight, adding latency and obscuring real activity. | All client API wrappers now only send `Content-Type: application/json` when a request has a body. |
| REL-012 | Thumbnail and YouTube-import polling could overlap; thumbnail responses had no stale-response guard. | An older response could overwrite newer state, causing confusing UI reversions or excess polling traffic. | Import polling now shares an in-flight request; thumbnail refreshes reject stale responses, matching Studio's reel refresh guard. |
| OBS-004 | A successful publish could hide a degraded thumbnail/cover outcome, and review thumbnails could silently fall back to flat artwork. | The final reel may be published without the intended visual treatment and the user had no durable diagnostic record. | These partial-success paths now create Operations warnings, including Instagram permalink delay. |
| OBS-005 | Story-bank/hygiene scheduler errors and partial trend-scout failures were terminal-only output. | Scheduled maintenance and content discovery could degrade without a persistent diagnostic trail. | They now emit durable `system`/`external` Operations records. |
| PUB-001 | YouTube and Instagram publishers always uploaded `reel.outputUrl`, even when an extra channel had its own `dest.outputUrl`. | A primary account’s branded outro could be uploaded to another platform/account. The picker also allowed unrendered/unconfigured accounts. | Publishing now resolves the exact ready `platform + channelId` destination before queue/provider work; the UI disables unavailable accounts, failed outros no longer silently fall back to the body, and S3 reconciliation retains destination media. Shared-body, draft, caption, revoice, replan, and part-teaser edits now invalidate affected finals before old media is deleted. |
| PUB-002 | The Studio description input and Instagram publish fallback shared the YouTube review description. | Editing YouTube copy could alter an Instagram Reel caption. | Metadata ownership is now platform-specific: YouTube uses `review`, Instagram uses `instagramSettings` only; a blank Instagram field never falls back to YouTube copy. |
| PUB-003 | A blank Instagram caption could be queued and posted after platform-copy isolation, and the caption field had no occurrence-aware hashtag cap. | A Reel could publish as video-only; repeated hashtags could evade a unique-tag counter. | Instagram now requires its own non-empty saved caption before queueing/publishing, caps every caption at five hashtag occurrences in the UI and server, logs only caption length/count, and sends the media-create/publish payloads as form fields. New review packages generate an independent AI Instagram caption from story material, persist its provenance/model, and append its LLM spend; legacy reels expose an explicit paid regeneration action. |
| REL-001 | A BullMQ reel job could mark a reel `failed` at its first failed attempt even though BullMQ would retry it. | UI showed a terminal failure while a retry was pending. | The worker now records retry vs final failure separately and keeps the reel retryable until attempts are exhausted. |
| REL-002 | Failures deleting superseded S3 media were intentionally ignored. | Storage leaks were invisible. | Best-effort behavior is preserved, but every reel, template, character, and cover superseded-asset deletion now records an Operations warning. |
| REL-003 | Part 2+ did not carry the series structure decision saved on Part 1. | The client incorrectly blocked Generate after successful “Keep current.” | Shared advice/decision is mirrored to all parts; Studio reads the series ledger as a compatibility fallback. |
| FIN-001 | TTS spend disappeared when OpenRouter’s generation receipt lagged. | Cost totals were understated. | Uses provider model pricing for a labelled estimate, preserves actual receipts where available, and historical affected reels were backfilled. |
| SEC-001 | Reddit share-link resolution had server-side fetch exposure. | SSRF risk through off-site redirects. | Verified the resolver now restricts initial and redirected URLs to approved Reddit hosts. |

## Open reliability and architecture findings

### Critical decisions needed

| ID | Finding | Why it matters now | Recommended decision |
| --- | --- | --- | --- |
| ARC-001 | There is no application authentication or authorization, while routes can render, publish, delete, connect accounts, run maintenance, and now inspect operational metadata. CORS is permissive. | Any party able to reach the server can operate it. This becomes more serious as Operations exposes system behavior. | Put the app behind authenticated access immediately; then restrict CORS to trusted origins and add role checks to destructive/publishing/Operations routes. |
| ARC-002 | A multipart restructuring/update flow writes several Mongo documents and may delete/recreate parts without a Mongo transaction or durable saga record. | A crash between writes can leave part counts, structure decision, assets, and series membership inconsistent. | Use Mongo transactions where deployment supports replica sets; otherwise introduce an explicit series-operation record with resumable phases and idempotency keys. |
| ARC-003 | The cost ledger is an append-only field on one Reel document, not a first-class immutable usage table. | Recovery/backfill needs text reconstruction; external receipt reconciliation is difficult. | Create a `UsageRecord` model keyed by provider generation ID/run ID, then derive the display ledger. Keep current ledger as a materialized UI view. |
| ARC-004 | `reel.youtube` is a single publish-result object while the UI can queue several YouTube accounts. | The media selected for each target is now correct, but concurrent YouTube jobs can overwrite the displayed status/result. | Move YouTube publishing to a per-destination publish ledger matching the existing Instagram array before supporting multi-YouTube distribution as fully state-isolated. |

### High priority engineering work

| ID | Finding | Evidence | Recommendation |
| --- | --- | --- | --- |
| REL-004 | Most multi-step service mutations are not idempotent at HTTP level. Queue duplicate detection is implemented for produce/publish, but many scene/settings/series mutations rely on the client not double-clicking. | `queue/queues.ts` only guards selected job types; route mutations have no idempotency key. | Adopt `Idempotency-Key` for paid/destructive actions and persist the action result. Disable duplicate UI controls while requests are in flight. |
| REL-005 | Cached body, assembly, narration, title, part-outro, branded-outro, subtitle, and destination outputs have many invalidation paths. | `reel.service`, `reel-edit.service`, `reel-edit-draft.service`, `reel-outro.service`. | Add table-driven regression tests for each edit x strategy x destination combination. A cache dependency graph should drive invalidation rather than hand-maintained conditionals. |
| REL-006 | Operations records request success/failure and behavior-changing fallbacks, but it intentionally does not log each successful external call. | Privacy/volume trade-off documented in `operations/observability.md`. Superseded-S3 deletion failures are now recorded rather than swallowed. | If per-provider latency/SLO analysis is needed, add sampled, redacted provider span records with a separate retention policy. |
| REL-007 | Best-effort cleanup may leave S3 objects on failure. | `deleteS3Urls` now records warnings, but does not retry. | Add a bounded cleanup queue or feed failed keys to the existing S3 reconciliation job. |
| REL-008 | Client polling is distributed across Studio, Zustand, and import hooks. | `StudioScreen`, `useReelSync`, `useYtImportPoll`, `useYtImportsPoll`. | Standardize on one query/polling abstraction with request cancellation, stale-result protection, exponential backoff, and visible “last updated / retrying” state. |
| REL-009 | The readable `GET reel status` endpoint can write to Mongo while sanitizing an old cost ledger. | `getReel` in `reel.service`. | Move migration/sanitization to an explicit maintenance migration or a background repair; reads should not normally mutate state. |
| REL-010 | Existing logs are user-deletable and TTL-expire by design. | `OperationLog` model. | If compliance/audit evidence is ever required, separate immutable security audit events from operational diagnostics. |

### Medium priority product/UX work

| ID | Finding | User effect | Recommendation |
| --- | --- | --- | --- |
| UX-001 | “Completed” is overloaded: it can mean render complete, review-ready, destination-ready, or publishing has started/completed. | Sidebar counts and studio controls need extra interpretation. | Use a derived lifecycle: Draft → Planning → Needs review → Producing → Ready to publish → Publishing → Published/Failed. Keep technical substeps separately. |
| UX-002 | Mutations expose errors differently by screen: some inline, some modal, some only state/poll driven. | A successful HTTP response can still leave the user unsure what changed. | Introduce a shared action result/toast pattern that includes request ID, affected entity, next state, and link to Operations. |
| UX-003 | Non-fatal fallbacks historically changed behavior without a visible cue. | Default gameplay/style/catalogue behavior can surprise users. | Keep Operations as source of detail, and add a compact “degraded mode” chip next to the affected control when a fallback is active. |
| UX-004 | The Studio inspector aggregates source, structure, rendering, outputs, publishing, and editing controls. | High cognitive load and easy misinterpretation of what will spend credits. | Separate “Plan”, “Produce”, “Publish”, and “Diagnostics”; show cost/side effects on each primary action, not in a distant panel. |
| UX-005 | Re-split/restructure uses expensive editorial work but the decision state was not obviously series-scoped. | This caused the Part 2 block that triggered the current repair. | Display a single Series Plan status banner across every part, with owner part and accepted decision/time. |
| UX-006 | Operations UI is new but is a full-screen diagnostic destination. | It helps investigation but not immediate action feedback. | Add a recent-errors badge/link from Studio and Accounts; prefilter by the active reel or destination. |
| UX-007 | The client build reports a statically and dynamically imported `StoryPickerStep`, plus a >500 kB main bundle. | Larger first load and an ineffective code split. | Remove the duplicate static import path and split heavy Studio/thumbnail dependencies deliberately. |

## Deliberately preserved behavior

- Reddit original-post comments are **not** mined for updates; discovery uses author posts and embedded links only.
- Manual update links are force-included when valid and excluded when they resolve to the original seed post.
- TTS estimates are visible as estimates rather than silently omitted when provider receipt timing is delayed.
- S3 cleanup remains non-blocking for user edits/renders; it is now observable instead of pretending it always succeeded.
- Rendering/publishing is queued and retryable; a worker retry is no longer shown as an irrevocable failure.

## Regression suite still required

There is no automated end-to-end test suite covering paid providers, social publishing, multipart restructuring, and cache invalidation. Before expanding features, add fixture-backed tests for:

1. Every series decision path from every part (manual/recommended, stale advice, resplit, update append/recut).
2. Every edit invalidation path for image and gameplay strategies, including multi-destination branded outros.
3. Queue retry vs final failure and duplicate submit behavior.
4. Cost ledger paths: actual receipt, delayed receipt estimate, rerender, cached narration, outro-only rerender, review-stage spend.
5. Publish metadata propagation for YouTube and Instagram without republishing.
6. Operations redaction, retention, API filtering, deletion, and no logging-recursion failure.
7. Browser state races: route switch during poll, stale poll after mutation, repeated Confirm clicks, and offline recovery.

## Verification completed after the changes

- `server`: `bun run typecheck` and `bun run lint`.
- `client`: `bun run build`.
- Live local API: Operations endpoint returns persistent backend request rows, `X-Request-Id`, and correctly records a controlled HTTP 400 as `api.request_failed`; redaction, single-row deletion, and full wipe were verified.
- Live UI: `/operations` loaded 50 rows without browser-console errors and showed real recorded activity without transport preflight noise.
- Build warning retained as an open issue: `StoryPickerStep` still has duplicate static/dynamic imports and the main bundle is 619.88 kB before gzip.

The remaining findings above are not silent: they are the prioritized work register for the next reliability pass.
